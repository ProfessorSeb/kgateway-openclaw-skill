# kGateway Security

Complete examples for securing your API gateway.

## Table of Contents
- [TLS Termination](#tls-termination)
- [TLS Passthrough](#tls-passthrough)
- [Backend TLS (mTLS)](#backend-tls-mtls)
- [CORS](#cors)
- [CSRF Protection](#csrf-protection)
- [Certificate Management](#certificate-management)
- [IP Allowlisting](#ip-allowlisting)

---

## TLS Termination

### Basic HTTPS Listener
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: gateway-tls
  namespace: kgateway-system
type: kubernetes.io/tls
data:
  tls.crt: <base64-encoded-certificate>
  tls.key: <base64-encoded-private-key>
---
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: https-gateway
  namespace: kgateway-system
spec:
  gatewayClassName: kgateway
  listeners:
  - name: https
    port: 443
    protocol: HTTPS
    tls:
      mode: Terminate
      certificateRefs:
      - name: gateway-tls
    allowedRoutes:
      namespaces:
        from: All
```

### Multiple Certificates (SNI)
```yaml
listeners:
- name: https-api
  port: 443
  protocol: HTTPS
  hostname: "api.example.com"
  tls:
    mode: Terminate
    certificateRefs:
    - name: api-tls-cert
- name: https-admin
  port: 443
  protocol: HTTPS
  hostname: "admin.example.com"
  tls:
    mode: Terminate
    certificateRefs:
    - name: admin-tls-cert
```

### Wildcard Certificate
```yaml
listeners:
- name: https-wildcard
  port: 443
  protocol: HTTPS
  hostname: "*.example.com"
  tls:
    mode: Terminate
    certificateRefs:
    - name: wildcard-tls-cert
```

---

## TLS Passthrough

Pass TLS directly to backend (no termination):

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
spec:
  listeners:
  - name: tls-passthrough
    port: 443
    protocol: TLS
    tls:
      mode: Passthrough
    allowedRoutes:
      kinds:
      - kind: TLSRoute
---
apiVersion: gateway.networking.k8s.io/v1alpha2
kind: TLSRoute
metadata:
  name: tls-passthrough-route
spec:
  parentRefs:
  - name: https-gateway
    sectionName: tls-passthrough
  hostnames:
  - "secure.example.com"
  rules:
  - backendRefs:
    - name: secure-backend
      port: 443
```

---

## Backend TLS (mTLS)

### Upstream TLS to Backend
```yaml
apiVersion: gateway.kgateway.dev/v1alpha1
kind: Upstream
metadata:
  name: secure-backend
spec:
  kube:
    serviceName: backend-service
    serviceNamespace: default
    servicePort: 8443
  sslConfig:
    sni: backend.internal
    secretRef:
      name: backend-client-cert
      namespace: kgateway-system
```

### Backend TLS Policy
```yaml
apiVersion: gateway.networking.k8s.io/v1alpha2
kind: BackendTLSPolicy
metadata:
  name: backend-tls
spec:
  targetRef:
    group: ""
    kind: Service
    name: backend-service
  tls:
    caCertRefs:
    - name: backend-ca-cert
      kind: ConfigMap
    hostname: backend.internal
```

---

## CORS

### Basic CORS Policy
```yaml
apiVersion: gateway.kgateway.dev/v1alpha1
kind: TrafficPolicy
metadata:
  name: cors-policy
  namespace: default
spec:
  targetRefs:
  - group: gateway.networking.k8s.io
    kind: HTTPRoute
    name: api-routes
  cors:
    allowOrigins:
    - exact: "https://app.example.com"
    - exact: "https://admin.example.com"
    allowMethods:
    - GET
    - POST
    - PUT
    - DELETE
    - OPTIONS
    allowHeaders:
    - Authorization
    - Content-Type
    - X-Requested-With
    exposeHeaders:
    - X-Request-Id
    - X-RateLimit-Remaining
    maxAge: 86400s
    allowCredentials: true
```

### Wildcard CORS (Development)
```yaml
cors:
  allowOrigins:
  - regex: ".*"
  allowMethods:
  - "*"
  allowHeaders:
  - "*"
  allowCredentials: false
```

### CORS with Regex Origins
```yaml
cors:
  allowOrigins:
  - regex: "https://.*\\.example\\.com"
  - regex: "https://localhost:\\d+"
  allowMethods:
  - GET
  - POST
```

---

## CSRF Protection

### Enable CSRF
```yaml
apiVersion: gateway.kgateway.dev/v1alpha1
kind: TrafficPolicy
metadata:
  name: csrf-protection
spec:
  targetRefs:
  - kind: HTTPRoute
    name: web-routes
  csrf:
    filterEnabled: true
    shadowEnabled: false
    additionalOrigins:
    - exact: "https://trusted-partner.com"
```

### CSRF with Shadow Mode (Logging Only)
```yaml
csrf:
  filterEnabled: false
  shadowEnabled: true  # Log violations without blocking
```

---

## Certificate Management

### Cert-Manager Integration
```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: gateway-cert
  namespace: kgateway-system
spec:
  secretName: gateway-tls
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  dnsNames:
  - "api.example.com"
  - "*.api.example.com"
---
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: admin@example.com
    privateKeySecretRef:
      name: letsencrypt-account-key
    solvers:
    - http01:
        gatewayHTTPRoute:
          parentRefs:
          - name: http-gateway
            namespace: kgateway-system
```

### Auto-Rotate Certificates
```yaml
# Cert-manager handles rotation automatically
# Gateway watches Secret and reloads on changes
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
spec:
  listeners:
  - name: https
    tls:
      certificateRefs:
      - name: gateway-tls  # Managed by cert-manager
```

---

## IP Allowlisting

### Source IP Filter
```yaml
apiVersion: gateway.kgateway.dev/v1alpha1
kind: TrafficPolicy
metadata:
  name: ip-allowlist
spec:
  targetRefs:
  - kind: HTTPRoute
    name: admin-routes
  rbac:
    principals:
    - ipBlocks:
      - cidr: "10.0.0.0/8"
      - cidr: "192.168.1.0/24"
      - cidr: "203.0.113.50/32"  # Specific IP
```

### Deny Specific IPs
```yaml
rbac:
  action: DENY
  principals:
  - ipBlocks:
    - cidr: "192.168.1.100/32"
    - cidr: "10.0.0.0/8"
```

---

## Security Headers

### Add Security Headers via Response Modifier
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
spec:
  rules:
  - filters:
    - type: ResponseHeaderModifier
      responseHeaderModifier:
        set:
        - name: X-Content-Type-Options
          value: nosniff
        - name: X-Frame-Options
          value: DENY
        - name: X-XSS-Protection
          value: "1; mode=block"
        - name: Strict-Transport-Security
          value: "max-age=31536000; includeSubDomains; preload"
        - name: Content-Security-Policy
          value: "default-src 'self'"
        - name: Referrer-Policy
          value: strict-origin-when-cross-origin
        - name: Permissions-Policy
          value: "geolocation=(), microphone=(), camera=()"
    backendRefs:
    - name: backend
      port: 8080
```

---

## TLS Minimum Version

### Enforce TLS 1.3
```yaml
# Via Helm values
gateway:
  listenerOptions:
    tlsParams:
      tlsMinimumProtocolVersion: TLSv1_3
      cipherSuites:
      - TLS_AES_256_GCM_SHA384
      - TLS_CHACHA20_POLY1305_SHA256
      - TLS_AES_128_GCM_SHA256
```

### Enforce TLS 1.2+
```yaml
gateway:
  listenerOptions:
    tlsParams:
      tlsMinimumProtocolVersion: TLSv1_2
      tlsMaximumProtocolVersion: TLSv1_3
```
