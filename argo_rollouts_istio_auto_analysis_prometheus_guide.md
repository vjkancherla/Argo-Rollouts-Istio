# Istio with Argo Rollouts and Analysis Templates for Automatic Verification and Rollback Guide

Argo Rollouts provides several ways to perform analysis to drive progressive delivery. This document describes how to achieve
progressive delivery, varying the point in time analysis is performed, its frequency, and occurrence.

Analysis can be run in the background -- while the canary is progressing through its rollout steps.

The following example gradually increments the canary weight by 25% every 2 minutes until it reaches 100%.

In the background, an AnalysisRun is started based on the AnalysisTemplates - "random-fail" and "always-pass"

The rollout will not progress to the following step until the AnalysisRun is complete. A failure/error of the analysis will cause the rollout's update to abort, and set the canary weight to zero.

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
k apply -n my-demo -f analysis_prometheus_example/services.yaml
```

### Istio Gateway

```bash
# gateway.yaml
k apply -n my-demo -f analysis_prometheus_example/gateway.yaml
```

### Istio VirtualService with Header Routing for Canary Testing

```bash
# virtualsvc.yaml
k apply -n my-demo -f analysis_prometheus_example/virtualsvc.yaml
```

### Analysis Templates for Automated Safety

```bash
# analysis-templates.yaml

# AnalysisTemplate is referenced, starting from the second step, which starts an AnalysisRun after
# the setWeight step. The rollout will not progress to the following step until the
# AnalysisRun is complete. A failure/error of the analysis will cause the rollout's update to
# abort, and set the canary weight to zero.

# PRODUCTION TEMPLATES (for real deployments):
# - success-rate: Istio-based success rate monitoring via Prometheus
# - error-rate: General error rate monitoring (5xx errors)
# - server-error-rate: Strict server error monitoring (5xx only)
# - comprehensive-error-rate: Complete error monitoring (4xx + 5xx)
# - client-error-rate: Client error monitoring (4xx only)

k apply -n my-demo -f analysis_prometheus_example/analysis-templates.yaml
```

### Application Rollout

```bash
# rollout.yaml
k apply -n my-demo -f analysis_prometheus_example/rollout.yaml
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
  Step:          11/11
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
⟳ demo-app                            Rollout     ✔ Healthy  2m31s
└──# revision:1
   └──⧉ demo-app-788d74444d           ReplicaSet  ✔ Healthy  2m31s  stable
      ├──□ demo-app-788d74444d-v9mjm  Pod         ✔ Running  2m31s  ready:2/2
      ├──□ demo-app-788d74444d-vshgw  Pod         ✔ Running  2m31s  ready:2/2
      ├──□ demo-app-788d74444d-x9j55  Pod         ✔ Running  2m31s  ready:2/2
      ├──□ demo-app-788d74444d-hqpm6  Pod         ✔ Running  2m30s  ready:2/2
      └──□ demo-app-788d74444d-jgczs  Pod         ✔ Running  2m30s  ready:2/2
```

## Performing a Progressive Update

### Update the App version

```bash
k apply -n my-demo -f example/rollout-update.yaml
```


### Continuously Monitor and verify the application in separate terminals
Run these in separate terminals:
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

# Round-robin to all pods. Based on the current Traffic weights, ISTIO will send requests to the stable/canary service correspondingly
while true; do curl http://demo-app.127.0.0.1.nip.io:8080 ; sleep 5; done

# Only target the Canary pods (see ISTIO VirtualService's "header" rule match). GREAT FOR VALIDATING/TESTING CANARY PODS
while true; do curl -H "x-canary-user: true" http://demo-app.127.0.0.1.nip.io:8080 ; sleep 5; done
```

### Check the state of the Canary Rollout

Argo Rollout performs a canary deployment with the following configuration:
- 11 total steps with gradual weight increases: 10% → 25% → 50% → 75% → 100%
- Analysis runs at multiple steps to verify canary health
- Automatic pause steps between weight changes
- Rollback capability if analysis fails

#### Initial Canary Deployment (Step 0 → Step 1)

After applying the rollout update, the system immediately begins the canary process:

**Step 0 (14:32:20):** Rollout becomes "Progressing" 
- Creates new ReplicaSet `demo-app-6b8478797` 
- Deploys first canary pod
- Status shows "more replicas need to be updated"
- Current: 6 pods (5 stable + 1 canary), Updated: 1, SetWeight: 10%, ActualWeight: 0%

**Step 1 (14:32:31):** First pause at 10% weight
- Canary pod becomes ready (2/2 containers)
- Status changes to "Paused" with message "CanaryPauseStep"
- ActualWeight reaches 10%
- Istio VirtualService updated with traffic split: 90% stable, 10% canary

```
Name:            demo-app
Namespace:       my-demo
Status:          ॥ Paused
Message:         CanaryPauseStep
Strategy:        Canary
  Step:          1/11
  SetWeight:     10
  ActualWeight:  10
Images:          nginx:1.20 (canary, stable)
Replicas:
  Desired:       5
  Current:       6
  Updated:       1
  Ready:         6
  Available:     6

NAME                                  KIND        STATUS     AGE    INFO
⟳ demo-app                            Rollout     ॥ Paused   5m58s  
├──# revision:2                                                     
│  └──⧉ demo-app-6b8478797            ReplicaSet  ✔ Healthy  19s    canary
│     └──□ demo-app-6b8478797-b2pwx   Pod         ✔ Running  19s    ready:2/2
└──# revision:1                                                     
   └──⧉ demo-app-788d74444d           ReplicaSet  ✔ Healthy  5m58s  stable
      ├──□ demo-app-788d74444d-v9mjm  Pod         ✔ Running  5m58s  ready:2/2
      ├──□ demo-app-788d74444d-vshgw  Pod         ✔ Running  5m58s  ready:2/2
      ├──□ demo-app-788d74444d-x9j55  Pod         ✔ Running  5m58s  ready:2/2
      ├──□ demo-app-788d74444d-hqpm6  Pod         ✔ Running  5m57s  ready:2/2
      └──□ demo-app-788d74444d-jgczs  Pod         ✔ Running  5m57s  ready:2/2
```

#### Step 2-3: Progression to 25% Weight with Analysis

**Step 2 (14:34:24):** Analysis begins
- Second canary pod starts deploying
- First AnalysisRun `demo-app-6b8478797-2` begins
- Status: "Progressing" - "more replicas need to be updated"
- Current: 7 pods, Updated: 2, SetWeight: 25%, ActualWeight: 10%

**Step 3 (14:34:34):** Pause at 25% with active analysis
- Both canary pods ready
- AnalysisRun continues running background checks
- Traffic weight successfully updated to 25%

```
Name:            demo-app
Namespace:       my-demo
Status:          ॥ Paused
Message:         CanaryPauseStep
Strategy:        Canary
  Step:          3/11
  SetWeight:     25
  ActualWeight:  25
Images:          nginx:1.20 (canary, stable)
Replicas:
  Desired:       5
  Current:       7
  Updated:       2
  Ready:         7
  Available:     7

NAME                                  KIND         STATUS     AGE    INFO
⟳ demo-app                            Rollout      ॥ Paused   8m1s   
├──# revision:2                                                      
│  ├──⧉ demo-app-6b8478797            ReplicaSet   ✔ Healthy  2m22s  canary
│  │  ├──□ demo-app-6b8478797-b2pwx   Pod          ✔ Running  2m22s  ready:2/2
│  │  └──□ demo-app-6b8478797-n2dpn   Pod          ✔ Running  13s    ready:2/2
│  └──α demo-app-6b8478797-2          AnalysisRun  ◌ Running  13s    ✔ 3
└──# revision:1                                                      
   └──⧉ demo-app-788d74444d           ReplicaSet   ✔ Healthy  8m1s   stable
      ├──□ demo-app-788d74444d-v9mjm  Pod          ✔ Running  8m1s   ready:2/2
      ├──□ demo-app-788d74444d-vshgw  Pod          ✔ Running  8m1s   ready:2/2
      ├──□ demo-app-788d74444d-x9j55  Pod          ✔ Running  8m1s   ready:2/2
      ├──□ demo-app-788d74444d-hqpm6  Pod          ✔ Running  8m     ready:2/2
      └──□ demo-app-788d74444d-jgczs  Pod          ✔ Running  8m     ready:2/2
```

#### Extended Analysis Phase (Steps 4-7)

The rollout spends significant time in analysis phases, with multiple AnalysisRuns executing:

**Step 4-7 (14:35:36 - 14:38:21):** Comprehensive analysis period
- Analysis runs for ~3 minutes with multiple checks
- AnalysisRun shows progressive success indicators: ✔ 2, ✔ 3, ✔ 4, etc.
- Additional AnalysisRuns start: `demo-app-6b8478797-2-4`, `demo-app-6b8478797-2-7`
- System remains at 25% weight during entire analysis phase

```
Name:            demo-app
Namespace:       my-demo
Status:          ◌ Progressing
Message:         more replicas need to be updated
Strategy:        Canary
  Step:          7/11
  SetWeight:     50
  ActualWeight:  50
Images:          nginx:1.20 (canary, stable)

NAME                                  KIND         STATUS         AGE    INFO
⟳ demo-app                            Rollout      ◌ Progressing  9m24s  
├──# revision:2                                                          
│  ├──⧉ demo-app-6b8478797            ReplicaSet   ✔ Healthy      3m45s  canary
│  ├──α demo-app-6b8478797-2          AnalysisRun  ◌ Running      4m31s  ✔ 9
│  ├──α demo-app-6b8478797-2-4        AnalysisRun  ✔ Successful   3m33s  ✔ 5
│  └──α demo-app-6b8478797-2-7        AnalysisRun  ◌ Running      32s    ✔ 6
```

#### Step 8-9: Progression to 50% and 75%

**Step 8 (14:36:17):** Scale to 50% weight
- Third canary pod `demo-app-6b8478797-7xz72` deployed
- Current: 8 pods total, Updated: 3
- Multiple analysis runs complete successfully

**Step 9 (14:42:59):** Progression to 75% weight  
- Fourth canary pod `demo-app-6b8478797-8l5lx` deployed
- All previous AnalysisRuns show ✔ Successful
- Current: 9 pods total, Updated: 4

```
Name:            demo-app
Namespace:       my-demo
Status:          ॥ Paused
Message:         CanaryPauseStep
Strategy:        Canary
  Step:          9/11
  SetWeight:     75
  ActualWeight:  75
Images:          nginx:1.20 (canary, stable)
Replicas:
  Desired:       5
  Current:       9
  Updated:       4
  Ready:         9
  Available:     9

NAME                                  KIND         STATUS        AGE    INFO
⟳ demo-app                            Rollout      ॥ Paused      16m    
├──# revision:2                                                         
│  ├──⧉ demo-app-6b8478797            ReplicaSet   ✔ Healthy     10m    canary
│  │  ├──□ demo-app-6b8478797-b2pwx   Pod          ✔ Running     10m    ready:2/2
│  │  ├──□ demo-app-6b8478797-n2dpn   Pod          ✔ Running     8m38s  ready:2/2
│  │  ├──□ demo-app-6b8478797-7xz72   Pod          ✔ Running     6m50s  ready:2/2
│  │  └──□ demo-app-6b8478797-8l5lx   Pod          ✔ Running     9s     ready:2/2
│  ├──α demo-app-6b8478797-2          AnalysisRun  ✔ Successful  8m38s  ✔ 15
│  ├──α demo-app-6b8478797-2-4        AnalysisRun  ✔ Successful  7m30s  ✔ 5
│  └──α demo-app-6b8478797-2-7        AnalysisRun  ✔ Successful  4m39s  ✔ 15
```

#### Step 10-11: Final Analysis and Completion

**Step 10 (14:44:01):** Final analysis phase
- New AnalysisRun `demo-app-6b8478797-2-10` starts for final verification
- Analysis runs for ~5 minutes with continuous monitoring
- All previous analyses show ✔ Successful

**Step 11 (14:48:30):** Full deployment completion
- Fifth and final canary pod `demo-app-6b8478797-c7rnf` deployed  
- Status: "Progressing" - "waiting for all steps to complete"
- SetWeight: 100%, ActualWeight: 100%
- Current: 10 pods total (5 canary + 5 stable), Updated: 5

#### Final State: Healthy Deployment

**Completion (14:48:40):** Rollout reaches Healthy state
- Status changes to ✔ Healthy
- All 5 canary pods ready and stable
- Canary ReplicaSet becomes the new "stable" 
- Old stable ReplicaSet begins scale-down process

```
Name:            demo-app
Namespace:       my-demo
Status:          ✔ Healthy
Strategy:        Canary
  Step:          11/11
  SetWeight:     100
  ActualWeight:  100
Images:          nginx:1.20 (stable)
Replicas:
  Desired:       5
  Current:       10
  Updated:       5
  Ready:         10
  Available:     10

NAME                                  KIND         STATUS        AGE    INFO
⟳ demo-app                            Rollout      ✔ Healthy     22m    
├──# revision:2                                                         
│  ├──⧉ demo-app-6b8478797            ReplicaSet   ✔ Healthy     16m    stable
│  │  ├──□ demo-app-6b8478797-b2pwx   Pod          ✔ Running     16m    ready:2/2
│  │  ├──□ demo-app-6b8478797-n2dpn   Pod          ✔ Running     14m    ready:2/2
│  │  ├──□ demo-app-6b8478797-7xz72   Pod          ✔ Running     12m    ready:2/2
│  │  ├──□ demo-app-6b8478797-8l5lx   Pod          ✔ Running     5m51s  ready:2/2
│  │  └──□ demo-app-6b8478797-c7rnf   Pod          ✔ Running     14s    ready:2/2
│  ├──α demo-app-6b8478797-2          AnalysisRun  ✔ Successful  14m    ✔ 15
│  ├──α demo-app-6b8478797-2-4        AnalysisRun  ✔ Successful  13m    ✔ 5
│  ├──α demo-app-6b8478797-2-7        AnalysisRun  ✔ Successful  10m    ✔ 15
│  └──α demo-app-6b8478797-2-10       AnalysisRun  ✔ Successful  4m44s  ✔ 25
└──# revision:1                                                         
   └──⧉ demo-app-788d74444d           ReplicaSet   ✔ Healthy     22m    delay:24s
```

#### Cleanup Phase: Old ReplicaSet Termination

**Final Cleanup (14:49:12):** Old pods terminated
- Previous stable ReplicaSet scaled down to 0
- All old pods show "Terminating" status, then removed
- System reaches final desired state: 5 pods total, all from new version

```
Name:            demo-app
Namespace:       my-demo
Status:          ✔ Healthy
Strategy:        Canary
  Step:          11/11
  SetWeight:     100
  ActualWeight:  100
Images:          nginx:1.20 (stable)
Replicas:
  Desired:       5
  Current:       5
  Updated:       5
  Ready:         5
  Available:     5

NAME                                 KIND         STATUS        AGE    INFO
⟳ demo-app                           Rollout      ✔ Healthy     22m    
├──# revision:2                                                        
│  ├──⧉ demo-app-6b8478797           ReplicaSet   ✔ Healthy     17m    stable
│  ├──α demo-app-6b8478797-2         AnalysisRun  ✔ Successful  15m    ✔ 15
│  ├──α demo-app-6b8478797-2-4       AnalysisRun  ✔ Successful  13m    ✔ 5
│  ├──α demo-app-6b8478797-2-7       AnalysisRun  ✔ Successful  11m    ✔ 15
│  └──α demo-app-6b8478797-2-10      AnalysisRun  ✔ Successful  5m25s  ✔ 25
└──# revision:1                                                        
   └──⧉ demo-app-788d74444d          ReplicaSet   • ScaledDown  22m    
```

### Key Observations from the Rollout

**Total Duration:** ~17 minutes (14:32:20 - 14:49:12)

**Analysis Success:** All AnalysisRuns completed successfully:
- Primary analysis: ✔ 15 successful checks
- Secondary analyses: ✔ 5, ✔ 15, ✔ 25 successful checks respectively

**Traffic Management:** Istio VirtualService automatically updated weights:
- 10% → 25% → 50% → 75% → 100% canary traffic
- Seamless traffic shifting with no downtime

**Safety Mechanisms:** 
- Multiple analysis templates validated canary health
- Pause steps allowed monitoring between changes  
- Automatic rollback capability (not triggered due to successful analysis)

**Resource Scaling:**
- Gradual replica increase: 1 → 2 → 3 → 4 → 5 canary pods
- Stable replicas maintained at 5 throughout rollout
- Clean termination of old pods after successful completion