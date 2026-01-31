# kGateway Authentication (Enterprise)

Enterprise authentication and authorization features.

## Table of Contents
- [JWT Authentication](#jwt-authentication)
- [OIDC/OAuth 2.0](#oidcoauth-20)
- [API Key Authentication](#api-key-authentication)
- [Basic Authentication](#basic-authentication)
- [External Auth Service](#external-auth-service)
- [OPA Authorization](#opa-authorization)
- [Multi-Auth Chains](#multi-auth-chains)

---

## JWT Authentication

### Basic JWT Validation
```yaml
apiVersion: enterprise.kgateway.solo.io/v1alpha1
kind: AuthConfig
metadata:
  name: jwt-auth
  namespace: kgateway-system
spec:
  configs:
  - jwt:
      issuer: "https://auth.example.com/"
      audiences:
      - "my-api"
      jwksUri: "https://auth.example.com/.well-known/jwks.json"
---
apiVersion: gateway.kgateway.dev/v1alpha1
kind: TrafficPolicy
metadata:
  name: jwt-policy
spec:
  targetRefs:
  - kind: HTTPRoute
    name: api-routes
  extAuth:
    configRef:
      name: jwt-auth
      namespace: kgateway-system
```

### JWT with Claims Extraction
```yaml
spec:
  configs:
  - jwt:
      issuer: "https://auth.example.com/"
      jwksUri: "https://auth.example.com/.well-known/jwks.json"
      claimsToHeaders:
      - claim: sub
        header: x-user-id
      - claim: email
        header: x-user-email
      - claim: roles
        header: x-user-roles
      - claim: tenant_id
        header: x-tenant-id
```

### JWT from Cookie
```yaml
spec:
  configs:
  - jwt:
      issuer: "https://auth.example.com/"
      jwksUri: "https://auth.example.com/.well-known/jwks.json"
      tokenSource:
        headers:
        - Authorization
        cookies:
        - session_token
```

### Multiple JWT Issuers
```yaml
spec:
  configs:
  - jwt:
      issuer: "https://auth0.example.com/"
      jwksUri: "https://auth0.example.com/.well-known/jwks.json"
  - jwt:
      issuer: "https://okta.example.com/"
      jwksUri: "https://okta.example.com/oauth2/default/v1/keys"
```

---

## OIDC/OAuth 2.0

### Authorization Code Flow
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
          name: oidc-client-secret
          namespace: kgateway-system
        issuerUrl: "https://accounts.google.com"
        scopes:
        - openid
        - email
        - profile
        callbackPath: /callback
        logoutPath: /logout
        session:
          cookieOptions:
            maxAge: 3600
            path: /
            domain: example.com
            secure: true
            httpOnly: true
---
apiVersion: v1
kind: Secret
metadata:
  name: oidc-client-secret
  namespace: kgateway-system
type: Opaque
stringData:
  client-secret: "your-client-secret"
```

### OIDC with PKCE
```yaml
oauth2:
  oidcAuthorizationCode:
    clientId: "spa-client"
    issuerUrl: "https://auth.example.com"
    pkce:
      enabled: true
      challengeMethod: S256
```

### Keycloak Integration
```yaml
oauth2:
  oidcAuthorizationCode:
    clientId: "kgateway-client"
    clientSecretRef:
      name: keycloak-secret
    issuerUrl: "https://keycloak.example.com/realms/myrealm"
    scopes:
    - openid
    - profile
    - roles
```

### Access Token Validation (Resource Server)
```yaml
spec:
  configs:
  - oauth2:
      accessTokenValidation:
        introspectionUrl: "https://auth.example.com/oauth/introspect"
        clientId: "resource-server"
        clientSecretRef:
          name: introspection-secret
        # Or use local JWT validation
        jwt:
          issuer: "https://auth.example.com/"
          jwksUri: "https://auth.example.com/.well-known/jwks.json"
```

---

## API Key Authentication

### Header-Based API Key
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
        app: my-api
---
# API Key secrets
apiVersion: v1
kind: Secret
metadata:
  name: api-key-user1
  labels:
    app: my-api
    team: backend
type: Opaque
stringData:
  api-key: "ak_live_xxxxxxxxxxxx"
  user-id: "user-123"
  rate-limit: "1000"
```

### Query Parameter API Key
```yaml
apiKeyAuth:
  queryParam: api_key
  labelSelector:
    app: my-api
```

### API Key with Metadata
```yaml
# Secret with metadata passed to headers
apiVersion: v1
kind: Secret
metadata:
  name: api-key-premium
  labels:
    app: my-api
    tier: premium
type: Opaque
stringData:
  api-key: "ak_premium_xxxxxxxxxxxx"
  user-id: "premium-user-1"
  org-id: "org-456"
  rate-limit: "10000"
```

---

## Basic Authentication

```yaml
apiVersion: enterprise.kgateway.solo.io/v1alpha1
kind: AuthConfig
metadata:
  name: basic-auth
spec:
  configs:
  - basicAuth:
      apr:
        users:
          admin:
            hashedPassword: "$apr1$xyz..."  # htpasswd format
          readonly:
            hashedPassword: "$apr1$abc..."
```

### Basic Auth from Secret
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: basic-auth-users
type: Opaque
stringData:
  htpasswd: |
    admin:$apr1$...
    user1:$apr1$...
    user2:$apr1$...
---
spec:
  configs:
  - basicAuth:
      apr:
        secretRef:
          name: basic-auth-users
```

---

## External Auth Service

### Custom gRPC Auth Service
```yaml
apiVersion: enterprise.kgateway.solo.io/v1alpha1
kind: AuthConfig
metadata:
  name: custom-auth
spec:
  configs:
  - passThroughAuth:
      grpc:
        address: auth-service.auth.svc.cluster.local:9001
        connectionTimeout: 5s
      # Pass headers to auth service
      request:
        allowedHeaders:
        - Authorization
        - X-Forwarded-For
        - X-Request-ID
      # Pass response headers back
      response:
        allowedUpstreamHeaders:
        - x-user-id
        - x-user-roles
```

### HTTP Auth Service
```yaml
passThroughAuth:
  http:
    url: "http://auth-service.auth.svc.cluster.local:8080/auth"
    method: POST
    connectionTimeout: 5s
```

---

## OPA Authorization

### Basic OPA Policy
```yaml
apiVersion: enterprise.kgateway.solo.io/v1alpha1
kind: AuthConfig
metadata:
  name: opa-auth
spec:
  configs:
  - opaAuth:
      modules:
      - name: policy
        namespace: kgateway-system
      query: "data.policy.allow == true"
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: policy
  namespace: kgateway-system
data:
  policy.rego: |
    package policy

    default allow = false

    # Allow GET requests to public paths
    allow {
      input.request.method == "GET"
      startswith(input.request.path, "/public/")
    }

    # Allow admins everywhere
    allow {
      input.jwt.claims.roles[_] == "admin"
    }

    # Allow users to their own resources
    allow {
      input.request.method == "GET"
      input.request.path == concat("/", ["users", input.jwt.claims.sub])
    }
```

### OPA with External Server
```yaml
opaAuth:
  serverAddr: "http://opa.opa-system.svc.cluster.local:8181"
  query: "data.authz.allow"
  options:
    fastInputConversion: true
```

---

## Multi-Auth Chains

### JWT + OPA (Auth + Authz)
```yaml
apiVersion: enterprise.kgateway.solo.io/v1alpha1
kind: AuthConfig
metadata:
  name: jwt-opa-chain
spec:
  configs:
  # First: Validate JWT
  - jwt:
      issuer: "https://auth.example.com/"
      jwksUri: "https://auth.example.com/.well-known/jwks.json"
      claimsToHeaders:
      - claim: sub
        header: x-user-id
      - claim: roles
        header: x-user-roles
  # Then: Check OPA policy
  - opaAuth:
      modules:
      - name: authz-policy
        namespace: kgateway-system
      query: "data.authz.allow"
```

### API Key OR JWT
```yaml
spec:
  # Either auth method can succeed
  booleanExpr: "apikey || jwt"
  configs:
  - name: apikey
    apiKeyAuth:
      headerName: x-api-key
      labelSelector:
        app: my-api
  - name: jwt
    jwt:
      issuer: "https://auth.example.com/"
      jwksUri: "https://auth.example.com/.well-known/jwks.json"
```

### Path-Specific Auth
```yaml
# Public paths - no auth
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: public-routes
spec:
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /public/
    backendRefs:
    - name: api
      port: 8080
---
# Protected paths - JWT required
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: protected-routes
spec:
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /api/
    backendRefs:
    - name: api
      port: 8080
---
apiVersion: gateway.kgateway.dev/v1alpha1
kind: TrafficPolicy
metadata:
  name: protected-auth
spec:
  targetRefs:
  - kind: HTTPRoute
    name: protected-routes  # Only applies to protected
  extAuth:
    configRef:
      name: jwt-auth
```
