# Enterprise DevOps Playbook: Triaging Unbound PVC Volume Failures (Q84)

This production lab models an infrastructure deadlock scenario where a stateful application deployment freezes in a `ContainerCreating` state due to an unfulfilled, unbound PersistentVolumeClaim (PVC) dependency.

## 🏢 Infrastructure Incident Creation
Execute the setup script to provision the multi-node testing infrastructure and trigger the storage engine failure:
```bash
chmod +x setup-lab.sh
./setup-lab.sh
```

---

## 🕵️‍♂️ Enterprise Incident Response & Triage Pipeline

### 1. Evaluate Blast Radius (Impact Assessment)
Query the running state of the primary data tier namespace:
```bash
kubectl get pods -n production-data
```
*System State:* Replicas freeze permanently in the `ContainerCreating` status, stalling microservice storage handshakes.

### 2. Isolate Pod Infrastructure Events
Query the system metadata logs from the Kubelet agent to locate the storage barrier:
```bash
# Capture the database pod name dynamically
POD_NAME=$(kubectl get pods -n production-data -l app=postgres-db -o jsonpath='{.items.metadata.name}')

# Inspect node events
kubectl describe pod $POD_NAME -n production-data
```
Navigate straight to the **Events:** data block at the bottom of the system output:
```text
Events:
  Type     Reason       Age                From               Message
  ----     ------     ---                ----               -------
  Warning  FailedMount  12s (x5 over 45s)  kubelet            MountVolume.SetUp failed for volume "db-storage" : source PersistentVolumeClaim "postgres-pvc" is not bound
```

### 3. Deep Dive into the Storage Tier Abstraction
Because the pod points directly to an unbound PVC failure, track the diagnostic properties of the storage controller resource:
```bash
kubectl get pvc -n production-data
kubectl describe pvc postgres-pvc -n production-data
```
Review the PVC controller event trace output:
```text
Events:
  Type     Reason                    Age   From                         Message
  ----     ------                    ---   ----                         -------
  Warning  VolumeProvisioningFailed  10s   persistentvolume-controller  storageclass.storage.k8s.io "aws-ebs-gp3-premium" not found
```
*Triage Discovery:* The storage provider layer is failing because the requested storage engine blueprint (`storageClassName: aws-ebs-gp3-premium`) does not exist on this active cluster network topology.

---

## 🛠️ Production-Safe GitOps Resolution

Following enterprise infrastructure-as-code patterns, correct the source configuration repository file directly to align with a functional infrastructure class:

1. Discover valid storage backends actively supported by your current cluster:
   ```bash
   kubectl get storageclass
   ```
2. Open the deployment manifest file in your code editor:
   ```bash
   nano lab-manifest-q84-pvc.yaml
   ```
3. Update the `storageClassName` property to target a verified available class block:
   ```yaml
   spec:
     accessModes:
       - ReadWriteOnce
     resources:
       requests:
         storage: 10Gi
     storageClassName: standard # ✅ Corrected from aws-ebs-gp3-premium to standard
   ```
4. Apply the updated code manifest declaratively to the cluster control plane:
   ```bash
   kubectl apply -f lab-manifest-q84-pvc.yaml
   ```

### 5. Monitor Microservice Recovery
Monitor the rollout status thread to confirm stable infrastructure convergence:
```bash
kubectl rollout status deployment/production-db -n production-data
```
*Expected Output:* `deployment "production-db" successfully rolled out`.

---

## 🧹 Post-Incident Environment Cleanup
Tear down the lab configuration and cluster components to free up local machine resources:
```bash
kind delete cluster --name troubleshooting-cluster
rm lab-manifest-q84-pvc.yaml kind-config.yaml
```

<img width="1834" height="1033" alt="image" src="https://github.com/user-attachments/assets/4044447e-80a6-47b0-b676-1698e5069e5f" />
<img width="1619" height="772" alt="image" src="https://github.com/user-attachments/assets/5e3c400d-8e86-4a3e-95cb-7d6c19fb9a3b" />


