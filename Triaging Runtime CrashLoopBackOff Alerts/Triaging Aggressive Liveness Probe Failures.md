# Enterprise DevOps Playbook: Triaging Container Exit Code 127 Failures (Q82)

This production lab models an deployment failure where a microservice manifest targets an unavailable system binary path inside a secured, minimal container base image.

## 🏢 Infrastructure Incident Creation
Execute the setup script to provision the multi-node testing infrastructure and trigger the binary route failure:
```bash
chmod +x setup-lab.sh
./setup-lab.sh
```

---

## 🕵️‍♂️ Enterprise Incident Response & Triage Pipeline

### 1. Evaluate Blast Radius (Impact Assessment)
Query the running pods in the target namespace:
```bash
kubectl get pods -n production-shipping
```
*System State:* Replicas instantly trip into `CrashLoopBackOff` or `Error` states with high restart frequencies.

### 2. Isolate Container Runtime Exit Parameters
Query the pod configuration metadata to extract the exact termination signatures returned by the container environment:
```bash
# Capture an active pod name dynamically
POD_NAME=$(kubectl get pods -n production-shipping -l app=tracking-api -o jsonpath='{.items.metadata.name}')

# Inspect container states
kubectl describe pod $POD_NAME -n production-shipping
```
Navigate to the container status section and analyze the `Last State:` properties:
```text
Containers:
  shipping-engine:
    State:          Waiting
      Reason:       CrashLoopBackOff
    Last State:     Terminated
      Reason:       Error
      Exit Code:    127              <--- ❌ Standard System Code for "Command/Binary Not Found"
```

### 3. Review OCI Container Engine Logs
Query the log buffer stream to view the error thrown during the initial bootstrap hook:
```bash
kubectl logs $POD_NAME -n production-shipping
```
*Triage Discovery:* The container engine explicitly outputs an execution failure:
```text
OCI runtime create failed: exec: "/bin/bash": stat /bin/bash: no such file or directory
```
This isolates the root cause: The deployment manifest explicitly requests an environment shell execution path (`/bin/bash`) that does not exist inside standard minimal base layers like Alpine Linux.

---

## 🛠️ Production-Safe GitOps Resolution

Following enterprise infrastructure-as-code patterns, update the source manifest file to leverage correct shell parameters:

1. Open the deployment manifest file in your code editor:
   ```bash
   nano lab-manifest-q82-127.yaml
   ```
2. Locate the container properties block and map the command argument list to an available shell executable path (`/bin/sh`):
   ```yaml
   spec:
     containers:
     - name: shipping-engine
       image: alpine:latest
       command: ["/bin/sh", "-c"] # ✅ Corrected from /bin/bash
   ```
3. Apply the updated code manifest declaratively to trigger a managed rolling update update:
   ```bash
   kubectl apply -f lab-manifest-q82-127.yaml
   ```

### 4. Monitor Infrastructure Recovery
Track the deployment update progress until complete:
```bash
kubectl rollout status deployment/tracking-service -n production-shipping
```
*Expected Output:* `deployment "tracking-service" successfully rolled out`.

---

## 🧹 Post-Incident Environment Cleanup
Tear down the lab configuration and cluster components to free up local machine resources:
```bash
kind delete cluster --name troubleshooting-cluster
rm lab-manifest-q82-127.yaml kind-config.yaml
```
