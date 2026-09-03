# Enterprise DevOps Playbook: Triaging ContainerCreating Dependency Failures (Q84)

This production lab environment models an infrastructure roadblock scenario where an enterprise microservice rollout freezes in a `ContainerCreating` phase due to an unfulfilled volume dependency.

## 🏢 Infrastructure Incident Creation
Execute the setup script to provision the multi-node testing infrastructure and trigger the dependency blockage:
```bash
chmod +x setup-lab.sh
./setup-lab.sh
```

---

## 🕵️‍♂️ Enterprise Incident Response & Triage Pipeline

### 1. Evaluate Blast Radius (Impact Assessment)
Query the running pods inside the targeted line-of-business tracking namespace:
```bash
kubectl get pods -n production-finance
```
*System State:* Replicas freeze permanently in the `ContainerCreating` state, completely stalling service initializations.

### 2. Isolate Node Level Runtime Events
Since the container has not reached execution, internal container log streams do not exist. Query the infrastructure event trace log records directly from the Kubelet agent:
```bash
# Capture an active pod name dynamically
POD_NAME=$(kubectl get pods -n production-finance -l app=ledger-api -o jsonpath='{.items.metadata.name}')

# Inspect node execution metadata
kubectl describe pod $POD_NAME -n production-finance
```
Navigate straight to the **Events:** data block at the bottom of the system output:
```text
Events:
  Type     Reason       Age                From               Message
  ----     ------       ---                ----               -------
  Warning  FailedMount  12s (x5 over 45s)  kubelet            MountVolume.SetUp failed for volume "app-config-volume" : configmap "missing-ledger-properties" not found
```

### 3. Root Cause Analysis
The warning string `configmap "..." not found` isolates the operational ticket. The scheduling phase succeeded, but the local node container runner (`containerd`) cannot construct the initial security and storage bounds because a referenced tracking object asset (`ConfigMap` or `Secret`) does not exist inside the tracking namespace.

---

## 🛠️ Production-Safe GitOps Resolution

Following enterprise infrastructure-as-code patterns, do not apply manual imperative CLI patches. Append the missing object tracking parameters directly inside your configuration source control layout file:

1. Open the deployment manifest file in your code editor:
   ```bash
   nano lab-manifest-q84.yaml
   ```
2. Append the missing data block to declare the necessary configuration mapping dependencies:
   ```yaml
   ---
   apiVersion: v1
   kind: ConfigMap
   metadata:
     name: missing-ledger-properties
     namespace: production-finance
   data:
     ledger.conf: |
       database.host=prod-finance-db.cluster.internal
       security.mode=strict
   ```
3. Apply the complete configuration parameters file declaratively to the cluster control plane:
   ```bash
   kubectl apply -f lab-manifest-q84.yaml
   ```

### 4. Monitor Microservice Recovery
Monitor the rollout status thread to confirm stable infrastructure convergence:
```bash
kubectl rollout status deployment/ledger-service -n production-finance
```
*Expected Output:* `deployment "ledger-service" successfully rolled out`.

---

## 🧹 Post-Incident Environment Cleanup
Tear down the lab configuration and cluster components to free up local machine resources:
```bash
kind delete cluster --name troubleshooting-cluster
rm lab-manifest-q84.yaml kind-config.yaml
```
