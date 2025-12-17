# UNDERSTANDING NODE POOLS(a.k.a. Node Groups) IN K8S 
The nodepool is a group of nodes that share the same configuration (CPU, Memory, Networking, OS, maximum number of pods, etc.). By default, one single (system) nodepool is created within the cluster. However, we can add nodepools during or after cluster creation. We can also remove these nodepools at any time 

There are 2 types of nodepools:
1. System nodepool: used to preferably deploy system pods.
2. User nodepool: used to preferably deploy application pods. 

## Why Node Pools Exist (Real Problem They Solve)
In real systems, not all workloads are the same.

Examples:
* Frontend → low CPU, moderate memory
* Backend APIs → balanced CPU/memory
* Jobs / batch → high CPU
* ML / media → GPU
* System pods → stable, critical

Running all of these on identical nodes is:
* Inefficient
* Expensive
* Risky

👉 Node pools allow us to separate workloads at infrastructure level.

## Kubernetes view
* Kubernetes itself does not create node pools.
* Kubernetes only sees nodes
* Node pools are created by:
    * Cloud providers (EKS, AKS, GKE)
    * Or cluster tools (eksctl, Terraform) 

## How Node Pools Are Created in EKS
There are 3 real, supported ways:
1. eksctl (most common, simplest)
2. Terraform / IaC (production standard)
3. AWS Console / CLI (manual, not preferred)  

## Creating a Node Pool Using eksctl (Most Common)
Create a new node pool
```
eksctl create nodegroup \
  --cluster my-eks-cluster \
  --name general-pool \
  --node-type t3.medium \
  --nodes 2 \
  --nodes-min 2 \
  --nodes-max 5 \
  --node-labels workload=cpu \
  --managed
``` 

Now nodes will have labels like:
```
workload=cpu
```

View nodes:
```
kubectl get nodes
```

See labels
```
kubectl get nodes --show-labels
``` 

Use it in Deployment: 
```
nodeSelector:
  workload: cpu
``` 
Pods now run only on this node pool. 

### kubectl Command to List Nodes in a Specific Node Pool
```
kubectl get nodes -l eks.amazonaws.com/nodegroup=<nodegroup-name> 
```

### If You’re Not Sure About the Node Pool Name
List all node pools
```
kubectl get nodes --show-labels
``` 
You’ll see something like:
```
eks.amazonaws.com/nodegroup=general-pool
eks.amazonaws.com/nodegroup=cpu-pool
``` 

### Custom Labels (If Your Team Added Them)
If your team labeled node pools like this:
```
workload=cpu
``` 
Then you can filter using that label:
```
kubectl get nodes -l workload=cpu
```  

## our requirement: moving image proctoring and question generator pods to particular nodepool

## What Is “Node Class” in EKS Auto Mode?
In EKS Auto Mode, a Node Class is:
    A template that defines how EKS should create and manage nodes 

You should CREATE A NEW NODE CLASS: Do NOT reuse default 

```
EKS Auto Mode
 ├── Node Class: default
 │    └── small workloads
 │
 └── Node Class: ai-ml
      └── heavy AI/ML workloads
``` 

**Node Class = how nodes are created**
**Node Pool / Group = how workloads are isolated** 

## steps to create new Node Class(AWS console)
1. Go to:
```
AWS Console → EKS → Your Cluster → Node configuration → Node classes
``` 
2. Click Create node class
3. Configure:
* Node class name:
    `ai-ml-node-class`
* Instance type:
    `m6a.xlarge` or `c6a.large` 
* IAM role:
    `AmazonEKSAutoNodeRole (reuse existing)`
* Networking / AMI: keep defaults (recommended) 

4. Create the node class 

## Update AI/ML Pod Specification
In only the AI/ML workloads, update the pod template
```
spec:
  template:
    spec:
      nodeSelector:
        eks.amazonaws.com/node-class: ai-ml-node-class
```

## EKS Auto Mode Provisions New Nodes Automatically
**What Happens Internally** 
* Scheduler sees pending pods
* EKS Auto Mode detects:
    * Required node class = ai-ml-node-class
* AWS automatically:
    * Launches new m6a.xlarge EC2 instances
    * Applies the node class configuration
    * Joins nodes to the cluster

Result
1. New nodes appear in kubectl get nodes
2. Nodes belong to ai-ml-node-class
3. AI/ML pods get scheduled on them

## Move Existing AI/ML Pods
Pods that were already running will not move automatically.You must recreate them safely. 
```
kubectl rollout restart deployment <ai-ml-deployment-name> 
``` 
* This ensures:
    * Old pods terminate
    * New pods are created
    * New pods land on AI/ML node class nodes

## Observe Cluster Rebalancing
* What You Will See
    * AI/ML pods → running on new m6a.xlarge nodes
    * Small pods → still running on default Auto Mode nodes
    * Old nodes may now be:
        * Empty
        * Underutilized 

## EKS Auto Mode Scales Down Unused Nodes
* This Happens Automatically If:
    * Nodes are empty or underutilized
    * Pods can be rescheduled elsewhere

## final steady state
```
Node Class: default
 └─ Small / lightweight pods
    (Auto Mode managed, auto scale up/down)

Node Class: ai-ml-node-class
 └─ AI/ML heavy pods
    (Dedicated m6a.xlarge nodes)
``` 

# Understanding Node IAM Role in EKS Auto Mode 

When you create an EKS cluster with Auto Mode enabled, AWS automatically:
* Creates a Node IAM Role
* Attaches the required AWS-managed policies
* Associates this role with the default node class

we can reuse `AmazonEKSAutoNodeRole` when we create the new node class 
**Why reusing is best**  
* Same cluster 
* Same trust boundary
* No special AWS service access required (yet)
* Keeps configuration simple
* AWS officially supports this 

but if we want to create a custom IAM role for the new node class, we need to attach following policies
1. Trust Policy
```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ec2.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
``` 
2. AWS Managed Policies
* AmazonEKSWorkerNodePolicy --> Allows node to talk to EKS control plane
* AmazonEKS_CNI_Policy --> Allows pod networking (ENI, IPs)
* AmazonEC2ContainerRegistryReadOnly --> Allows pulling images from ECR 

When creating your new node class:
* IAM Role field → choose:
    * AmazonEKSAutoNodeRole (recommended now)
    * or your custom role (if created)
* EKS Auto Mode will then:
* Launch EC2 instances
* Attach that IAM role
* Nodes assume the role automatically 





**Note:**
* existing nodes are NOT automatically added to a newly created node pool / node group, even if they are the same instance type `(m6a.xlarge).`

**EKS + eksctl do ALL of this automatically:** 
1. Create a Managed Node Group
2. Create an Auto Scaling Group
3. Set instance type = m6a.xlarge
4. Launch EC2 instances of type m6a.xlarge
5. Install:
    * container runtime (containerd)
    * kubelet
    * CNI 
6. Join nodes to the cluster
7. Apply labels & taints (if provided) 

👉 Result:
All nodes in that node pool are m6a.xlarge
Scaling up/down adds/removes only m6a.xlarge nodes 

* once AI/ML pods are moved out, any Auto Mode nodes that become empty or underutilized will be automatically scaled down by EKS after the stabilization period  

