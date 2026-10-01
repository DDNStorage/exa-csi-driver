# Canary kernel upgrade

Pause workers, upgrade masters, rebuild `builder-base`, read the new
kernel, then roll workers one node at a time.

KMM pushes the moduleloader to
`image-registry.openshift-image-registry.svc:5000/openshift-kmm/lustre-client-moduleloader:${KERNEL_FULL_VERSION}`
(MOFED: `*-mofed`). The in-cluster registry uses PVC
`image-registry-storage`. Modules have no `spec.imageRepoSecret`.

A node belongs to one MachineConfigPool. MCO cordons and drains.
`${KERNEL_FULL_VERSION}` selects the moduleloader image per node.

Live DaemonSet: `lnet-configuration` (or `lnet-configuration-<current-ocp>`
after a previous upgrade). Target: `lnet-configuration-<target-ocp>`
(example: `lnet-configuration-4.22`).

TCP: `lnet-mod.yaml`, `lustre-dockerfile-configmap.yaml`,
`upgrade/lnet-lustre-configuration-ds.yaml`.
MOFED: `lnet-mod-mofed.yaml`, `lustre-dockerfile-mofed-configmap.yaml`,
`upgrade/lnet-lustre-configuration-ds-mofed.yaml` (image tag `*-mofed`).

Run `oc apply` from the repository root (`exascaler-csi-file-driver`).
Pin the live DaemonSet with `oc patch`. Apply only
`upgrade/lnet-lustre-configuration-ds*.yaml`.

Live and target DaemonSets use `updateStrategy: OnDelete`. Labeling
`worker-canary` unschedules the live pod on that node.

## 0. Pin the live stack (current kernel)

Replace the ConfigMap if the tarball, entitlements, or Dockerfile
changed. Apply the Module. Pin the live DS to the current worker
kernel, exclude `node-role.kubernetes.io/worker-canary`, grace 120.

```bash
LIVE=lnet-configuration
OLD=$(oc get node -l node-role.kubernetes.io/worker \
  -o jsonpath='{.items[0].status.nodeInfo.kernelVersion}')
echo "$LIVE $OLD"
oc get ds -n openshift-kmm "$LIVE" \
  -o jsonpath='selector={.spec.template.spec.nodeSelector}{"\n"}affinity={.spec.template.spec.affinity.nodeAffinity}{"\n"}grace={.spec.template.spec.terminationGracePeriodSeconds}{"\n"}'
```

Set `OnDelete` first (not a template field; does not replace pods):

```bash
oc patch ds -n openshift-kmm "$LIVE" --type=merge -p '{"spec":{"updateStrategy":{"type":"OnDelete"}}}'
oc get ds -n openshift-kmm "$LIVE" -o jsonpath='{.spec.updateStrategy.type}{"\n"}'
```

If affinity or grace are missing:

```bash
oc patch ds -n openshift-kmm "$LIVE" --type=strategic -p '{
  "spec":{"template":{"spec":{
    "terminationGracePeriodSeconds":120,
    "affinity":{"nodeAffinity":{"requiredDuringSchedulingIgnoredDuringExecution":{
      "nodeSelectorTerms":[{"matchExpressions":[{
        "key":"node-role.kubernetes.io/worker-canary","operator":"DoesNotExist"
      }]}]}}
    }
  }}}
}'
```

If the kernel selector is missing:

```bash
oc patch ds -n openshift-kmm "$LIVE" --type=json -p='[
  {"op":"add","path":"/spec/template/spec/nodeSelector/feature.node.kubernetes.io~1kernel-version.full","value":"'"$OLD"'"}
]'
```

## 1. Pause workers, upgrade masters

Pause `mcp/worker` **before** `oc adm upgrade`. Masters (`mcp/master`)
still update. Workers stay on `$OLD` until you canary them.

```bash
oc patch mcp/worker --type=merge -p '{"spec":{"paused":true}}'
oc get mcp
oc adm upgrade --to=<version>
oc get clusterversion
oc get mcp -w
```

Wait until `mcp/master` is `UPDATED=True` / `UPDATING=False`.
`mcp/worker` stays paused (`UPDATED=False` is expected).

Read the new kernel from a master:

```bash
NEW=$(oc get node -l node-role.kubernetes.io/master \
  -o jsonpath='{.items[0].status.nodeInfo.kernelVersion}')
echo "$NEW"
# compact / no master role:
# NEW=$(oc get node -l node-role.kubernetes.io/control-plane \
#   -o jsonpath='{.items[0].status.nodeInfo.kernelVersion}')
```

## 2. Rebuild `builder-base`

Rebuild `builder-base` if DTK, the tarball, or entitlements changed.
The registry PVC is `image-registry-storage.yaml` in the parent
directory.

```bash
oc get pvc image-registry-storage -n openshift-image-registry

ls deploy/openshift/lustre-module/Dockerfile \
   deploy/openshift/lustre-module/entitlement.pem \
   deploy/openshift/lustre-module/entitlement-key.pem \
   deploy/openshift/lustre-module/lustre-*.tar.gz

oc get bc -n openshift-kmm builder-base || \
  oc new-build -n openshift-kmm --binary --name=builder-base --strategy=docker

oc start-build -n openshift-kmm builder-base \
  --from-dir=deploy/openshift/lustre-module/ --follow

oc run pull-builder -n openshift-kmm --restart=Never --rm -it \
  --image=image-registry.openshift-image-registry.svc:5000/openshift-kmm/builder-base:latest \
  --overrides='{"spec":{"serviceAccountName":"kmm-operator-module-loader"}}' \
  --command -- echo ok
```

If `lnet-build-*` / `ko2iblnd-build-*` for `$NEW` already failed, delete
those Builds by name so KMM retries:

```bash
oc get builds -n openshift-kmm
oc delete build -n openshift-kmm <build-name>
```

## 3. Apply the target DaemonSet

In `upgrade/lnet-lustre-configuration-ds.yaml` (or `-mofed.yaml`) set:

- `metadata.name` / selector / pod label: `lnet-configuration-<target-ocp>`
- `feature.node.kubernetes.io/kernel-version.full`: `$NEW`
- `kmm.node.kubernetes.io/openshift-kmm.lnet.ready`
  (MOFED also: `kmm.node.kubernetes.io/openshift-kmm.ko2iblnd.ready`
  and `network.nvidia.com/operator.mofed.wait: "false"`)
- `&lustre-image`: `…/lustre-client-moduleloader:$NEW` (MOFED: `$NEW-mofed`)

```bash
oc apply -n openshift-kmm -f deploy/openshift/lustre-module/upgrade/lnet-lustre-configuration-ds.yaml
# MOFED: upgrade/lnet-lustre-configuration-ds-mofed.yaml
oc get ds -n openshift-kmm lnet-configuration-4.22 \
  -o jsonpath='selector={.spec.template.spec.nodeSelector}{"\n"}image={.spec.template.spec.containers[0].image}{"\n"}'
```

`DESIRED=0` until a worker runs `$NEW` and KMM has set the ready labels.

## 4. Canary MachineConfigPool

```bash
oc apply -f deploy/openshift/lustre-module/upgrade/mcp-worker-canary.yaml
oc patch mcp/worker --type=json -p='[
  {"op":"add","path":"/spec/nodeSelector/matchExpressions","value":[
    {"key":"node-role.kubernetes.io/worker","operator":"Exists"},
    {"key":"node-role.kubernetes.io/worker-canary","operator":"DoesNotExist"}
  ]}
]'
```

If `mcp/worker` already has `matchExpressions`, inspect
`oc get mcp/worker -o yaml` and edit instead of adding a second selector.

## 5. First worker

```bash
NODE=<worker>
oc label node "$NODE" node-role.kubernetes.io/worker-canary=""
oc get mcp
```

Keep `node-role.kubernetes.io/worker`. `mcp/worker` MACHINECOUNT must drop
by one; `mcp/worker-canary` must be 1. Wait for live DS `preStop`:

```bash
oc wait -n openshift-kmm --for=delete pod \
  -l name="$LIVE" --field-selector spec.nodeName="$NODE" --timeout=180s
```

MCO drains and reboots `$NODE`. Wait until `mcp/worker-canary` is
`UPDATED=True` and the node is Ready on `$NEW`. Confirm kernel, then set
the NFD label:

```bash
oc get node "$NODE" -o jsonpath='{.status.nodeInfo.kernelVersion}{"\n"}'
oc label node "$NODE" \
  feature.node.kubernetes.io/kernel-version.full="$NEW" --overwrite
oc get builds -n openshift-kmm
oc get pods -n openshift-kmm -o wide --field-selector spec.nodeName="$NODE"
```

KMM starts `lnet-build-<kernel>` for `$NEW` if that tag is missing. Wait
until the Build is `Complete` (MOFED: 20–30 min). Then:

```bash
oc debug node/"$NODE" -- chroot /host lsmod | grep -E 'lnet|ko2iblnd|ptlrpc|lustre|mgc'
```

That node must run `kmm-worker-…-lnet` and `lnet-configuration-4.22-*`,
not `$LIVE`. `lsmod` must include `lustre` and `mgc`.

Confirm `$NEXT` can pull the tag:

```bash
# MOFED:
IMG=image-registry.openshift-image-registry.svc:5000/openshift-kmm/lustre-client-moduleloader:${NEW}-mofed
# TCP: drop -mofed

NEXT=<next-worker>
oc delete pod -n openshift-kmm pull-next --ignore-not-found
oc run pull-next -n openshift-kmm --restart=Never --image="$IMG" \
  --overrides='{"spec":{"nodeName":"'"$NEXT"'","serviceAccountName":"kmm-operator-module-loader"}}' \
  --image-pull-policy=Always --command -- echo ok
oc wait -n openshift-kmm --for=jsonpath='{.status.phase}'=Succeeded pod/pull-next --timeout=120s
oc logs -n openshift-kmm pull-next
oc delete pod -n openshift-kmm pull-next
```

`Succeeded` / `ok` before canarying `$NEXT`.

## 6. Remaining workers

Repeat section 5 for each remaining worker after `pull-next` on that
node is `Succeeded`.

## 7. Finish

After every worker is on `$NEW` with `lustre` and `mgc` in `lsmod`:

1. Unlabel.

```bash
for n in $(oc get nodes -l node-role.kubernetes.io/worker-canary \
  -o jsonpath='{.items[*].metadata.name}'); do
  oc label node "$n" node-role.kubernetes.io/worker-canary-
done
```

2. Confirm every worker is back on `mcp/worker` and `mcp/worker-canary`
   `MACHINECOUNT=0`. Do **not** delete the canary pool yet. Nodes still
   reference `rendered-worker-canary-<hash>`, and that MachineConfig is
   owned by `mcp/worker-canary`. Deleting the pool deletes the object
   while MCD is still syncing it (`not found` → `DEGRADED`).

```bash
oc get mcp
oc get node -l node-role.kubernetes.io/worker \
  -o jsonpath='{range .items[*]}{.metadata.name} desired={.metadata.annotations.machineconfiguration\.openshift\.io/desiredConfig} current={.metadata.annotations.machineconfiguration\.openshift\.io/currentConfig} state={.metadata.annotations.machineconfiguration\.openshift\.io/state}{"\n"}{end}'
```

3. Unpause `mcp/worker` **before** deleting the canary pool. A paused pool
   does not rewrite `desiredConfig`, so step 1 alone never moves nodes off
   `rendered-worker-canary-<hash>`. Unpause lets MCO point them at
   `rendered-worker-<hash>` (same hash, no reboot). Wait until every
   worker `desired` and `current` are `rendered-worker-<hash>` and
   `state=Done`. `DEGRADED=False`.

```bash
oc patch mcp/worker --type=merge -p '{"spec":{"paused":false}}'
oc get node -l node-role.kubernetes.io/worker \
  -o jsonpath='{range .items[*]}{.metadata.name} desired={.metadata.annotations.machineconfiguration\.openshift\.io/desiredConfig} current={.metadata.annotations.machineconfiguration\.openshift\.io/currentConfig} state={.metadata.annotations.machineconfiguration\.openshift\.io/state}{"\n"}{end}'
oc get mcp
```

4. Only then delete the empty canary pool and restore the worker selector:

```bash
oc delete mcp worker-canary
oc patch mcp/worker --type=json -p='[
  {"op":"remove","path":"/spec/nodeSelector/matchExpressions"}
]'
oc get mcp
```

Patch `lnet-configuration-4.22` with the same canary-exclude affinity and
grace 120 as section 0. Delete `$LIVE`. Next upgrade:
`$LIVE=lnet-configuration-4.22`, target `lnet-configuration-4.23`.
