# Storage for VMs

## Bare Metal Storage Classes

odf-storage - created by LSO (you provide the name when you create the storageSystem)
ocs-storagecluster-ceph-rbd  
ocs-storagecluster-ceph-rgw  
ocs-storagecluster-cephfs  
openshift-storage.noobaa.io  

## Cloud Default Storage Classes

AWS - gp3
Azure - managed-premium

## Resource File

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: dbdata
spec:
  storageClassName: ocs-external-storagecluster-ceph-rbd  
  accessModes:
    - ReadWriteOnce  
  resources:
    requests:
      storage: 10Gi  
  volumeMode: Filesystem  
```

> Note: volumeMode can be `Filesystem` (default)l; volume has a filesystem that you can mount inside the container or `Block`, volume is presented as a raw devices (i.e. /dev/xvda).

## PVCs for OCPv

With PVCs in Block mode, OpenShift Virtualization transfers the disk image into the volume. The VM disk uses the volume as its back-end device.  Better option is when the CSI driver supports snapshots/cloning and the source PVC is cloned.

With PVCs in Filesystem mode, OpenShift Virtualization creates a disk.img file at the root of the PV file system and then copies the disk image into the file. The VM disk uses the disk.img file as its back-end device.

### Data Volumes

```
apiVersion: cdi.kubevirt.io/v1beta1
kind: DataVolume
metadata:
  name: osdisk
spec:
  preallocation: true
  storage:  
    resources:
      requests:
        storage: 10Gi
    storageClassName: ocs-external-storagecluster-ceph-rbd  
  source:  
    http:
      url: http://images..example.com/rhel-8.5-x86_64-kvm.qcow2

```
> Note: Using `preallocation` will reserve the entire requested space with the back-end storage; this can improve write performance.

### Select a Data Volume Source

* Blank (creates PVC)  
The new Persistent Volume Claim (PVC) provides an unformatted raw device. You can use your operating system tools to partition and then format the new block device from inside the VM.  

* Import via URL (creates PVC)  
Red Hat OpenShift Virtualization downloads the virtual disk image from the URL that you provide and then extracts it into the new volume. The virtual disk image must be in the raw format or the QEMU copy on write version 2 (qcow2) format. The disk usually includes file systems that you can mount to access their data from inside the VM. Because the disk is read/write, Red Hat OpenShift Virtualization preserves all the changes you make across VM restarts.  

* Use an existing PVC  
OpenShift Virtualization attaches the PVC you provide as a disk. That PVC might come from a data volume you detached from another VM and that you want to reuse with your VM.  

* Clone existing PVC (creates PVC)  
OpenShift Virtualization copies the PVC you provide and then uses that copy as the VM disk. Because the source PVC must not be in use for the cloning process to start, you must stop any pods or VMs that are actively using it.  

* Import via Registry (creates PVC)  
OpenShift Virtualization downloads the given container image from a container registry. The container image contains a disk image in the raw or qcow2 format under the /disk/ directory. OpenShift Virtualization extracts that disk image and copies it into the volume.  

This source option is similar to importing disks from URLs, but it is useful when you already have a container registry deployed in your organization. Instead of installing a new file server to distribute your disk images, you can store these images in your container registry.  

Because container registries can only manage images, you must store your disk images in container images under the /disk/ directory. These images are not meant to be run as containers, and they usually only contain the /disk/ directory and the disk image.  

* Container (ephemeral)
OpenShift Virtualization downloads the given container image from a container registry and uses the disk in that container image as the VM disk. This source option is similar to the preceding option except that OpenShift Virtualization does not create DataVolume, PVC, or PV resources and therefore does not consume space on your back-end storage. 

You can mount the disk read/write into the VM, but the data you write are not persistent. Every time you restart the VM, OpenShift Virtualization discards your data and restores the disk image to its initial state. 

You can use ephemeral storage when you do not need to preserve data, such as temporary files.  

## Resizing

Verify the storage class supports resizing of PVCs:

```
oc get sc
oc get sc/ocs-storagecluster-ceph-rbd-virtualization -o yaml
allowVolumeExpansion: true
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
...
```

## Creating a PV

Kubernetes does not delete the static PV when you delete the PVC. If you do not need the PV anymore, then ask your cluster administrator to delete it. If you want to create a PVC that reuses the PV, then your cluster administrator must release the PV first. To release the PV, your cluster administrator can edit the resource and remove the uid parameter from the claimRef section:

```
...
  claimRef:
    apiVersion: v1
    kind: PersistentVolumeClaim
    name: websrv1-staticimgs
    namespace: vm-project
    resourceVersion: "325655"
    # uid: d9c1a805-f1e8-4370-8d6e-c450dc9c3ef3
    ...
```

### FC

```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: fc-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  volumeMode: Block
  claimRef:
    name: dbsrv1-binlog
    namespace: vm-project
  fc:
    targetWWNs:
      - "50060e801049cfd1"
    lun: 0    
```
> Note: Implies that the cluster nodes have HBAs connected to a FC storage array

### NFS

```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-pv
spec:
  capacity:
    storage: 20Gi
  accessModes:
    - ReadWriteMany  1
  volumeMode: Filesystem  2
  claimRef:
    name: websrv1-logs
    namespace: vm-project
  nfs:  3
    path: /exports-ocp/vm135
    server: 10.20.42.42

```

### iSCSI
```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: iscsi-pv
spec:
  capacity:
    storage: 50Gi 
  accessModes:
    - ReadWriteOnce 
  volumeMode: Block 
  claimRef: 
    name: websrv1-staticimgs
    namespace: vm-project
  iscsi: 
     targetPortal: 192.168.51.40:3260
     iqn: iqn.1986-03.com.ibm:2145.disk1
     lun: 0
     initiatorName: iqn.1994-05.com.redhat:openshift-nodes

```

## Inject a Disk Image into a Volume

Use the virtctl command to inject a disk image:
```
virtctl image-upload pvc <pvcName> --image-path=./webimgfs.qcow2 --no-create
```

