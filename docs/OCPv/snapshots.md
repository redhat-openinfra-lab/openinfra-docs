# VM Snapshots and Clones

## Snapshots
When the VM is running, OCPv communicates with the QEMU guest agent running inside the VM to quiesce the file systems before taking the snapshot. This mechanism ensures that the file systems are in a consistent state. If you do not install the QEMU guest agent, then you can still take live snapshots, but the file systems might not be consistent. The QEMU guest agent is available for Linux and Microsoft Windows systems.

If one VM disk relies on back-end storage that does not support snapshots, then OCPv displays a warning message when you take the snapshot. When you revert the snapshot, OCPv does not restore the data for that specific disk, which stays unchanged after the revert.

A VM snapshot also records the VM configuration, such as the number of CPUs or the amount of memory. If you change the VM configuration after taking the snapshot, then OCPv restores the original configuration when you revert the snapshot.

### Custom Resources

* VirtualMachineSnapshot: respresents a user request to create a snapshot, contains information about the current state of the VM.  
* VirtualMachineSnapshotContent: represents a provisioned resource on the cluster (snapshot).  Created by the VM snapshot controller and contains references to all resources required to restore the VM.  
* VirtualMachineRestore: represents a user request to restore a VM from a snapshot


### Snapshot Yaml
```
apiVersion: snapshot.kubevirt.io/v1alpha1
kind: VirtualMachineSnapshot
metadata:
  name: mariadb-server-2022-3-23
spec:
  source:
    apiGroup: kubevirt.io
    kind: VirtualMachine
    name: mariadb-server
```
### Details on Snapshot

```
oc describe vmsnapshot <snapshot_name>
```

> Note: vlues of the `status.indications` parameters can be online, GuestAgent, NoGuestAgent.

### Restore Yaml
```
oc get virtualmachinesnapshot
```

```
apiVersion: snapshot.kubevirt.io/v1alpha1
kind: VirtualMachineRestore
metadata:
  name: restore-db
spec:
  target:
    apiGroup: kubevirt.io
    kind: VirtualMachine
    name: mariadb-server
  virtualMachineSnapshotName: mariadb-server-2022-3-23

```

> Note: A VM snapshot includes the VM configuration. If you change the VM disk configuration after you have taken a snapshot, review the following considerations before restoring the snapshot:  

>  * If you removed a disk, then OpenShift Virtualization recreates the disk. If the initial disk does not support snapshots, then OCPv attaches a new blank disk with the same characteristics as the original disk. With this type of disk, OCPv does not restore the data present at the time you took the snapshot. 

>  * If you add a disk, then OCPv removes the disk when you restore the snapshot. If you used a data volume resource to create the associated PVC, by using the OpenShift web console, for example, then OCPv deletes the PVC and all its data.


### Delete Snapshots

```
oc delete virtualmachinesnapshot <snapName>
```


## Clones

Be aware that a clone will likely have machine-specific data and configuration settings from the original VM. For example, Microsoft Windows clones get the same machine Security Identifier (SID). Red Hat Enterprise Linux clones get the same SSL certificates, SSH host keys, and Red Hat subscriptions.

To prevent the clones from having these same identifying settings, you usually seal the original VM before using it to create clones. Sealing the VM is a process where you clear the machine-specific information.

OCPv provides the virtctl guestfs command to start a container that includes the virt-sysprep tool. The container also attaches the VM's PVC as the /dev/vda block device. 

```
$ virtctl guestfs -n <nameSp> <pvcName>
Use image: registry.redhat.io/container-native-virtualization/libguestfs-tools@sha256:591e...64ae
The PVC has been mounted at /dev/vda
Waiting for container libguestfs still in pending, reason: ContainerCreating, message:
...output omitted...
If you don't see a command prompt, try pressing enter.
[root@libguestfs-tools-dsk1 /]# virt-sysprep -a /dev/vda
[   0.0] Examining the guest ...
[   3.4] Performing "abrt-data" ...
[   3.4] Performing "backup-files" ...
[   4.1] Performing "bash-history" ...
...output omitted...
[   4.7] Setting a random seed
[   4.7] Setting the machine ID in /etc/machine-id
[   4.9] Performing "lvm-uuids" ...
[root@libguestfs-tools-dsk1 /]# exit
exit
[user@host ~]$
```

DataVolume section of a Virtual Machine resource:
```hl_lines="5 10 16 18"
apiVersion: kubevirt.io/v1
kind: VirtualMachine
...output omitted...
spec:
  dataVolumeTemplates:
  - metadata:
      creationTimestamp: null
      name: front1-vm1-bvle4
    spec:
      pvc:
        accessModes:
        - ReadWriteMany
        resources:
          requests:
            storage: 10Gi
        storageClassName: ocs-external-storagecluster-ceph-rbd
        volumeMode: Block
      source:
        pvc:
          name: vm1
          namespace: golden-vms
  - metadata:
      creationTimestamp: null
      name: front1-webroot-diaki
    spec:
      pvc:
        accessModes:
        - ReadWriteMany
        resources:
          requests:
            storage: 1Gi
        storageClassName: ocs-external-storagecluster-ceph-rbd
        volumeMode: Block
      source:
        pvc:
          name: webroot
          namespace: golden-vms
...output omitted...
```


### Clone VM Disk using Data Volume

The StorageProfile resource uses the cloneStrategy parameter to specify the cloning method:

* csi-clone
* snapshot
* copy; OCPv starts Kubernetes pods to perform a copy

> Note: if not specified, OCPv uses snapshots if possible, otherwise the copy method is used.

Clone a individual disk using a DataVolume resource:

```
apiVersion: cdi.kubevirt.io/v1beta1
kind: DataVolume
metadata:
  name: documentroot-clone1
spec:
  storage:
    resources:
      requests:
        storage: 1Gi
    storageClassName: ocs-external-storagecluster-ceph-rbd
  source:
    pvc:
      name: documentroot
      namespace: golden-vms
```


