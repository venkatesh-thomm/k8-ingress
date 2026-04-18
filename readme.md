# Kubernetes Ingress — Evolution & Problems

## Stage 1: Old ALB Ingress Controller

Created an ALB but had serious problems:

- **target-type: instance** — ALB registered EC2 nodes, not pods. Traffic had to go through NodePort on every node before reaching the pod (extra hop)
- **One ALB per app** — no concept of grouping. 10 apps = 10 ALBs = high cost
- **v1beta1 API** — flat backend syntax, no pathType, ingressClassName via annotation. This API was removed in K8s 1.22

```
Internet → ALB (per app) → EC2 node (NodePort) → pod
```

---

## Stage 2: AWS Load Balancer Controller (Current)

Fixed all the above problems:

- **target-type: ip** — ALB routes directly to pod IP, no NodePort needed
- **group.name** — multiple apps share one ALB, much cheaper
- **v1 API** — proper pathType, nested backend, ingressClassName in spec

```
Internet → ALB (shared) → pod (directly)
```

But Ingress itself still has problems — see below.

---

## Problems with Ingress API (Why Move to Gateway)

**1. Annotation hell**
Any feature beyond basic host/path routing goes into annotations — no validation, no schema, easy to mistype. And annotations are vendor-specific, so nginx annotations don't work on ALB and vice versa.

**2. No role separation**
Infra config (TLS, ports, scheme) and app routing rules live in the same Ingress file. Dev team and infra team both touch the same resource — causes conflicts in real teams.

**3. Not extensible**
Ingress was designed for simple use cases. Advanced routing (header-based, weight-based canary, traffic splitting) is not in the spec — every vendor hacks it in differently via annotations.

---

## Stage 3: Gateway API (Future)

Splits the Ingress into 3 clean resources:

| Resource | Owner | Purpose |
|---|---|---|
| `GatewayClass` | Cluster Admin | Which controller to use |
| `Gateway` | Infra Team | LB config — ports, TLS, scheme |
| `HTTPRoute` | Dev Team | Routing rules — host, path, backend |

```
Internet → ALB (Gateway) → pod
```

Same ALB underneath — only the K8s side is cleaner. Dev team only touches HTTPRoute, infra team owns Gateway. No annotation hacks needed.

**Status:** GA since K8s 1.31. Ingress is in maintenance mode — no new features.

---

## What Changed in the API (Interview)

| | v1beta1 (old) | v1 (current) |
|---|---|---|
| apiVersion | `extensions/v1beta1` → `networking.k8s.io/v1beta1` | `networking.k8s.io/v1` |
| backend | flat: `serviceName`, `servicePort` | nested: `service.name`, `service.port.number` |
| pathType | not required | mandatory |
| ingressClassName | annotation | spec field |

## pathType

| pathType | Behavior |
|---|---|
| `Exact` | `/api` matches only `/api` |
| `Prefix` | `/api` matches `/api`, `/api/users`, `/api/orders` |
| `ImplementationSpecific` | controller decides |


# Why Move from Kubernetes Ingress to Gateway API

## TL;DR

> Ingress API is **feature-frozen** — it works today and will continue to work, but no new features will ever be added. All Kubernetes networking innovation is happening in **Gateway API**. Existing Ingress setups don't need immediate migration, but new projects should start with Gateway API directly.

---

## Current State of Ingress API

| Aspect | Status |
|---|---|
| Bug fixes | ✅ Active |
| Security patches | ✅ Active |
| New features | ❌ Frozen |
| Active development | ❌ Stopped |

The Kubernetes community has officially stated:
> *"Ingress is frozen. No new features will be added. Gateway API is the future."*

**Gateway API** became **GA (Generally Available)** in **Kubernetes 1.28 (2023)** and is now where all active development happens.

---

## Key Reasons to Move

### 1. Feature Frozen
- Ingress gets **only** bug fixes and security patches
- No new capabilities will ever be added
- Gateway API is where all new Kubernetes networking features land

### 2. Role Separation (Real Production Benefit)

**Old Ingress** — everything mixed in one resource, app developer needs AWS/infra knowledge:

```yaml
# Developer had to know infra details
annotations:
  alb.ingress.kubernetes.io/scheme: internet-facing
  alb.ingress.kubernetes.io/target-type: ip
spec:
  rules:          # routing also in the same resource
  - host: myapp.com
```

**Gateway API** — clean separation of concerns:

```
Platform/Infra Team  →  GatewayClass + Gateway  (AWS annotations live here)
App/Dev Team         →  HTTPRoute only           (zero AWS knowledge needed)
```

| Role | Resource | Owns |
|---|---|---|
| Infrastructure Team | `GatewayClass` | Controller config, provider |
| Platform/Ops Team | `Gateway` | Load balancer config, TLS, ports |
| App/Dev Team | `HTTPRoute` | Routing rules, path matching |

### 3. Richer Routing — No More Annotation Hacks

| Feature | Ingress | Gateway API |
|---|---|---|
| Canary deployments | Annotation hack | Native spec |
| Header matching | Annotation hack | Native spec |
| Traffic splitting | Annotation hack | Native spec |
| Retries & timeouts | Annotation hack | Native spec |
| Redirects | Annotation hack | Native spec |

### 4. Multi-Protocol Support

| Protocol | Ingress | Gateway API |
|---|---|---|
| HTTP/HTTPS | ✅ | ✅ |
| TCP | ❌ | ✅ |
| UDP | ❌ | ✅ |
| gRPC | ❌ | ✅ |

---

## API Version History (for reference)

```yaml
# OLD — deprecated, removed in k8s 1.22
apiVersion: extensions/v1beta1

# INTERMEDIATE — also deprecated
apiVersion: networking.k8s.io/v1beta1

# CURRENT — still valid, but feature-frozen
apiVersion: networking.k8s.io/v1

# FUTURE — actively developed
apiVersion: gateway.networking.k8s.io/v1
```

---

## Ingress vs Gateway API — Full Comparison

| | Ingress | Gateway API |
|---|---|---|
| API version | `networking.k8s.io/v1` | `gateway.networking.k8s.io/v1` |
| Status | Stable, feature-frozen | GA since k8s 1.28, actively developed |
| Routing config | Inside Ingress spec | Separate `HTTPRoute` resource |
| Controller config | Annotation-based | `GatewayClass` resource |
| Load balancer config | Annotation-based | `Gateway` resource |
| Role separation | ❌ None | ✅ Class / Gateway / Route |
| Traffic splitting | ❌ Annotation hack | ✅ Native |
| Protocol support | HTTP/HTTPS only | HTTP, TCP, UDP, gRPC |

---

## Migration Recommendation

| Situation | Recommendation |
|---|---|
| Existing Ingress in production | ✅ No rush — it still works fine |
| New cluster / new project | 🚀 Start with Gateway API directly |
| Planning for long term | 📅 Plan migration to Gateway API |

---

## Analogy

Think of it like **Python 2 vs Python 3**:
- **Python 2** → still worked, got security patches, no new features → eventually EOL
- **Ingress** → same path — works today, but Gateway API is the future

---

## One Line Summary

> *"Ingress is a stable legacy system — reliable but going nowhere. Gateway API is the future — better role separation, richer features, and all active Kubernetes networking development. Migrate when you can, but don't panic if you haven't yet."*