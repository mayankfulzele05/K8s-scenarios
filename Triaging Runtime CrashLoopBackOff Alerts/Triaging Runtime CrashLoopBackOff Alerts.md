# Enterprise DevOps Playbook: Triaging Runtime CrashLoopBackOff Alerts (Q82)

This enterprise lab simulates a critical corporate production outage (P1) where an application microservice passes static container builds but crashes continuously at runtime due to missing infrastructure configurations.

## 🏢 Infrastructure Incident Creation
Execute the setup script to provision the multi-node testing infrastructure and trigger the corporate deployment failure:
```bash
chmod +x setup-lab.sh
./setup-lab.sh
```

---

## 🕵️‍♂️ Enterprise Incident Response & Triage Pipeline

### 1. Evaluate Blast Radius (Impact Assessment)
Query the high-level deployment health to evaluate how hard user traffic is being dropped:
```bash
kubectl get deployments -n production-services
```
Identify the live states of the individual application nodes:
```bash
kubectl get pods -n production-services
```
*System State:* The deployment shows `0/3` available replicas. Replicas are caught in an infinite cycle, triggering the `CrashLoopBackOff` state.

### 2. Audit Container Termination Exit Codes
Isolate an unstable node and extract the exact container exit parameters returned by the container runtime:
```bash
# Capture a pod name dynamically
POD_NAME=$(kubectl get pods -n production-services -l app=order-api -o jsonpath='{.items.metadata.name}')

# Inspect node events and state
kubectl describe pod $POD_NAME -n production-services
```
Navigate to the `Containers -> Last State` block:
*   If `Exit Code: 137` ➡️ The application exceeded cgroup container limits and was killed by the OS kernel (**OOMKilled**).
*   If `Exit Code: 1` ➡️ The application process encountered a fatal inner software exception or configuration failure.

*Actual State:* The system reports `Exit Code: 1`, pointing to an application-level failure.

### 3. Extract Historical Crash Dump Logs
Because the container process is terminated, standard logs may capture blank initialization screens. Query the buffer stream from the **previous dead container instance**:
```bash
kubectl logs $POD_NAME -n production-services --previous
```
*Extracted Root Cause Output:*
```text
Booting Order Processor Microservice v2.4.1...
[FATAL RUNTIME ERROR] Database connection string is NULL or empty!
Thread 'main' java.lang.NullPointerException at com.enterprise.order.DBConnect.init
```

---

## 🛠️ Production-Safe GitOps Resolution

Following enterprise declarative workflows, do not apply manual imperative CLI patches. Modify the source configuration deployment infrastructure-as-code file:

1. Open the deployment manifest file in your code editor:
   ```bash
   nano lab-manifest-q82.yaml
   ```
2. Locate the container block and map the missing environment variables required by the application runtime layer:
   ```yaml
   env:
   - name: DB_CONNECTION_STRING
     value: "postgresql://prod-db-user:EncryptedTokenPassword@aws-rds-cluster:5432/orders"
   ```
3. Apply the corrected configuration manifest to trigger an automated rolling update:
   ```bash
   kubectl apply -f lab-manifest-q82.yaml
   ```

### 4. Monitor Managed Infrastructure Recovery
Track the zero-downtime microservice transition in real-time to close the corporate ticket:
```bash
kubectl rollout status deployment/order-processor -n production-services
```
*Expected Output:* `deployment "order-processor" successfully rolled out`.

---

## 🧹 Post-Incident Environment Cleanup
Tear down the lab configuration and cluster components to free up local machine resources:
```bash
kind delete cluster --name troubleshooting-cluster
rm lab-manifest-q82.yaml kind-config.yaml
```
