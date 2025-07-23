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