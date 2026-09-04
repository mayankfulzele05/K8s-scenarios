# Enterprise DevOps Playbook: Resolving Availability Zone Storage Mismatches

This playbook outlines the incident response engineering pipeline for resolving a production infrastructure failure where a Pod freezes in a `Pending` or `ContainerCreating` cycle due to an Availability Zone (AZ) affinity mismatch with its cloud storage disk.

## 🕵️‍♂️ The Architectural Scenario
In cloud infrastructures (such as AWS, GCP, or Azure), block storage volumes (like AWS EBS or GCP Persistent Disks) are tightly bounded to a single **Availability Zone (AZ)** (e.g., `us-east-1a`). They cannot physically bridge across geographical server farms.

**The Incident Trap:** If a Pod relies on an existing static cloud volume that lives in zone `us-east-1a`, but the Kubernetes scheduler blindly assigns that Pod to a worker node sitting inside zone `us-east-1b`, the volume attachment fails. The container runtime cannot access the hardware block device across zones, trapping the rollout.

---

## 🕵️‍♂️ Enterprise Incident Triage Pipeline

### 1. Track Deployment Stalls
Query the active namespace status thread when a cloud migration or node failure recovery hangs:
```bash
kubectl get pods -n production-data
```
*System State:* The stateful pod is stuck in a `Pending` or `ContainerCreating` state.

### 2. Extract Volume Attachment Warnings
Pull the node engine infrastructure events directly from the API server:
```bash
POD_NAME=$(kubectl get pods -n production-data -o jsonpath='{.items[0].metadata.name}')
kubectl describe pod $POD_NAME -n production-data
```
Navigate straight to the **Events:** block at the bottom of the system output:
```text
Events:
  Type     Reason                  Age   From               Message
  ----     ------                  ---   ----               -------
  Warning  FailedScheduling        45s   default-scheduler  0/3 nodes are available: 1 node(s) had volume node affinity conflict, 2 Insufficient memory.
  # OR IF STUCK IN CONTAINERCREATING:
  Warning  FailedAttachVolume      30s   attachdetach-cntl  Multi-Attach error for volume "pvc-xxxx": Volume is already attached to another node or in a different AZ zone.
```
*Triage Analysis:* The explicit string **`volume node affinity conflict`** or **`different AZ zone`** confirms a geographical topological mismatch across your data centers.

### 3. Verify Volume Zone Properties vs Node Labels
Locate where your cloud disk lives compared to your worker nodes:
```bash
# Check the zone mapping assigned to your PersistentVolume
kubectl get pv -o custom-columns=NAME:.metadata.name,ZONE:.metadata.labels.topology\.kubernetes\.io/zone

# Check the zone mappings of your active cluster hardware nodes
kubectl get nodes --show-labels | grep topology.kubernetes.io/zone
```
*Triage Discovery:* The PersistentVolume is pinned strictly to `us-east-1a`, but your only available healthy worker nodes with free memory resources are sitting inside `us-east-1b`.

---

## 🛠️ Production-Safe GitOps Resolution

To permanently fix this topological conflict using standard enterprise declarative code workflows, you instruct the scheduler to respect the volume's location by injecting a strict **`nodeAffinity`** or **`nodeSelector` constraint** into the Pod template.

1. Open your deployment manifest configuration file:
   ```bash
   nano database-deployment.yaml
   ```
2. Locate the Pod `spec` structure and force a hard topological scheduling rule matching the exact zone coordinates of your physical storage device:
   ```yaml
   spec:
     # 🍏 Node Affinity injected to force the pod into the volume's home zone
     affinity:
       nodeAffinity:
         requiredDuringSchedulingIgnoredDuringExecution:
           nodeSelectorTerms:
           - matchExpressions:
             自由- key: topology.kubernetes.io/zone
               operator: In
               values:
               - us-east-1a # ✅ Explicitly matches the exact zone footprint where the cloud disk is physically provisioned
     containers:
     - name: db-engine
       image: postgres:15
   ```
3. Apply the updated code manifest declaratively to update the cluster definitions:
   ```bash
   kubectl apply -f database-deployment.yaml
   ```

### 4. Monitor Managed Infrastructure Recovery
Track the zero-downtime rolling update process:
```bash
kubectl rollout status deployment/database-deployment -n production-data
```
*Expected Resolution:* The `kube-scheduler` reads the new affinity constraints, bypasses the nodes in zone `1b`, and places the Pod on a node inside zone `1a`. The storage engine attaches the block layer successfully, and the deployment moves cleanly into a healthy **`Running`** status state.
