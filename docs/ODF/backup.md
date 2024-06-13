# Backup Restore

## OADP

Features:

* Backup - all applications and can filter the resources by type, namespace, or label.  Kubernetes objects and internal images are saved as an archive to object storage.  PVs are backed up by creating a snapshot with the CSI or cloud-native snapshot API.
* Restore - All objects in a backup can be restored or filter the objects by namespace, PV or label.
* Schedule - Can schedule backups at specified intervals.
* Hooks - can be used both pre and post backup or restore.  Restore hooks can run in an init container or in the application container.

Data Mover is now fully supported for both containerized and VM workloads.  This allows you to move CSI volume snapshots to remote object stores.  It uses Kopia as the uploader mechanism to read the snapshot data and to write to the Unified Repository.

File system backup supports both the Restic and Kopia libraries.  This is specified during the installation through the uploader-type flag (restic/kopia and defaults to kopia).  This cannot be changed after installation.

Supported S3 compatible object storage providers:

* MinIO
* MCG
* AWS S3
* IBM Cloud S3

Supported object storage providers with their own plugins:

* GCP
* Azure


## Velero CLI

```
alias velero='oc -n openshift-adp exec deployment/velero -c velero -it -- ./velero'
```

## Configure OADP DPA
Create OBC in the `openshift-adp` or `default` namespace.

Get the access/secret keys, bucket name, and bucket hostname/url
```
oc get configmap oadp-bucket -n openshift-adp -o jsonpath='{.data.BUCKET_NAME}{"\n"}'
```
```
oc get configmap oadp-bucket -n openshift-adp -o jsonpath='{.data.BUCKET_HOST}{"\n"}'
```
```
oc get secret oadp-bucket -n openshift-adp -o jsonpath='{.data.AWS_ACCESS_KEY_ID}{"\n"}' | base64 -d
```
```
oc get secret oadp-bucket -n openshift-adp -o jsonpath='{.data.AWS_SECRET_ACCESS_KEY}{"\n"}' | base64 -d
```

Install the OADP Operator; accept the default values.  

Create the `cloud credentials` secret in the openshift-adp namespace.

Create the Data Protection Application in the OADP Operator

```hl_lines="10 12 13 14 20 24-37 42"
kind: DataProtectionApplication  
apiVersion: oadp.openshift.io/v1alpha1
metadata:
  name: backup-odf
  namespace: openshift-adp
spec:
  backupLocations:
    - velero:
        config:
          insecureSkipTLSVerify: 'true'
          profile: default
          region: local
          s3ForcePathStyle: 'true'
          s3Url: https://s3-openshift-storage.apps.blm-ocp.hexo.lab
        credential:
          key: cloud
          name: cloud-credentials
        default: true
        objectStorage:
          bucket: oadp-bucket-b4991e9c-99fa-48f1-ba12-aaf6940277f4
          prefix: oadp-backups
        provider: aws
  configuration:
    nodeAgent:
      enable: true
      uploaderType: kopia
    velero:
      featureFlags: 
        - EnableCSI
      podConfig:
        resourceAllocations:
          limits:
            cpu: "1"
            memory: 1024Mi
          requests:
            cpu: 200m
            memory: 256Mi
      defaultPlugins:
        - openshift
        - aws
        - kubevirt
        - csi
      resourceTimeout: 10m
  snapshotLocations:
    - velero:
        config:
          profile: default
          region: local
        provider: aws

```

Update the VolumeSnapShotClasses ocs-storagecluster-rbdplugin-snapclass; change the deletionPolicy to `Retain` and add the `label`

```hl_lines="2 8 9"
apiVersion: snapshot.storage.k8s.io/v1
deletionPolicy: Retain
driver: openshift-storage.rbd.csi.ceph.com
kind: VolumeSnapshotClass
metadata:
  creationTimestamp: '2024-05-09T16:22:18Z'
  generation: 1
  labels:
    velero.io/csi-volumesnapshot-class: 'true'	
  managedFields:
...
parameters:
  clusterID: openshift-storage
  csi.storage.k8s.io/snapshotter-secret-name: rook-csi-rbd-provisioner
  csi.storage.k8s.io/snapshotter-secret-namespace: openshift-storage
```


## Backup Application Resources

Backup of the application resource for stateless applications:
```
oc get deployment/example -o json | jq '. | del(
    .metadata.uid,
    .metadata.selfLink,
    .metadata.generation,
    .metadata.resourceVersion,
    .metadata.creationTimestamp,
    .metadata.managedFields,
    .metadata.annotations."deployment.kubernetes.io/revision",
    .metadata.annotations."kubectl.kubernetes.io/last-applied-configuration",
    .status
)' > deployment-backup.json
```

## Backup of Stateful Applications 

Requirements:  

* Application must be stopped to backup the data  
* Application has a specialized backup tool (i.e. mysqldump)  



Scale the application to zero replicas if app cannot be paused
```
oc scale deployment/myApp --replicas=0
```

Create a job to mounts the persistent volume and copies the data to a backup location
```hl_lines="19-22 24-35"
---
apiVersion: batch/v1
kind: Job
metadata:
  name: backup
  namespace: application
  labels:
    app: backup
spec:
  backoffLimit: 1
  template:
    metadata:
      labels:
        app: backup
    spec:
      containers:
      - name: backup
        image: registry.access.redhat.com/ubi8/ubi:8.4-209
        command: 
        - /bin/bash
        - -vc
        - 'dnf -qy install rsync && rsync -avH /var/application /opt/backup'
        resources: {}
        volumeMounts: 
        - name: application-data
          mountPath: /var/application
        - name: backup
          mountPath: /opt/backup
      volumes: 
      - name: application-data
        persistentVolumeClaim:
          claimName: pvc-application
      - name: backup
        persistentVolumeClaim:
          claimName: pvc-backup
      restartPolicy: Never
```
> NOTE: The command section is used to install and execute rsync.  The volumeMounts section mounts both the original PVC and the PVC to hold the copy of the data.  The volume section references the PVC names.

```
oc apply -f backup-job.yaml
```

Scale the application back to original state when complete
```
oc scale deployment/myApp --replicas=1 
```

## Using Specialized Tool

Example using MYSQL
```
oc exec -it deployment/mariadb -- /bin/bash
...
$ mysql -u ${MYSQL_USER} -p "${MYSQL_PASSWORD}" ${MYSQL_DATABASE}
...
MariaDB [example]> FLUSH TABLES WITH READ LOCK ;
...
```

Create a job to dump the database to another PVC:
```hl_lines="20-23 25-41"
---
apiVersion: batch/v1
kind: Job
metadata:
  name: mariadb-backup
  namespace: application
  labels:
    app: mariadb-backup
spec:
  backoffLimit: 1
  template:
    metadata:
      labels:
        app: mariadb-backup
    spec:
      containers:
      - name: mariadb-backup
        image: mariadb:10.5
        workingDir: /opt/backup
        command: 
        - /bin/bash
        - -vc
        - 'mysqldump -v -h "${MYSQL_HOST}" -P "${MYSQL_PORT}" -u "root" -p"${MYSQL_ROOT_PASSWORD}" --databases "${MYSQL_DATABASE}" > backup.sql'
        resources: {}
        env: 
        - name: MYSQL_HOST
          value: mariadb.application
        - name: MYSQL_PORT
          value: "3306"
        envFrom: 
        - configMapRef:
            name: mariadb
        - secretRef:
            name: mariadb
        volumeMounts: 
        - name: backup
          mountPath: /opt/backup
      volumes:  
      - name: backup
        persistentVolumeClaim:
          claimName: pvc-backup
      restartPolicy: Never
```
> NOTE: Command section is used to run the mysqldump.  The env section is used to pass the `mariadb` service information.  The envFrom is used to pass the database name and credentials.  The volumeMounts indicate where to mount the backup PVC and the volumes section references the PVC to use.

```
oc apply -f backup-job.yaml
```