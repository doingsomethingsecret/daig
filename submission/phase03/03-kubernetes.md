# Phase 3 - Kubernetes Reliability

## Objective

In this phase, I deployed the Daig application on a local Kubernetes cluster and tested Kubernetes reliability features.

The main things tested were:

- Kustomize configuration
- Kubernetes deployment
- Pod self-healing
- HPA configuration
- Manual scaling
- Failed image rollout
- Rollback
- Crash-loop diagnosis
- Recovery after failure

---

# 1. Kustomize Validation

I first checked the Kubernetes manifests using Kustomize.

## Render Kubernetes manifests

```bash
kubectl kustomize k8s/base
```

The manifests rendered successfully.

## Count Kubernetes objects

```bash
kubectl kustomize k8s/base | grep '^kind:' | wc -l
```

Output:

```text
34
```

So the Kustomize configuration generated 34 Kubernetes objects.

## Check for `:latest`

```bash
kubectl kustomize k8s/base | grep ':latest'
```

There was no output.

This means there were no `:latest` image references in the rendered application manifests.

The application images were pinned to version `1.0.0`.

The main application images were:

```text
ghcr.io/adeelarshad414/daig/orders:1.0.0
ghcr.io/adeelarshad414/daig/kitchen:1.0.0
ghcr.io/adeelarshad414/daig/dispatch:1.0.0
ghcr.io/adeelarshad414/daig/web:1.0.0
```

### Evidence

`submission/phase03/screenshots/16-kustomize-validation.png`

---

# 2. Local Kubernetes Cluster

I created a local Kind Kubernetes cluster for testing.

The cluster was created with:

```bash
kind create cluster --name daig
```

Then I checked the Kubernetes node:

```bash
kubectl get nodes
```

The control-plane node was in the `Ready` state.

I then applied the Daig Kubernetes configuration:

```bash
kubectl apply -k k8s/base
```

The application images were built locally and loaded into the Kind cluster.

The local application images were tagged with the image names expected by the Kubernetes manifests.

The images used were:

```text
ghcr.io/adeelarshad414/daig/orders:1.0.0
ghcr.io/adeelarshad414/daig/kitchen:1.0.0
ghcr.io/adeelarshad414/daig/dispatch:1.0.0
ghcr.io/adeelarshad414/daig/web:1.0.0
```

---

# 3. Initial Kubernetes Application Failure

After deploying the application, some application Pods did not start correctly.

I checked the Pods using:

```bash
kubectl -n daig get pods
```

The Orders, Kitchen, and Dispatch application Pods were initially failing.

I checked the Orders logs:

```bash
kubectl -n daig logs deployment/orders --tail=80
```

The important error was:

```text
{"level":"fatal","service":"secrets","msg":"No secret store configured while NODE_ENV=production","exit_code":78,"remedy":"Configure OpenBao (BAO_ADDR/BAO_ROLE_ID/BAO_SECRET_ID), or deliver secrets as *_FILE Docker/Swarm secrets. See vault/README.md."}
```

I then checked the Orders Deployment configuration:

```bash
kubectl -n daig get deployment orders -o yaml | grep -A20 -B5 -E 'NODE_ENV|secret|envFrom'
```

The Deployment was using:

```text
PORT=3001
NODE_ENV=production
```

and was reading application secrets from the Kubernetes Secret.

I also checked the ConfigMap:

```bash
kubectl -n daig get configmap daig-config -o yaml
```

The ConfigMap contained:

```text
NODE_ENV: production
```

The problem was that the application was running in production mode but there was no OpenBao secret store configured in the local cluster.

For the local Kubernetes exercise, I temporarily changed the environment to development:

```bash
kubectl -n daig patch configmap daig-config -p '{"data":{"NODE_ENV":"development"}}'
```

I then restarted the affected Deployments:

```bash
kubectl -n daig rollout restart deployment orders kitchen dispatch
```

After the restart, the affected application Pods became healthy.

I verified the Pods with:

```bash
kubectl -n daig get pods
```

The application Pods reached:

```text
1/1 Running
```

This was only a local testing workaround. Production secret management should use the proper secret-store configuration and is addressed later in the DevSecOps phase.

### Evidence

`submission/phase03/screenshots/18-kubernetes-initial-failure.png`

`submission/phase03/screenshots/20-kubernetes-recovered.png`

---

# 4. Kubernetes Pod Self-Healing

After the application was healthy, I checked the Orders Pods:

```bash
kubectl -n daig get pods -l app=orders
```

There were multiple running Orders Pods.

I deliberately deleted one Orders Pod:

```bash
kubectl -n daig delete pod <orders-pod-name>
```

I then watched the Orders Pods:

```bash
kubectl -n daig get pods -l app=orders -w
```

The deleted Pod disappeared and Kubernetes automatically created a replacement Pod.

The replacement eventually reached:

```text
1/1 Running
```

I stopped the watch with:

```text
Ctrl+C
```

This demonstrated Kubernetes self-healing.

The Orders application is managed by a Deployment. The Deployment uses a ReplicaSet to maintain the required number of Pods.

When one Pod is deleted:

```text
Desired Pods = N
Actual Pods  = N-1
```

Kubernetes detects the difference and creates a replacement.

### Evidence

`submission/phase03/screenshots/17-kubernetes-self-healing.png`

`submission/phase03/screenshots/21-kubernetes-pod-self-heal.png`

---

# 5. Check HPA

I checked the Horizontal Pod Autoscalers:

```bash
kubectl -n daig get hpa
```

The output was:

```text
NAME       REFERENCE             TARGETS              MINPODS   MAXPODS   REPLICAS   AGE
dispatch   Deployment/dispatch   cpu: <unknown>/60%   2         12        2          108m
kitchen    Deployment/kitchen    cpu: <unknown>/60%   2         12        2          108m
orders     Deployment/orders     cpu: <unknown>/60%   3         20        3          108m
web        Deployment/web        cpu: <unknown>/70%   2         6         2          108m
```

I then checked the detailed Orders HPA:

```bash
kubectl -n daig describe hpa orders
```

The important configuration was:

```text
Metrics:                                               ( current / target )
  resource cpu on pods  (as a percentage of request):  <unknown> / 60%

Min replicas:                                          3
Max replicas:                                          20
```

The HPA behavior was:

```text
Scale Up:
  Stabilization Window: 0 seconds
  Select Policy: Max
  Type: Percent
  Value: 100
  Period: 15 seconds

Scale Down:
  Stabilization Window: 300 seconds
  Select Policy: Max
  Type: Percent
  Value: 25
  Period: 60 seconds
```

The HPA conditions showed:

```text
AbleToScale:      True
ScalingActive:    False
```

The reason was:

```text
FailedGetResourceMetric
```

The HPA could not get CPU metrics because the local cluster did not have a working:

```text
metrics.k8s.io
```

API.

The reported error was:

```text
failed to get cpu utilization:
unable to fetch metrics from resource metrics API
```

So the HPA itself existed and was configured, but automatic CPU-based scaling was not active in this local cluster.

---

# 6. Manual HPA Scaling Test

I manually changed the Orders Deployment to five replicas:

```bash
kubectl -n daig scale deployment/orders --replicas=5
```

I then checked the Orders Pods:

```bash
kubectl -n daig get pods -l app=orders
```

I also checked the Deployment:

```bash
kubectl -n daig get deployment orders
```

The Orders Deployment reached five replicas.

This demonstrated that the Deployment could be manually scaled even though CPU metrics were unavailable.

When Metrics Server is available, the HPA can automatically change the number of replicas between its configured minimum and maximum values.

For Orders:

```text
Minimum replicas = 3
Maximum replicas = 20
CPU target        = 60%
```

### Evidence

`submission/phase03/screenshots/22-kubernetes-hpa-scale.png`

---

# 7. Deliberately Broken Image Rollout

Next, I tested what happens when a Deployment is given an invalid image.

I changed the Orders image to a nonexistent image:

```bash
kubectl -n daig set image deployment/orders orders=daig-orders:doesnotexist
```

I then checked the rollout:

```bash
kubectl -n daig rollout status deployment/orders --timeout=30s
```

The rollout stalled because Kubernetes could not start the new Pods using the nonexistent image.

I checked the Orders Pods:

```bash
kubectl -n daig get pods -l app=orders
```

I also checked the Deployment:

```bash
kubectl -n daig describe deployment orders
```

The new revision could not become healthy.

---

# 8. Why the Bad Rollout Did Not Cause an Outage

The Orders Deployment uses:

```text
maxSurge: 1
maxUnavailable: 0
```

The important setting is:

```text
maxUnavailable: 0
```

This means Kubernetes should not intentionally remove currently available Pods while the new revision is being rolled out.

The result was:

```text
Old healthy Pods
       |
       v
New bad image
       |
       v
New Pods fail
       |
       v
Old healthy Pods remain available
```

So the bad rollout was detected without immediately taking the existing Orders service offline.

This is an important Kubernetes availability feature.

### Evidence

`submission/phase03/screenshots/23-kubernetes-bad-image-rollout.png`

---

# 9. Rollback the Failed Deployment

After testing the bad image, I rolled the Orders Deployment back:

```bash
kubectl -n daig rollout undo deployment/orders
```

I then waited for the rollback:

```bash
kubectl -n daig rollout status deployment/orders --timeout=60s
```

Finally, I checked the Orders Pods:

```bash
kubectl -n daig get pods -l app=orders
```

The Orders Deployment returned to the working revision.

Kubernetes keeps Deployment revision history, so a failed rollout can be reverted to a previous working revision.

### Evidence

`submission/phase03/screenshots/24-kubernetes-rollout-rollback.png`

---

# 10. Crash-Loop Chaos Test

I then ran the Phase 3 crash-loop test:

```bash
./chaos/day3-crashloop.sh break
```

The script returned:

```text
[chaos] container daig-chaos-orders is unhealthy.
[chaos] They must distinguish: config missing, config wrong, or
[chaos] environment wrong. All three look identical from 'it is down'.
[chaos] Exit code is the fastest discriminator: docker inspect.
```

I checked the Kubernetes Pods:

```bash
kubectl -n daig get pods
```

The Kubernetes application Pods were still running.

This showed that this particular chaos script creates a separate Docker container called:

```text
daig-chaos-orders
```

It does not directly create a Kubernetes CrashLoopBackOff Pod.

Therefore, I used Docker commands to diagnose this specific failure.

---

# 11. Crash-Loop Exit Code

I inspected the Docker container:

```bash
docker inspect daig-chaos-orders --format 'ExitCode={{.State.ExitCode}} Status={{.State.Status}}'
```

The result was:

```text
ExitCode=78 Status=restarting
```

The exit code gave the first important clue.

The container was repeatedly restarting because the application was exiting.

---

# 12. Crash-Loop Logs

I then checked the container logs:

```bash
docker logs daig-chaos-orders
```

The logs repeatedly showed:

```text
{"level":"info","msg":"telemetry started","service":"orders","endpoint":"http://otel-collector:4318"}
```

followed by:

```text
{"level":"fatal","service":"secrets","msg":"No secret store configured while NODE_ENV=production","exit_code":78,"remedy":"Configure OpenBao (BAO_ADDR/BAO_ROLE_ID/BAO_SECRET_ID), or deliver secrets as *_FILE Docker/Swarm secrets. See vault/README.md."}
```

The important error was:

```text
No secret store configured while NODE_ENV=production
```

and:

```text
exit_code=78
```

---

# 13. Crash-Loop Root Cause

The root cause was that the application was running with:

```text
NODE_ENV=production
```

but no production secret store was configured.

The application deliberately stopped with exit code:

```text
78
```

The Docker restart behavior then started the container again, causing the repeated failure.

The failure flow was:

```text
NODE_ENV=production
        |
        v
No secret store configured
        |
        v
Application exits
        |
        v
Exit code 78
        |
        v
Container restarts
        |
        v
Crash-loop
```

The exit code was useful because different problems can all look like "the service is down". Checking the exit code first gave a quick way to identify the failure type.

### Evidence

`submission/phase03/screenshots/25-kubernetes-crashloop-diagnosis.png`

---

# 14. Recover From the Crash-Loop

After diagnosing the problem, I reverted the chaos test:

```bash
./chaos/day3-crashloop.sh fix
```

The command returned:

```text
[chaos] reverted
```

I checked whether the chaos container was still present:

```bash
docker ps -a --filter name=daig-chaos-orders
```

The result showed no container:

```text
CONTAINER ID   IMAGE     COMMAND   CREATED   STATUS    PORTS     NAMES
```

I then checked the Kubernetes application:

```bash
kubectl -n daig get pods
```

The application Pods were healthy.

The important Pods showed:

```text
1/1 Running
```

This confirmed that the deliberate failure had been reverted successfully.

---

# 15. Phase 3 Screenshot Evidence

The screenshots created during Phase 3 are:

```text
submission/phase03/screenshots/16-kustomize-validation.png
submission/phase03/screenshots/17-kubernetes-self-healing.png
submission/phase03/screenshots/18-kubernetes-initial-failure.png
submission/phase03/screenshots/19-kubernetes-stalled-rollout.png
submission/phase03/screenshots/20-kubernetes-recovered.png
submission/phase03/screenshots/21-kubernetes-pod-self-heal.png
submission/phase03/screenshots/22-kubernetes-hpa-scale.png
submission/phase03/screenshots/23-kubernetes-bad-image-rollout.png
submission/phase03/screenshots/24-kubernetes-rollout-rollback.png
submission/phase03/screenshots/25-kubernetes-crashloop-diagnosis.png
```

---

# 16. Phase 3 Results

| Test | Result |
|---|---|
| Kustomize rendering | Passed |
| Kubernetes object count | 34 |
| `:latest` image check | No output |
| Kind cluster | Created successfully |
| Kubernetes deployment | Completed |
| Application recovery | Completed |
| Pod self-healing | Demonstrated |
| HPA configuration | Verified |
| CPU metrics | Unavailable locally |
| Manual scaling to 5 replicas | Passed |
| Bad image rollout | Successfully stalled |
| Existing healthy Pods | Remained available |
| Deployment rollback | Completed |
| Crash-loop diagnosis | Exit code 78 identified |
| Chaos recovery | Completed |
| Final Kubernetes state | Healthy |

---

# 17. What I Learned

## Kubernetes self-healing

When a Pod managed by a Deployment is deleted, Kubernetes automatically creates a replacement to maintain the desired state.

## HPA

HPA can automatically increase or decrease replicas based on metrics.

In my local cluster, CPU metrics were unavailable because the `metrics.k8s.io` API was not available.

## Manual scaling

I manually scaled Orders to five replicas using:

```bash
kubectl -n daig scale deployment/orders --replicas=5
```

This showed how the Deployment replica count can be changed directly.

## Rolling updates

A Deployment can update Pods gradually instead of replacing everything at once.

The Orders Deployment uses:

```text
maxSurge: 1
maxUnavailable: 0
```

This helped keep the existing healthy Pods available when the new image failed.

## Rollback

When a new Deployment revision fails, Kubernetes can return to a previous working revision using:

```bash
kubectl -n daig rollout undo deployment/orders
```

## Failure diagnosis

I learned not to assume the cause from the first symptom.

For the crash-loop test, I first checked the container exit code:

```bash
docker inspect daig-chaos-orders --format 'ExitCode={{.State.ExitCode}} Status={{.State.Status}}'
```

The result was:

```text
ExitCode=78 Status=restarting
```

Then I checked the logs:

```bash
docker logs daig-chaos-orders
```

The logs showed:

```text
No secret store configured while NODE_ENV=production
```

This gave the actual root cause.

---

# 18. Phase 3 Checkpoint Answers

## Why did the deleted Pod come back?

The Pod was managed by a Deployment and ReplicaSet. Kubernetes detected that the actual number of Pods was lower than the desired number and created a replacement.

## Why did the bad image not immediately take the application down?

The Deployment uses:

```text
maxUnavailable: 0
```

So Kubernetes kept the existing available Pods while trying to deploy the new revision.

## What happened to the new Pods?

The new Pods could not start correctly because the image:

```text
daig-orders:doesnotexist
```

does not exist.

The rollout therefore stalled.

## Why was the HPA showing `<unknown>`?

The local Kubernetes cluster did not have a working `metrics.k8s.io` API, so the HPA could not get CPU utilization.

## What are the Orders HPA limits?

```text
Minimum replicas = 3
Maximum replicas = 20
CPU target        = 60%
```

## Why did I manually scale Orders to five replicas?

The assignment required demonstrating scaling behavior. Since CPU metrics were unavailable, manual scaling was used to demonstrate that the Deployment can run five replicas.

## What caused the crash-loop?

The application was running in production mode without a configured production secret store.

The application exited with code `78`, and the container restarted repeatedly.

## Why was the exit code important?

It gave a quick clue about the failure before looking deeper into the logs.

## Was the crash-loop a Kubernetes Pod?

No. The repository's `day3-crashloop.sh` script created a standalone Docker container named:

```text
daig-chaos-orders
```

Therefore Docker inspection and Docker logs were used for this specific test.

## How was the crash-loop fixed?

The chaos test was reverted using:

```bash
./chaos/day3-crashloop.sh fix
```

Then the container and Kubernetes Pods were checked again.

---

# 19. Phase 3 Conclusion

Phase 3 was completed by testing Kubernetes reliability and failure recovery.

I verified:

```text
Kustomize
   ↓
Kubernetes Deployment
   ↓
Self-Healing
   ↓
HPA
   ↓
Manual Scaling
   ↓
Bad Rollout
   ↓
Rollback
   ↓
Crash Diagnosis
   ↓
Recovery
```

The main reliability features demonstrated were Kubernetes self-healing, Deployment rolling updates, HPA configuration, rollback, and failure diagnosis.

The final Kubernetes application was checked after the tests and the application Pods were running successfully.
