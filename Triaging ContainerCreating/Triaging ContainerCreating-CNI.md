# Enterprise DevOps Playbook: Triaging CNI Network Plugin Failures (Q84)

This production lab environment models a critical microservice deployment failure where a Pod gets stuck permanently in a `ContainerCreating` phase due to a `FailedCreatePodSandBox` CNI network configuration exception.

## 🏢 Infrastructure Incident Creation
Execute the setup script to provision the multi-node testing infrastructure and trigger the CNI sandbox failure:
```bash
chmod +x setup-lab.sh
./setup-lab.sh
```

---

## 🕵️‍♂️ Enterprise Incident Response & Triage Pipeline

### 1. Evaluate Blast Radius (Impact Assessment)
Query the running pods in the target networking namespace:
```bash
kubectl get pods -n production-network-cni
```
*System State:* Replicas freeze permanently in the `ContainerCreating` status, completely halting infrastructure convergence loops.

### 2. Isolate Node Level Networking Event Traces
Since the container engine cannot bind a virtual interface link, pull the runtime diagnostics from the node's Kubelet agent:
```bash
# Capture an active pod name dynamically
POD_NAME=$(kubectl get pods -n production-network-cni -l app=billing-api -o jsonpath='{.items.metadata.name}')

# Inspect node events
kubectl describe pod \$POD_NAME -n production-network-cni
```
Navigate straight to the **Events:** data block at the bottom of the system output:
```text
Events:
  Type     Reason                  Age                From               Message
  ----     ------                  ---                ----               -------
  Warning  FailedCreatePodSandBox  10s (x4 over 35s)  kubelet            Failed to create pod sandbox: rpc error: code = Unknown desc = failed to setup network for sandbox
```

### 3. Root Cause Classification Matrix (CNI Troubleshooting)
When a Pod freezes at `FailedCreatePodSandBox`, isolate the failure using this enterprise playbook blueprint:
*   **Scenario A: IP Address Exhaustion** ➡️ The error explicitly prints `no IP addresses available`. This happens commonly with the AWS VPC CNI when a worker node instance type runs completely out of elastic network secondary private IPs.
*   **Scenario B: CNI DaemonSet Crash** ➡️ The error flags `CNI network not initialized`. This means the underlying system network routing pods (Calico, Cilium) are dead or crash-looping in the `kube-system` namespace.
*   **Scenario C: Invalid Manifest Settings** ➡️ The deployment calls an unavailable or misconfigured custom `runtimeClassName` parameter, blocking `containerd` from processing the interface handshake hook.

*Actual State:* The manifest references an invalid network configuration parameter, causing a runtime sandbox setup timeout.

---

## 🛠️ Production-Safe GitOps Resolution

Following enterprise infrastructure-as-code patterns, modify the source tracking configuration manifest file directly to strip out the invalid sandbox parameters:

1. Open the deployment manifest file in your code editor:
   ```bash
   nano lab-manifest-q84-cni.yaml
   ```
2. Locate the Pod specification template block and delete the broken networking constraint line:
   ```yaml
   spec:
     # ❌ Delete this broken line to restore standard container runtime configuration
     runtimeClassName: non-existent-cni-runtime 
   ```
3. Apply the updated code manifest declaratively to trigger a managed Kubernetes rolling update:
   ```bash
   kubectl apply -f lab-manifest-q84-cni.yaml
   ```

### 4. Monitor Microservice Recovery
Monitor the rollout status thread to confirm stable infrastructure convergence:
```bash
kubectl rollout status deployment/billing-service -n production-network-cni
```
*Expected Output:* `deployment "billing-service" successfully rolled out`.

---

## 🧹 Post-Incident Environment Cleanup
Tear down the lab configuration and cluster components to free up local machine resources:
```bash
kind delete cluster --name troubleshooting-cluster
rm lab-manifest-q84-cni.yaml kind-config.yaml
```
