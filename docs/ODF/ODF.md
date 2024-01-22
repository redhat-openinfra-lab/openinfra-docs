# Data Foundation Storage 

ODF can make use of external Ceph when the OCP cluster is running on:

* Bare Metal
* VMware vSphere
* OSP (Tech Preview)

cccccbkrlteillethekiungfreuhfrcutrklddjrieuu

## Deployment

Label the respective nodes (in this example the worker nodes):
```
oc label nodes \
-l node-role.kubernetes.io/worker= \
cluster.ocs.openshift.io/openshift-storage=
```

Get the nodes the LSO operator discovered:
```
oc get localvolumediscoveryresult -n openshift-local-storage 
```

Get the disks available on one of the nodes:
```
oc describe localvolumediscoveryresult/discovery-result-worker01 -n openshift-local-storage
```
> NOTE: Use the Device ID when adding devices to the rook-ceph operator

Get the PVs in use:
```
oc get pv -n openshift-local-storage
```

Get the PV on a specific node using the label option:
```
oc get pv -l kubernetes.io/hostname=worker01
```
> NOTE: Status of Available means it can be added to the rook-ceph operator

Example `LocalVolume` manifest:
```
apiVersion: local.storage.openshift.io/v1
kind: LocalVolume
metadata:
  name: expanded-local-block
  namespace: openshift-local-storage
spec:
  nodeSelector:
    nodeSelectorTerms:
    - matchExpressions:
        - key: cluster.ocs.openshift.io/openshift-storage
          operator: In
          values:
          - ""
  storageClassDevices:
    - storageClassName: localblock
      volumeMode: Block
      devicePaths:
        - /dev/disk/by-id/virtio-aaf40cdd-da3d-4d6d-9
        - /dev/disk/by-id/virtio-fe40db23-a129-4dff-9
        - /dev/disk/by-id/virtio-a0d9e043-c4b9-4a6d-8
```

Apply the yaml file with the new device paths:
```
oc apply -f file.yaml 
```

Use the `oc patch` or `oc edit` commands to increase the `storageDeviceSets` attribute 
```
oc patch storagecluster/ocs-storagecluster -n openshift-storage --type json -p \
'[{"op": "replace", "path": "/spec/storageDeviceSets/0/count", "value": 2}]'
```

To add nodes to the cluster:
```
oc label nodes -l node-role.kubernetes.io/worker= cluster.ocs.openshift.io/openshift-storage=
```

Get the osd-pods:
```
oc get pods -n openshift-storage -l app=rook-ceph-osd
```

## Storage Classes
```
oc get storageclasses -o name
```

```
oc describe sc <storageClass>
```

Create storage class for Windows VMs in OCPv with mapOptions parameter:
```hl_lines="18"
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: windows-vms
parameters:
  clusterID: openshift-storage
  csi.storage.k8s.io/controller-expand-secret-name: rook-csi-rbd-provisioner
  csi.storage.k8s.io/controller-expand-secret-namespace: openshift-storage
  csi.storage.k8s.io/fstype: ext4
  csi.storage.k8s.io/node-stage-secret-name: rook-csi-rbd-node
  csi.storage.k8s.io/node-stage-secret-namespace: openshift-storage
  csi.storage.k8s.io/provisioner-secret-name: rook-csi-rbd-provisioner
  csi.storage.k8s.io/provisioner-secret-namespace: openshift-storage
  imageFeatures: layering
  imageFormat: "2"
  pool: ocs-storagecluster-windowsblockpool
  mounter: rbd
  mapOptions: "krbd:rxbounce"
provisioner: openshift-storage.rbd.csi.ceph.com
reclaimPolicy: Delete
volumeBindingMode: Immediate
allowVolumeExpansion: true
```

### Persistent Volume Claim

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: production-application
spec:
  accessModes:
  - ReadWriteMany
  storageClassName: ocs-storagecluster-cephfs
  resources:
    requests:
      storage: 10Gi
```

Patch the PVC to increase the size:
```
oc patch pvc/examplePVC -p '{spec:{resources:{requests:{storage: 20Gi}}}}'
```

```
oc rollout latest deploymentconfig.apps.openshift.io/exampleDeployentConfig
```

### Logs

Get listing of ODF pods:
``` 
oc get pods -n openshift-storage
```

Monitor Logs
```
oc logs rook-ceph-mon-a-blah -n openshift-storage -c mon
```

OSD Logs
```
oc logs rook-ceph-osd-0-blah -n openshift-storage -c osd
```

Rook Operator Logs:
```
oc logs rook-ceph-operator-blah -n openshift-storage
```

RBD Logs:
```
oc logs csi-rbdplugin-blah -n openshift-storage -c csi-rbdplugin
```

CephFS Logs:
```
oc logs csi-cephfsplugin-blah -n openshift-storage -c cephfsplugin
```

## Cluster Config Tool

Ceph CLI
```
oc exec -it pod/rook-ceph-operator-blah -n openshift-storage -c rook-ceph-operator -- /bin/bash
```

Cluster Health
```
ceph -c /var/lib/rook/openshift-storage/openshift-storage.config health
```

Cluster Status
```
ceph -c /var/lib/rook/openshift-storage/openshift-storage.config -s
```

Storage Usage
```
ceph -c /var/lib/rook/openshift-storage/openshift-storage.config df
```

Tool - must-gather
```
oc adm must-gather --image=registry.redhat.io/ocs4/ocs-must-gather-rhel8:v4.7 --dest-dir=must-gather
```

## Rook-Ceph Toolbox

To deploy the toolbox, navigate to Administration --> Custom Resource Definitions --> OCSInitialization --> Instances.  Select the `ocsinit` instance and upddate the YAML with the following:

```
spec:
  enableCephTools: true 
```

To access the toolbox, navigate to Workloads --> Pods --> openshift-storage namespace, scroll to bottom of list of pods and select the rook-ceph-toolbox pod.  From there, select Terminal from the menu bar.

To access via the command line:

```
oc rsh $(oc get pods -l app=rook-ceph-tools -o name) 
```
> NOTE: Make sure you are in the openshift-storage project or use the -n parameter to pass the namespace  

### Debug 

Enable debug logging for rook-ceph-operator container:
```
oc edit cm rook-ceph-operator-config
...
data:
...
  ROOK_LOG_LEVEL: DEBUG
```

Get the list of devices available on worker node:
```
oc debug node/worker01 -- lsblk --paths --nodeps
```

### Adding Disks to an Internal Cluster

Create `LocalVolume` 
```
apiVersion: local.storage.openshift.io/v1
kind: LocalVolume
metadata:
  name: expanded-local-block
  namespace: openshift-local-storage
spec:
  nodeSelector:
    nodeSelectorTerms:
    - matchExpressions:
        - key: cluster.ocs.openshift.io/openshift-storage
          operator: In
          values:
          - ""
  storageClassDevices:
    - storageClassName: localblock
      volumeMode: Block
      devicePaths:
        - /dev/disk/by-id/virtio-aaf40cdd-da3d-4d6d-9
        - /dev/disk/by-id/virtio-fe40db23-a129-4dff-9
        - /dev/disk/by-id/virtio-a0d9e043-c4b9-4a6d-8
```

Patch or edit the storage cluster to increase the `storageDeviceSets`.
```
oc patch storagecluster/ocs-storagecluster -n openshift-storage --type json \
-p '[{op: replace, path: /spec/storageDeviceSets/0/count, value: 2}]'
```
> NOTE: This triggers the creation of new OSD pods

### Adding Nodes to an Internal Cluster

Add the new node name to the `LocalVolumeDiscovery` and `LocalVolumeSet` objects or label the nodes with the `node-role.kubernetes.io/worker=` and `cluster.ocs.storage.io/openshift-storage=` labels to make the storage cluster use the nodes automatically.
```
oc label nodes -l node-role.kubernetes.io/worker= cluster.ocs.openshift.io/openshift-storage=
```
