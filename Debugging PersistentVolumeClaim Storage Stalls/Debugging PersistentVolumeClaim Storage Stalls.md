# Enterprise DevOps Playbook: Debugging PersistentVolumeClaim Storage Stalls

This production lab environment models an infrastructure configuration failure where an enterprise application (Jenkins Master) fails to initialize and remains stuck in a `Pending` state due to an invalid or unconfigured CSI StorageClass backend claim.

## 🏢 Infrastructure Incident Creation
Execute the setup script to provision the multi-node testing infrastructure and trigger the storage boundary blockages:
```bash
chmod +x setup-lab.sh
./setup-lab.sh
```

---

## 🕵️‍♂️ Enterprise Incident Response & Triage Pipeline

### 1. Evaluate Storage Layer Degradation
Query the structural state parameters inside the targeted line-of-business tracking namespace:
```bash
kubectl get deployments,pvc,pods -n production-jenkins
```
*System State:* The Deployment indicates `0/1` active nodes, with the application Pod and its corresponding PersistentVolumeClaim locked together in a `Pending` phase loop.

### 2. Extract Infrastructure Provisioner Warning Traces
Since the resource is blocked before volume assembly, pull the central storage system logs directly from the cluster control plane:
```bash
kubectl describe pvc jenkins-home-pvc -n production-jenkins
```
Navigate to the **Events:** data tracing array at the bottom of the system output:
```text
Events:
  Type     Reason              Age   From                         Message
  ----     ------              ---   ----                         -------
  Warning  ProvisioningFailed  10s   persistentvolume-controller  storageclass.storage.k8s.io "aws-ebs-premium-gp3" not found
```

### 3. Root Cause Assessment
The explicit warning string `storageclass ... not found` isolates the configuration mismatch. The deployment manifest requests an AWS cloud storage profile (`aws-ebs-premium-gp3`) that is non-existent within our current bare-metal cluster topologies. 

To determine active operational drivers available for reclamation, audit cluster storage providers:
```bash
kubectl get storageclass
```

---

## 🛠️ Production-Safe GitOps Resolution

Following enterprise infrastructure-as-code hardening standards, update the core tracking manifest files directly. Because the `storageClassName` property is structurally immutable after creation, pass a `--force` instruction to replace the resource cleanly:

1. Open the deployment manifest file in your code editor:
   ```bash
   nano lab-manifest-pvc-pending.yaml
   ```
2. Locate the `PersistentVolumeClaim` properties block and align the target storage manager with a verified cluster provider:
   ```yaml
   spec:
     accessModes:
       - ReadWriteOnce
     storageClassName: standard   # ✅ Aligned to match native cluster storage parameters
     resources:
       requests:
         storage: 10Gi
   ```
3. Execute a forced declarative update to replace the blocked claim asset inside ETCD:
   ```bash
   kubectl replace -f lab-manifest-pvc-pending.yaml --force
   ```

### 4. Monitor Microservice Recovery
Monitor the namespace status thread to confirm successful local CSI dynamic provisioning and Pod initialization:
```bash
kubectl get pvc,pods -n production-jenkins
```
*Expected Output:* The PVC switches to `Bound` and the Jenkins server transitions seamlessly into a steady `Running` status state.

---

## 🧹 Post-Incident Environment Cleanup
Tear down the lab configuration and cluster components to free up local machine resources:
```bash
kind delete cluster --name troubleshooting-cluster
rm lab-manifest-pvc-pending.yaml kind-config.yaml
```
