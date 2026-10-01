# Lustre client modules on OpenShift (MOFED / InfiniBand)

Kernel Module Management (KMM) Module `lnet` loads `libcfs`, `lnet`, and
`ksocklnd`. Module `ko2iblnd` loads `ko2iblnd`. DaemonSet
`lnet-configuration` then configures LNet NIDs and loads `ptlrpc`,
`lustre`, and `mgc` from the same moduleloader image (`modprobe -d /opt`).

In-cluster builds compile MOFED and Lustre against `${DTK_AUTO}` (the
Driver Toolkit for the node's kernel) so `ko2iblnd` symbol CRCs match the
MOFED drivers on the node. Image `builder-base` supplies only the Lustre
tarball and RHSM entitlements. Rebuild `builder-base` when the tarball or
entitlements change, or when `builder-base:latest` is missing from the
in-cluster registry.

KMM tags and pushes the moduleloader to
`image-registry.openshift-image-registry.svc:5000/openshift-kmm/lustre-client-moduleloader:${KERNEL_FULL_VERSION}-mofed`.
The DaemonSet image tag must match that string. Modules have no
`spec.imageRepoSecret`.

For TCP, see [README.md](README.md).
To upgrade an existing deployment and roll workers onto a new kernel,
see [upgrade/README.md](upgrade/README.md). Use the MOFED ConfigMap,
Module, and DaemonSet files listed in that document.

## Prerequisites

- NVIDIA Network Operator installed, with MOFED driver pods running on
  every worker
- InfiniBand interface on each worker with an IP in the same subnet as
  the Lustre servers (configure IP before this DaemonSet; this procedure
  does not assign addresses)
- Client IP range present in the Lustre server nodemap
- KMM operator in namespace `openshift-kmm`
- Node Feature Discovery (NFD), so each worker has
  `feature.node.kubernetes.io/kernel-version.full`
- Lustre client tarball `lustre-*.tar.gz` and RHSM entitlement files in
  this directory

```bash
oc get pods -n nvidia-network-operator
oc get pods -n openshift-kmm
oc debug node/<node-name> -- chroot /host ip addr show <ib-interface>
ls deploy/openshift/lustre-module/Dockerfile \
   deploy/openshift/lustre-module/entitlement.pem \
   deploy/openshift/lustre-module/entitlement-key.pem \
   deploy/openshift/lustre-module/lustre-*.tar.gz
```

If the entitlement files are missing:

```bash
oc get secret etc-pki-entitlement -n openshift-config-managed -o json \
  | jq -r '.data["entitlement.pem"]' | base64 -d \
  > deploy/openshift/lustre-module/entitlement.pem
oc get secret etc-pki-entitlement -n openshift-config-managed -o json \
  | jq -r '.data["entitlement-key.pem"]' | base64 -d \
  > deploy/openshift/lustre-module/entitlement-key.pem
```

On the MGS, add the client range to the nodemap if it is not already
present:

```bash
lctl nodemap_add_range --name <nodemap> --range <client-ip-range>
```

## Deploy

Run the following from the repository root (`exascaler-csi-file-driver`).

### 1. Match the DOCA / MOFED versions

Set `DOCA_IMAGE_TAG` and `MOFED_VERSION` in `lnet-mod-mofed.yaml`
before applying the ConfigMap.

```bash
POD=$(oc get pods -n nvidia-network-operator -o name | grep mofed | head -1)

# DOCA_IMAGE_TAG: last path component of the image
oc get -n nvidia-network-operator "$POD" \
  -o jsonpath='{.spec.containers[*].image}{"\n"}'

# MOFED_VERSION: strip the MLNX_OFED_LINUX- prefix and trailing colon
oc exec -n nvidia-network-operator "$POD" -- ofed_info -s
```

### 2. Build `builder-base`

```bash
oc get bc -n openshift-kmm builder-base || \
  oc new-build -n openshift-kmm --binary --name=builder-base --strategy=docker

oc start-build -n openshift-kmm builder-base \
  --from-dir=deploy/openshift/lustre-module/ --follow
```

Confirm the image pulls:

```bash
oc run pull-builder -n openshift-kmm --restart=Never --rm -it \
  --image=image-registry.openshift-image-registry.svc:5000/openshift-kmm/builder-base:latest \
  --overrides='{"spec":{"serviceAccountName":"kmm-operator-module-loader"}}' \
  --command -- echo ok
```

The command must print `ok`.

### 3. Apply the KMM Dockerfile ConfigMap

```bash
oc apply -n openshift-kmm -f deploy/openshift/lustre-module/lustre-dockerfile-mofed-configmap.yaml
```

The first MOFED + Lustre build typically takes 20–30 minutes.

### 4. In-cluster registry

KMM pushes the moduleloader to the OpenShift internal registry. Modules
have no `spec.imageRepoSecret`.

Set `kubernetes.io/hostname` in
`image-registry-storage.yaml` to the worker that will hold
`/var/lib/registry`, then:

```bash
oc apply -f deploy/openshift/lustre-module/image-registry-storage.yaml
oc debug node/<registry-worker> -- chroot /host \
  chown -R 1000310000:1000310000 /var/lib/registry
oc patch configs.imageregistry.operator.openshift.io cluster --type=merge -p '{
  "spec": {
    "replicas": 1,
    "rolloutStrategy": "Recreate",
    "nodeSelector": {
      "kubernetes.io/hostname": "<registry-worker>",
      "kubernetes.io/os": "linux"
    },
    "storage": {
      "pvc": { "claim": "image-registry-storage" }
    }
  }
}'
oc get pvc image-registry-storage -n openshift-image-registry
```

### 5. Apply Modules `lnet` and `ko2iblnd`

If `cpu_npartitions` is required, set it in `lnet-mod-mofed.yaml` to a
value no greater than the node's vCPU count. Module `ko2iblnd` selects
`network.nvidia.com/operator.mofed.wait: "false"` so KMM loads it after
NVIDIA MOFED (`mlx_compat` / `ib_core`).

```bash
oc apply -n openshift-kmm -f deploy/openshift/lustre-module/lnet-mod-mofed.yaml
oc apply -n openshift-kmm -f deploy/openshift/lustre-module/ko2iblnd-mod-mofed.yaml
oc get module -n openshift-kmm
```

Confirm the LNet stack on a worker:

```bash
oc debug node/<node-name> -- chroot /host lsmod | grep -E "libcfs|lnet|ksocklnd|ko2iblnd"
```

### 6. Edit and apply DaemonSet `lnet-configuration`

In `lnet-lustre-configuration-ds-mofed.yaml`:

- `&lustre-image`: `…/lustre-client-moduleloader:<uname -r>-mofed`
- `feature.node.kubernetes.io/kernel-version.full`: worker `uname -r`
- `kmm.node.kubernetes.io/openshift-kmm.lnet.ready`
- `kmm.node.kubernetes.io/openshift-kmm.ko2iblnd.ready`
- `network.nvidia.com/operator.mofed.wait: "false"`
- `NET_TYPE`: `o2ib`
- `NET_IFACE`: InfiniBand interface that already has an IP

```bash
oc apply -n openshift-kmm -f deploy/openshift/lustre-module/lnet-lustre-configuration-ds-mofed.yaml
oc rollout status -n openshift-kmm ds/lnet-configuration
```

## Verify

```bash
oc get ds -n openshift-kmm lnet-configuration

POD=$(oc get pods -n openshift-kmm -l name=lnet-configuration \
  -o jsonpath='{.items[0].metadata.name}')
oc logs -n openshift-kmm "$POD" -c configure-lnet
oc exec -n openshift-kmm "$POD" -- lnetctl net show
oc debug node/<node-name> -- chroot /host lsmod | grep -E "lnet|ko2iblnd|ptlrpc|lustre|mgc"
```

`lnetctl net show` must list the InfiniBand NID as `up`. The CSI node
DaemonSet waits until `lustre` appears in `/proc/modules`.
