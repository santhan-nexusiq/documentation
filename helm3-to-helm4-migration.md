# HELM-3 TO HELM-4 MIGRATION 

**Helm 4 Overview:** 
* Helm v4 represents a significant evolution from v3, introducing breaking changes, new architectural patterns, and enhanced functionality while maintaining backwards compatibility for charts  

## New Features

### WebAssembly-Based Plugin System
* Helm plugins are add-on tools that extend the core features and functionality of the Helm command-line interface (CLI) 
* Helm 3 plugins are native executables (Linux ELF, macOS Mach-O, Windows PE) with unrestricted system access. A malicious plugin can access your kubeconfig, arbitrary file systems, environment variables. 
* Helm 4 introduces an optional WebAssembly-based runtime(WASM) for enhanced security and expanded capabilities. WebAssembly (WASM) modules running in sandboxed runtime with explicit capabilities, so that they cannot access kubeconfig, arbitrary file systems, environment variables. 
* Helm 4 launches with three plugin types: CLI plugins, getter plugins, and post-renderer plugins, plus a system that enables new plugin types for customizing additional core functionality  

### kstatus Integration (Accurate Deployment Monitoring) 
* Helm 3's biggest UX flaw: marking deployments "successful" before pods actually run:

```
# Helm 3
helm install myapp ./chart
# Output: "Release myapp installed successfully"
# Reality: Pods in CrashLoopBackOff

kubectl get pods
NAME                    READY   STATUS             RESTARTS   AGE
myapp-5d7c8f9-abcde     0/1     CrashLoopBackOff   5          2m
```

Helm 4 with kstatus:

```# Helm 4 waits for actual readiness
helm install myapp ./chart --wait

# Helm monitors:
# - Deployments: All replicas available
# - StatefulSets: All pods ready in order
# - DaemonSets: Pods running on all nodes
# - Jobs: Completed successfully
# - Services: Endpoints available

# Real-time status (similar to kubectl rollout status)
Installing myapp...
→ Deployment myapp-api: 0/3 replicas ready
→ Deployment myapp-api: 1/3 replicas ready
→ Deployment myapp-api: 2/3 replicas ready
→ Deployment myapp-api: 3/3 replicas ready ✓
→ Service myapp-api: Endpoints ready ✓
Release myapp installed successfully
```

Failure Detection:

```
# If pods fail to start, Helm 4 detects and reports
helm install myapp ./chart --wait --timeout 5m

# Output:
Error: release myapp failed: 
  Deployment myapp-api: 0/3 replicas ready after 5m
  Pod myapp-api-abc123: CrashLoopBackOff
    Container api: Error: ImagePullBackOff
    Image: registry.example.com/myapp:v1.0.0 not found

# Helm automatically rolls back (with --rollback-on-failure)
``` 

### Enhanced OCI Support
* An OCI Registry (Open Container Initiative Registry) is a standardized, secure location (like Docker Hub, AWS ECR) that stores various cloud-native artifacts, including container images and Helm charts, using the same distribution model. 
* Helm 3.8+ introduced OCI registry support (experimental). Helm 4 makes OCI production-ready  

```
# Helm 3: Install by tag (mutable, tag can change)
helm install myapp oci://registry.example.com/charts/myapp:1.0.0

# Helm 4: Install by SHA256 digest (immutable)
helm install myapp oci://registry.example.com/charts/myapp@sha256:abc123...

# Guarantees exact chart content (prevents supply chain attacks)
# If chart modified, digest changes (installation fails)
```

### Multi-Document Values
* Split complex values across multiple YAML files. Perfect for testing different environment configs. 
* Helm 3 forces single values.yaml:

```
# Helm 3: One massive values.yaml (hard to maintain)
# values.yaml
global:
  environment: production
database:
  host: prod-db.example.com
  replicas: 3
  resources:
    requests:
      cpu: 2000m
      memory: 8Gi
redis:
  enabled: true
  sentinel: true
monitoring:
  prometheus: true
  grafana: true
# ... 500 more lines ...
```

Helm 4: Split values across multiple files:

```
# values-global.yaml
global:
  environment: production

# values-database.yaml
database:
  host: prod-db.example.com
  replicas: 3

# values-monitoring.yaml
monitoring:
  prometheus: true

# Helm 4: Merge multiple values files
helm install myapp ./chart \
  -f values-global.yaml \
  -f values-database.yaml \
  -f values-monitoring.yaml

# Deep merge: Later files override earlier ones
# Easier maintenance: Separate concerns by domain
```

### Server-Side Apply Integration
* Better conflict resolution when multiple tools manage the same resources. Kubernetes 1.22+ introduced server-side apply (SSA) where API server tracks field ownership solving multi-tool conflicts:
    * Helm and Argo CD/Flux coexist without conflicts
    * Clear ownership: Helm manages image tags, Argo manages replicas/autoscaling
    * Audit trail: `kubectl get deployment -o yaml` shows which tool modified each field 

```
# Helm 3 (client-side apply, default)
helm upgrade myapp ./chart

# Helm 4 (opt-in server-side apply)
helm upgrade myapp ./chart --server-side

# Future: Server-side apply will become default in Helm 4.1
```

### Custom Template Functions
* Extend Helm's templating with your own functions through plugins. Great for organization-specific templating needs. 


## Breaking Changes & Deprecations
### CLI Flag Renaming**
Helm 4 renames common flags for clarity:
`--atomic` --> `--rollback-on-failure`
`--force` --> `--force-replace`

**Migration Action:**
```
# Helm 3 commands
helm upgrade myapp ./chart --atomic
helm upgrade myapp ./chart --force

# Helm 4 equivalents
helm upgrade myapp ./chart --rollback-on-failure
helm upgrade myapp ./chart --force-replace
```

### Plugin System Overhaul 
* While existing native plugins work, Helm 4 deprecates some plugin hook behaviors:

    * Removed: `install` and `delete` hooks (use `pre-install`, `post-install`) 
    * Changed: Environment variable passing to plugins (use plugin.yaml config) 

### Chart API Version (No Change)
* Helm 4 continues using Chart API v2 (same as Helm 3)
* All Helm 3 charts work in Helm 4 without modification

## Helm 4 Migration Guide (Step-by-Step) 

### Prerequisites
* Kubernetes Version: 1.27+ (1.24-1.26 supported with caveats)
* Helm 3 Version: Upgrade to Helm 3.14+ before migrating to Helm 4
* Chart API Version: Charts must use apiVersion: v2 (Helm 2 charts not supported)
* Kubectl Access: Verify kubectl context configured correctly

### Step 1: Install Helm 4 CLI (Parallel to Helm 3)

```
# Download Helm 4 (does not overwrite Helm 3)
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-4 | bash

# Verify installation
helm version
# Output: version.BuildInfo{Version:"v4.0.0", ...}

# Check Helm 3 still accessible (if needed)
helm3 version  # Helm 4 installer creates "helm3" symlink

# List existing releases (Helm 4 reads Helm 3 release history)
helm list --all-namespaces
```
### Step 2: Audit Existing Charts and Plugins
```
# List installed plugins
helm plugin list

# Check for plugins needing updates:
# - Post-renderer plugins: Convert to plugin name format
# - Native plugins with security concerns: Migrate to WASM

# Scan charts for deprecated patterns
helm lint ./mychart

# Check CI/CD scripts for deprecated flags
grep -r "--atomic" .github/workflows/
grep -r "--force" .gitlab-ci.yml
```

### Step 3: Update CLI Flags in Automation
```
# Example: Update GitHub Actions workflow
# .github/workflows/deploy.yml

# BEFORE (Helm 3)
- name: Deploy to Staging
  run: |
    helm upgrade myapp ./chart \
      --install \
      --atomic \
      --timeout 5m

# AFTER (Helm 4)
- name: Deploy to Staging
  run: |
    helm upgrade myapp ./chart \
      --install \
      --rollback-on-failure \
      --timeout 5m
```

### Step 4: Test in Staging Environment
```
# Deploy test application with Helm 4
helm install test-app ./chart \
  --namespace staging \
  --wait \
  --timeout 10m

# Verify kstatus monitoring
helm status test-app -n staging

# Test upgrade workflow
helm upgrade test-app ./chart \
  --set image.tag=v2.0.0 \
  --rollback-on-failure

# Test rollback
helm rollback test-app 1 -n staging

# Validate server-side apply (if enabled)
kubectl get deployment test-app -n staging -o yaml | grep managedFields
```

### Step 5: Enable New Helm 4 Features Progressively 
```
# Enable server-side apply for non-critical apps
helm upgrade myapp ./chart --server-side

# Migrate to OCI charts with digest-based installs
helm upgrade myapp oci://registry.example.com/charts/myapp@sha256:abc123...

# Adopt WebAssembly plugins
helm plugin install oci://registry.example.com/plugins/secure-plugin:1.0.0

# Implement multi-document values
helm upgrade myapp ./chart -f values-base.yaml -f values-prod.yaml
```

