# kGateway Resiliency

Complete examples for making your gateway resilient.

## Table of Contents
- [Retries](#retries)
- [Timeouts](#timeouts)
- [Circuit Breakers](#circuit-breakers)
- [Health Checks](#health-checks)
- [Load Balancing](#load-balancing)
- [Failover](#failover)
- [Buffering](#buffering)

---

## Retries

### Basic Retries
```yaml
apiVersion: gateway.kgateway.dev/v1alpha1
kind: TrafficPolicy
metadata:
  name: retry-policy
spec:
  targetRefs:
  - kind: HTTPRoute
    name: api-routes
  retry:
    attempts: 3
    perTryTimeout: 5s
    retryOn:
    - 5xx
    - reset
    - connect-failure
```

### Advanced Retry Configuration
```yaml
retry:
  attempts: 5
  perTryTimeout: 10s
  retryOn:
  - 5xx
  - reset
  - connect-failure
  - retriable-4xx
  - refused-stream
  - cancelled
  - deadline-exceeded
  - resource-exhausted
  backOff:
    baseInterval: 100ms
    maxInterval: 2s
  retryHostPredicate:
  - name: envoy.retry_host_predicates.previous_hosts
  hostSelectionRetryMaxAttempts: 3
```

### Retry on Specific Status Codes
```yaml
retry:
  attempts: 3
  retryOn:
  - retriable-status-codes
  retriableStatusCodes:
  - 502
  - 503
  - 504
```

### Retry Budget
```yaml
retry:
  attempts: 3
  retryBudget:
    budgetPercent: 25.0  # Max 25% of requests can be retries
    minRetriesPerSecond: 10
```

---

## Timeouts

### Request Timeout
```yaml
apiVersion: gateway.kgateway.dev/v1alpha1
kind: TrafficPolicy
metadata:
  name: timeout-policy
spec:
  targetRefs:
  - kind: HTTPRoute
    name: api-routes
  timeout:
    request: 30s
```

### Idle Timeout
```yaml
timeout:
  request: 30s
  idle: 5m  # Close idle connections after 5 minutes
```

### Per-Route Timeouts
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
spec:
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /slow-api
    timeouts:
      request: 120s
      backendRequest: 60s
    backendRefs:
    - name: slow-service
      port: 8080
  - matches:
    - path:
        type: PathPrefix
        value: /fast-api
    timeouts:
      request: 5s
    backendRefs:
    - name: fast-service
      port: 8080
```

---

## Circuit Breakers

### Basic Circuit Breaker
```yaml
apiVersion: gateway.kgateway.dev/v1alpha1
kind: TrafficPolicy
metadata:
  name: circuit-breaker
spec:
  targetRefs:
  - kind: HTTPRoute
    name: api-routes
  circuitBreaker:
    maxConnections: 1000
    maxPendingRequests: 100
    maxRequests: 1000
    maxRetries: 3
```

### Advanced Circuit Breaker
```yaml
circuitBreaker:
  maxConnections: 1000
  maxPendingRequests: 100
  maxRequests: 1000
  maxRetries: 3
  trackRemaining: true
  outlierDetection:
    consecutive5xx: 5
    interval: 10s
    baseEjectionTime: 30s
    maxEjectionPercent: 50
    consecutiveGatewayFailure: 3
    enforcingConsecutive5xx: 100
    enforcingSuccessRate: 100
    successRateMinimumHosts: 3
    successRateRequestVolume: 100
    successRateStdevFactor: 1900
```

---

## Health Checks

### Active Health Checks
```yaml
apiVersion: gateway.kgateway.dev/v1alpha1
kind: Upstream
metadata:
  name: backend-upstream
spec:
  kube:
    serviceName: backend
    serviceNamespace: default
    servicePort: 8080
  healthChecks:
  - timeout: 5s
    interval: 10s
    unhealthyThreshold: 3
    healthyThreshold: 2
    httpHealthCheck:
      path: /health
      expectedStatuses:
      - start: 200
        end: 299
```

### gRPC Health Check
```yaml
healthChecks:
- timeout: 5s
  interval: 10s
  grpcHealthCheck:
    serviceName: "myapp.HealthService"
```

### TCP Health Check
```yaml
healthChecks:
- timeout: 5s
  interval: 10s
  tcpHealthCheck:
    send:
      text: "ping"
    receive:
    - text: "pong"
```

---

## Load Balancing

### Round Robin (Default)
```yaml
apiVersion: gateway.kgateway.dev/v1alpha1
kind: Upstream
spec:
  loadBalancer:
    roundRobin: {}
```

### Least Connections
```yaml
loadBalancer:
  leastRequest:
    choiceCount: 3
```

### Random
```yaml
loadBalancer:
  random: {}
```

### Ring Hash (Consistent Hashing)
```yaml
loadBalancer:
  ringHash:
    hashKey:
      header:
        headerName: x-user-id
    ringHashConfig:
      minimumRingSize: 1024
      maximumRingSize: 8192
```

### Maglev (Consistent Hashing)
```yaml
loadBalancer:
  maglev:
    hashKey:
      cookie:
        name: session_id
        ttl: 0s  # Session cookie
```

### Weighted Endpoints
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
spec:
  rules:
  - backendRefs:
    - name: backend-a
      port: 8080
      weight: 70
    - name: backend-b
      port: 8080
      weight: 30
```

---

## Failover

### Priority-Based Failover
```yaml
apiVersion: gateway.kgateway.dev/v1alpha1
kind: Upstream
metadata:
  name: failover-upstream
spec:
  failover:
    priorityGroups:
    - priority: 0  # Primary
      - kube:
          serviceName: primary-backend
          serviceNamespace: default
          servicePort: 8080
    - priority: 1  # Secondary
      - kube:
          serviceName: secondary-backend
          serviceNamespace: default
          servicePort: 8080
    - priority: 2  # DR
      - static:
          hosts:
          - addr: dr-backend.example.com
            port: 443
```

### Multi-Cluster Failover
```yaml
# With Solo Enterprise multicluster
failover:
  localPriority: 0
  remotePriority: 1
  clusters:
  - name: cluster-east
    priority: 0
  - name: cluster-west
    priority: 1
```

---

## Buffering

### Request Buffering
```yaml
apiVersion: gateway.kgateway.dev/v1alpha1
kind: TrafficPolicy
metadata:
  name: buffer-policy
spec:
  targetRefs:
  - kind: HTTPRoute
    name: api-routes
  buffer:
    maxRequestBytes: 10485760  # 10MB
```

### Response Buffering
```yaml
buffer:
  maxRequestBytes: 10485760
  maxResponseBytes: 52428800  # 50MB
```

---

## Connection Limits

### Per-Connection Limits
```yaml
apiVersion: gateway.kgateway.dev/v1alpha1
kind: ListenerOption
metadata:
  name: connection-limits
spec:
  targetRefs:
  - kind: Gateway
    name: http-gateway
  options:
    perConnectionBufferLimitBytes: 32768
    connectionLimit:
      maxActiveConnections: 10000
```

### TCP Keepalive
```yaml
tcpKeepalive:
  probes: 9
  time: 7200s
  interval: 75s
```

---

## Complete Resiliency Example

```yaml
apiVersion: gateway.kgateway.dev/v1alpha1
kind: TrafficPolicy
metadata:
  name: full-resiliency
spec:
  targetRefs:
  - kind: HTTPRoute
    name: critical-api
  # Timeouts
  timeout:
    request: 30s
    idle: 5m
  # Retries
  retry:
    attempts: 3
    perTryTimeout: 10s
    retryOn:
    - 5xx
    - reset
    - connect-failure
    backOff:
      baseInterval: 100ms
      maxInterval: 1s
  # Circuit breaker
  circuitBreaker:
    maxConnections: 1000
    maxPendingRequests: 100
    maxRequests: 1000
    outlierDetection:
      consecutive5xx: 5
      interval: 10s
      baseEjectionTime: 30s
      maxEjectionPercent: 50
  # Buffering
  buffer:
    maxRequestBytes: 10485760
```
