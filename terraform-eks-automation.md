Where You Are Right Now (Checkpoint)

✅ VPC module – tested
✅ EKS module – completed
✅ Auto Mode flags – correctly set
✅ IAM cluster + node roles – created
❌ No nodes yet (expected)
❌ No workloads yet

At this stage:

Your cluster exists, but cannot run pods yet — and that is by design

Cluster Access & Validation (Immediate Next Step)
1️⃣ Update kubeconfig
Once you apply later, first thing you’ll do:

aws eks update-kubeconfig \
  --region ap-south-2 \
  --name nexsuiq-test

2️⃣ Verify cluster
kubectl get nodes

Expected result:
No resources found

✅ This confirms:

Control plane is up
Auto Mode has no nodes yet


PHASE 2: Enable IRSA (Very Important – Do This Next) 
----------------------------------------------------
Why this is mandatory:
Auto Mode + Karpenter + AWS controllers require IAM Roles for Service Accounts.
You should:
    Create OIDC provider (if not already)
    Prepare for:
        Karpenter
        ALB Controller
        EBS CSI (already enabled at cluster level)  

👉 This is your next Terraform enhancement, before touching Karpenter. 


PHASE 4: NodeClass & NodePool Design (Core Auto Mode Logic)
--------------------------------------------------------
This is where real engineering decisions come in.

You’ll design:
    NodeClass (AMI, subnet, security, IAM role)
    NodePool (capacity, instance types, spot/on-demand)
Example strategy:
    general-purpose
    compute-optimized
    spot-pool

NodePools are policies, not nodes.

Pod arrives
  ↓
Scheduler cannot place it
  ↓
AWS-managed Karpenter reads NodePool rules
  ↓
EC2NodeClass tells AWS how to build the instance
  ↓
Node appears
  ↓
Pod runs


PHASE 5: Validation Test (Very Important)
----------------------------------------

After NodePools are applied:
```
Test case
apiVersion: apps/v1
kind: Deployment
spec:
  replicas: 2
```
What should happen
    Pod goes into Pending
    Karpenter sees unscheduled pod
    EC2 instance is launched
    Node joins cluster
    Pod runs

This confirms:
✅ Auto Mode works
✅ IAM is correct
✅ Networking is correct
