# Architecture

## Overview

The cluster-authentication-operator manages the authentication infrastructure for OpenShift clusters. It runs in the `openshift-authentication-operator` namespace and manages two operands: the OAuth server and the OAuth API server.

The operator's startup is organized into three preparation functions in `pkg/operator/starter.go`:

- **`prepareOauthOperator()`** — controllers for the OAuth server (deployment, routes, ingress, metadata, trust distribution)
- **`prepareOauthAPIServerOperator()`** — controllers for the OAuth API server (workload, encryption, API services, CSR approval)
- **`prepareExternalOIDC()`** — the External OIDC controller (feature-gated, manages auth-config for kube-apiserver and oauth-apiserver)

Mode detection is handled by `authConfigChecker.OIDCAvailable()`, which inspects the `Authentication.config.openshift.io/cluster` resource. When External OIDC is enabled, the operator disables the default OAuth stack and configures the cluster to use an external OIDC provider instead. The existing OAuth stack is considered feature complete; future feature development is focused on External OIDC.

## Operands

### OAuth Server (`openshift-authentication`)

The OAuth server handles user authentication via identity providers (HTPasswd, LDAP, OIDC, Request Header, etc.). It serves the `/oauth/authorize` and `/oauth/token` endpoints and manages login flows.

Key resources managed from `bindata/oauth-openshift/`:

| Resource | File | Purpose |
|---|---|---|
| Deployment | `deployment.yaml` | OAuth server pods |
| Service | `oauth-service.yaml` | Internal service |
| Route | `route.yaml` | External route for OAuth endpoints |
| ServiceAccount | `serviceaccount.yaml` | Pod identity |
| NetworkPolicy | `networkpolicy_oauth-server.yaml` | Ingress rules for OAuth server |
| Audit Policy | `audit-policy.yaml` | Audit logging configuration |
| Trust Distribution | `trust_distribution_role.yaml`, `trust_distribution_rolebinding.yaml` | Distributes OAuth serving certs |
| RoleBindingRestriction CRD | `authorization.openshift.io_rolebindingrestrictions.yaml` | Copied from `vendor/` via `make update-bindata` |

### OAuth API Server (`openshift-oauth-apiserver`)

The OAuth API server serves the OAuth API resources (`OAuthAccessTokens`, `OAuthAuthorizeTokens`, `OAuthClients`) and registers the `oauth.openshift.io` and `user.openshift.io` API groups via APIService resources.

Key resources managed from `bindata/oauth-apiserver/`:

| Resource | File | Purpose |
|---|---|---|
| Deployment | `deploy.yaml` | OAuth API server pods (standard mode) |
| Deployment | `externaloidc-deploy.yaml` | OAuth API server pods (External OIDC mode) |
| Service | `svc.yaml` | Internal service |
| ServiceAccount | `sa.yaml` | Pod identity |
| PodDisruptionBudget | `oauth-apiserver-pdb.yaml` | Availability guarantee |
| NetworkPolicy | `networkpolicy_oauth-apiserver.yaml` | Ingress rules for API server |
| RBAC | `RBAC/useroauthaccesstokens_clusterrole.yaml` | User access token permissions |

## Operator Startup Flow

```
cmd/authentication-operator/main.go
  → NewAuthenticationOperatorCommand()
    → operator.RunOperator()             [pkg/operator/starter.go]
      → CreateOperatorStarter()
        → prepareOauthOperator()         → OAuth server controllers
        → prepareOauthAPIServerOperator() → OAuth API server controllers
        → prepareExternalOIDC()          → External OIDC controller
      → operatorStarter.Start(ctx)
        → Sync() then Run() for each controller
```

All controllers use the library-go factory pattern. They are synced once at startup, then run continuously with informer-driven requeues.

## Controllers

### OAuth Server Controllers (`prepareOauthOperator`)

| Controller | Directory | Purpose |
|---|---|---|
| Deployment | `pkg/controllers/deployment/` | Manages OAuth server deployment workload |
| Custom Route | `pkg/controllers/customroute/` | Syncs custom OAuth route TLS configuration from ingress config |
| Ingress Nodes Available | `pkg/controllers/ingressnodesavailable/` | Validates that ingress/router nodes are available |
| Ingress State | `pkg/controllers/ingressstate/` | Monitors OAuth service endpoints and pod state |
| Metadata | `pkg/controllers/metadata/` | Publishes OAuth metadata ConfigMap (issuer, endpoints) |
| OAuth Clients | `pkg/controllers/oauthclientscontroller/` | Creates and manages default OAuth clients (console, browser) |
| OAuth Endpoints | `pkg/controllers/oauthendpoints/` | Health checks for OAuth route, service, and endpoints |
| Payload | `pkg/controllers/payload/` | Generates OAuth server config (IDPs, tokens, templates) |
| Proxy Config | `pkg/controllers/proxyconfig/` | Validates proxy configuration for OAuth route accessibility |
| Readiness | `pkg/controllers/readiness/` | Checks `.well-known/oauth-authorization-server` endpoint |
| Router Certs | `pkg/controllers/routercerts/` | Validates router certificates match ingress domain |
| Service CA | `pkg/controllers/serviceca/` | Syncs service CA bundle for certificate injection |
| Termination | `pkg/controllers/termination/` | Forces operator restart when console capability transitions |
| Trust Distribution | `pkg/controllers/trustdistribution/` | Distributes OAuth serving certificates to `openshift-config-managed` |

### OAuth API Server Controllers (`prepareOauthAPIServerOperator`)

| Controller | Directory | Purpose |
|---|---|---|
| Workload | `pkg/operator/workload/` | Manages oauth-apiserver deployment (replicas, config injection, KMS) |
| Webhook Authenticator | `pkg/controllers/webhookauthenticator/` | Configures webhook authenticator for kube-apiserver token validation |
| Config Observer | `pkg/operator/configobservation/` | Observes cluster config and syncs to oauth-apiserver |
| Encryption | *(library-go)* | Manages encryption of OAuth tokens at rest with key rotation |
| API Services | *(library-go)* | Registers `oauth.openshift.io` and `user.openshift.io` APIServices |
| CSR Approval | *(library-go)* | Approves CSRs for webhook authenticator certificates |

### External OIDC Controller (`prepareExternalOIDC`)

| Controller | Directory | Purpose |
|---|---|---|
| External OIDC | `pkg/controllers/externaloidc/` | Generates auth-config ConfigMap for kube-apiserver and oauth-apiserver when External OIDC is enabled |

## Config Observers

### OAuth Server Observers

Located in `pkg/controllers/configobservation/`. These observe cluster resources and produce the configuration for the OAuth server deployment.

| Observer | What it observes |
|---|---|
| `infrastructure.ObserveAPIServerURL` | API server URL from Infrastructure CR |
| `oauth.ObserveIdentityProviders` | IDP configuration from OAuth CR |
| `oauth.ObserveTemplates` | Login/error page templates from OAuth CR |
| `oauth.ObserveTokenConfig` | Token durations from OAuth CR |
| `oauth.ObserveAudit` | Audit policy from OAuth CR |
| `console.ObserveConsoleURL` | Console public URL (if console capability enabled) |
| `routersecret.ObserveRouterSecret` | Session secret from router |

### OAuth API Server Observers

Located in `pkg/operator/configobservation/configobservercontroller/`. These observe cluster resources and produce the configuration for the oauth-apiserver deployment.

| Observer | What it observes |
|---|---|
| `apiserver.ObserveAdditionalCORSAllowedOrigins` | CORS configuration from APIServer CR |
| `apiserver.ObserveTLSSecurityProfile` | TLS cipher suites and minimum version |
| `authentication.ObserveAPIAudiences` | Service account issuer audiences |
| `oauth.ObserveAccessTokenInactivityTimeout` | Token inactivity timeout from OAuth CR |
| `libgoetcd.ObserveStorageURLs` | etcd endpoints |
| `encryptobserver.NewEncryptionConfigObserver` | Encryption config secret location |
| `featuregates.NewObserveFeatureFlagsFunc` | Feature gates (e.g. CBOR support) |

## Feature Gates

The following feature gates affect the operator's behavior, all related to External OIDC:

| Feature Gate | Effect |
|---|---|
| `ExternalOIDC` | Enables External OIDC mode; disables the default OAuth stack |
| `ExternalOIDCWithUIDAndExtraClaimMappings` | Enables additional claim mappings (UID, extra claims) for OIDC providers |
| `ExternalOIDCWithUpstreamParity` | Enables upstream-parity features for OIDC authentication |
| `ExternalOIDCExternalClaimsSourcing` | Preserves the oauth-apiserver under OIDC (instead of deleting it) and enables external claims sourcing |

These gates are defined in `vendor/github.com/openshift/api/features/features.go` and checked via `featuregates.FeatureGate` throughout the controllers.

## Namespaces

| Namespace | Role |
|---|---|
| `openshift-authentication-operator` | The operator itself runs here |
| `openshift-authentication` | OAuth server deployment |
| `openshift-oauth-apiserver` | OAuth API server deployment |
| `openshift-config` | User-provided configuration (IdP secrets, TLS certs, CA bundles) |
| `openshift-config-managed` | Operator-managed configuration (trust distribution) |
| `openshift-console` | Console URL observation |
| `openshift-ingress` | Router/ingress pods and secrets |
| `openshift-ingress-operator` | Ingress controller configuration |
| `openshift-kube-apiserver` | Webhook authenticator configuration target |

## Key Directories

```
cmd/authentication-operator/          Operator entrypoint
pkg/operator/
  starter.go                          Controller registration and startup
  workload/                           OAuth API server workload controller
  configobservation/                  OAuth API server config observers
pkg/controllers/
  configobservation/                  OAuth server config observers
  deployment/                         OAuth server deployment controller
  externaloidc/                       External OIDC controller and config generation
  metadata/                           OAuth metadata controller
  oauthclientscontroller/             Default OAuth client management
  payload/                            OAuth server config payload generation
  routercerts/                        Router certificate validation
  webhookauthenticator/               Webhook authenticator for kube-apiserver
  ...                                 (other controllers listed above)
bindata/
  oauth-openshift/                    OAuth server manifests (embedded via //go:embed)
  oauth-apiserver/                    OAuth API server manifests (embedded via //go:embed)
manifests/                            Operator deployment, RBAC, monitoring
test/
  e2e/                                General e2e tests
  e2e-encryption/                     Encryption e2e tests (serial, 4h)
  e2e-encryption-rotation/            Encryption rotation e2e tests (serial, 4h)
  e2e-encryption-perf/                Encryption performance tests
  e2e-oidc/                           OIDC-specific e2e tests
  library/                            Shared test utilities
```
