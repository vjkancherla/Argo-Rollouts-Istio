# Istio with Argo Rollouts and Analysis Templates for Automatic verification and Rollback Guide

Argo Rollouts provides several ways to perform analysis to drive progressive delivery. 
This document describes how to achieve various forms of progressive delivery, varying the point in time analysis is performed, its frequency, and occurrence.

Analysis can be run in the background -- while the canary is progressing through its rollout steps.

The following example gradually increments the canary weight by 25% every 2 minutes until it reaches 100%.

In the background, an AnalysisRun is started based on the AnalysisTemplates - "random-fail" and "always-pass"

The rollout will not progress to the following step until the AnalysisRun is complete. A failure/error of the analysis will cause the rollout's update to abort, and set the canary weight to zero.

## Prerequisites Installation

See [argo_rollouts_istio_guide.md](argo_rollouts_istio_guide.md) for installing all the pre-requisities:
- Istio CRDs
- IstioD
- Istion Ingress Gateway
- Metal LB + extra config
- Argo Rollouts
- Argo Rollouts Plugin


---

## Core Application Configuration [deploy in the order defined below]

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

k apply -n my-demo -f complex_example/services.yaml
```

### Istio Gateway

```bash
# gateway.yaml

k apply -n my-demo -f complex_example/gateway.yaml
```

### Istio VirtualService with Header Routing for Canary Testing

```bash
# virtualsvc.yaml

k apply -n my-demo -f complex_example/virtualsvc.yaml
```

## Analysis Templates for Automated Safety

```bash
# analysis-templates.yaml

# AnalysisTemplate is referenced, starting from the second step, which starts an AnalysisRun after
# the setWeight step. The rollout will not progress to the following step until the
# AnalysisRun is complete. A failure/error of the analysis will cause the rollout's update to
# abort, and set the canary weight to zero.

k apply -n my-demo -f complex_example/analysis-templates.yaml

```

### Application Rollout

```bash
# rollout.yaml

k apply -n my-demo -f complex_example/rollout.yaml
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
Strategy:        Canary
  Step:          8/8
  SetWeight:     100
  ActualWeight:  100
Images:          nginx:1.20 (stable)
Replicas:
  Desired:       5
  Current:       5
  Updated:       5
  Ready:         5
  Available:     5

NAME                                  KIND        STATUS     AGE    INFO
⟳ demo-app                            Rollout     ✔ Healthy  12m
└──# revision:1
   └──⧉ demo-app-7d88bfc4bb           ReplicaSet  ✔ Healthy  9m45s  stable
      ├──□ demo-app-7d88bfc4bb-lr7ns  Pod         ✔ Running  9m35s  ready:2/2
      ├──□ demo-app-7d88bfc4bb-p6qfg  Pod         ✔ Running  9m35s  ready:2/2
      ├──□ demo-app-7d88bfc4bb-rr89d  Pod         ✔ Running  9m35s  ready:2/2
      ├──□ demo-app-7d88bfc4bb-xjrg2  Pod         ✔ Running  9m35s  ready:2/2
      └──□ demo-app-7d88bfc4bb-xppnd  Pod         ✔ Running  9m35s  ready:2/2
```

## Performing a Progressive Update

### Update the App version

```bash
k apply -n my-demo -f example/rollout-update.yaml
```

Run tthese in separate terminals:
```bash 
# Round-robin to all pods. Based on the current Traffic weights, ISIO will send requests to the stable/canary service correspondingly
while true; do curl http://demo-app.127.0.0.1.nip.io:8080 ; sleep 5; done

# Only target the Canary pods (see ISTIO VirtualService's "header" rule match). GREAT FOR VALIDATING/TESTING CANARY PODS
while true; do curl  -H "x-canary-user: true" http://demo-app.127.0.0.1.nip.io:8080 ; sleep 5; done
```

### Check the state of the Canary Rollout

Argo Rollout does the following:

####  Canary-Rollout-Step:
- `setWeight: 25` # Sets the ratio of canary ReplicaSet to 25%
- `pause: {duration: 2m}` # Pauses for 2 minutes 

**Important Notes:**
- The rollout controller will scale the canary to match the current trafficWeight of the current step. For example, if the current weight is 25%, and there are four replicas, then the canary will be scaled to 1, to match the traffic weight.
- The stable ReplicaSet is left scaled to 100% during the update. This has the advantage that if an abort occurs, traffic can be immediately shifted back to the stable ReplicaSet without delay.

**Actions Performed:**
- Created a new replica-set
- Created one new pod
- Updated `demo-app-service-canary` service's selector: 
  ```
  Selector: app=demo-app,rollouts-pod-template-hash=5cf4fb69f8
  ```
- Updated Istio VirtualService - `demo-app-vsvc` - and set the weights to correspond to the rollout

```yaml
  Route:
    Destination:
      Host:  demo-app-service-stable
    Weight:  75
    Destination:
      Host:  demo-app-service-canary
    Weight:  25
  ```

**Check Rollout Status:**
```bash
k argo rollouts -n my-demo get rollout demo-app
```

**Output:**
```
Name:            demo-app
Namespace:       my-demo
Status:          ॥ Paused
Message:         CanaryPauseStep
Strategy:        Canary
  Step:          1/8
  SetWeight:     10
  ActualWeight:  10
Images:          nginx:1.20 (canary, stable)
Replicas:
  Desired:       5
  Current:       6
  Updated:       1
  Ready:         6
  Available:     6

NAME                                                           KIND         STATUS        AGE    INFO
⟳ demo-app                                                     Rollout      ॥ Paused      35m
├──# revision:3
│  └──⧉ demo-app-6b8478797                                     ReplicaSet   ✔ Healthy     27s    canary
│     └──□ demo-app-6b8478797-xbbxs                            Pod          ✔ Running     27s    ready:2/2
├──# revision:2
│  ├──⧉ demo-app-788d74444d                                    ReplicaSet   ✔ Healthy     13m    stable
│  │  ├──□ demo-app-788d74444d-crq2b                           Pod          ✔ Running     13m    ready:2/2
│  │  ├──□ demo-app-788d74444d-tp7sv                           Pod          ✔ Running     11m    ready:2/2
│  │  ├──□ demo-app-788d74444d-vc5t4                           Pod          ✔ Running     9m51s  ready:2/2
│  │  ├──□ demo-app-788d74444d-z9tkx                           Pod          ✔ Running     7m44s  ready:2/2
│  │  └──□ demo-app-788d74444d-hg7qt                           Pod          ✔ Running     6m33s  ready:2/2
│  └──α demo-app-788d74444d-2                                  AnalysisRun  ✔ Successful  11m    ✔ 2
│     ├──⊞ 5fdfcaa4-f172-41c0-9ca3-c07e6cb77e44.random-fail.1  Job          ✔ Successful  11m
│     └──⊞ 5fdfcaa4-f172-41c0-9ca3-c07e6cb77e44.pass.1         Job          ✔ Successful  11m
└──# revision:1
   └──⧉ demo-app-7d88bfc4bb                                    ReplicaSet   • ScaledDown  32m
```

####  Further rollouts with Analysis steps
From Canary Step - 25% onwards, during each incremental step, the canary pods will be analysed using the AnalysisRun templates.
The rollout will not progress to the following step until the AnalysisRun is complete. A failure/error of the analysis will cause the rollout's update to abort, and set the canary weight to zero.

```bash
k argo rollouts -n my-demo get rollout demo-app
```

**Output:**
```
Name:            demo-app
Namespace:       my-demo
Status:          ॥ Paused
Message:         CanaryPauseStep
Strategy:        Canary
  Step:          3/8
  SetWeight:     25
  ActualWeight:  25
Images:          nginx:1.20 (canary, stable)
Replicas:
  Desired:       5
  Current:       7
  Updated:       2
  Ready:         7
  Available:     7

NAME                                                           KIND         STATUS        AGE    INFO
⟳ demo-app                                                     Rollout      ॥ Paused      37m
├──# revision:3
│  ├──⧉ demo-app-6b8478797                                     ReplicaSet   ✔ Healthy     2m33s  canary
│  │  ├──□ demo-app-6b8478797-xbbxs                            Pod          ✔ Running     2m33s  ready:2/2
│  │  └──□ demo-app-6b8478797-fgqmq                            Pod          ✔ Running     22s    ready:2/2
│  └──α demo-app-6b8478797-3                                   AnalysisRun  ◌ Running     22s
│     ├──⊞ b280b4a1-9868-460a-af49-307576ee646f.pass.1         Job          ◌ Running     22s
│     └──⊞ b280b4a1-9868-460a-af49-307576ee646f.random-fail.1  Job          ◌ Running     22s
├──# revision:2
│  ├──⧉ demo-app-788d74444d                                    ReplicaSet   ✔ Healthy     15m    stable
│  │  ├──□ demo-app-788d74444d-crq2b                           Pod          ✔ Running     15m    ready:2/2
│  │  ├──□ demo-app-788d74444d-tp7sv                           Pod          ✔ Running     13m    ready:2/2
│  │  ├──□ demo-app-788d74444d-vc5t4                           Pod          ✔ Running     11m    ready:2/2
│  │  ├──□ demo-app-788d74444d-z9tkx                           Pod          ✔ Running     9m50s  ready:2/2
│  │  └──□ demo-app-788d74444d-hg7qt                           Pod          ✔ Running     8m39s  ready:2/2
│  └──α demo-app-788d74444d-2                                  AnalysisRun  ✔ Successful  13m    ✔ 2
│     ├──⊞ 5fdfcaa4-f172-41c0-9ca3-c07e6cb77e44.random-fail.1  Job          ✔ Successful  13m
│     └──⊞ 5fdfcaa4-f172-41c0-9ca3-c07e6cb77e44.pass.1         Job          ✔ Successful  13m
└──# revision:1
   └──⧉ demo-app-7d88bfc4bb                                    ReplicaSet   • ScaledDown  34m
```


**Output:**
```
Name:            demo-app
Namespace:       my-demo
Status:          ॥ Paused
Message:         CanaryPauseStep
Strategy:        Canary
  Step:          7/8
  SetWeight:     75
  ActualWeight:  75
Images:          nginx:1.20 (canary, stable)
Replicas:
  Desired:       5
  Current:       9
  Updated:       4
  Ready:         9
  Available:     9

NAME                                                           KIND         STATUS        AGE    INFO
⟳ demo-app                                                     Rollout      ॥ Paused      40m
├──# revision:3
│  ├──⧉ demo-app-6b8478797                                     ReplicaSet   ✔ Healthy     5m41s  canary
│  │  ├──□ demo-app-6b8478797-xbbxs                            Pod          ✔ Running     5m41s  ready:2/2
│  │  ├──□ demo-app-6b8478797-fgqmq                            Pod          ✔ Running     3m30s  ready:2/2
│  │  ├──□ demo-app-6b8478797-pn27z                            Pod          ✔ Running     2m20s  ready:2/2
│  │  └──□ demo-app-6b8478797-lmqvk                            Pod          ✔ Running     13s    ready:2/2
│  └──α demo-app-6b8478797-3                                   AnalysisRun  ◌ Running     3m30s
│     ├──⊞ b280b4a1-9868-460a-af49-307576ee646f.pass.1         Job          ◌ Running     3m30s
│     └──⊞ b280b4a1-9868-460a-af49-307576ee646f.random-fail.1  Job          ◌ Running     3m30s
├──# revision:2
│  ├──⧉ demo-app-788d74444d                                    ReplicaSet   ✔ Healthy     18m    stable
│  │  ├──□ demo-app-788d74444d-crq2b                           Pod          ✔ Running     18m    ready:2/2
│  │  ├──□ demo-app-788d74444d-tp7sv                           Pod          ✔ Running     16m    ready:2/2
│  │  ├──□ demo-app-788d74444d-vc5t4                           Pod          ✔ Running     15m    ready:2/2
│  │  ├──□ demo-app-788d74444d-z9tkx                           Pod          ✔ Running     12m    ready:2/2
│  │  └──□ demo-app-788d74444d-hg7qt                           Pod          ✔ Running     11m    ready:2/2
│  └──α demo-app-788d74444d-2                                  AnalysisRun  ✔ Successful  16m    ✔ 2
│     ├──⊞ 5fdfcaa4-f172-41c0-9ca3-c07e6cb77e44.random-fail.1  Job          ✔ Successful  16m
│     └──⊞ 5fdfcaa4-f172-41c0-9ca3-c07e6cb77e44.pass.1         Job          ✔ Successful  16m
└──# revision:1
   └──⧉ demo-app-7d88bfc4bb                                    ReplicaSet   • ScaledDown  37m
``` 

**Output:**
```
Name:            demo-app
Namespace:       my-demo
Status:          ✔ Healthy
Strategy:        Canary
  Step:          8/8
  SetWeight:     100
  ActualWeight:  100
Images:          nginx:1.20 (stable)
Replicas:
  Desired:       5
  Current:       5
  Updated:       5
  Ready:         5
  Available:     5

NAME                                                           KIND         STATUS        AGE    INFO
⟳ demo-app                                                     Rollout      ✔ Healthy     42m
├──# revision:3
│  ├──⧉ demo-app-6b8478797                                     ReplicaSet   ✔ Healthy     7m36s  stable
│  │  ├──□ demo-app-6b8478797-xbbxs                            Pod          ✔ Running     7m36s  ready:2/2
│  │  ├──□ demo-app-6b8478797-fgqmq                            Pod          ✔ Running     5m25s  ready:2/2
│  │  ├──□ demo-app-6b8478797-pn27z                            Pod          ✔ Running     4m15s  ready:2/2
│  │  ├──□ demo-app-6b8478797-lmqvk                            Pod          ✔ Running     2m8s   ready:2/2
│  │  └──□ demo-app-6b8478797-7jwt5                            Pod          ✔ Running     60s    ready:2/2
│  └──α demo-app-6b8478797-3                                   AnalysisRun  ✔ Successful  5m25s  ✔ 2
│     ├──⊞ b280b4a1-9868-460a-af49-307576ee646f.pass.1         Job          ✔ Successful  5m25s
│     └──⊞ b280b4a1-9868-460a-af49-307576ee646f.random-fail.1  Job          ✔ Successful  5m25s
├──# revision:2
│  ├──⧉ demo-app-788d74444d                                    ReplicaSet   • ScaledDown  20m
│  └──α demo-app-788d74444d-2                                  AnalysisRun  ✔ Successful  18m    ✔ 2
│     ├──⊞ 5fdfcaa4-f172-41c0-9ca3-c07e6cb77e44.random-fail.1  Job          ✔ Successful  18m
│     └──⊞ 5fdfcaa4-f172-41c0-9ca3-c07e6cb77e44.pass.1         Job          ✔ Successful  18m
└──# revision:1
   └──⧉ demo-app-7d88bfc4bb                                    ReplicaSet   • ScaledDown  39m
```
