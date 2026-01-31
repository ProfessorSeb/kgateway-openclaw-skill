---
name: kgateway
description: |
  Solo.io kGateway skill for deploying and managing Envoy-based API gateways in Kubernetes. Covers both open source (kgateway.dev, CNCF sandbox) and Enterprise (Solo.io) editions. Use when:
  (1) Installing kGateway via Helm or ArgoCD
  (2) Configuring Kubernetes Gateway API resources (Gateway, HTTPRoute, GRPCRoute, TCPRoute)
  (3) Traffic management: routing, header manipulation, transformations, rewrites, redirects
  (4) Security: TLS termination, CORS, CSRF, mTLS, certificate management
  (5) Authentication: JWT, OIDC/OAuth 2.0, API keys, basic auth, external auth (Enterprise)
  (6) Rate limiting: local and global rate limiting
  (7) Resiliency: retries, timeouts, circuit breakers, health checks, failover
  (8) Observability: access logging, OpenTelemetry, metrics, tracing
  (9) Integrations: Istio service mesh, AWS Lambda, external processing
  (10) Enterprise features: advanced rate limiting, staged transformations, multicluster, OPA
metadata:
  author: ProfessorSeb
  version: "1.0.0"
---

# kGateway Skill

Envoy-based Kubernetes-native API gateway implementing the Gateway API.

## Quick Reference

| Edition | Docs | Helm Chart |
|---------|------|------------|
| **Open Source** | https://kgateway.dev | `oci://ghcr.io/kgateway-dev/charts/kgateway` |
| **Enterprise** | https://docs.solo.io/kgateway | `oci://us-docker.pkg.dev/solo-public/enterprise-kgateway/charts/enterprise-kgateway` |

## Feature Comparison

| Feature | Enterprise | Open Source |
|---------|------------|-------------|
| Envoy-based gateway proxy | ✅ | ✅ |
| Kubernetes Gateway API | ✅ | ✅ |
| HTTPRoute, GRPCRoute, TCPRoute | ✅ | ✅ |
| Traffic management (headers, transforms) | ✅ | ✅ |
| TLS, CORS, CSRF | ✅ | ✅ |
| Local rate limiting | ✅ | ✅ |
| Access logging, OpenTelemetry | ✅ | ✅ |
| Istio mesh integration | ✅ | ✅ |
| **Built-in rate limiting server** | ✅ | ❌ |
| **Staged transformations** | ✅ | ❌ |
| **AWS Lambda transformations** | ✅ | ❌ |
| **JWT, OIDC/OAuth 2.0, OPA auth** | ✅ | ❌ |
| **External auth service** | ✅ | ❌ |
| **Multicluster federation** | ✅ | ❌ |
| **FIPS/LTS builds** | ✅ | ❌ |
| **24x7 enterprise support** | ✅ | ❌ |

## Installation

### Open Source
```bash
# Gateway API CRDs
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.4.0/standard-install.yaml

# kGateway CRDs
helm upgrade -i --create-namespace \
  --namespace kgateway-system \
  kgateway-crds oci://ghcr.io/kgateway-dev/charts/kgateway-crds

# kGateway control plane
helm upgrade -i -n kgateway-system \
  kgateway oci://ghcr.io/kgateway-dev/charts/kgateway
```

### Enterprise
```bash
# Gateway API CRDs
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.4.0/standard-install.yaml

# Enterprise CRDs
helm upgrade -i --create-namespace \
  --namespace kgateway-system \
  --version 2.1.0 \
  enterprise-kgateway-crds oci://us-docker.pkg.dev/solo-public/enterprise-kgateway/charts/enterprise-kgateway-crds

# Enterprise control plane
helm upgrade -i -n kgateway-system \
  --version 2.1.0 \
  enterprise-kgateway oci://us-docker.pkg.dev/solo-public/enterprise-kgateway/charts/enterprise-kgateway
```

## Core Resources

### Gateway

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: http-gateway
  namespace: kgateway-system
spec:
  gatewayClassName: kgateway  # or enterprise-kgateway
  listeners:
  - name: http
    port: 80
    protocol: HTTP
    allowedRoutes:
      namespaces:
        from: All
  - name: https
    port: 443
    protocol: HTTPS
    tls:
      mode: Terminate
      certificateRefs:
      - name: gateway-tls-secret
    allowedRoutes:
      namespaces:
        from: All
```

### HTTPRoute

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: app-routes
  namespace: default
spec:
  parentRefs:
  - name: http-gateway
    namespace: kgateway-system
  hostnames:
  - "api.example.com"
  rules:
  # Path-based routing
  - matches:
    - path:
        type: PathPrefix
        value: /api/v1
    backendRefs:
    - name: api-v1-service
      port: 8080
  # Header-based routing
  - matches:
    - headers:
      - name: x-version
        value: beta
    backendRefs:
    - name: api-beta-service
      port: 8080
  # Method matching
  - matches:
    - method: POST
      path:
        type: Exact
        value: /submit
    backendRefs:
    - name: submission-service
      port: 8080
```

### GRPCRoute

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GRPCRoute
metadata:
  name: grpc-route
spec:
  parentRefs:
  - name: http-gateway
    namespace: kgateway-system
  hostnames:
  - "grpc.example.com"
  rules:
  - matches:
    - method:
        service: myapp.UserService
        method: GetUser
    backendRefs:
    - name: user-grpc-service
      port: 9090
```

## Traffic Management

See [references/traffic-management.md](references/traffic-management.md) for complete examples.

### Header Modification

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
spec:
  rules:
  - filters:
    - type: RequestHeaderModifier
      requestHeaderModifier:
        add:
        - name: X-Custom-Header
          value: my-value
        set:
        - name: Host
          value: backend.internal
        remove:
        - X-Forwarded-For
    - type: ResponseHeaderModifier
      responseHeaderModifier:
        add:
        - name: X-Response-Time
          value: "%RESPONSE_TIME%"
    backendRefs:
    - name: backend
      port: 8080
```

### URL Rewrite

```yaml
filters:
- type: URLRewrite
  urlRewrite:
    hostname: internal-api.default.svc
    path:
      type: ReplacePrefixMatch
      replacePrefixMatch: /v2
```

### Redirect

```yaml
filters:
- type: RequestRedirect
  requestRedirect:
    scheme: https
    hostname: www.example.com
    statusCode: 301
```

### Traffic Splitting (Canary)

```yaml
rules:
- backendRefs:
  - name: api-v1
    port: 8080
    weight: 90
  - name: api-v2
    port: 8080
    weight: 10
```

## Security

See [references/security.md](references/security.md) for complete examples.

### TLS Termination

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: gateway-tls-secret
type: kubernetes.io/tls
data:
  tls.crt: <base64-cert>
  tls.key: <base64-key>
---
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
spec:
  listeners:
  - name: https
    port: 443
    protocol: HTTPS
    tls:
      mode: Terminate
      certificateRefs:
      - name: gateway-tls-secret
```

### CORS (TrafficPolicy)

```yaml
apiVersion: gateway.kgateway.dev/v1alpha1
kind: TrafficPolicy
metadata:
  name: cors-policy
spec:
  targetRefs:
  - group: gateway.networking.k8s.io
    kind: HTTPRoute
    name: app-routes
  cors:
    allowOrigins:
    - exact: "https://example.com"
    - regex: "https://.*\\.example\\.com"
    allowMethods:
    - GET
    - POST
    - PUT
    - DELETE
    allowHeaders:
    - Authorization
    - Content-Type
    exposeHeaders:
    - X-Request-Id
    maxAge: 86400s
    allowCredentials: true
```

### CSRF Protection

```yaml
apiVersion: gateway.kgateway.dev/v1alpha1
kind: TrafficPolicy
spec:
  targetRefs:
  - kind: HTTPRoute
    name: app-routes
  csrf:
    filterEnabled: true
    shadowEnabled: false
    additionalOrigins:
    - exact: "https://trusted.example.com"
```

## Rate Limiting

### Local Rate Limiting (Both Editions)

```yaml
apiVersion: gateway.kgateway.dev/v1alpha1
kind: TrafficPolicy
spec:
  targetRefs:
  - kind: HTTPRoute
    name: app-routes
  rateLimit:
    local:
      tokensPerFill: 100
      fillInterval: 60s
      maxTokens: 100
```

### Global Rate Limiting (Enterprise)

```yaml
apiVersion: gateway.kgateway.dev/v1alpha1
kind: TrafficPolicy
spec:
  targetRefs:
  - kind: HTTPRoute
    name: app-routes
  rateLimit:
    global:
      descriptors:
      - entries:
        - key: remote_address
      limit:
        requestsPerUnit: 100
        unit: MINUTE
```

## Resiliency

### Retries

```yaml
apiVersion: gateway.kgateway.dev/v1alpha1
kind: TrafficPolicy
spec:
  targetRefs:
  - kind: HTTPRoute
    name: app-routes
  retry:
    attempts: 3
    perTryTimeout: 5s
    retryOn:
    - 5xx
    - reset
    - connect-failure
    - retriable-4xx
    backOff:
      baseInterval: 100ms
      maxInterval: 1s
```

### Timeouts

```yaml
apiVersion: gateway.kgateway.dev/v1alpha1
kind: TrafficPolicy
spec:
  targetRefs:
  - kind: HTTPRoute
    name: app-routes
  timeout:
    request: 30s
    idle: 5m
```

### Circuit Breaker

```yaml
apiVersion: gateway.kgateway.dev/v1alpha1
kind: TrafficPolicy
spec:
  targetRefs:
  - kind: HTTPRoute
    name: app-routes
  circuitBreaker:
    maxConnections: 1000
    maxPendingRequests: 100
    maxRequests: 1000
    maxRetries: 3
```

## Authentication (Enterprise)

See [references/authentication.md](references/authentication.md) for complete examples.

### JWT Authentication

```yaml
apiVersion: enterprise.kgateway.solo.io/v1alpha1
kind: AuthConfig
metadata:
  name: jwt-auth
spec:
  configs:
  - jwt:
      issuer: "https://auth.example.com/"
      audiences:
      - "my-api"
      jwksUri: "https://auth.example.com/.well-known/jwks.json"
      claimsToHeaders:
      - claim: sub
        header: x-user-id
      - claim: email
        header: x-user-email
```

### OIDC/OAuth 2.0

```yaml
apiVersion: enterprise.kgateway.solo.io/v1alpha1
kind: AuthConfig
metadata:
  name: oidc-auth
spec:
  configs:
  - oauth2:
      oidcAuthorizationCode:
        clientId: "my-client-id"
        clientSecretRef:
          name: oidc-secret
        issuerUrl: "https://accounts.google.com"
        scopes:
        - openid
        - email
        - profile
        callbackPath: /callback
        logoutPath: /logout
```

### API Key Authentication

```yaml
apiVersion: enterprise.kgateway.solo.io/v1alpha1
kind: AuthConfig
metadata:
  name: apikey-auth
spec:
  configs:
  - apiKeyAuth:
      headerName: x-api-key
      labelSelector:
        team: backend
```

## Observability

### Access Logging

```yaml
apiVersion: gateway.kgateway.dev/v1alpha1
kind: ListenerOption
metadata:
  name: access-logging
spec:
  targetRefs:
  - group: gateway.networking.k8s.io
    kind: Gateway
    name: http-gateway
  options:
    accessLoggingService:
      accessLog:
      - fileSink:
          path: /dev/stdout
          jsonFormat:
            timestamp: "%START_TIME%"
            method: "%REQ(:METHOD)%"
            path: "%REQ(X-ENVOY-ORIGINAL-PATH?:PATH)%"
            status: "%RESPONSE_CODE%"
            duration: "%DURATION%"
            upstream: "%UPSTREAM_HOST%"
```

### OpenTelemetry

```yaml
# Values for Helm installation
telemetry:
  enabled: true
  openTelemetry:
    endpoint: "otel-collector.telemetry:4317"
    serviceName: "kgateway"
```

## Common Commands

```bash
# Check gateways
kubectl get gateway -A

# Check routes
kubectl get httproute,grpcroute,tcproute -A

# Check gateway class
kubectl get gatewayclass

# Gateway status
kubectl describe gateway http-gateway -n kgateway-system

# Envoy config dump
kubectl exec -n kgateway-system deploy/http-gateway -- curl localhost:19000/config_dump

# Check control plane logs
kubectl logs -n kgateway-system deploy/enterprise-kgateway

# Check proxy logs
kubectl logs -n kgateway-system deploy/http-gateway
```

## Troubleshooting

| Issue | Check |
|-------|-------|
| Route not working | HTTPRoute status, parentRef matches Gateway |
| 404 errors | Path matching, hostname, listener port |
| TLS errors | Certificate secret exists, tls.mode correct |
| Timeout | Backend service reachable, timeout settings |
| Auth failures | AuthConfig attached, JWKS reachable |

## References

- [references/traffic-management.md](references/traffic-management.md) — Routing, transforms, rewrites
- [references/security.md](references/security.md) — TLS, CORS, CSRF, mTLS
- [references/authentication.md](references/authentication.md) — JWT, OIDC, API keys (Enterprise)
- [references/resiliency.md](references/resiliency.md) — Retries, timeouts, circuit breakers
