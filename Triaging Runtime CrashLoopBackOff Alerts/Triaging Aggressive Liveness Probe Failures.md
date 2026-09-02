# Enterprise DevOps Playbook: Triaging Aggressive Liveness Probe Failures (Q82)

This production lab models an infrastructure deadlock scenario where an application's required warm-up timeframe conflicts with aggressive cluster health thresholds, causing an endless automated termination loop.

## 🏢 Infrastructure Incident Creation
Execute the setup script to provision the multi-node testing infrastructure and trigger the health check failures:
```bash
chmod +x setup-lab.sh
./setup-lab.sh
```

---

## 🕵️‍♂️ Enterprise Incident Response & Triage Pipeline

### 1. Evaluate Blast Radius (Impact Assessment)
Query the running pods in the target business unit namespace:
```bash
kubectl get pods -n production-checkout
```
*System State:* Replicas flip repeatedly through `Running` ➡️ `Error` ➡️ `CrashLoopBackOff`, flagging an active critical availability impairment.

### 2. Isolate Control Plane Infrastructure Logs
Since the container reports cyclic restarts, pull the node engine event trace metrics from the local Kubelet agent:
```bash
# Capture an active pod name dynamically
POD_NAME=$(kubectl get pods -n production-checkout -l app=payment-processor -o jsonpath='{.items.metadata.name}')

# Inspect node events
kubectl describe pod $POD_NAME -n production-checkout
```
Navigate straight to the **Events:** block at the bottom of the system readout:
```text
Events:
  Type     Reason     Age                From               Message
  ----     ------     ---                ----               -------
  Warning  Unhealthy  25s (x2 over 28s)  kubelet            Liveness probe failed: cat: can't open '/tmp/healthy'
  Normal   Killing    25s                kubelet            Container gateway-app failed liveness probe, will be restarted
```

### 3. Cross-Reference Application Initialization Timings
Query the historical output data streams from the previous dead container shell before it was terminated:
```bash
kubectl logs \$POD_NAME -n production-checkout --previous
```
*Triage Discovery:* The app prints its boot script messages normally but terminates exactly 8–10 seconds into execution. Cross-referencing this with the `livenessProbe` specs (`initialDelaySeconds: 2` + `failureThreshold: 2`) proves that the cluster is killing the app before it completes its 15-second initialization.

---

## 🛠️ Production-Safe GitOps Resolution

Following cloud-native architecture patterns, do not remove safety probes entirely. Instead, implement a protective **`startupProbe`** configuration layer to shield the microservice during its initial load window:

1. Open the deployment manifest file in your code editor:
   ```bash
   nano lab-manifest-q82-probe.yaml
   ```
2. Inject a `startupProbe` block above the existing `livenessProbe` to handle the startup window:
   ```yaml
   startupProbe:
     exec:
       command:
       - cat
       - /tmp/healthy
     initialDelaySeconds: 5
     periodSeconds: 5
     failureThreshold: 5     # Grants up to 25 total seconds for initial boot
   livenessProbe:
     exec:
       command:
       - cat
       - /tmp/healthy
     periodSeconds: 10
   ```
3. Apply the updated configuration declaratively to trigger a managed rolling update update:
   ```bash
   kubectl apply -f lab-manifest-q82-probe.yaml
   ```

### 4. Verify Microservice Recovery
Monitor the rollout status thread to confirm stable infrastructure convergence:
```bash
kubectl rollout status deployment/payment-gateway -n production-checkout
```
*Expected Output:* `deployment "payment-gateway" successfully rolled out`.

---

## 🧹 Post-Incident Environment Cleanup
Tear down the lab configuration and cluster components to free up local machine resources:
```bash
kind delete cluster --name troubleshooting-cluster
rm lab-manifest-q82-probe.yaml kind-config.yaml
```
