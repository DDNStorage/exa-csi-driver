# Lustre Client Module Deployment on Kubernetes

Deploy Lustre client modules using KMM (Kernel Module Management) with automatic builds.

## Prerequisites

1. **Namespace**: Create the `lustre-kmm` namespace before deploying any resources

   ```bash
   kubectl create namespace lustre-kmm
   ```

2. **KMM Operator**: Running in `kmm-operator-system` namespace
3. **Container Registry**: Push access to registry where built images will be stored
4. **REGISTRY_URL**: Private container registry URL (e.g., `harbor.company.com/lustre`, `quay.io/myorg`)
5. **Lustre Source**: Tarball on HTTP server
6. **Node Labels**: `os-version` label on worker nodes

```bash
kubectl label node <node-name> os-version=<os_version>
```

Where `<os_version>` is one of: `ubuntu-20.04`, `ubuntu-22.04`, or `ubuntu-24.04`

Example:

```bash
kubectl label node worker-1 os-version=ubuntu-22.04
```

### InfiniBand (o2ib only)

**CRITICAL**: IB interfaces need IP in **same subnet** as Lustre servers.

1. Install MOFED drivers (if needed)
2. Configure IB IP:

   ```bash
   sudo ip addr add 172.25.81.10/16 dev ib0
   sudo ip link set ib0 up
   ```

## Supported Distributions

| OS               | Kernels                                   | Network   |
| ---------------- | ----------------------------------------- | --------- |
| **Ubuntu 20.04** | 5.4.x (GA), 5.15.x (HWE)                  | TCP, o2ib |
| **Ubuntu 22.04** | 5.15.x (GA), 6.8.x (HWE)                  | TCP, o2ib |
| **Ubuntu 24.04** | 6.8.0-\*, 6.11.0-\*, 6.14.0-\*, 6.17.0-\* | TCP, o2ib |

## Quick Start

Replace `<os_version>` with one of: `ubuntu20`, `ubuntu22`, or `ubuntu24`.

### 1. Configure Placeholders

Update these in manifest files:

```bash
export REGISTRY_URL="<REGISTRY_URL>"
export LUSTRE_TARBALL_NAME="exa-client-ddn246.tar.gz"
export LUSTRE_SOURCE_SERVER="10.10.10.10:8000"
export IFACE_NAME="ens160"
export KERNEL_VERSION=$(uname -r)

# Replace placeholders in all manifests
sed -i "s|<REGISTRY_URL>|${REGISTRY_URL}|g" *.yaml
sed -i "s|<LUSTRE_TARBALL_NAME>|${LUSTRE_TARBALL_NAME}|g" lustre-<os_version>-dockerfile-configmap.yaml
sed -i "s|<LUSTRE_SOURCE_SERVER>|${LUSTRE_SOURCE_SERVER}|g" lustre-<os_version>-dockerfile-configmap.yaml
sed -i "s|<LUSTRE_SOURCE_SERVER>|${LUSTRE_SOURCE_SERVER}|g" lnet-mod-<os_version>.yaml
sed -i "s|<LUSTRE_SOURCE_SERVER>|${LUSTRE_SOURCE_SERVER}|g" lustre-client-mod-<os_version>.yaml
sed -i "s|<LUSTRE_TARBALL_NAME>|${LUSTRE_TARBALL_NAME}|g" lnet-mod-<os_version>.yaml
sed -i "s|<LUSTRE_TARBALL_NAME>|${LUSTRE_TARBALL_NAME}|g" lustre-client-mod-<os_version>.yaml
sed -i "s|<IFACE_NAME>|${IFACE_NAME}|g" lnet-configuration-tcp-<os_version>.yaml
sed -i "s|<IFACE_NAME>|${IFACE_NAME}|g" lnet-configuration-o2ib-<os_version>.yaml
sed -i "s|\${KERNEL_FULL_VERSION}|${KERNEL_VERSION}|g" lnet-configuration-tcp-<os_version>.yaml
```

**Configure cleanup DaemonSet (required for proper cleanup later):**

```bash
export OS="ubuntu20"  # Match your deployment: ubuntu20, ubuntu22, or ubuntu24
export OS_VERSION="ubuntu-20.04"  # Match: ubuntu-20.04, ubuntu-22.04, or ubuntu-24.04
export NODE_LABEL="lnet-ubuntu20"  # Match: lnet-ubuntu20, lnet-ubuntu22, or lnet-ubuntu24

sed -i "s|<OS>|${OS}|g" lustre-cleanup-ds.yaml
sed -i "s|<OS_VERSION>|${OS_VERSION}|g" lustre-cleanup-ds.yaml
sed -i "s|<NODE_LABEL>|${NODE_LABEL}|g" lustre-cleanup-ds.yaml
sed -i "s|<KERNEL_VERSION>|${KERNEL_VERSION}|g" lustre-cleanup-ds.yaml
# <REGISTRY_URL> already replaced by the command above
```

### 2. Deploy

```bash
# Stage 1: Module CRD (libcfs, lnet, ksocklnd)
kubectl apply -f lustre-<os_version>-dockerfile-configmap.yaml
kubectl apply -f lnet-mod-<os_version>.yaml

# Monitor build status (builds can take 15-60 min depending on resources)
# Check module status:
kubectl get module lnet-<os_version> -n lustre-kmm

# Watch build pod logs in real-time:
kubectl logs -n lustre-kmm -l kmm.node.kubernetes.io/module.name=lnet-<os_version> -f

# Check build pod status:
kubectl get pods -n lustre-kmm -l kmm.node.kubernetes.io/module.name=lnet-<os_version>

# Wait until module shows 'Ready':
# Expected output: lnet-<os_version>   Ready   <age>
# Once Ready, proceed to Stage 2

# Stage 2: DaemonSet (10 filesystem modules)
kubectl apply -f lnet-configuration-tcp-<os_version>.yaml
```

### 3. Monitor Deployment

Check module build status:

```bash
# View module status (should show 'Ready' when complete)
kubectl get module lnet-<os_version> -n lustre-kmm

# Watch build progress in real-time
kubectl logs -n lustre-kmm -l kmm.node.kubernetes.io/module.name=lnet-<os_version> -f

# Check if build pods are running/completed
kubectl get pods -n lustre-kmm -l kmm.node.kubernetes.io/module.name=lnet-<os_version>
```

### 4. Verify Module Loading

After Stage 2 (DaemonSet) deployment:

```bash
# Get DaemonSet pod name
POD=$(kubectl get pods -n lustre-kmm -l name=lnet-configuration-tcp-<os_version> -o jsonpath='{.items[0].metadata.name}')

# Verify Lustre modules are loaded
kubectl exec -n lustre-kmm $POD -- lsmod | grep -E "lnet|lustre"

# Check LNet network status
kubectl exec -n lustre-kmm $POD -- lnetctl net show
```

## Deployment by Network Type

### TCP

```bash
kubectl apply -f lustre-<os_version>-dockerfile-configmap.yaml
kubectl apply -f lnet-mod-<os_version>.yaml
kubectl apply -f lnet-configuration-tcp-<os_version>.yaml
```

**Note**: Update `NET_IFACE` in `lnet-configuration-tcp-<os_version>.yaml` to match your TCP interface (e.g., `eth0`, `ens33`, `bond0`).

### InfiniBand (o2ib)

```bash
kubectl apply -f lustre-<os_version>-dockerfile-configmap.yaml
kubectl apply -f lnet-mod-<os_version>.yaml
kubectl apply -f lnet-configuration-o2ib-<os_version>.yaml
```

**Note**: Update `NET_IFACE` in `lnet-configuration-o2ib-<os_version>.yaml` to match your IB interface (e.g., `ib0`, `ibs1f1`).

## Advanced: Optional Lustre Client Module Loading

**⚠️ Important:** This is an advanced pattern. For standard deployments, skip this section and use the Quick Start workflow above.

**Default behavior:** The `lnet-configuration-*` DaemonSets automatically load all 13 Lustre modules (base + filesystem modules) on deployment. **No action needed for standard deployments.**

**Use the optional `lustre-client-mod-*.yaml` files ONLY if:**

1. **Custom kernel mapping** - Your nodes have non-standard kernel versions requiring different module builds per kernel
2. **Granular node targeting** - You need to load Lustre modules on specific node subsets with different `nodeSelector` labels
3. **Pre-deployment testing** - You want to validate module builds before deploying the full DaemonSet configuration
4. **Advanced debugging** - Isolating module load failures from network configuration issues

**Available files:**
- `lustre-client-mod-ubuntu20.yaml`
- `lustre-client-mod-ubuntu22.yaml`
- `lustre-client-mod-ubuntu24.yaml`

**Deployment** (if one of the above scenarios applies):

```bash
kubectl apply -f lustre-client-mod-<os_version>.yaml

# Verify module CRD is Ready
kubectl get module -n lustre-kmm

# Then deploy the network configuration DaemonSet
kubectl apply -f lnet-configuration-tcp-<os_version>.yaml
```

## Cleanup

**Important**: Kernel modules must be properly unloaded before removing Kubernetes resources. The cleanup DaemonSet handles safe module unloading.

### Step 1: Configure Cleanup DaemonSet (if not already done during deployment)

```bash
export OS="ubuntu20"  # Match your deployment: ubuntu20, ubuntu22, or ubuntu24
export OS_VERSION="ubuntu-20.04"  # Match: ubuntu-20.04, ubuntu-22.04, or ubuntu-24.04
export NODE_LABEL="lnet-ubuntu20"  # Match: lnet-ubuntu20, lnet-ubuntu22, or lnet-ubuntu24
export KERNEL_VERSION="5.4.0-216-generic"  # Your kernel version from deployment

sed -i "s|<OS>|${OS}|g" lustre-cleanup-ds.yaml
sed -i "s|<OS_VERSION>|${OS_VERSION}|g" lustre-cleanup-ds.yaml
sed -i "s|<NODE_LABEL>|${NODE_LABEL}|g" lustre-cleanup-ds.yaml
sed -i "s|<KERNEL_VERSION>|${KERNEL_VERSION}|g" lustre-cleanup-ds.yaml
# <REGISTRY_URL> should already be configured from deployment
```

### Step 2: Deploy Cleanup DaemonSet to Unload Modules

```bash
kubectl apply -f lustre-cleanup-ds.yaml

# Wait for cleanup pods to complete module unloading (30-60 seconds)
kubectl get pods -n lustre-kmm -l name=lustre-cleanup

# Verify modules are being unloaded
kubectl logs -n lustre-kmm -l name=lustre-cleanup
```

### Step 3: Remove Kubernetes Resources

For TCP network:

```bash
# Remove cleanup DaemonSet
kubectl delete -f lustre-cleanup-ds.yaml

# Remove LNet configuration DaemonSet
kubectl delete -f lnet-configuration-tcp-<os_version>.yaml

# Remove module CRDs
kubectl delete -f lnet-mod-<os_version>.yaml

# Remove Dockerfile ConfigMap
kubectl delete -f lustre-<os_version>-dockerfile-configmap.yaml
```

For InfiniBand (o2ib) network:

```bash
# Remove cleanup DaemonSet
kubectl delete -f lustre-cleanup-ds.yaml

# Remove LNet configuration DaemonSet
kubectl delete -f lnet-configuration-o2ib-<os_version>.yaml

# Remove module CRDs
kubectl delete -f lnet-mod-<os_version>.yaml

# Remove Dockerfile ConfigMap
kubectl delete -f lustre-<os_version>-dockerfile-configmap.yaml
```

## Important: Kernel Version Changes

**Critical**: After a node kernel upgrade, downgrade, or reboot with kernel change, you must update the DaemonSet image reference before applying the configuration.

When the kernel version changes:

1. KMM automatically builds the required module image for the new kernel (if the image is unavailable in the registry or node containers)
2. The `image:` field (e.g., `image: "<REGISTRY_URL>/lustre-client-<os_version>:${KERNEL_FULL_VERSION}"`) in the configuration manifest contains a kernel-specific version tag that must be updated
3. **You must update the `${KERNEL_FULL_VERSION}` placeholder** to match the new kernel before applying the DaemonSet

**Example workflow after kernel change:**

**Note**: Only `${KERNEL_FULL_VERSION}` needs updating. Other placeholders (`<REGISTRY_URL>`, `<IFACE_NAME>`, etc.) were already configured in the previous step.

```bash
# Get the new kernel version
export KERNEL_VERSION=$(uname -r)

# Update only the required DaemonSet manifest with new kernel version
sed -i "s|\${KERNEL_FULL_VERSION}|${KERNEL_VERSION}|g" lnet-configuration-tcp-<os_version>.yaml
# (or lnet-configuration-o2ib-<os_version>.yaml if using InfiniBand)

# Then apply the updated DaemonSet
kubectl apply -f lnet-configuration-tcp-<os_version>.yaml
```

**Why this is necessary**: The DaemonSet references a specific container image built for a particular kernel version. If the kernel version changes but the image tag doesn't, the DaemonSet will attempt to use an image built for the wrong kernel, causing module load failures.

## Troubleshooting

### Monitoring Build Progress

Check build status at any time:

```bash
# View module status summary
kubectl get module -n lustre-kmm

# Detailed module information
kubectl describe module lnet-<os_version> -n lustre-kmm

# Watch build pod logs (real-time)
kubectl logs -n lustre-kmm -l kmm.node.kubernetes.io/module.name=lnet-<os_version> -f

# Check all KMM-related pods
kubectl get pods -n lustre-kmm --show-labels

# View events for troubleshooting
kubectl get events -n lustre-kmm --sort-by='.lastTimestamp'
```

**Build Stages:**
1. **Pending**: Module CRD created, waiting for KMM operator
2. **Build**: Pod building kernel modules (15-60 min)
3. **InProgress**: Modules built, deploying to nodes
4. **Ready**: Modules loaded on all matching nodes

### Build Failures

#### Error: "No package kernel-devel available"

```bash
# Verify HTTP server accessible from cluster
kubectl run -it --rm test --image=curlimages/curl -- curl <LUSTRE_SOURCE_URL>

# Check if tarball URL is correct in ConfigMap
kubectl get configmap lustre-<os_version>-dockerfile -n lustre-kmm -o yaml | grep LUSTRE_SOURCE_URL
```

#### Build pod stuck in "ImagePullBackOff"

```bash
# Check build pod status
kubectl describe pod -n lustre-kmm -l kmm.node.kubernetes.io/module.name=lnet-<os_version>

# Verify registry URL is accessible
kubectl run -it --rm test --image=curlimages/curl -- curl -I <REGISTRY_URL>
```

#### Build pod shows "CrashLoopBackOff"

```bash
# View build logs for errors
kubectl logs -n lustre-kmm -l kmm.node.kubernetes.io/module.name=lnet-<os_version> --previous

# Common issues:
# - Incorrect LUSTRE_TARBALL_NAME in ConfigMap
# - Missing kernel-devel package for target kernel
# - Insufficient resources (increase memory/CPU limits)
```

### Module Load Failures

#### Error: "Unknown symbol in module"

```bash
# Kernel mismatch - modules built for wrong kernel version
kubectl logs -n lustre-kmm -l name=lnet-configuration-tcp-<os_version>

# Verify kernel version matches build
kubectl get nodes -o custom-columns=NAME:.metadata.name,KERNEL:.status.nodeInfo.kernelVersion

# Rebuild modules for correct kernel
kubectl delete module lnet-<os_version> -n lustre-kmm
# Update kernel version in manifests, then reapply
```

#### Modules not loading after DaemonSet deployment

```bash
# Check DaemonSet status
kubectl get daemonset -n lustre-kmm

# View DaemonSet pod logs
kubectl logs -n lustre-kmm -l name=lnet-configuration-tcp-<os_version>

# Verify Stage 1 (KMM Module) completed successfully
kubectl get module lnet-<os_version> -n lustre-kmm
# Must show 'Ready' before deploying Stage 2
```

### Network Configuration

#### LNet not showing interface

```bash
# Verify interface exists on node
kubectl debug node/<node-name> -- chroot /host ip addr show <NET_IFACE>

# Check LNet configuration
POD=$(kubectl get pods -n lustre-kmm -l name=lnet-configuration-tcp-<os_version> -o jsonpath='{.items[0].metadata.name}')
kubectl exec -n lustre-kmm $POD -- lnetctl net show
```
