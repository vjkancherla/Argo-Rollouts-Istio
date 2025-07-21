# Header-based Routing During Canary Lifecycle

## Key Concept: Subset Labels vs Pod Scaling

The crucial point is that **Istio routing is based on subset definitions, not individual pods**. Let me show you exactly what happens:

### DestinationRule Subset Definition
```yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: demo-app-dr
spec:
  host: demo-app-service
  subsets:
  - name: stable
    labels:
      app: demo-app              # ← This matches ALL stable pods
  - name: canary
    labels:
      app: demo-app              # ← This matches ALL canary pods
```

### VirtualService with Header Routing
```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: demo-app-vs
spec:
  hosts:
  - demo-app-service
  http:
  # Header-based routing ALWAYS works
  - match:
    - headers:
        x-canary-user:
          exact: "true"
    route:
    - destination:
        host: demo-app-service
        subset: canary            # ← Routes to ALL canary pods
      weight: 100
  
  # Regular traffic split (managed by Argo Rollouts)
  - name: primary
    route:
    - destination:
        host: demo-app-service
        subset: stable
      weight: 90                  # ← Argo manages this
    - destination:
        host: demo-app-service
        subset: canary
      weight: 10                  # ← Argo manages this
```

---

## Lifecycle Scenarios

### Scenario 1: Initial Canary Deployment (10% traffic)

**Pod State:**
```
Stable ReplicaSet:  [v1.0] [v1.0] [v1.0] [v1.0] [v1.0]
Canary ReplicaSet:  [v1.21]
```

**Routing Behavior:**
```
Regular traffic (no headers):
├─ 90% → stable subset → any of 5 stable pods
└─ 10% → canary subset → the 1 canary pod

Header traffic (x-canary-user: true):
└─ 100% → canary subset → the 1 canary pod
```

**Key Point:** Header-based routing works even with just 1 canary pod!

---

### Scenario 2: Mid-deployment (50% traffic)

**Pod State:**
```
Stable ReplicaSet:  [v1.0] [v1.0] [v1.0]
Canary ReplicaSet:  [v1.21] [v1.21] [v1.21]
```

**Routing Behavior:**
```
Regular traffic (no headers):
├─ 50% → stable subset → load balanced across 3 stable pods
└─ 50% → canary subset → load balanced across 3 canary pods

Header traffic (x-canary-user: true):
└─ 100% → canary subset → load balanced across 3 canary pods
```

**Key Point:** Headers still work - now with load balancing across multiple canary pods!

---

### Scenario 3: Near completion (75% traffic)

**Pod State:**
```
Stable ReplicaSet:  [v1.0] [v1.0]
Canary ReplicaSet:  [v1.21] [v1.21] [v1.21] [v1.21]
```

**Routing Behavior:**
```
Regular traffic (no headers):
├─ 25% → stable subset → load balanced across 2 stable pods
└─ 75% → canary subset → load balanced across 4 canary pods

Header traffic (x-canary-user: true):
└─ 100% → canary subset → load balanced across 4 canary pods
```

---

### Scenario 4: Full Promotion (100% canary)

**Pod State:**
```
Stable ReplicaSet:  (terminated)
Canary ReplicaSet:  [v1.21] [v1.21] [v1.21] [v1.21] [v1.21]
```

**What happens to header routing?**

#### Option A: Headers become irrelevant (both routes go to same pods)
```yaml
# After promotion, VirtualService looks like this:
http:
- match:
  - headers:
      x-canary-user:
        exact: "true"
  route:
  - destination:
      host: demo-app-service
      subset: canary          # ← All pods are now "canary" (v1.21)
    weight: 100

- name: primary
  route:
  - destination:
      host: demo-app-service
      subset: stable          # ← No pods match this anymore
    weight: 0
  - destination:
      host: demo-app-service
      subset: canary          # ← All pods are here
    weight: 100
```

**Result:** Both header and non-header traffic go to the same v1.21 pods.

#### Option B: Clean up headers after promotion (Recommended)
```yaml
# After promotion, update VirtualService to:
http:
- name: primary
  route:
  - destination:
      host: demo-app-service
      subset: stable          # ← Now points to v1.21 pods
    weight: 100
```

---

## Argo Rollouts Label Management

### How Argo Rollouts Manages Pod Labels

```yaml
# During rollout, Argo Rollouts creates pods with different labels:

# Stable pods (old ReplicaSet):
metadata:
  labels:
    app: demo-app
    rollouts-pod-template-hash: "7bf8c7c8f9"    # ← Stable hash
    
# Canary pods (new ReplicaSet):  
metadata:
  labels:
    app: demo-app
    rollouts-pod-template-hash: "86d4c9c7f2"    # ← Canary hash
```

### More Precise Subset Definition (Advanced)

```yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: demo-app-dr
spec:
  host: demo-app-service
  subsets:
  - name: stable
    labels:
      app: demo-app
      # Optionally add more specific labels
  - name: canary
    labels:
      app: demo-app
      # Argo Rollouts automatically manages these labels
```

---

## Production Recommendations

### 1. Persistent Header Routing (Recommended)
```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: demo-app-vs
spec:
  hosts:
  - demo-app-service
  http:
  # Beta users always get latest version
  - match:
    - headers:
        x-user-type:
          exact: "beta"
    route:
    - destination:
        host: demo-app-service
        subset: canary          # During rollout: new version
      weight: 100               # After rollout: becomes "stable"
  
  # Internal services get canary first
  - match:
    - headers:
        x-service-type:
          exact: "internal"
    route:
    - destination:
        host: demo-app-service
        subset: canary
      weight: 100
  
  # Regular traffic split
  - name: primary
    route:
    - destination:
        host: demo-app-service
        subset: stable
      weight: 100
    - destination:
        host: demo-app-service
        subset: canary
      weight: 0
```

### 2. Post-Promotion Cleanup
```bash
# After successful promotion, you might want to:

# Option 1: Remove header routing rules
kubectl patch virtualservice demo-app-vs --type='json' -p='[
  {"op": "remove", "path": "/spec/http/0"}
]'

# Option 2: Update header routing to point to "stable"
# (so beta users get the promoted version immediately in next rollout)
```

### 3. Automated Header Management
```yaml
# Use Argo Rollouts hooks for automatic cleanup
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: demo-app
spec:
  strategy:
    canary:
      # ... other config ...
      hooks:
        postPromotion:
        - templateName: cleanup-headers
```

---

## Key Takeaways

### ✅ **Headers Always Work Throughout Rollout**
- Header routing is based on subset names, not individual pods
- As more canary pods are created, headers still route to the canary subset
- Load balancing happens automatically across all pods in the subset

### ✅ **No Manual Updates Needed**
- You don't need to update headers as pods scale up/down
- Argo Rollouts manages pod labels automatically
- Istio handles service discovery and load balancing

### ✅ **After Full Promotion**
- Headers might become redundant (both routes go to same pods)
- Consider cleaning up header rules post-promotion
- Or keep them for the next deployment cycle

### ⚠️ **Consider Your Use Case**
- **Temporary canary access:** Remove headers after promotion
- **Persistent beta access:** Keep headers pointing to "canary" subset
- **Internal testing:** Keep separate routing for internal services

The beauty is that **you don't need to manage individual pod routing** - Istio and Argo Rollouts handle all the complexity while your header-based routing rules remain simple and consistent!