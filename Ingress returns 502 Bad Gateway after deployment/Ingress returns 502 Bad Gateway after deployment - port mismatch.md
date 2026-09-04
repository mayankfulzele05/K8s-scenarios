# Enterprise DevOps Playbook: Triaging Ingress HTTP 502 Bad Gateway Outages (Q97)

This production lab models an edge routing incident response scenario where an enterprise microservice release introduces an upstream port mapping mismatch, triggering edge-level HTTP 502 Bad Gateway rejections.

## 🏢 Infrastructure Incident Creation
Execute the setup script to provision the multi-node testing infrastructure, initialize the Nginx Ingress proxy, and inject the routing faults:
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
*System State:* The connection returns an explicit error payload: `HTTP/1.1 502 Bad Gateway`. This confirms the edge controller accepted the request but cannot connect to the backend microservice.

### 2. Query Edge Ingress Controller Proxy Logs
Pull the log buffer history from inside the core ingress control unit to trace the upstream forward paths:
```bash
kubectl logs -n ingress-nginx -l app.kubernetes.io/component=controller --tail=20
```
*Proxy Trace Data Discovery:*
```text
[error] 32#32: *1154236 connect() failed (111: Connection refused) while connecting to upstream, upstream: "http://10.244.1"
```
This confirms that the Ingress engine is routing packets to port `8080`, where connection requests are actively being dropped.

### 3. Validate Internal Endpoint Routing Arrays
Verify if the target abstraction tracking service has located valid, healthy application pod addresses:
```bash
kubectl get endpoints portal-service -n production-traffic
```
*Triage Discovery:* The system returns active pod IPs bound to port `80` (e.g., `10.244.1.8:80`). Cross-referencing this with the proxy traces proves that the core Ingress manifest is targeting an incorrect port (`8080`), creating an infrastructure routing mismatch.

---

## 🛠️ Production-Safe GitOps Resolution

Following cloud-native architecture standards, do not hot-patch variables via direct CLI configuration overrides. Update your declarative deployment configuration tracking files:

1. Open the deployment manifest file in your code editor:
   ```bash
   nano lab-manifest-q97.yaml
   ```
2. Locate the backend networking rules section under your Ingress metadata block and correct the target port variable parameters:
   ```yaml
   spec:
     ingressClassName: nginx
     rules:
     - http:
         paths:
         - path: /
           backend:
             service:
               name: portal-service
               port:
                 number: 80 # ✅ Aligned from 8080 to match structural Service limits
   ```
3. Apply the updated manifest configuration file declaratively to the cluster control plane to reload the routing matrices:
   ```bash
   kubectl apply -f lab-manifest-q97.yaml
   ```

### 4. Verify Edge Restabilisation
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
rm lab-manifest-q97.yaml kind-config.yaml
```
