# Lustre client modules on OpenShift (TCP)

Kernel Module Management (KMM) Module `lnet` loads `libcfs`, `lnet`, and
`ksocklnd`. DaemonSet `lnet-configuration` then configures LNet NIDs and
loads `ptlrpc`, `lustre`, and `mgc` from the same moduleloader image
(`modprobe -d /opt`).

In-cluster builds compile against `${DTK_AUTO}` (the Driver Toolkit for
the node's kernel). Image `builder-base` supplies only the Lustre tarball
and RHSM entitlements. Rebuild `builder-base` when the tarball or
entitlements change, or when `builder-base:latest` is missing from the
in-cluster registry. Modules have no `spec.imageRepoSecret`.

For InfiniBand / MOFED, see [README-MOFED.md](README-MOFED.md).
To upgrade an existing deployment and roll workers onto a new kernel,
see [upgrade/README.md](upgrade/README.md).

## Prerequisites

- KMM operator in namespace `openshift-kmm`
- Node Feature Discovery (NFD), so each worker has
  `feature.node.kubernetes.io/kernel-version.full`
- Lustre client tarball `lustre-*.tar.gz` in this directory
- RHSM entitlement files `entitlement.pem` and `entitlement-key.pem` in
  this directory

```bash
oc get pods -n openshift-kmm
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

## Deploy

Run the following from the repository root (`exascaler-csi-file-driver`).

### 1. Build `builder-base`

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

### 2. Apply the KMM Dockerfile ConfigMap

```bash
oc apply -n openshift-kmm -f deploy/openshift/lustre-module/lustre-dockerfile-configmap.yaml
```

### 3. In-cluster registry

KMM pushes the moduleloader to
`image-registry.openshift-image-registry.svc:5000/openshift-kmm/lustre-client-moduleloader:${KERNEL_FULL_VERSION}`.
Modules have no `spec.imageRepoSecret`.

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

### 4. Apply Module `lnet`

If `cpu_npartitions` is required, set it in `lnet-mod.yaml` to a value
no greater than the node's vCPU count.

```bash
oc apply -n openshift-kmm -f deploy/openshift/lustre-module/lnet-mod.yaml
```

### 5. Edit and apply DaemonSet `lnet-configuration`

In `lnet-lustre-configuration-ds.yaml`:

- `&lustre-image`: `…/lustre-client-moduleloader:<uname -r>`
- `feature.node.kubernetes.io/kernel-version.full`: worker `uname -r`
- `kmm.node.kubernetes.io/openshift-kmm.lnet.ready`
- `NET_IFACE`: interface that reaches the Lustre servers
- `NET_TYPE`: `tcp`

```bash
oc apply -n openshift-kmm -f deploy/openshift/lustre-module/lnet-lustre-configuration-ds.yaml
oc rollout status -n openshift-kmm ds/lnet-configuration
```

## Verify

```bash
oc get module -n openshift-kmm
oc get ds -n openshift-kmm lnet-configuration

POD=$(oc get pods -n openshift-kmm -l name=lnet-configuration \
  -o jsonpath='{.items[0].metadata.name}')
oc logs -n openshift-kmm "$POD" -c configure-lnet
oc exec -n openshift-kmm "$POD" -- lnetctl net show
oc debug node/<node-name> -- chroot /host lsmod | grep -E "lnet|ptlrpc|lustre|mgc"
```

The CSI node DaemonSet waits until `lustre` appears in `/proc/modules`.
