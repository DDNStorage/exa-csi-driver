# Exascaler-csi-file-driver

## Supported kubernetes versions matrix

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

## Requirements

- Kubernetes cluster must allow privileged pods, this flag must be set for the API server and the kubelet
  ([instructions](https://github.com/kubernetes-csi/docs/blob/735f1ef4adfcb157afce47c64d750b71012c8151/book/src/Setup.md#enable-privileged-pods)):
  ```
  --allow-privileged=true
  ```
- Required the API server and the kubelet feature gates
  ([instructions](https://github.com/kubernetes-csi/docs/blob/735f1ef4adfcb157afce47c64d750b71012c8151/book/src/Setup.md#enabling-features)):
  ```
  --feature-gates=,VolumePVCDataSource=true,ExpandInUsePersistentVolumes=true,ExpandCSIVolumes=true,ExpandPersistentVolumes=true
  ```
- Mount propagation must be enabled, the Docker daemon for the cluster must allow shared mounts
  ([instructions](https://github.com/kubernetes-csi/docs/blob/735f1ef4adfcb157afce47c64d750b71012c8151/book/src/Setup.md#enabling-mount-propagation))


## Installation

1. Untar driver
   ```bash
   tar -xzvf /opt/exascaler-csi-file-driver.tar.gz
   cd exascaler-csi-file-driver
   docker load -i bin/exascaler-csi-file-driver.tar
   ```
2. Edit `deploy/kubernetes/exascaler-csi-file-driver-config.yaml` file. Driver configuration example:
   ```yaml
   mountPoint: /exaFS
   exaFS: 10.3.196.24@tcp:/csi                                 # default Exascaler data IP
   debug: true                                                 # more logs
   exaMountUser: ubuntu                                        # non-root user synced between EXA clients and server
   ```
   **Note:** exaMountUser should exist and have identical ID on Kubernetes host side and on EXAScaler side
3. Create Kubernetes secret from the file:
   ```bash
   kubectl create secret generic exascaler-csi-file-driver-config --from-file=deploy/kubernetes/exascaler-csi-file-driver-config.yaml
   ```
4. Register driver to Kubernetes:
   ```bash
   kubectl apply -f deploy/kubernetes/exascaler-csi-file-driver.yaml
   ```

## Usage

### Dynamically provisioned volumes

For dynamic volume provisioning, the administrator needs to set up a _StorageClass_ pointing to the driver.
In this case Kubernetes generates volume name automatically (for example `pvc-ns-cfc67950-fe3c-11e8-a3ca-005056b857f8`).
Default driver configuration may be overwritten in `parameters` section:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: exascaler-csi-file-driver-sc-nginx-dynamic
provisioner: exascaler-csi-file-driver.tintri.com
mountOptions:                        # list of options for `mount -o ...` command
#  - noatime                         #
```

#### Example

Run Nginx pod with dynamically provisioned volume:

```bash
kubectl apply -f examples/exa-dynamic-nginx.yaml

# to delete this pod:
kubectl delete -f examples/exa-dynamic-nginx.yaml
```

### Pre-provisioned volumes

The driver can use already existing Exasaler filesystem,
in this case, _StorageClass_, _PersistentVolume_ and _PersistentVolumeClaim_ should be configured.

#### _StorageClass_ configuration

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: exascaler-csi-driver-sc-nginx-persistent
provisioner: exascaler-csi-driver.tintri.com
mountOptions:                        # list of options for `mount -o ...` command
#  - noatime                         #
```

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
    driver: exascaler-csi-driver.tintri.com
    volumeHandle: /exaFS/nginx-persistent
  #mountOptions:  # list of options for `mount` command
  #  - noatime    #
```

CSI Parameters:

| Name           | Description                                                       | Example                              |
|----------------|-------------------------------------------------------------------|--------------------------------------|
| `driver`       | installed driver name "exascaler-csi-file-driver.tintri.com"        | `exascaler-csi-file-driver.tintri.com` |
| `volumeHandle` | EXAScaler server IP and path to existing EXAScaler filesystem | `/exaFS/nginx-persistent`               |

#### _PersistentVolumeClaim_ (pointed to created _PersistentVolume_)

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
kubectl apply -f examples/nginx-persistent-volume.yaml

# to delete this pod:
kubectl delete -f examples/nginx-persistent-volume.yaml
```

## Updating the driver version
To update to a new driver version, you need to follow the following steps:

1. Download the new driver version
2. Update version in exascaler-csi-file-driver.yaml
3. Remove the old version
```bash
kubectl delete -f deploy/kubernetes/exascaler-csi-file-driver.yaml
kubectl delete secrets exascaler-csi-file-driver-config
```
4. Load new image
```bash
docker load -i bin/exascaler-csi-file-driver.tar
```
5. Apply new driver
```bash
kubectl create secret generic exascaler-csi-file-driver-config --from-file=./deploy/kubernetes/exascaler-csi-file-driver-config.yaml
kubectl apply -f deploy/kubernetes/exascaler-csi-file-driver.yaml
```
