# Enterprise DevOps Playbook: Triaging Registry Authentication Failures (Q83)

This production lab models an infrastructure outage where a microservice rollout fails due to an unauthorized or invalid container registry credentials block, triggering an `ImagePullBackOff` loop.

## 🏢 Infrastructure Incident Creation
Execute the setup script to provision the multi-node testing infrastructure and trigger the credential block:
```bash
chmod +x setup-lab.sh
./setup-lab.sh
```

---

## 🕵️‍♂️ Enterprise Incident Response & Triage Pipeline

### 1. Evaluate Blast Radius (Impact Assessment)
Query the runtime states inside the targeted namespace to verify container availability:
```bash
kubectl get pods -n production-secure
```
*System State:* Replicas display an unstable operational phase, getting caught permanently in an `ErrImagePull` / `ImagePullBackOff` loop.

### 2. Isolate Node Registry Communication Diagnostics
Query the pod configuration metadata to extract the exact error string returned by the node's container engine runner:
```bash
# Capture an active pod name dynamically
POD_NAME=$(kubectl get pods -n production-secure -l app=auth-api -o jsonpath='{.items.metadata.name}')

# Inspect node events
kubectl describe pod \$POD_NAME -n production-secure
```
Navigate straight to the **Events:** block at the bottom of the system output stream:
```text
Events:
  Type     Reason     Age                From               Message
  ----     ------     ---                ----               -------
  Warning  Failed     14s (x2 over 40s)  kubelet            Failed to pull image "...": rpc error: code = Unknown desc = pull access denied, repository does not exist or may require 'docker login'
```

### 3. Root Cause Classification Matrix
When reading the `kubectl describe` events for an image pull failure, apply this enterprise triage matrix to locate the error:
*   **If `code = NotFound`** ➡️ Typo in the image name or tag string.
*   **If `access denied / unauthorized`** ➡️ Missing or expired registry secret keys (`imagePullSecrets`).
*   **If `connection timed out / context deadline exceeded`** ➡️ Network Firewall blockage or missing NAT Gateway routes on the worker nodes.

*Actual State:* The engine logs report `pull access denied`, isolating the issue to registry authentication failures.

---

## 🛠️ Production-Safe GitOps Resolution

Following enterprise infrastructure-as-code patterns, update the configuration tracking file to map your container workloads to a validated image asset path:

1. Open the deployment manifest file in your code editor:
   ```bash
   nano lab-manifest-q83-private.yaml
   ```
2. Locate the container target image string and correct the production target route:
   ```yaml
   spec:
     containers:
     - name: secure-vault
       image: registry.k8s.io/pause:3.9 # ✅ Corrected path to an authenticated infrastructure mirror
   ```
3. Apply the updated code manifest declaratively to trigger a managed Kubernetes rolling update:
   ```bash
   kubectl apply -f lab-manifest-q83-private.yaml
   ```

### 4. Monitor Microservice Recovery
Monitor the rollout status thread to confirm stable infrastructure convergence:
```bash
kubectl rollout status deployment/authentication-service -n production-secure
```
*Expected Output:* `deployment "authentication-service" successfully rolled out`.

---

## 🧹 Post-Incident Environment Cleanup
Tear down the lab configuration and cluster components to free up local machine resources:
```bash
kind delete cluster --name troubleshooting-cluster
rm lab-manifest-q83-private.yaml kind-config.yaml
```


<img width="1897" height="1025" alt="image" src="https://github.com/user-attachments/assets/c73027e3-2284-408a-b3ca-feef6cd580b4" />
<img width="1841" height="244" alt="image" src="https://github.com/user-attachments/assets/77c4a752-8484-4264-a448-012c35079f4c" />


