# Enterprise DevOps Playbook: Triaging Intermittent DNS Failures (Q91)

This production lab models an advanced infrastructure networking failure where high-frequency UDP lookup collisions trigger intermittent connection drops (`Temporary failure in name resolution`) due to Linux kernel conntrack race conditions.

## 🏢 Infrastructure Incident Creation
Execute the setup script to provision the multi-node testing infrastructure and start the packet transmission stream:
```bash
chmod +x setup-lab.sh
./setup-lab.sh
```

---

## 🕵️‍♂️ Enterprise Incident Response & Triage Pipeline

### 1. Identify Service Degradation Signs
Monitor application log streams to confirm the frequency of name resolution anomalies:
```bash
kubectl logs -f deployment/client-gateway -n production-core
```
*System State:* The service functions normally for most loops but flags random intermittent lookup warnings: `nslookup: can't resolve 'google.com'`.

### 2. Isolate the CoreDNS Infrastructure Layer
Rule out system provider capacity blocks by validating the operational resource profiles of the cluster DNS engines:
```bash
kubectl get deployment coredns -n kube-system
kubectl top pods -n kube-system -l k8s-app=kube-dns
```
If the pods have stable resource usage and `kubectl logs -n kube-system -l k8s-app=kube-dns` reports zero internal failures, the drops are occurring at the worker node's Linux conntrack table wrapper layer.

### 3. Review the Injected Pod Resolver Specifications
Query the runtime internal network resolver specifications file:
```bash
POD_NAME=\$(kubectl get pods -n production-core -l app=API-gateway -o jsonpath='{.items.metadata.name}')
kubectl exec -it \$POD_NAME -n production-core -- cat /etc/resolv.conf
```
*Triage Discovery:* The system returns an active `options ndots:5` property. This triggers a 5x query amplification cascade for external connections, overwhelming the node sockets and inducing parallel `A`/`AAAA` conntrack conflicts.

---

## 🛠️ Production-Safe GitOps Resolution

Following enterprise network-hardening standards, update your declarative deployment configuration manifests to enforce explicit path constraints and conntrack workarounds:

1. Open the deployment manifest file in your code editor:
   ```bash
   nano lab-manifest-q91.yaml
   ```
2. Inject a protective `dnsConfig` block directly into the Pod spec layer to change the lookup behavior:
   ```yaml
   spec:
     dnsConfig:
       options:
       - name: ndots
         value: "1"                 # Reduce lookups by forcing direct external lookups instantly
       - name: single-request-reopen # Workaround for Linux kernel UDP conntrack race bugs
     containers:
     - name: network-tester
   ```
3. Apply the updated manifest declaratively to execute a clean zero-downtime update:
   ```bash
   kubectl apply -f lab-manifest-q91.yaml
   ```

### 4. Monitor Infrastructure Recovery
Track the deployment rollout status to confirm cluster-wide network stabilization:
```bash
kubectl rollout status deployment/client-gateway -n production-core
```
*Expected Output:* `deployment "client-gateway" successfully rolled out`.

---

## 🧹 Post-Incident Environment Cleanup
Tear down the lab configuration and cluster components to free up local machine resources:
```bash
kind delete cluster --name troubleshooting-cluster
rm lab-manifest-q91.yaml kind-config.yaml
```

<img width="1918" height="632" alt="image" src="https://github.com/user-attachments/assets/f48f2242-a036-4bf5-9a30-a54bacd895e0" />

