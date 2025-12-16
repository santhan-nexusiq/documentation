# Understanding imagePullPolicy in kubernetes
imagePullPolicy tells Kubernetes whether to pull a fresh image from the registry or reuse an existing image already present on the node. 

**Kubernetes supports three values:**
    1. IfNotPresent
    2. Always
    3. Never  

## Difference Between IfNotPresent and Always: 

**IfNotPresent:** 
1. Kubernetes checks if the image already exists on the node(Each Kubernetes worker node maintains a local image cache via its container runtime)(Images are stored locally on the node’s disk)(The container runtime pulls images) 
2. If it exists → uses local image 
3. If it does NOT exist → pulls from registry

**Always:** 
1. Kubernetes always pulls the image from the registry
2. Even if the image already exists on the node 

## When to Use Each Option?
**Use `IfNotPresent` When:**
1. Production environments
2. Versioned images (v1.0.0, commit SHA, release tag) -- Image tag is immutable
3. No chance of image changing behind the same tag 
4. CI/CD pipelines that generate unique tags per build 

**Use `Always` When:**
1. Development/QA/Testing environments:  
2. When using mutable tags like latest -- The image content changes, The tag stays the same
Without Always, Kubernetes might reuse an old cached image and: New code won’t appear

**In most real projects:**
v22 = a specific release
Once pushed → image content should never change
Next change → new tag (v23)
If your team follows this (which most teams do), then:
v22 is immutable 

* `ifnotpresent` suits to our requirement because: 
1. Commit SHA tags are immutable by design  
2. With IfNotPresent:
    Node uses cached image
    Pods start immediately
    Less blast radius during infra events
3. Reduced ECR dependency during critical moments 

* Using `Always` with commit SHA tags means:
1. Pulling the same immutable image again and again
2. Slower pod starts
3. More ECR traffic
4. No functional benefit 