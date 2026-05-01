# Lustre Client with MOFED/InfiniBand Support on OpenShift

This guide explains how to deploy Lustre client modules with MOFED (Mellanox OFED) support for InfiniBand connectivity on OpenShift.

## Prerequisites

Before deploying Lustre client support, ensure the following are configured:

### 1. NVIDIA Network Operator
- Installed and configured with MOFED drivers
- MOFED driver pods running on all worker nodes
- Verify: `oc get pods -n nvidia-network-operator`

### 2. InfiniBand Network Configuration
**CRITICAL:** InfiniBand interfaces must be configured with IP addresses in the same subnet as your Lustre servers.

- **Interface name**: Identify your IB interface (e.g., `ibs1f1`, `ibP1p1s0f1`)
- **IP subnet**: Must match Lustre MGS/MDS/OSS network subnet
- **Configuration methods**:
  - DHCP on InfiniBand network (recommended for production)
  - nmstate / NetworkManager
  - Manual IP assignment
  - Network Operator IPoIB CNI

**Verification:**
```bash
# Check interface has IP in correct subnet
oc debug node/<node-name> -- chroot /host ip addr show <ib-interface>
# Example: inet 172.25.81.10/16 scope global ibs1f1
```

### 3. Lustre Server Configuration
- **Nodemap access**: Ensure client IP range is added to Lustre server's nodemap
  ```bash
  # On MGS server:
  lctl nodemap_add_range --name <nodemap> --range <client-ip-range>
  # Example: lctl nodemap_add_range --name exa_servers --range 172.25.81.[10-250]@o2ib
  ```
- Verify Lustre filesystem is accessible from client subnet

### 4. Kernel Module Management (KMM)
- KMM operator installed in `openshift-kmm` namespace
- Verify: `oc get pods -n openshift-kmm`

## Architecture

The deployment uses a multi-stage build approach:

1. **Stage 1a**: Extract MOFED source from NVIDIA DOCA driver image
2. **Stage 1b**: Build MOFED 25.10 against target kernel using builder-base (with entitlements)
3. **Stage 2**: Build Lustre 2.14.0 with o2ib support against the compiled MOFED

This ensures symbol CRCs match between Lustre's `ko2iblnd` module and the MOFED drivers running on nodes.

## Quick Start

Edit lustre-dockerfile-mofed-configmap.yaml with the correct doca driver version

```bash
oc get pod -n nvidia-network-operator mofed-rhel9.6-6974f4b749-ds-5qfqt -oyaml | grep doca3

```


Create the ConfigMap and build the modules

```bash
# 1. Build Lustre client image
oc create -n openshift-kmm -f lustre-dockerfile-mofed-configmap.yaml
oc apply -n openshift-kmm -f lnet-mod-mofed.yaml

# 2. Deploy kernel modules
oc apply -n openshift-kmm -f ko2iblnd-mod-mofed.yaml

# 3. Edit and deploy LNet configuration (UPDATE NET_IFACE!)
vi lnet-lustre-configuration-ds-mofed.yaml
oc apply -n openshift-kmm -f lnet-lustre-configuration-ds-mofed.yaml

# 4. Verify
oc get pods -n openshift-kmm
POD=$(oc get pods -n openshift-kmm -l name=lnet-configuration -o jsonpath='{.items[0].metadata.name}')
oc exec -n openshift-kmm $POD -- lnetctl net show
```

## Detailed Deployment Steps

### Step 1: Build the Lustre Client Image and load Lnet Module

```bash
# Create the Dockerfile ConfigMap
oc create -n openshift-kmm -f lustre-dockerfile-mofed-configmap.yaml

# Deploy the lnet module (triggers the build)
oc apply -n openshift-kmm -f lnet-mod-mofed.yaml

# Watch the build progress
oc logs -n openshift-kmm -f $(oc get pods -n openshift-kmm | grep lnet-build | awk '{print $1}')
```

The build takes approximately 20-30 minutes as it compiles both MOFED and Lustre from source.

### Step 2: Deploy o2iblnd Kernel Module

Deploy the kernel modules for LNet and InfiniBand support:

```bash
# Deploy ko2iblnd module (InfiniBand LNet support)
oc apply -n openshift-kmm -f ko2iblnd-mod-mofed.yaml
```

Wait for modules to load on all nodes:

```bash
# Check module status
oc get module -n openshift-kmm
```

### Step 3: Verify Kernel Modules

Once the build completes, verify the modules are loaded:

```bash
# Check module status
oc get module lnet -n openshift-kmm

# Verify on a worker node
oc debug node/<node-name> -- chroot /host lsmod | grep -E "lnet|o2iblnd"
```

Expected output:
```
ko2iblnd              266240  0
ksocklnd              200704  0
lnet                  724992  2 ko2iblnd,ksocklnd
libcfs                589824  3 lnet,ko2iblnd,ksocklnd
```

### Step 4: Configure LNet Network Settings

Edit `lnet-lustre-configuration-ds-mofed.yaml` to specify your InfiniBand interface:

```bash
# Edit the DaemonSet configuration
vi lnet-lustre-configuration-ds-mofed.yaml
```

Update the environment variables:

```yaml
env:
  - name: NET_TYPE
    value: "o2ib"           # Network type (o2ib for InfiniBand)
  - name: NET_IFACE
    value: "ibs1f1"         # YOUR InfiniBand interface name
```

**Important:**
- Replace `ibs1f1` with your actual InfiniBand interface name
- The interface must already have an IP address configured (see Prerequisites)
- The IP must be in the same subnet as your Lustre servers

### Step 5: Deploy LNet Configuration DaemonSet

```bash
oc apply -n openshift-kmm -f lnet-lustre-configuration-ds-mofed.yaml
```

This DaemonSet runs on all worker nodes and:
1. Configures LNet with the specified network type
2. Adds the o2ib network using the configured interface
3. Loads Lustre client modules (lustre, mgc, mdc, etc.)
4. Runs `depmod` to ensure modules are discoverable

### Step 6: Verify LNet Configuration

```bash
# Check DaemonSet status
oc get ds lnet-configuration -n openshift-kmm

# Check init container logs (configuration happens here)
POD=$(oc get pods -n openshift-kmm -l name=lnet-configuration -o jsonpath='{.items[0].metadata.name}')
oc logs -n openshift-kmm $POD -c configure-lnet

# Verify LNet is configured
oc exec -n openshift-kmm $POD -- lnetctl net show
```

Expected output showing LNet configured with your InfiniBand network:
```
net:
    - net type: lo
      local NI(s):
        - nid: 0@lo
          status: up
    - net type: o2ib
      local NI(s):
        - nid: 172.25.81.11@o2ib
          status: up
          interfaces:
              0: ibs1f1
```

Verify all Lustre modules are loaded:
```bash
oc exec -n openshift-kmm $POD -- lsmod | grep -E "lustre|mgc|lnet"
```

Expected modules:
```
mgc                   110592  0
lustre               1241088  0
mdc                   315392  1 lustre
lov                   401408  2 mdc,lustre
lmv                   241664  1 lustre
lnet                  724992  9 osc,ko2iblnd,obdclass,ptlrpc,mgc,ksocklnd,lmv,lustre
```
