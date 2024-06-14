# Live Migrations

To perform a live migration, the VM must meet the following conditions:

* The underlying persistent volume claim (PVC) must use ReadWriteMany (RWX) access mode.  
* The pod network is not configured with the bridge binding type.  
* Ports 49152 and 49153 must be available in the VM's virt-launcher pod; live migration fails if these ports are specified in a masquerade network interface.  

You must enable the sriovLiveMigration feature gate in the HyperConverged custom resource (CR) to perform a live migration for VMs attached to an SR-IOV network interface.

Default Limits:

| Key | Description | Default Value | 
| --- | --- | --- |
| parallelMigrationsPerCluster | Max migrations that can run in parallel | 5 | 
| parallelOutboundMigrationsPerNode | Max migrations moving from a node | 2 |
| bandwidthPerMigration | Limits the bandwidth; in MiB/s | 64 | 
| completionTimeoutPerGiB | If migration exceeds the defined time, in GiB/s of memory, migration is canceled. | 800 |
| progressTimeout | The migration is canceled if the memory copy fails to make progress in this time (seconds) | 150 |

## Node Selector

Kubernetes default schedule can be used or install the [Secondary Scheduler](https://redhat-openinfra-lab.github.io/openinfra-docs/How%20To/Proof%20Of%20Concept%20Topics/Openshift/secondaryScheduler/) Operator which takes load into consideration. 

 
## Affinity

Use node affinity rules to schedule a VM to run on a group of nodes. You can also specify anti-affinity rules to prevent a VM from running on a particular group of nodes. To allow more flexibility during scheduling, you can specify if the affinity rule is required for the VM to run, or preferred but not required.

```
metadata:
  name: example-vm-node-affinity
apiVersion: kubevirt.io/v1
kind: VirtualMachine
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: example.io/example-key
            operator: In
            values:
            - example-value-1
            - example-value-2
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1
        preference:
          matchExpressions:
          - key: example-node-label-key
            operator: In
            values:
            - example-node-label-value
...
```

## Tolerations

A toleration specifies that a VM can be scheduled on a node if the VM's taint matches the taints on the node. However, if a VM is configured to tolerate a taint, it is not required to be scheduled onto a node configured with said taint.

```
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: example-vm-tolerations
spec:
  tolerations:
  - key: "key"
    operator: "Equal"
    value: "virtualization"
    effect: "NoSchedule"
...
```

Scheduler Profiles:  

* LowNodeUtilization   
* HighNodeUtilization  
* NoScoring - low latency profile that strives for the quickest scheduling cycle by disabling all score plug-ins  

## Live Migration via CLI

Storage class must support RWX; patch the Kubernetes storage class if necessary:

```
oc patch -n openshift-cnv cm kubervirt-storage-class-defaults -p \
'{"data":{"nfs-storage.accessMode":"ReadWriteMany"}}'
```

```
$ oc get vmi -n <nameSp>

```
NAME                     AGE     PHASE     IP             NODENAME      READY
fedora-competent-orca    18h     Running   10.131.1.129   kni-worker2   True
rhel9-xenial-crocodile   7d21h   Running   10.128.2.247   kni-worker3   True

```
apiVersion: kubevirt.io/v1
kind: VirtualMachineInstanceMigration
metadata:
  name: migration-job
spec:
  vmiName: rhel9-xenial-crocodile
```

```
oc create -f yaml.file -n <nameSp>
```

## Node Maintenance 

Node Maintenance Resource:

```
apiVersion: nodemaintenance.kubevirt.io/v1beta1
kind: NodeMaintenance
metadata:
  name: maintenance-node2
spec:
  nodeName: node2
  reason: "Node maintenance"
```

Using the CLI

```
oc adm cordon node2
```

```
oc adm drain node2 --delete-empty-dir-data --ignore-daemonsets --force
```
> Note:

> * The --delete-emptydir-data option prevents the drain operation from failing when some pods or VMs use the local node storage as an ephemeral volume. In that case, RHOCP restarts the pod or VM on the other nodes with a new empty volume.  
* With the --ignore-daemonsets option, RHOCP skips moving the Daemon Set pods.  
* The --force option prevents the drain operation from failing when some pods that RHOCP does not manage are running on the node.  

To remove the node from maintenance mode:

```
oc adm uncordon node2
```

During node drain, OCPv performs live migrations for the VMs with the `evictionStrategy` set to LiveMigrate.  For VMs without the `evictionStrategy` parameter, OCPv shuts down the VMs and restarts them on another node.

If the `evictionStrategy` is set but the underlying storage used doesn't support live migrations, the drain process will be blocked.  For these VMs, remove the `evictionStrategy` parameter to prevent the drain process from blocking. To remove the parameter, stop the VM, navigate to the Details tab, and then scroll down to the Scheduling and resource requirements section. Remove the LiveMigrate flag from the Eviction Strategy parameter.


## Article on VMware - Change Block Tracking

https://kb.synology.com/en-us/DSM/tutorial/How_to_enable_CBT_manually_for_a_virtual_machine
