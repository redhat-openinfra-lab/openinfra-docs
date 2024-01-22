# Backup Restore

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

Require:
* application must be stopped to backup the data
* application has a specialized backup tool (i.e. mysqldump)

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