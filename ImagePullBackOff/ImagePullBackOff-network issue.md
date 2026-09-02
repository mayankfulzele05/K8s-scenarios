# Enterprise DevOps Playbook: Triaging Pull Access Denied Authentication Failures (Q83)

This production lab environment models a critical rolling update failure where a microservice deployment gets stuck in an `ImagePullBackOff` state due to an invalid or expired corporate registry authentication token (`imagePullSecrets`).

## 🏢 Infrastructure Incident Creation
Execute the setup script to provision the multi-node testing infrastructure and trigger the access denial failure:
```bash
chmod +x setup-lab.sh
./setup-lab.sh
```

---

## 🕵️‍♂️ Enterprise Incident Response & Triage Pipeline

### 1. Evaluate Blast Radius (Impact Assessment)
Query the running pods in the target business namespace:
```bash
kubectl get pods -n production-secure
```
*System State:* Replicas flip repeatedly through `ErrImagePull` and `ImagePullBackOff`, completely blocking the deployment.

### 2. Isolate Node Registry Communication Diagnostics
Pull the system metadata log trace directly from the worker node's Kubelet infrastructure layer:
```bash
# Capture an active pod name dynamically
POD_NAME=$(kubectl get pods -n production-secure -l app=vault-api -o jsonpath='{.items.metadata.name}')

# Inspect node events
kubectl describe pod $POD_NAME -n production-secure
```
Navigate straight to the **Events:** block at the bottom of the system readout:
```text
Events:
  Type     Reason     Age                From               Message
  ----     ------     ---                ----               -------
  Warning  Failed     14s (x2 over 40s)  kubelet            Failed to pull image "docker.io/enterprise-secure-vault/core-api:v1": rpc error: code = Unknown desc = failed to pull and unpack image: failed to resolve reference: pull access denied for enterprise-secure-vault/core-api, repository does not exist or may require 'docker login'
```

### 3. Root Cause Assessment
The explicit string `pull access denied... or may require 'docker login'` isolates the problem directly to the security authentication layer. Network routes are clear, but the credentials passed inside the manifest's `imagePullSecrets` array are expired, missing, or unauthorized on the destination container registry server.

---

## 🛠️ Production-Safe GitOps Resolution

Following enterprise infrastructure-as-code patterns, do not execute random imperative CLI patches. Update the source tracking configuration manifest file directly:

1. Open the deployment manifest file in your code editor:
   ```bash
   nano lab-manifest-q83-auth.yaml
   ```
2. Correct the container image string target to point to a validated, authorized repository location:
   ```yaml
   spec:
     containers:
     - name: vault-engine
       image: registry.k8s.io/pause:3.9 # ✅ Updated to an authorized infrastructure mirror asset
   ```
3. Apply the updated code manifest declaratively to trigger an automated Kubernetes rolling update:
   ```bash
   kubectl apply -f lab-manifest-q83-auth.yaml
   ```

### 4. Monitor Microservice Recovery
Monitor the rollout status thread to confirm stable infrastructure convergence:
```bash
kubectl rollout status deployment/secure-auth-vault -n production-secure
```
*Expected Output:* `deployment "secure-auth-vault" successfully rolled out`.

---

## 🧹 Post-Incident Environment Cleanup
Tear down the lab configuration and cluster components to free up local machine resources:
```bash
kind delete cluster --name troubleshooting-cluster
rm lab-manifest-q83-auth.yaml kind-config.yaml
```
