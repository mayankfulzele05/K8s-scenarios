# Enterprise DevOps Playbook: Triage Guide for Liveness Probe Failures & Restart Loops

This production lab models an infrastructure deadlock scenario where an application's startup requirements conflict with aggressive cluster validation thresholds, causing an infinite container reboot loop.

## 🏢 Infrastructure Incident Creation
Execute the setup script to provision the multi-node testing infrastructure and initiate the restart loop:
```bash
chmod +x setup-lab.sh
./setup-lab.sh
```

---

## 🕵️‍♂️ Enterprise Incident Triage Pipeline

### 1. Identify the Core Symptom Checklist
Query the runtime states inside the namespace:
```bash
kubectl get pods -n production-core
```
*System State:* The Pod shows a status of `Running`, but the `RESTARTS` count increases every minute, preventing service initialization.

### 2. Isolate the Termination Authority
Query the cluster metadata profile to see why the container engine is restarting the workload:
```bash
# Capture the pod name dynamically
POD_NAME=$(kubectl get pods -n production-core -l app=warehouse-service -o jsonpath='{.items.metadata.name}')

# Inspect node events
kubectl describe pod $POD_NAME -n production-core
```
Navigate straight to the **Events:** block at the bottom of the system readout:
```text
Events:
  Type     Reason     Age   From     Message
  ----     ------     ---   ----     -------
  Warning  Unhealthy  10s   kubelet  Liveness probe failed: cat: can't open '/tmp/app-ready'
  Normal   Killing    10s   kubelet  Container api-engine failed liveness probe, will be restarted
```

### 3. Review Historical Log Buffer Traces
Query the log buffer from the previous dead container instance to confirm manual compatibility:
```bash
kubectl logs \$POD_NAME -n production-core --previous
```
*Triage Discovery:* The app prints its bootstrap logs perfectly but is terminated exactly 6 seconds into execution. Cross-referencing this with the `livenessProbe` specs (`initialDelaySeconds: 2` + `failureThreshold: 2`) confirms that the cluster is killing the app before it completes its 12-second warm-up sequence.

---

## 🛠️ Production-Safe GitOps Resolution

Following enterprise infrastructure-as-code hardening standards, implement a protective **`startupProbe`** configuration layer to shield the microservice during its initial boot window:

1. Open the deployment manifest file in your code editor:
   ```bash
   nano lab-manifest-liveness.yaml
   ```
2. Inject a `startupProbe` block directly above your existing `livenessProbe` block:
   ```yaml
   startupProbe:
     exec:
       command:
       - cat
       - /tmp/app-ready
     initialDelaySeconds: 5
     periodSeconds: 5
     failureThreshold: 4     # Grants up to 20 total seconds for initial warm-up
   livenessProbe:
     exec:
       command:
       - cat
       - /tmp/app-ready
     periodSeconds: 10       # Standard runtime check frequency
   ```
3. Apply the updated configuration manifest file declaratively to the cluster control plane to reload the parameters:
   ```bash
   kubectl apply -f lab-manifest-liveness.yaml
   ```

### 4. Monitor Infrastructure Recovery
Track the deployment update progress until complete:
```bash
kubectl rollout status deployment/data-warehouse-api -n production-core
```
*Expected Output:* `deployment "data-warehouse-api" successfully rolled out`.

---

## 🧹 Post-Incident Environment Cleanup
Tear down the lab configuration and cluster components to free up local machine resources:
```bash
kind delete cluster --name troubleshooting-cluster
rm lab-manifest-liveness.yaml kind-config.yaml
```

<img width="1864" height="1031" alt="image" src="https://github.com/user-attachments/assets/38ac46a1-7e68-44d1-9cf2-f73f25710c4c" />
