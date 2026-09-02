# DevOps Practice Lab: Resolving K8s Pod Scheduling Blocks (Q81)

This hands-on environment tests your ability to triage infrastructure bottleneck tickets where a Pod fails to schedule and gets stuck permanently in the `Pending` state.

## 🚀 Lab Initialization
Initialize your local testing cluster and inject the broken production application manifest:
```bash
chmod +x setup-lab.sh
./setup-lab.sh
```

---

## 🔬 Triage & Diagnosis Workflow

### 1. Identify the Stuck Resource
Query the live deployment status inside the target tracking namespace:
```bash
kubectl get pods -n devops-lab-q81
```

### 2. Inspect Control Plane Scheduling Events
Since the pod hasn't started, container application logs do not exist yet. Pull the engine diagnostics from the API server:
```bash
kubectl describe pod q81-pending-pod -n devops-lab-q81
```
Navigate to the **Events:** block at the bottom of the output to extract the system failure root cause:
```text
Events:
  Warning  FailedScheduling  10s   default-scheduler  0/3 nodes are available: 3 Insufficient cpu.
```

---

## 🛠️ Declarative Resolution Execution

Instead of running temporary inline command-line overrides, follow standard declarative Infrastructure-as-Code workflows by editing the manifest source file:

1. Open the manifest file in a Linux text editor:
   ```bash
   nano lab-manifest-q81.yaml
   ```
2. Locate the container resource requirements block:
   ```yaml
   resources:
     requests:
       cpu: "500"  # ❌ CHANGE THIS VALUE
       memory: "2Gi"
   ```
3. Update the `cpu` allocation line to match a realistic footprint:
   ```yaml
   cpu: "100m"     #  FIXED (100 millicores / 0.1 CPU)
   ```
4. Force Kubernetes to tear down the stuck Pod and replace it using the corrected code:
   ```bash
   kubectl replace -f lab-manifest-q81.yaml --force
   ```

### Verify Infrastructure Restabilisation
Track the real-time scheduling transition:
```bash
kubectl get pods -n devops-lab-q81 -w
```
The state will shift rapidly from `Pending` ➡️ `ContainerCreating` ➡️ `Running`.

---

## 🧹 Infrastructure Cleanup
Tear down the lab configuration and cluster components to free up local machine resources:
```bash
kind delete cluster --name troubleshooting-cluster
rm lab-manifest-q81.yaml kind-config.yaml
```
