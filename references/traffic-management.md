# kGateway Traffic Management

Complete examples for routing, transformations, and traffic manipulation.

## Table of Contents
- [Path-Based Routing](#path-based-routing)
- [Header-Based Routing](#header-based-routing)
- [Query Parameter Matching](#query-parameter-matching)
- [Method Matching](#method-matching)
- [Header Modification](#header-modification)
- [URL Rewriting](#url-rewriting)
- [Redirects](#redirects)
- [Traffic Splitting](#traffic-splitting)
- [Request Mirroring](#request-mirroring)
- [Transformations (Enterprise)](#transformations-enterprise)

---

## Path-Based Routing

### Exact Path
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: exact-path-route
spec:
  parentRefs:
  - name: http-gateway
    namespace: kgateway-system
  rules:
  - matches:
    - path:
        type: Exact
        value: /api/health
    backendRefs:
    - name: health-service
      port: 8080
```

### Path Prefix
```yaml
rules:
- matches:
  - path:
      type: PathPrefix
      value: /api/v1/
  backendRefs:
  - name: api-v1
    port: 8080
- matches:
  - path:
      type: PathPrefix
      value: /api/v2/
  backendRefs:
  - name: api-v2
    port: 8080
```

### Regex Path (Experimental)
```yaml
# Requires experimental Gateway API CRDs
rules:
- matches:
  - path:
      type: RegularExpression
      value: "^/users/[0-9]+$"
  backendRefs:
  - name: user-service
    port: 8080
```

---

## Header-Based Routing

### Exact Header Match
```yaml
rules:
- matches:
  - headers:
    - name: x-api-version
      value: "2"
  backendRefs:
  - name: api-v2
    port: 8080
```

### Multiple Headers (AND)
```yaml
rules:
- matches:
  - headers:
    - name: x-api-version
      value: "2"
    - name: x-tenant
      value: "premium"
  backendRefs:
  - name: premium-api-v2
    port: 8080
```

### Header Presence
```yaml
rules:
- matches:
  - headers:
    - name: Authorization
      type: RegularExpression
      value: ".*"  # Header exists with any value
  backendRefs:
  - name: authenticated-service
    port: 8080
```

---

## Query Parameter Matching

```yaml
rules:
- matches:
  - queryParams:
    - name: version
      value: beta
  backendRefs:
  - name: beta-service
    port: 8080
```

### Multiple Query Parameters
```yaml
rules:
- matches:
  - queryParams:
    - name: format
      value: json
    - name: verbose
      value: "true"
  backendRefs:
  - name: verbose-json-service
    port: 8080
```

---

## Method Matching

```yaml
rules:
# Read operations
- matches:
  - method: GET
    path:
      type: PathPrefix
      value: /api/
  backendRefs:
  - name: read-replica
    port: 8080
# Write operations
- matches:
  - method: POST
    path:
      type: PathPrefix
      value: /api/
  backendRefs:
  - name: write-primary
    port: 8080
- matches:
  - method: PUT
    path:
      type: PathPrefix
      value: /api/
  backendRefs:
  - name: write-primary
    port: 8080
- matches:
  - method: DELETE
    path:
      type: PathPrefix
      value: /api/
  backendRefs:
  - name: write-primary
    port: 8080
```

---

## Header Modification

### Add/Set Request Headers
```yaml
rules:
- filters:
  - type: RequestHeaderModifier
    requestHeaderModifier:
      add:
      - name: X-Request-ID
        value: "%REQ(X-REQUEST-ID)%"
      - name: X-Forwarded-Proto
        value: https
      set:
      - name: Host
        value: internal-api.default.svc.cluster.local
  backendRefs:
  - name: backend
    port: 8080
```

### Remove Request Headers
```yaml
filters:
- type: RequestHeaderModifier
  requestHeaderModifier:
    remove:
    - X-Forwarded-For
    - X-Real-IP
    - Cookie
```

### Add Response Headers
```yaml
filters:
- type: ResponseHeaderModifier
  responseHeaderModifier:
    add:
    - name: X-Content-Type-Options
      value: nosniff
    - name: X-Frame-Options
      value: DENY
    - name: Strict-Transport-Security
      value: "max-age=31536000; includeSubDomains"
    set:
    - name: Cache-Control
      value: "no-cache, no-store, must-revalidate"
```

---

## URL Rewriting

### Rewrite Hostname
```yaml
filters:
- type: URLRewrite
  urlRewrite:
    hostname: internal-service.namespace.svc.cluster.local
```

### Rewrite Path Prefix
```yaml
# /api/v1/users → /users
filters:
- type: URLRewrite
  urlRewrite:
    path:
      type: ReplacePrefixMatch
      replacePrefixMatch: /
```

### Full Path Replacement
```yaml
# Any path → /v2/handler
filters:
- type: URLRewrite
  urlRewrite:
    path:
      type: ReplaceFullPath
      replaceFullPath: /v2/handler
```

---

## Redirects

### HTTP to HTTPS
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: https-redirect
spec:
  parentRefs:
  - name: http-gateway
    sectionName: http  # HTTP listener
  rules:
  - filters:
    - type: RequestRedirect
      requestRedirect:
        scheme: https
        statusCode: 301
```

### Domain Redirect
```yaml
filters:
- type: RequestRedirect
  requestRedirect:
    hostname: www.example.com
    statusCode: 301
```

### Path Redirect
```yaml
filters:
- type: RequestRedirect
  requestRedirect:
    path:
      type: ReplaceFullPath
      replaceFullPath: /new-location
    statusCode: 302
```

---

## Traffic Splitting

### Canary Deployment (90/10)
```yaml
rules:
- backendRefs:
  - name: app-stable
    port: 8080
    weight: 90
  - name: app-canary
    port: 8080
    weight: 10
```

### Blue/Green (50/50)
```yaml
rules:
- backendRefs:
  - name: app-blue
    port: 8080
    weight: 50
  - name: app-green
    port: 8080
    weight: 50
```

### A/B Testing with Headers
```yaml
rules:
# Test group gets new version
- matches:
  - headers:
    - name: x-test-group
      value: "true"
  backendRefs:
  - name: app-experimental
    port: 8080
# Everyone else gets stable
- backendRefs:
  - name: app-stable
    port: 8080
```

---

## Request Mirroring

Mirror traffic to a secondary backend for testing:

```yaml
rules:
- filters:
  - type: RequestMirror
    requestMirror:
      backendRef:
        name: shadow-service
        port: 8080
      percent: 10  # Mirror 10% of traffic
  backendRefs:
  - name: primary-service
    port: 8080
```

---

## Transformations (Enterprise)

### Inja Templates
```yaml
apiVersion: gateway.kgateway.dev/v1alpha1
kind: RouteOption
metadata:
  name: transform-request
spec:
  targetRefs:
  - kind: HTTPRoute
    name: app-routes
  options:
    stagedTransformations:
      early:
        requestTransforms:
        - requestTransformation:
            transformationTemplate:
              headers:
                x-user-id:
                  text: "{{ header(\"authorization\") | split(\" \") | last | base64_decode | json | get(\"sub\") }}"
              body:
                text: |
                  {
                    "original": {{ body() }},
                    "timestamp": "{{ env(\"TIMESTAMP\") }}",
                    "source": "{{ header(\"x-forwarded-for\") }}"
                  }
```

### Extract from Path
```yaml
options:
  stagedTransformations:
    early:
      requestTransforms:
      - matchers:
        - prefix: /users/
        requestTransformation:
          transformationTemplate:
            extractors:
              userId:
                header: ':path'
                regex: '/users/([^/]+)'
                subgroup: 1
            headers:
              x-user-id:
                text: "{{ userId }}"
```

### Response Transformation
```yaml
options:
  stagedTransformations:
    regular:
      responseTransforms:
      - responseTransformation:
          transformationTemplate:
            body:
              text: |
                {
                  "data": {{ body() }},
                  "meta": {
                    "requestId": "{{ request_header(\"x-request-id\") }}",
                    "duration": "{{ dynamic_metadata(\"envoy.filters.http.router\", \"upstream_request_time\") }}"
                  }
                }
```
