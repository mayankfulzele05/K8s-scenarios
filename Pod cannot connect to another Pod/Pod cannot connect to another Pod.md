# Enterprise DevOps Playbook: Debugging Pod-to-Pod Network Outages

This advanced infrastructure simulation models a complex, multi-layered networking failure across boundaries where a client application fails to deliver network payloads to a core backend component.

## 🏢 Infrastructure Incident Creation
Execute the setup script to provision the multi-node testing infrastructure, isolate sub-spaces, and inject the layout faults:
```bash
chmod +x setup-lab.sh
./setup-lab.sh
```

---

## 🕵️‍♂️ Enterprise Incident Response Triage Pipeline

When an application logs connection timeouts or resolution failures, follow this 5-step checklist:

### 1. The Service Name Check
Verify that the service name the client application is hitting matches the actual resource name inside the cluster database [pdf_mpSGEi.pdf, 1.2.5]. 

### 2. The Namespace Check (DNS Isolation)
If applications communicate **across different namespaces**, the standard name string resolution fails [pdf_mpSGEi.pdf, 1.2.8]. You must query the target using its Fully Qualified Domain Name (FQDN) [pdf_mpSGEi.pdf, 1.2.5]:
```text
<target-service-name>.<target-namespace>.svc.cluster.local
```

### 3. The Label Selector Check
If your network requests return connection drops, check the mapping endpoints array:
```bash
kubectl get endpoints <service-name> -n <namespace>
```
If `ENDPOINTS` outputs `<none>`, the Service's `spec.selector` is misaligned with the Pods' metadata `labels` [pdf_mpSGEi.pdf, 1.2.10]. Cross-reference configurations using:
```bash
kubectl get svc <service-name> -n <namespace> -o jsonpath='{.spec.selector}'
kubectl get pods -n <namespace> --show-labels
```

### 4. The Port & TargetPort Alignment Check
Verify that the communication port channels match between components [pdf_mpSGEi.pdf, 1.2.5]:
*   **`port`:** The port exposed by the Service abstraction layer [pdf_mpSGEi.pdf, 1.2.5].
*   **`targetPort`:** The exact inner container process port (`containerPort`) the application code binds to [pdf_mpSGEi.pdf, 1.2.5].

### 5. The NetworkPolicy Verification Check
If DNS resolves and endpoints are populated but requests still time out, an enterprise security wall is blocking traffic [pdf_mpSGEi.pdf, 1.2.2]. Check the active security policies in the namespace [pdf_mpSGEi.pdf, 1.2.2]:
```bash
kubectl get networkpolicies -n <namespace>
kubectl describe networkpolicy <policy-name> -n <namespace>
```
Ensure that an ingress rule is explicitly configured to whitelist traffic originating from the client's namespace or pod label scope [pdf_mpSGEi.pdf, 1.2.2].

---

## 🧹 Post-Incident Environment Cleanup
Tear down the lab configuration and cluster components to free up local machine resources:
```bash
kind delete cluster --name troubleshooting-cluster
rm lab-networking-incident.yaml lab-manifest-fixed.yaml kind-config.yaml
```
