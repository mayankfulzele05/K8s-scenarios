# Enterprise DevOps Playbook: Resolving App-vs-Container OOM Discrepancies (Q89)

This production lab models an advanced incident response scenario where an application's internal logging streams report healthy memory boundaries, yet the container is terminated by the host kernel with an `OOMKilled` (Exit Code 137) signature.

## 🏢 Infrastructure Incident Creation
Execute the setup script to provision the multi-node testing infrastructure and trigger the cgroup breach:
```bash
chmod +x setup-lab.sh
./setup-lab.sh
```

---

## 🕵️‍♂️ Enterprise Incident Response & Triage Pipeline

### 1. Evaluate Blast Radius (Impact Assessment)
Query the active states of the microservice architecture inside the targeted namespace:
```bash
kubectl get pods -n production-finance
```
*System State:* Replicas cycle through `Error` and `CrashLoopBackOff` states, causing service interruptions.

### 2. Isolate Kernel Termination Properties
Extract the container termination properties to confirm why the process runner stopped:
```bash
# Capture the pod name dynamically
POD_NAME=$(kubectl get pods -n production-finance -l app=billing-app -o jsonpath='{.items.metadata.name}')

# Inspect infrastructure metadata
kubectl describe pod $POD_NAME -n production-finance
```
Navigate to the container configuration block and inspect the `Last State:` properties:
```text
Last State:     Terminated
  Reason:       OOMKilled        <--- 🧠 Confirms Linux cgroup boundary breach
  Exit Code:    137              <--- ❌ Confirms kernel SIGKILL intervention
```

### 3. Identify the Logging Paradox
Query the historical log buffer stream from the previous dead container instance to review the application context:
```bash
kubectl logs $POD_NAME -n production-finance --previous
```
*Triage Discovery:* The application logs actively report stable and healthy managed internal profiles right up to the crash point:
```text
[MONITOR LOG] Heap status: HEALTHY - Current Heap Usage: 15MB / 30MB Max
```
This isolates the root cause: The application's managed environment (e.g., JVM Heap) is fine, but the container's **Total Resident Set Size (RSS)** breached the container's hard resource ceiling due to off-heap overhead or native memory leaks.

---

## 🛠️ Production-Safe GitOps Resolution

Following enterprise infrastructure-as-code patterns, update the master configuration tracking manifest file directly to scale the container memory allocations to cover off-heap structures:

1. Open the deployment manifest file in your code editor:
   ```bash
   nano lab-manifest-q89.yaml
   ```
2. Adjust the container `resources` specification block to grant appropriate container ceiling parameters:
   ```yaml
   resources:
     requests:
       memory: "64Mi"
     limits:
       memory: "256Mi" # ✅ Scaled from 64Mi to provide structural headroom for off-heap overhead
   ```
3. Apply the updated code manifest declaratively to trigger a managed Kubernetes rolling update:
   ```bash
   kubectl apply -f lab-manifest-q89.yaml
   ```

### 4. Monitor Microservice Recovery
Track the zero-downtime microservice transition to verify infrastructure restabilisation:
```bash
kubectl rollout status deployment/billing-processor -n production-finance
```
*Expected Output:* `deployment "billing-processor" successfully rolled out`.

---

## 🧹 Post-Incident Environment Cleanup
Tear down the lab configuration and cluster components to free up local machine resources:
```bash
kind delete cluster --name troubleshooting-cluster
rm lab-manifest-q89.yaml kind-config.yaml
```
