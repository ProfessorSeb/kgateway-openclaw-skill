# kGateway Skill for OpenClaw

Envoy-based Kubernetes-native API gateway skill implementing the Gateway API.

## Installation

```bash
clawdhub install kgateway
```

Or clone directly:
```bash
git clone https://github.com/ProfessorSeb/kgateway-openclaw-skill.git ~/.openclaw/workspace/skills/kgateway
```

## Coverage

| Feature | Open Source | Enterprise |
|---------|-------------|------------|
| Gateway API (HTTPRoute, GRPCRoute, TCPRoute) | ✅ | ✅ |
| Envoy-based data plane | ✅ | ✅ |
| Traffic management (routing, headers, transforms) | ✅ | ✅ |
| TLS, CORS, CSRF | ✅ | ✅ |
| Local rate limiting | ✅ | ✅ |
| Retries, timeouts, circuit breakers | ✅ | ✅ |
| Access logging, OpenTelemetry | ✅ | ✅ |
| Built-in rate limiting server | - | ✅ |
| JWT, OIDC/OAuth 2.0, OPA | - | ✅ |
| External auth service | - | ✅ |
| Staged transformations | - | ✅ |
| Multicluster federation | - | ✅ |
| FIPS/LTS builds | - | ✅ |
| 24x7 support | - | ✅ |

## Documentation

- [SKILL.md](SKILL.md) — Main skill instructions
- [references/traffic-management.md](references/traffic-management.md) — Routing, rewrites, transforms
- [references/security.md](references/security.md) — TLS, CORS, CSRF, mTLS
- [references/authentication.md](references/authentication.md) — JWT, OIDC, API keys (Enterprise)
- [references/resiliency.md](references/resiliency.md) — Retries, timeouts, circuit breakers

## Links

- Open Source: https://kgateway.dev (CNCF Sandbox)
- Enterprise: https://docs.solo.io/kgateway
- GitHub: https://github.com/kgateway-dev/kgateway
