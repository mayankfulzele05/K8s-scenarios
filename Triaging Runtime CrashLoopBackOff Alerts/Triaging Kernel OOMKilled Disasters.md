# Enterprise DevOps Playbook: Triaging Kernel OOMKilled Disasters (Exit Code 137)

This production lab environment models a critical service failure caused by a memory limit mismatch, where the node runtime forces an operational termination on an analytics component.

## 🏢 Infrastructure Incident Creation
Execute the setup script to provision the multi-node testing infrastructure and trigger the resource exhaustion:
```bash
chmod +x setup-lab.sh
./setup-lab.sh
```

---

## 🕵️‍♂️ Enterprise Incident Response & Triage Pipeline

### 1. Evaluate Blast Radius (Impact Assessment)
Query the runtime states inside the targeted namespace:
```bash
kubectl get pods -n production-analytics
```
*System State:* Replicas display unstable uptimes, continuously resetting and moving back into the `CrashLoopBackOff` control state.

### 2. Isolate Node Kernel Termination Properties
Query the system metadata profile for an active broken node instance to isolate why the container engine closed the process:
```bash
# Capture an active pod name dynamically
POD_NAME=$(kubectl get pods -n production-analytics -l app=analytics-engine -o jsonpath='{.items[0].metadata.name}')

# Inspect node execution states
kubectl describe pod $POD_NAME -n production-analytics
```
Navigate to the container configuration tracking block and inspect the `Last State:` properties:
```text
State:          Waiting
  Reason:       CrashLoopBackOff
Last State:     Terminated
  Reason:       OOMKilled        <--- 🧠 Node Engine explicitly caught resource limit breach
  Exit Code:    137              <--- ❌ Confirms kernel SIGKILL termination (128 + Signal 9)
```

### 3. Review Historical Log Buffer Traces
Query the log buffer from the previous active container execution block before the runtime termination occurred:
```bash
kubectl logs \$POD_NAME -n production-analytics --previous
```
*Triage Discovery:* The logs contain standard memory consumption steps but freeze instantly without generating internal error reports. This confirms that an external process manager forced the shutdown.

---

## 🛠️ Production-Safe GitOps Resolution

Following cloud-native declarative configuration lifecycle patterns, update the infrastructure-as-code template file to allocate appropriate memory resource limits:

1. Open the deployment manifest file in your code editor:
   ```bash
   nano lab-manifest-q82-oom.yaml
   ```
2. Locate the container `resources` specification block and scale the memory maximum parameters to cover the workload needs:
   ```yaml
   resources:
     requests:
       memory: "64Mi"
     limits:
       memory: "512Mi" # ✅ Scaled from 50Mi to ensure application stability
   ```
3. Apply the updated code manifest declaratively to trigger a managed Kubernetes rolling update:
   ```bash
   kubectl apply -f lab-manifest-q82-oom.yaml
   ```

### 4. Verify Microservice Recovery
Monitor the rollout progression to confirm infrastructure stabilization:
```bash
kubectl rollout status deployment/report-generator -n production-analytics
```
*Expected Output:* `deployment "report-generator" successfully rolled out`.

---

## 🧹 Post-Incident Environment Cleanup
Tear down the lab configuration and cluster components to free up local machine resources:
```bash
kind delete cluster --name troubleshooting-cluster
rm lab-manifest-q82-oom.yaml kind-config.yaml
```
