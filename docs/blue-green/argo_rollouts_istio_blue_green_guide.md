# Istio with Argo Rollouts Blue Green Deployment Guide

Argo Rollouts provides several deployment strategies to achieve progressive delivery. This document describes how to implement Blue Green deployments with Istio, providing instant traffic switching between stable and preview versions with zero downtime.

## Why Istio + Argo Rollouts vs Just Argo Rollouts?

**The KEY advantage of using Istio with Argo Rollouts for Blue Green deployments is the ability to precisely target and test the Green (preview) environment through header-based routing BEFORE promoting it to production.**

### Without Istio (Argo Rollouts Only):
- ❌ Limited testing of Green environment before promotion
- ❌ Green environment testing requires separate service endpoints
- ❌ No sophisticated traffic routing capabilities
- ❌ Difficult to simulate real user scenarios on Green environment

### With Istio + Argo Rollouts:
- ✅ **Header-based routing**: Target Green environment with specific headers (`x-preview-version: true`)
- ✅ **Production-like testing**: Test Green environment using the same ingress path as production
- ✅ **Confidence in promotion**: Thoroughly validate Green environment before switching traffic
- ✅ **Seamless integration**: Single endpoint serves both Blue and Green environments based on routing rules
- ✅ **Advanced traffic management**: Leverage Istio's full traffic management capabilities

**This header-based routing capability gives you the confidence to promote the Green environment, knowing it has been thoroughly tested under production-like conditions through the same ingress gateway and routing infrastructure.**

## Blue Green Deployment Strategy Overview

Blue Green deployment is a technique that reduces downtime and risk by running two identical production environments called Blue and Green. At any time, only one of the environments is live, with the other serving as a staging environment for the next release.

**How Blue Green Works with Argo Rollouts:**

1. **Initial State**: The "Blue" environment serves all production traffic (stable/active)
2. **Deployment**: A new "Green" environment is created with the updated application (preview)
3. **Testing**: The Green environment can be tested independently via the preview service
4. **Promotion**: Traffic is instantly switched from Blue to Green environment
5. **Cleanup**: The old Blue environment is scaled down after successful promotion

**Key Advantages:**
- **Instant rollbacks**: Switch back to previous version immediately if issues arise
- **Zero downtime**: Traffic switching happens instantaneously
- **Independent testing**: Preview environment allows thorough testing before promotion
- **Resource efficiency**: Only runs two environments (current + next)

For a comprehensive understanding of Blue Green deployments with Argo Rollouts, see the [complete Blue Green implementation guide](https://github.com/vjkancherla/Argo-Rollouts/blob/main/Argo-Rollouts-BlueGreen.txt).

## Prerequisites Installation

See [argo_rollouts_istio_guide.md](argo_rollouts_istio_guide.md) for installing all the prerequisites:
- Istio CRDs
- IstioD
- Istio Ingress Gateway
- Metal LB + extra config
- Argo Rollouts
- Argo Rollouts Plugin

---

## Core Application Configuration

**Deploy in the order defined below**

### Create a namespace for the demo

```bash
k create ns my-demo
```

### Enable Istio SideCar injection

```bash
k label namespace my-demo istio-injection=enabled
```

### Kubernetes Services

```bash
# services.yaml
k apply -n my-demo -f ../../examples/blue-green/basic/services.yaml
```

### Istio Gateway

```bash
# gateway.yaml
k apply -n my-demo -f ../../examples/blue-green/basic/gateway.yaml
```

### Istio VirtualService with Header Routing for Preview Testing

```bash
# virtualsvc.yaml
k apply -n my-demo -f ../../examples/blue-green/basic/virtualsvc.yaml
```

### Application Rollout

```bash
# rollout.yaml
k apply -n my-demo -f ../../examples/blue-green/basic/rollout.yaml
```

### Verify external access to the Application, via the Istio-Ingress-Gateway

**Traffic Flow:**
```
Internet -> Istio-Ingress-Gateway -> Gateway -> VirtualService -> Service -> Pod
```

```bash
k port-forward -n istio-ingress svc/istio-ingressgateway 8080:80

# nip.io allows you to map any IP Address to a hostname. Here demo-app.127.0.0.1.nip.io maps to 127.0.0.1/localhost
http://demo-app.127.0.0.1.nip.io:8080

curl http://demo-app.127.0.0.1.nip.io:8080
```

### Check the initial state of the Rollout

```bash
k argo rollouts -n my-demo get rollout demo-app
```

**Expected Output:**
```
Name:            demo-app
Namespace:       my-demo
Status:          ✔ Healthy
Strategy:        BlueGreen
Images:          nginx:1.20 (stable, active)
Replicas:
  Desired:       5
  Current:       5
  Updated:       5
  Ready:         5
  Available:     5

NAME                                  KIND        STATUS     AGE  INFO
⟳ demo-app                            Rollout     ✔ Healthy  14m
└──# revision:1
   └──⧉ demo-app-85d7cbc77b           ReplicaSet  ✔ Healthy  14m  stable,active
      ├──□ demo-app-85d7cbc77b-8jnmk  Pod         ✔ Running  14m  ready:2/2
      ├──□ demo-app-85d7cbc77b-g8nc8  Pod         ✔ Running  14m  ready:2/2
      ├──□ demo-app-85d7cbc77b-nkh2n  Pod         ✔ Running  14m  ready:2/2
      ├──□ demo-app-85d7cbc77b-cqftr  Pod         ✔ Running  14m  ready:2/2
      └──□ demo-app-85d7cbc77b-m2jf9  Pod         ✔ Running  14m  ready:2/2
```

Note that "revision:1" is marked as "stable,active" - this is the Blue environment serving all production traffic.

## Performing a Blue Green Update

### Update the App version

```bash
k apply -n my-demo -f ../../examples/blue-green/basic/rollout-update.yaml
```

### Continuously Monitor and verify the application in separate terminals

Run these in separate terminals to see the Istio routing advantage in action:
```bash 
# Monitor and log the rollout status every 10 seconds
while true; do
    {
        kubectl argo rollouts -n my-demo get rollout demo-app
        echo "---"
        date
    } >> rollout.log
    sleep 10
done

# Tail the rollout log
tail -f rollout.log

# PRODUCTION TRAFFIC - goes to active (Blue) environment
# This is your real users' experience - unaffected during Green testing
while true; do curl http://demo-app.127.0.0.1.nip.io:8080 ; sleep 5; done

# PREVIEW TRAFFIC - goes to preview (Green) environment for testing
# THIS IS THE ISTIO MAGIC: Same URL, different routing based on headers!
# Test your Green environment thoroughly before promoting
while true; do curl -H "x-preview-version: true" http://demo-app.127.0.0.1.nip.io:8080 ; sleep 5; done
```

### Check the state of the Blue Green Rollout

After applying the update, Argo Rollout performs the following:

**Blue Green Deployment Process:**

1. **Green Environment Creation**: Creates a new ReplicaSet with updated pods (Green environment)
2. **Preview Service Update**: Updates the preview service to point to Green environment
3. **Pause State**: Rollout pauses automatically for manual verification
4. **Testing Phase**: Green environment accessible via preview service with header routing

**Actions Performed:**
- Created a new replica-set for the Green environment
- Created new pods with updated application version
- Updated `demo-app-service-preview` service's selector to point to Green pods
- Updated Istio VirtualService to route preview traffic to Green environment
- Blue environment continues serving production traffic

**Check Rollout Status:**
```bash
k argo rollouts -n my-demo get rollout demo-app
```

**Output during Blue Green deployment:**
```
Name:            demo-app
Namespace:       my-demo
Status:          ॥ Paused
Message:         BlueGreenPause
Strategy:        BlueGreen
Images:          nginx:1.20 (active, preview, stable)
Replicas:
  Desired:       5
  Current:       10
  Updated:       5
  Ready:         5
  Available:     5

NAME                                  KIND        STATUS     AGE  INFO
⟳ demo-app                            Rollout     ॥ Paused   26m
├──# revision:2
│  └──⧉ demo-app-8597cbc469           ReplicaSet  ✔ Healthy  11m  preview
│     ├──□ demo-app-8597cbc469-484vp  Pod         ✔ Running  11m  ready:2/2
│     ├──□ demo-app-8597cbc469-bm7h8  Pod         ✔ Running  11m  ready:2/2
│     ├──□ demo-app-8597cbc469-h9zws  Pod         ✔ Running  11m  ready:2/2
│     ├──□ demo-app-8597cbc469-pxcwh  Pod         ✔ Running  11m  ready:2/2
│     └──□ demo-app-8597cbc469-r92bn  Pod         ✔ Running  11m  ready:2/2
└──# revision:1
   └──⧉ demo-app-85d7cbc77b           ReplicaSet  ✔ Healthy  26m  stable,active
      ├──□ demo-app-85d7cbc77b-8jnmk  Pod         ✔ Running  26m  ready:2/2
      ├──□ demo-app-85d7cbc77b-g8nc8  Pod         ✔ Running  26m  ready:2/2
      ├──□ demo-app-85d7cbc77b-nkh2n  Pod         ✔ Running  26m  ready:2/2
      ├──□ demo-app-85d7cbc77b-cqftr  Pod         ✔ Running  26m  ready:2/2
      └──□ demo-app-85d7cbc77b-m2jf9  Pod         ✔ Running  26m  ready:2/2
```

**Key Observations:**
- **revision:1** (Blue) is marked as "stable,active" - serving production traffic
- **revision:2** (Green) is marked as "preview" - available for testing
- Both environments are running simultaneously
- Total replicas: 10 (5 Blue + 5 Green)

### Testing the Green Environment - The Istio Advantage

**This is where Istio's header-based routing becomes crucial for Blue Green success:**

During the paused state, you can thoroughly test the Green environment using the SAME ingress endpoint as production, but with routing headers:

```bash
# Test the GREEN environment (v2.0) via preview header - SAME URL as production!
curl -H "x-preview-version: true" http://demo-app.127.0.0.1.nip.io:8080
# Returns: <html><body><h1>Demo App - Version v2.0</h1></body></html>

# Production traffic still goes to BLUE environment (v1.0) - SAME URL!
curl http://demo-app.127.0.0.1.nip.io:8080
# Returns: <html><body><h1>Demo App - Version v1.0</h1></body></html>
```

**Why this is powerful:**
- **Same Infrastructure**: Green environment tested through identical ingress gateway, VirtualService, and routing rules
- **Production Parity**: Network path, security policies, and traffic handling identical to production
- **Comprehensive Testing**: Run full integration tests, load tests, and user scenarios against Green environment
- **Risk Mitigation**: Discover routing, DNS, certificate, or infrastructure issues before promotion
- **Confidence Building**: Know that Green environment works exactly as production will

**Without Istio, you'd need separate service endpoints or port-forwards to test the Green environment, which doesn't give you the same confidence in the production routing path.**

## Promoting the Blue Green Release

Once testing is complete and you're satisfied with the Green environment, promote the rollout:

```bash
k argo rollouts -n my-demo promote demo-app
```

**Promotion Process:**
1. **Traffic Switch**: Active service instantly switches to point to Green environment
2. **Blue Becomes Inactive**: Blue environment stops receiving traffic
3. **Cleanup**: After a brief delay, Blue environment is scaled down

**Post-Promotion Status:**
```bash
k argo rollouts -n my-demo get rollout demo-app
```

**Output after promotion:**
```
Name:            demo-app
Namespace:       my-demo
Status:          ✔ Healthy
Strategy:        BlueGreen
Images:          nginx:1.20 (stable, active)
Replicas:
  Desired:       5
  Current:       5
  Updated:       5
  Ready:         5
  Available:     5

NAME                                  KIND        STATUS        AGE  INFO
⟳ demo-app                            Rollout     ✔ Healthy     28m
├──# revision:2
│  └──⧉ demo-app-8597cbc469           ReplicaSet  ✔ Healthy     13m  stable,active
│     ├──□ demo-app-8597cbc469-484vp  Pod         ✔ Running     13m  ready:2/2
│     ├──□ demo-app-8597cbc469-bm7h8  Pod         ✔ Running     13m  ready:2/2
│     ├──□ demo-app-8597cbc469-h9zws  Pod         ✔ Running     13m  ready:2/2
│     ├──□ demo-app-8597cbc469-pxcwh  Pod         ✔ Running     13m  ready:2/2
│     └──□ demo-app-8597cbc469-r92bn  Pod         ✔ Running     13m  ready:2/2
└──# revision:1
   └──⧉ demo-app-85d7cbc77b           ReplicaSet  • ScaledDown  28m
```

Now **revision:2** (formerly Green) is marked as "stable,active" and serves all production traffic.

## Rollback Process

If issues are discovered after promotion, you can instantly rollback:

```bash
# Rollback to previous revision
k argo rollouts -n my-demo undo demo-app

# Promote the rollback (if auto-promotion is disabled)
k argo rollouts -n my-demo promote demo-app
```

## Traffic Monitoring Results

Based on the monitoring output from `blue-green-argo.txt`, here's what you'll observe during the entire Blue Green deployment cycle:

### Regular Traffic Monitoring
```bash
# Shows instant switch from v1.0 to v2.0 after promotion
<html><body><h1>Demo App - Version v1.0</h1></body></html>  # Blue environment
<html><body><h1>Demo App - Version v2.0</h1></body></html>  # After promotion
<html><body><h1>Demo App - Version v1.0</h1></body></html>  # After rollback
```

### Preview Traffic Monitoring
```bash
# Consistently shows v2.0 during preview phase, then switches after rollback
<html><body><h1>Demo App - Version v2.0</h1></body></html>  # Green environment
<html><body><h1>Demo App - Version v1.0</h1></body></html>  # After rollback
```

## Key Blue Green Commands

```bash
# Deploy Blue version
k apply -n my-demo -f ../../examples/blue-green/basic/rollout.yaml

# Deploy Green version  
k apply -n my-demo -f ../../examples/blue-green/basic/rollout-update.yaml

# Promote Green to active
k argo rollouts -n my-demo promote demo-app

# Rollback to Blue
k argo rollouts -n my-demo undo demo-app

# Promote rollback (if needed)
k argo rollouts -n my-demo promote demo-app
```

## Summary

Blue Green deployment with Argo Rollouts and Istio provides:

- **Zero-downtime deployments** with instant traffic switching
- **Risk mitigation** through independent testing of new versions
- **Instant rollbacks** if issues are detected
- **Resource efficiency** by maintaining only two environments
- **Production-grade reliability** with Istio's advanced traffic management

**The killer feature: Istio's header-based routing enables comprehensive testing of the Green environment using the exact same production infrastructure and routing path, giving you complete confidence to promote the deployment.**

Key Istio advantages:
- **Same URL testing**: Test Green environment through production ingress gateway
- **Production parity**: Identical network path, security, and routing rules
- **Confidence building**: Validate everything works before switching production traffic
- **Seamless integration**: Single endpoint, multiple environments via intelligent routing

**Without Istio, you're essentially deploying blind - with Istio, you deploy with complete confidence.**