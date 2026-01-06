# kubernetes backup 
* Backing up a Kubernetes cluster involves two primary approaches: focusing solely on the cluster's core state data (etcd) or using a comprehensive tool to capture both the cluster state and application data (Persistent Volumes).  

* **Method 1:** Backing Up the etcd Database (Cluster State Only)
All Kubernetes object configurations and states are stored in the etcd key-value store. Periodically backing this up is crucial for disaster recovery of the cluster's control plane. 

* **Method 2:** Using a Dedicated Backup Tool (Cluster State + Application Data) 
For production environments with stateful applications using Persistent Volumes (PVs), a more robust, application-aware solution is recommended. Tools like **Velero** automate the capture of both Kubernetes resources (manifests, config maps, secrets, etc.) and associated PV snapshots. 

## Backing Up etcd
**Overview:**
* etcd is the distributed key-value store that holds the entire Kubernetes control-plane state.
* All Kubernetes objects (Deployments, Services, ConfigMaps, Secrets, RBAC, CRDs, etc.) are persisted in etcd.
* Backing up etcd typically involves taking periodic point-in-time snapshots using etcdctl.
* This approach is mainly applicable to self-managed Kubernetes clusters. 

## Disadvantages of etcd Backup
**1. Not supported in managed Kubernetes services**
In AWS EKS (Auto Mode), etcd is fully managed by AWS and cannot be accessed or backed up manually.

**2. No application data protection**
etcd backups do not include Persistent Volume (PV) data or external data stores.

**3. High operational risk**
Restoring etcd requires stopping control-plane components and can lead to cluster-wide outages if misconfigured.

**4. Poor scalability for SaaS environments**
Not practical for frequent backups, partial restores, or multi-environment recovery scenarios.

## Why Velero Is Preferred Over etcd Backup
**1. Designed for production Kubernetes environments**
* Supports backup and restore of Kubernetes resources at cluster, namespace, or application level.

**2. Fully compatible with managed services**
* Works natively with AWS EKS, including Auto Mode clusters.

**3. Granular recovery**
* Restore individual namespaces, applications, or resources without impacting the entire cluster.

**4. Application-aware backups**
* Captures Kubernetes manifests, metadata, and optional PV snapshots when required.

**5. Cloud-native storage**
* Stores backups securely in object storage (e.g., Amazon S3) with encryption and retention policies.

**GitOps-friendly**
* Complements Git as the source of truth while covering runtime and generated cluster state not stored in Git.

**etcd backup vs velero**

| Aspect               | etcd Backup         | Velero                  |
| -------------------- | ------------------- | ----------------------- |
| Managed EKS support  | ❌ Not supported     | ✅ Fully supported       |
| Restore granularity  | ❌ Full cluster only | ✅ Namespace / app level |
| Operational risk     | High                | Low                     |
| PV awareness         | ❌ No                | ✅ Optional              |
| GitOps compatibility | ❌ Poor              | ✅ Excellent             |
| Production readiness | Limited             | High                    |


## why we need velero in our setup? 
* eventhough we are taking the backup of MongoDB atlas where our application data is storing, velero protect us us from operational failures and human errors

**GitOps tells us what should exist, Velero captures what actually exists at runtime**

**Example 1: Accidental Namespace Deletion**
“If someone accidentally deletes a namespace during a production window”
* Without Velero:
    * Recreate infra
    * Reapply Helm
    * Re-debug configs
    * 30–60 minutes impact 
* With Velero:
    * Restore namespace in minutes
    * Minimal disruption 

* restore command
```
velero restore create restore-exam-prod \
  --from-backup daily-backup \
  --include-namespaces exam-prod
```

**Example 2: Runtime Hotfix During an Exam**
“During an exam window, an engineer may apply a temporary hotfix directly to the cluster to stabilize a service.”
* That fix:
    * Exists only in the cluster
    * Is not in Git yet 

* If the cluster breaks:
    * Git-based redeploy loses the fix
    * Velero restores exact working state 

**Example 3: CRD or Controller Misconfiguration**
“If a CRD or admission webhook is deleted or misconfigured…”

* Git redeploy may not fully restore runtime state
* Controllers can enter broken states
* Debugging is slow and risky 
* Velero restores:
    * CRDs
    * Controller resources
    * Cluster back to last-known-good state 

* restore command 
```
velero restore create restore-crd \
  --from-backup pre-crd-change
```

“Yes, we can operate without it, but it increases recovery time and operational risk during human errors. Given the low cost and high impact, Velero is a sensible safety layer for a production SaaS platform.”  

