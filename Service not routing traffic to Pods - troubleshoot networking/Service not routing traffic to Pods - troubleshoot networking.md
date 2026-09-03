# Enterprise DevOps Playbook: Diagnosing Service Endpoint Routing Outages (Q86)

This production lab environment models a high-severity microservice communication impairment where an internal load-balancer Service fails to route incoming client traffic down to functional backend application Pods.

## 🏢 Infrastructure Incident Creation
Execute the setup script to provision the multi-node testing network and inject the label selector mismatch:
```bash
chmod +x setup-lab.sh
./setup-lab.sh
```

---

## 🕵️‍♂️ Enterprise Incident Response & Triage Pipeline

### 1. Audit Microservice Execution Health
Query the running state of the targeted backend pods to verify compute availability:
```bash
kubectl get pods -n production-network-svc
```
*System State:* The compute infrastructure layers report `STATUS: Running` and `READY: 1/1`, isolating the bottleneck entirely to the network abstractions layer.

### 2. Inspect Load Balancer Core Abstractions
Inspect the metadata and endpoint registers of the live Service object:
```bash
kubectl describe service order-service -n production-network-svc
```
Locate the **`Endpoints:`** parameter line:
```text
Name:              order-service
Namespace:         production-network-svc
Type:              ClusterIP
IP Family Policy:  SingleStack
IP Families:       IPv4
IP:                10.96.25.14
Port:              <unset>  80/TCP
TargetPort:        80/TCP
Selector:          app=order-api
Endpoints:         <none>             <--- ❌ CRITICAL BLINDSPOT IDENTIFIED
```
*Triage Discovery:* The network controller maps `Endpoints: <none>`. This proves that while the Service IP exists, it has zero valid backing Pod routes registered to forward client packets to.

### 3. Diagnose Selector Definition Drift
Audit the specific query selectors used by the Service against the active structural tags attached to the pods:
```bash
# Extract Service requirements
kubectl get svc order-service -n production-network-svc -o jsonpath='{.spec.selector}'

# Extract Pod active labels
kubectl get pods -n production-network-svc --show-labels
```
*Root Cause:* The Service filters traffic for `app=order-api`, but the deployment workloads wear the tag label `app=order-backend`. The metadata mismatch blocks automatic discovery routines.

---

## 🛠️ Production-Safe GitOps Resolution

Following enterprise infrastructure-as-code patterns, update the master configuration tracking manifest file directly to align the metadata tags:

1. Open the deployment manifest file in your code editor:
   ```bash
   nano lab-manifest-q86.yaml
   ```
2. Locate the Service resource definition block at the bottom and align its selector key value string:
   ```yaml
   spec:
     ports:
     - port: 80
       targetPort: 80
     selector:
       app: order-backend # ✅ Corrected from order-api to match template labels
   ```
3. Apply the updated code manifest declaratively to the cluster control plane:
   ```bash
   kubectl apply -f lab-manifest-q86.yaml
   ```

### 4. Verify Traffic Endpoint Convergence
Re-run the description check to confirm the Kubernetes network layer successfully maps the private Pod IP allocations:
```bash
kubectl describe service order-service -n production-network-svc
```
*Expected Output:* The `Endpoints:` array populates with the private node IP addresses of the running Pod instances, indicating restabilised load-balanced routes.

---

## 🧹 Post-Incident Environment Cleanup
Tear down the lab configuration and cluster components to free up local machine resources:
```bash
kind delete cluster --name troubleshooting-cluster
rm lab-manifest-q86.yaml kind-config.yaml
```

<img width="1911" height="814" alt="image" src="https://github.com/user-attachments/assets/8feb0ef6-fde4-4b6e-a6a8-a65a3b416ce2" />

