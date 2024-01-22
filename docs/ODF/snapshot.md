# Snapshots and Clones

Volumes Clones are created by referencing the source PVC in the dataSource parameter in the yaml file of a PVC:
```hl_lines="13-15"
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata: 1
  name: postgresql-data-clone
  namespace: backup-volume
  labels:
    app: postgresql-data-clone
spec:
  accessModes: 2
    - ReadWriteOnce
  storageClassName: ocs-storagecluster-ceph-rbd 3
  dataSource: 4
    kind: PersistentVolumeClaim
    name: postgresql-data
  resources:
    requests:
      storage: 1Gi
```

Snapshots are created by referencing the source PVC in the persistentVolumeClaimName parameter and specifying the volume snapshot class for Ceph RBD or CephFS.
```hl_lines="7 9"
---
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: postgresql-data 
spec:
  volumeSnapshotClassName: ocs-storagecluster-rbdplugin-snapclass 
  source:
    persistentVolumeClaimName: postgresql-data 
```

Once the snapshot is created, a new PVC can be created.
```hl_lines="13-16"
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata: 1
  name: postgresql-data-restore
  namespace: backup-volume
  labels:
    app: postgresql-data-snapshot
spec:
  accessModes: 2
  - ReadWriteOnce
  storageClassName: ocs-storagecluster-ceph-rbd 3
  dataSource: 4
    apiGroup: snapshot.storage.k8s.io
    kind: VolumeSnapshot
    name: postgresql-data
  resources:
    requests:
      storage: 1Gi
```
