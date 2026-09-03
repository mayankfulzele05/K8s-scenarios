# Enterprise DevOps Playbook: Diagnosing & Recovering NotReady Compute Nodes (Q85)

This production simulation environment models an infrastructure failure where a cluster worker node loses its connection agent daemon (`kubelet`), dropping into a `NotReady` state and threatening service availability.

## 🏢 Infrastructure Incident Creation
Execute the setup script to provision the multi-node cluster topology and trigger the system level process failure:
```bash
chmod +x setup-lab.sh
./setup-lab.sh
```

---

## 🕵️‍♂️ Enterprise Incident Response & Triage Pipeline

### 1. Evaluate Compute Cluster Integrity
Query the high-level infrastructure tracking layers to locate the broken node:
```bash
kubectl get nodes
```
*System State:* The target compute host `troubleshooting-cluster-worker` flips to `STATUS: NotReady`.

### 2. Isolate the Broken Node (Stop the Bleeding)
Prevent the scheduler from assigning new application workloads to the unstable host during the triage window:
```bash
kubectl cordon troubleshooting-cluster-worker
```

### 3. Review Node Structural Conditions
Extract the last known heartbeat environmental conditions stored by the API manager:
```bash
kubectl describe node troubleshooting-cluster-worker
```
Analyze the core system pressure matrix:
*   **If MemoryPressure = True** ➡️ The machine ran out of physical memory. Triage application memory leaks.
*   **If DiskPressure = True** ➡️ The local root drive partition is full. Clean container storage layers (`/var/lib/containerd`).
*   **If All Pressures = False but Ready = False** ➡️ The system service daemon itself crashed or lost its network routes.

### 4. Direct OS Host Debugging
Execute a secure shell link onto the underlying Linux host layer to verify the status of the `kubelet` system service process:
```bash
# Open node host access terminal
docker exec -it troubleshooting-cluster-worker bash

# Audit system daemon execution status (Real VM: systemctl status kubelet)
ps aux | grep kubelet

# Extract system event error logs (Real VM: journalctl -u kubelet -n 50)
cat /kind/kubelet.log | tail -n 20
```
*Root Cause:* The `kubelet` binary execution loop was stopped, causing the machine to miss its mandatory lease check-in intervals with the API control plane.

---

## 🛠️ Production-Safe Resolution

1. Re-initialize and boot the background system manager process inside the worker node terminal shell:
   ```bash
   kubelet &
   exit
   ```
2. Track the cluster connection handshake protocol to verify node state stabilization:
   ```bash
   kubectl get nodes -w
   ```
3. Once status returns to `Ready`, safely restore standard application resource routing:
   ```bash
   kubectl uncordon troubleshooting-cluster-worker
   ```

---

## 🧹 Post-Incident Environment Cleanup
Tear down the lab configuration and cluster components to free up local machine resources:
```bash
kind delete cluster --name troubleshooting-cluster
rm lab-manifest-q85.yaml kind-config.yaml
```
