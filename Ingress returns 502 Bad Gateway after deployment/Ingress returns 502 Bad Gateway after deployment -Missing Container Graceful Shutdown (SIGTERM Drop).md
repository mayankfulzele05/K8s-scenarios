# Enterprise DevOps Playbook: Triaging Empty Endpoints & Readiness Probe Outages (Q97)

This production lab models an edge routing incident response scenario where an unverified health path configuration blocks microservices from passing cluster readiness bounds, forcing Ingress proxies to drop an HTTP 502/503 response.

## 🏢 Infrastructure Incident Creation
Execute the setup script to provision the multi-node testing infrastructure, initialize the Nginx Ingress proxy, and inject the broken health check paths:
```bash
chmod +x setup-lab.sh
./setup-lab.sh
```

---

## 🕵️‍♂️ Enterprise Incident Response & Triage Pipeline

### 1. Replicate External Client Exposure
Execute an edge payload lookup to confirm the status code returned to the external internet:
```bash
curl -I http://localhost/
```
*System State:* The gateway returns an explicit error payload: `HTTP/1.1 502 Bad Gateway`.

### 2. Query Ingress Controller Error Logs
Pull the log buffer history from inside the core proxy engine to trace the upstream mapping states:
```bash
kubectl logs -n ingress-nginx -l app.kubernetes.io/component=controller --tail=20
```
*Proxy Trace Data Discovery:*
```text
[error] 32#32: *2145122 no live upstreams while connecting to upstream, upstream: "http://portal-service/"
```
This confirms that the Ingress proxy engine is live and reachable, but has zero backend node locations mapped to drop traffic onto.

### 3. Validate Internal Endpoint Routing Arrays
Verify if the target abstraction tracking service has located valid, healthy application pod addresses:
```bash
kubectl get endpoints portal-service -n production-traffic
```
*Triage Discovery:* The system returns `ENDPOINTS: <none>`. This isolates the problem directly to health check exclusions. The pods are running, but the cluster manager has removed them from traffic routing tables.

### 4. Isolate Node Kubelet Probe Events
Extract the lifecycle event logs from the underlying worker node hosting the application pods:
```bash
# Capture a pod name dynamically
POD_NAME=$(kubectl get pods -n production-traffic -l app=portal-ui -o jsonpath='{.items[0].metadata.name}')

# Inspect node events
kubectl describe pod $POD_NAME -n production-traffic
```
Navigate to the **Events:** block at the bottom of the system output:
```text
Events:
  Type     Reason     Age   From     Message
  ----     ------     ---   ----     -------
  Warning  Unhealthy  12s   kubelet  Readiness probe failed: HTTP probe failed with statuscode: 404
```
This isolates the root cause: The deployment configuration manifest mandates a readiness probe path (`/healthz`) that does not exist on the application web server, causing continuous health-check failures.

---

## 🛠️ Production-Safe GitOps Resolution

Following enterprise infrastructure-as-code hardening standards, do not hot-patch variables via direct CLI configuration overrides. Update your declarative deployment configuration files:

1. Open the deployment manifest file in your code editor:
   ```bash
   nano lab-manifest-q97-probe.yaml
   ```
2. Locate the container `readinessProbe` section and align the HTTP path parameters to a verified existing route endpoint:
   ```yaml
   spec:
     containers:
     - name: ui-engine
       readinessProbe:
         httpGet:
           path: / # ✅ Aligned from /healthz to protect traffic tables
           port: 80
   ```
3. Apply the updated manifest configuration file declaratively to the cluster control plane to reload the routing matrices:
   ```bash
   kubectl apply -f lab-manifest-q97-probe.yaml
   ```

### 4. Monitor Infrastructure Recovery
Track the deployment update progress until complete:
```bash
kubectl rollout status deployment/dynamic-web-portal -n production-traffic
```
Re-execute the external lookup command thread to verify complete traffic recovery across your architecture:
```bash
curl -I http://localhost/
```
*Expected Output:* `HTTP/1.1 200 OK`.

---

## 🧹 Post-Incident Environment Cleanup
Tear down the lab configuration and cluster components to free up local machine resources:
```bash
kind delete cluster --name troubleshooting-cluster
rm lab-manifest-q97-probe.yaml kind-config.yaml
```
