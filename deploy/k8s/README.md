# Kubernetes Manifests — SorobanPay

This directory contains Kubernetes manifests for deploying SorobanPay backend
services to a cluster.

## Redis StatefulSet (issue #1444)

Redis is used for:
- **Distributed rate limiting** — `ioredis` + `rate-limiter-flexible` in the
  backend so rate limits are shared across all backend replicas.
- **MRR analytics caching** — Payout summary queries are cached in Redis to
  reduce database load during report generation.

### Files

| File | Purpose |
|------|---------|
| `redis-config.yaml` | ConfigMap with `redis.conf` tuned for caching (256MB max, LRU eviction) |
| `redis-service.yaml` | ClusterIP (`redis`) + headless (`redis-headless`) services |
| `redis-statefulset.yaml` | StatefulSet with PVC, resource limits (256Mi / 0.25 CPU), health probes |
| `kustomization.yaml` | Kustomize manifest |

### Resource limits

Per the issue acceptance criteria:

| Resource | Request | Limit |
|----------|---------|-------|
| CPU | 125m | 250m (0.25) |
| Memory | 128Mi | 256Mi |

### Health probes

| Probe | Command | Initial Delay | Period | Failure Threshold |
|-------|---------|--------------|--------|-------------------|
| Startup | `redis-cli PING` | 5s | 5s | 6 (30s max) |
| Liveness | `redis-cli PING` | 10s | 15s | 3 |
| Readiness | `redis-cli PING` | 5s | 10s | 3 |

### Configuration

The Redis instance is configured via the `redis-config` ConfigMap. Key settings:

| Setting | Value | Rationale |
|---------|-------|-----------|
| `maxmemory` | 256mb | Matches container memory limit |
| `maxmemory-policy` | allkeys-lru | Evict least-recently-used keys; safe for ephemeral cache |
| `save` | `""` (disabled) | No RDB persistence — cache data is not durable |
| `appendonly` | no | No AOF — same reason |

To enable persistence for disaster recovery, change `save` and `appendonly`
in `redis-config.yaml` and increase the PVC size in `redis-statefulset.yaml`.

### Apply

```bash
kubectl apply -k deploy/k8s/
```

### Verify

```bash
# Check the StatefulSet
kubectl get statefulset redis -n sorobanpay

# Check the pods
kubectl get pods -n sorobanpay -l app=redis

# Verify Redis responds
kubectl exec -n sorobanpay redis-0 -- redis-cli PING
# Expected: PONG
```

### Backend integration

The backend connects to Redis using the `REDIS_URL` environment variable:

```env
REDIS_URL=redis://redis.sorobanpay.svc.cluster.local:6379
```

When `REDIS_URL` is set:
- Rate limiting uses `rate-limiter-flexible` with `ioredis` (distributed)
- MRR/summary queries are cached in Redis (60s TTL)
- The `/health` endpoint reports Redis connectivity status

When `REDIS_URL` is unset:
- Rate limiting falls back to in-memory (per-replica, not shared)
- Caching is disabled (all queries hit the database directly)
- The `/health` endpoint reports `redis: "disabled"` (not a failure)
