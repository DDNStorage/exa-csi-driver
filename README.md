# Exascaler-csi-file-driver

Releases can be found here - https://github.com/DDNStorage/exa-csi-driver/releases

## Feature List
|Feature|Feature Status|CSI Driver Version|CSI Spec Version|Kubernetes Version|
|--- |--- |--- |--- |--- |
|Static Provisioning|GA|>= 1.0.0|>= 1.0.0|>=1.18|
|Dynamic Provisioning|GA|>= 1.0.0|>= 1.0.0|>=1.18|
|RW mode|GA|>= 1.0.0|>= 1.0.0|>=1.18|
|RO mode|GA|>= 1.0.0|>= 1.0.0|>=1.18|
|Expand volume|GA|>= 1.0.0|>= 1.1.0|>=1.18|
|StorageClass Secrets|GA|>= 1.0.0|>=1.0.0|>=1.18|
|Mount options|GA|>= 1.0.0|>= 1.0.0|>=1.18|
|Topology|GA|>= v2.0.0|>= v1.0.0|>=1.17|

## Access Modes support
|Access mode| Supported in version|
|--- |--- |
|ReadWriteOnce| >=1.0.0 |
|ReadOnlyMany| >=2.3.0 |
|ReadWriteMany| >=1.0.0 |
|ReadWriteOncePod| >=2.3.0 |

## Openshift Certification
|Openshift Version| CSI driver Version| EXA Version|
|---|---|---|
|v4.13|v2.2.3|v6.3.0|

## Requirements

- Kubernetes cluster must allow privileged pods, this flag must be set for the API server and the kubelet
  ([instructions](https://github.com/kubernetes-csi/docs/blob/735f1ef4adfcb157afce47c64d750b71012c8151/book/src/Setup.md#enable-privileged-pods)):
  ```
  --allow-privileged=true
  ```
- Required API server and kubelet feature gates for k8s version < 1.16 (skip this step for k8s >= 1.16)
  ([instructions](https://github.com/kubernetes-csi/docs/blob/735f1ef4adfcb157afce47c64d750b71012c8151/book/src/Setup.md#enabling-features)):
  ```
  --feature-gates=ExpandInUsePersistentVolumes=true,ExpandCSIVolumes=true,ExpandPersistentVolumes=true
  ```
- Mount propagation must be enabled, the Docker daemon for the cluster must allow shared mounts
  ([instructions](https://github.com/kubernetes-csi/docs/blob/735f1ef4adfcb157afce47c64d750b71012c8151/book/src/Setup.md#enabling-mount-propagation))

## Installation
Clone or untar driver (depending on where you get the driver)
```bash
git clone -b <driver version> https://github.com/DDNStorage/exa-csi-driver.git /opt/exascaler-csi-file-driver
```
e.g:-
```bash
git clone -b 2.2.3 https://github.com/DDNStorage/exa-csi-driver.git /opt/exascaler-csi-file-driver
```
or
```bash
rpm -Uvh exa-csi-driver-1.0-1.el7.x86_64.rpm
```

## Openshift
    Make sure that `openshift: true` in `deploy/openshift/exascaler-csi-file-driver-config.yaml`.

### Building lustre rpms
You will need a vm with the kernel version matching that of the Openshift nodes. To check on nodes:
```bash
oc get nodes
oc debug node/c1-pk6k4-worker-0-2j6w8
uname -r
```

On the builder vm, install the matching kernel and log in to it, example for Rhel 9.2:
```bash
# login to subscription-manager
subscription-manager register --username <username> --password <password> --auto-attach
# list available kernels
yum --showduplicates list available kernel
yum install kernel-<version>-<revision>.<arch>  # e.g.: yum install kernel-5.14.0-284.25.1.el9_2.x86_64
grubby --info=ALL | grep title
title="Red Hat Enterprise Linux (5.14.0-284.11.1.el9_2.x86_64) 9.2 (Plow)"                <---- 0
title="Red Hat Enterprise Linux (5.14.0-284.25.1.el9_2.x86_64) 9.2 (Plow)"                <---- 1

grub2-set-default 1
reboot
```

Copy EXAScaler client tar from the EXAScaler server:
```bash
scp root@<exa-server-ip>:/scratch/EXAScaler-<version>/exa-client-<version>.tar.gz .
tar -xvf exa-client-<version>.tar.gz
cd exa-client
./exa_client_deploy.py -i
```

This will build the rpms and install the client.
Upload the rpms to any repository available from the cluster and change deploy/openshift/lustre-module/lustre-dockerfile-configmap.yaml lines 12-13 accordingly.
  ```
  RUN git clone https://github.com/Qeas/rpms.git # change this to your repo with matching rpms
  RUN yum -y install rpms/*.rpm
  ```

### Loading lustre modules in Openshift

Before loading the lustre modules, make sure to install Openshift Kernel Module Management (KMM) via Openshift console.

```bash
oc create -n openshift-kmm -f deploy/openshift/lustre-module/lustre-dockerfile-configmap.yaml
oc apply -n openshift-kmm -f deploy/openshift/lustre-module/lnet-mod.yaml
```
Wait for the builder pod (e.g. `lnet-build-5f265-build`) to finish. After builder finishes,
you should have `lnet-8b72w-6fjwh` running on each worker node.
```bash
# run ko2iblnd-mod if you are using Infiniband network
oc apply -n openshift-kmm -f deploy/openshift/lustre-module/ko2iblnd-mod.yaml
```

Make changes to `deploy/openshift/lustre-module/lnet-configuration-ds.yaml` line 38 according to the cluster's network
```
lnetctl net add --net tcp --if br-ex # change interface according to your cluster
```
Configure lnet and install lustre
```
oc apply -n openshift-kmm -f deploy/openshift/lustre-module/lnet-configuration-ds.yaml
oc apply -n openshift-kmm -f deploy/openshift/lustre-module/lustre-mod.yaml
```

### Installing the driver

```bash
oc create -n openshift-kmm secret generic exascaler-csi-file-driver-config --from-file=deploy/openshift/exascaler-csi-file-driver-config.yaml
oc apply -n openshift-kmm -f deploy/openshift/exascaler-csi-file-driver.yaml
```

### Uninstall

```bash
oc delete -n openshift-kmm secret exascaler-csi-file-driver-config
oc delete -n openshift-kmm -f deploy/openshift/exascaler-csi-file-driver.yaml
oc delete -n openshift-kmm -f deploy/openshift/lustre-module/lustre-dockerfile-configmap.yaml
oc delete -n openshift-kmm -f deploy/openshift/lustre-module/lnet-mod.yaml
oc delete -n openshift-kmm -f deploy/openshift/lustre-module/ko2iblnd-mod.yaml
oc delete -n openshift-kmm -f deploy/openshift/lustre-module/lnet-configuration-ds.yaml
oc delete -n openshift-kmm -f deploy/openshift/lustre-module/lustre-mod.yaml
oc get images | grep lustre-client-moduleloader | awk '{print $1}' | xargs oc delete image
```

## Kubernetes
### Using helm chart

Make changes to `deploy/helm-chart/values.yaml` and `deploy/helm-chart/exascaler-csi-file-driver-config.yaml` according to your Kubernetes and EXAScaler clusters environment

`helm install -n ${namespace} exascaler-csi-file-driver deploy/helm-chart/`

#### Uninstall
`helm uninstall exascaler-csi-file-driver`

#### Upgrade
1. Make any necessary changes to the chart, for example new driver version: `tag: "v2.2.3"` in `deploy/helm-chart/values.yaml`.
2. Run `helm upgrade -n ${namespace} exascaler-csi-file-driver deploy/helm-chart/`

### Using docker load and kubectl commands
   ```bash
   docker load -i /opt/exascaler-csi-file-driver/bin/exascaler-csi-file-driver.tar
   ```

2. Copy /opt/exascaler-csi-file-driver/deploy/kubernetes/exascaler-csi-file-driver-config.yaml to /etc/exascaler-csi-file-driver-v1.0/exascaler-csi-file-driver-config.yaml
   ```
   cp /opt/exascaler-csi-file-driver/deploy/kubernetes/exascaler-csi-file-driver-config.yaml /etc/exascaler-csi-file-driver-v1.0/exascaler-csi-file-driver-config.yaml 
   ```
Edit `/etc/exascaler-csi-file-driver-v1.0/exascaler-csi-file-driver-config.yaml` file. Driver configuration example:
   ```yaml
   exascaler_map:
     exa1:
       mountPoint: /exaFS                                          # mountpoint on the host where the exaFS will be mounted
       exaFS: 192.168.88.114@tcp2 :/testfs                            # default path to exa filesystem
       managementIp: 10.204.86.114@tcp                           # network for management operations, such as create/delete volume
       zone: zone-1

     exa2:
       mountPoint: /exaFS-zone-2                                          # mountpoint on the host where the exaFS will be mounted
       exaFS: 192.168.78.112@tcp2:/testfs/zone-2                            # default path to exa filesystem
       managementIp: 10.204.86.114@tcp                           # network for management operations, such as create/delete volume
       zone: zone-2

     exa3:
       mountPoint: /exaFS-zone-3                                          # mountpoint on the host where the exaFS will be mounted
       exaFS: 192.168.98.113@tcp2:/testfs/zone-3                            # default path to exa filesystem
       managementIp: 10.204.86.114@tcp                           # network for management operations, such as create/delete volume
       zone: zone-3

   debug: true
   ```

3. Create Kubernetes secret from the file:
   ```bash
   kubectl create secret generic exascaler-csi-file-driver-config --from-file=/etc/exascaler-csi-file-driver-v1.0/exascaler-csi-file-driver-config.yaml
   ```

4. Register driver to Kubernetes:
   ```bash
   kubectl apply -f /opt/exascaler-csi-file-driver/deploy/kubernetes/exascaler-csi-file-driver.yaml
   ```

## Usage

### Dynamically provisioned volumes

For dynamic volume provisioning, the administrator needs to set up a _StorageClass_ in the PV yaml (/opt/exascaler-csi-file-driver/examples/exa-dynamic-nginx.yaml for this example) pointing to the driver.
For dynamically provisioned volumes, Kubernetes generates volume name automatically (for example `pvc-ns-cfc67950-fe3c-11e8-a3ca-005056b857f8-projectId-1001`).

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: exascaler-csi-file-driver-sc-nginx-dynamic
provisioner: exa.csi.ddn.com
allowedTopologies:
- matchLabelExpressions:
  - key: topology.exa.csi.ddn.com/zone
    values:
    - zone-1
mountOptions:                        # list of options for `mount -o ...` command
#  - noatime                         #
parameters:
  projectId: "100001"      # Points to EXA project id to be used to set volume quota. Automatically generated by the driver if not provided.
  exaMountUid: "1001"      # Uid which will be used to access the volume in pod. Should be synced between EXA server and clients.
  bindMount: "false"       # Determines, whether volume will bind mounted or as a separate lustre mount.
  exaFS: "10.204.86.114@tcp:/testfs"   # Overrides exaFS value from config. Use this to support multiple EXA filesystems.
  mountPoint: /exaFS       # Overrides mountPoint value from config. Use this to support multiple EXA filesystems.
  mountOptions: ro,noflock
```

#### Example

Run Nginx pod with dynamically provisioned volume:

```bash
kubectl apply -f /opt/exascaler-csi-file-driver/examples/exa-dynamic-nginx.yaml

# to delete this pod:
kubectl delete -f /opt/exascaler-csi-file-driver/examples/exa-dynamic-nginx.yaml
```

### Static (pre-provisioned) volumes

The driver can use already existing Exasaler filesystem,
in this case, _StorageClass_, _PersistentVolume_ and _PersistentVolumeClaim_ should be configured.

#### _StorageClass_ configuration

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: exascaler-csi-driver-sc-nginx-persistent
provisioner:  exa.csi.ddn.com
allowedTopologies:
- matchLabelExpressions:
  - key: topology.exa.csi.ddn.com/zone
    values:
    - zone-1
mountOptions:                         # list of options for `mount -o ...` command
#  - noatime                         #
```

### Topology configuration
In order to configure CSI driver with kubernetes topology, use the `zone` parameter in driver config or storageClass parameters. Example config file with zones:
  ```bash

  exascaler_map:
    exa1:
      exaFS: 10.3.196.24@tcp:/csi
      mountPoint: /exaFS
      zone: us-west
    exa2:
      exaFS: 10.3.196.24@tcp:/csi-2
      mountPoint: /exaFS-zone2
      zone: us-east
  ```

This will assign volumes to be created on Exascaler cluster that correspond with the zones requested by allowedTopologies values.


#### _PersistentVolume_ configuration

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: exascaler-csi-driver-pv-nginx-persistent
  labels:
    name: exascaler-csi-driver-pv-nginx-persistent
spec:
  storageClassName: exascaler-csi-driver-sc-nginx-persistent
  accessModes:
    - ReadWriteMany
  capacity:
    storage: 1Gi
  csi:
    driver: exa.csi.ddn.com
    volumeHandle: exa1:/exaFS/nginx-persistent
    volumeAttributes:         # volumeAttributes are the alternative of storageClass params for static (precreated) volumes.
      exaMountUid: "1001"     # Uid which will be used to access the volume in pod.
      #mountOptions: ro, flock # list of options for `mount` command
```

CSI Parameters:

| Name           | Description                                                       | Example                              |
|----------------|-------------------------------------------------------------------|--------------------------------------|
| `driver`       | [required] installed driver name " exa.csi.ddn.com"        | `exa.csi.ddn.com` |
| `volumeHandle` | [required] EXAScaler server IP and path to existing EXAScaler filesystem | `exa1:/exaFS/nginx-persistent`               |
| `exaMountUid`  | Uid which will be used to access the volume from the pod. | `1015` |
| `projectId`    | Points to EXA project id to be used to set volume quota. Automatically generated by the driver if not provided. | `100001` |
| `exaFS`        | Overrides exaFS value from config. Use this to support multiple EXA filesystems. | `10.0.1.1@tcp:/fs01` |
| `managementIp` | Should be used if there is a separate network configured for management operations, such as create/delete volumes. This network should have access to all Exa filesystems in a isolated zones environment | `192.168.10.20@tcp2` |
| `mountPoint`   | Overrides Mountpoint where EXAscaler filesystem will be mounted on the host | `/sc-mnt` |
| `bindMount`    | Determines, whether volume will bind mounted or as a separate lustre mount. | `true` |
| `mountOptions` | Options that will be passed to mount command (-o <opt1,opt2,opt3>)  | `ro,flock` |
| `minProjectId` | Minimum project ID number for automatic generation. Only used when projectId is not provided. | 10000 |
| `maxProjectId` | Maximum project ID number for automatic generation. Only used when projectId is not provided. | 4294967295 |
| `generateProjectIdRetries` | Maximum retry count for generating random project ID. Only used when projectId is not provided. | `5` |
| `zone`        | Topology zone to control where the volume should be created. Should match topology.exa.csi.ddn.com/zone label on node(s). | `us-west` |
| `v1xCompatible` | [Optional] Only used when upgrading the driver from v1.x.x to v2.x.x. Provides compatibility for volumes that were created beore the upgrade. Set it to `true` to point to the Exa cluster that was configured before the upgrade | `false` |
| `tempMountPoint` | [Optional] Used when `exaFS` points to a subdirectory that does not exist on Exascaler and will be automatically created by the driver. This parameter sets the directory where Exascaler filesystem will be temporarily mounted to create the subdirectory. | `/tmp/exafs-mnt` |

#### _PersistentVolumeClaim_ (pointing to created _PersistentVolume_)

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: exascaler-csi-driver-pvc-nginx-persistent
spec:
  storageClassName: exascaler-csi-driver-cs-nginx-persistent
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
  selector:
    matchLabels:
      # to create 1-1 relationship for pod - persistent volume use unique labels
      name: exascaler-csi-file-driver-pv-nginx-persistent
```

#### Example

Run nginx server using PersistentVolume.

**Note:** Pre-configured filesystem should exist on the EXAScaler:
`/exaFS/nginx-persistent`.

```bash
kubectl apply -f /opt/exascaler-csi-file-driver/examples/nginx-persistent-volume.yaml

# to delete this pod:
kubectl delete -f /opt/exascaler-csi-file-driver/examples/nginx-persistent-volume.yaml
```

## Updating the driver version
To update to a new driver version, you need to follow the following steps:

1. Remove the old driver version
```bash
kubectl delete -f /opt/exascaler-csi-file-driver/deploy/kubernetes/exascaler-csi-file-driver.yaml
kubectl delete secrets exascaler-csi-file-driver-config
rpm -evh exa-csi-driver
```
2. Download the new driver version (git clone or new ISO)
3. Copy and edit config file
  ```bash
  cp /opt/exascaler-csi-file-driver/deploy/kubernetes/exascaler-csi-file-driver-config.yaml /etc/exascaler-csi-file-driver-v1.1/exascaler-csi-file-driver-config.yaml
  ```

  If you are upgrading from v1.x.x to v2.x.x, config file structure will change to a map of Exascaler clusters instead of a flat structure. To support old volume that were created using v1.x.x, old config should be put in the config map with `v1xCompatible: true`. For example:

  v1.x.x config
  ```bash
  exaFS: 10.3.196.24@tcp:/csi
  mountPoint: /exaFS
  debug: true
  ```
  v2.x.x config with support of previously created volumes
  ```bash

  exascaler_map:
    exa1:
      exaFS: 10.3.196.24@tcp:/csi
      mountPoint: /exaFS
      v1xCompatible: true

    exa2:
      exaFS: 10.3.1.200@tcp:/csi-fs
      mountPoint: /mnt2

  debug: true
```

Only one of the Exascaler clusters can have `v1xCompatible: true` since old config supported only 1 cluster.

4. Update version in /opt/exascaler-csi-file-driver/deploy/kubernetes/exascaler-csi-file-driver.yaml
```
          image: exascaler-csi-file-driver:v2.2.3
```
5. Load new image
```bash
docker load -i /opt/exascaler-csi-file-driver/bin/exascaler-csi-file-driver.tar
```
6. Apply new driver
```bash
kubectl create secret generic exascaler-csi-file-driver-config --from-file=/etc/exascaler-csi-file-driver-v1.1/exascaler-csi-file-driver-config.yaml
kubectl apply -f /opt/exascaler-csi-file-driver/deploy/kubernetes/exascaler-csi-file-driver.yaml
```

## Troubleshooting
Logs can be found in /var/log/containers directory.
To collect all driver related logs, you can use the following command
```
mkdir -p /tmp/exascaler-csi-file-driver-logs/
cp /var/log/containers/exascaler-csi-* /tmp/exascaler-csi-file-driver-logs/
```
