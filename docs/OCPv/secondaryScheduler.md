# Secondary Scheduler

The kube-scheduler binary includes multiple plugins.  The plugins are configured by creating one or more scheduler profiles.  The example below uses Trimaran; a collection of load-aware plugins.

* TargetLoadPacking: Implements a packing policy up to a configured CPU utilization, then switches to a spreading policy among the hot nodes. (Supports CPU resource.)  
* LoadVariationRiskBalancing: Equalizes the risk, defined as a combined measure of average utilization and variation in utilization, among nodes. (Supports CPU and memory resources.)  
* LowRiskOverCommitment: Evaluates the performance risk of overcommitment and selects the node with lowest risk by taking into consideration (1) the resource limit values of pods (limit-aware) and (2) the actual load (utilization) on the nodes (load-aware). Thus, it provides a low risk environment for pods and alleviate issues with overcommitment, while allowing pods to use their limits.  


## Install the Secondary Scheduler Operator

Create a namespace to install the secondary scheduler in.

Install the Secondary Scheduler from Operator Hub in the newly created namespace.

Before creating an instance of the secondary scheduler, create the config map.

```
apiVersion: kubescheduler.config.k8s.io/v1beta3
kind: KubeSchedulerConfiguration
leaderElection:
  leaderElect: false
profiles:
  - schedulerName: secondary-scheduler
    plugins:
      score:
        disabled:
          - name: NodeResourcesBalancedAllocation
          - name: NodeResourcesLeastAllocated
        enabled:
          - name: TargetLoadPacking
    pluginConfig:
      - name: TargetLoadPacking
        args:
          defaultRequests:
            cpu: "2000m"
          defaultRequestsMultiplier: "1"
          targetUtilization: 70
          metricProvider:
            type: Prometheus
            address: ${PROM_URL}
            token: ${PROM_TOKEN}
```
> NOTE:  Update the PROM_URL and the PROM_TOKEN with the appropriate values.  
> ```
> PROM_URL=https://`oc get routes prometheus-k8s -n openshift-monitoring -o json |jq ".status.ingress"|jq ".[0].host"|sed 's/"//g'`  
> PROM_TOKEN=`oc create token prometheus-k8s -n openshift-monitoring`
> ```

Apply the new configMap:
```
oc create -n openshift-secondary-scheduler-operator configmap secondary-scheduler-config --from-file=config.yaml
```

Create an instance of the secondary scheduler

```
apiVersion: operator.openshift.io/v1
kind: SecondaryScheduler
metadata:
  name: cluster
  namespace: openshift-secondary-scheduler-operator
spec:
  logLevel: Normal
  managementState: Managed
  operatorLogLevel: Normal
  schedulerConfig: secondary-scheduler-config
  schedulerImage: 'registry.k8s.io/scheduler-plugins/kube-scheduler:v0.28.9'
```
> NOTE: See [scheduler-plugin-images](https://github.com/kubernetes-sigs/scheduler-plugins#compatibility-matrix) for container image compatibility.


## Schedule VM using the Secondary Scheduler

Update the VMs yaml:
```hl_lines="30"
...
spec:
  dataVolumeTemplates:
    - apiVersion: cdi.kubevirt.io/v1beta1
      kind: DataVolume
      metadata:
        creationTimestamp: null
        name: centos-stream9-blue-snipe-20
      spec:
        sourceRef:
          kind: DataSource
          name: centos-stream9
          namespace: openshift-virtualization-os-images
        storage:
          resources:
            requests:
              storage: 30Gi
  running: true
  template:
    metadata:
      annotations:
        vm.kubevirt.io/flavor: small
        vm.kubevirt.io/os: centos-stream9
        vm.kubevirt.io/workload: server
      creationTimestamp: null
      labels:
        kubevirt.io/domain: centos-stream9-blue-snipe-20
        kubevirt.io/size: small
    spec:
      schedulerName: secondary-scheduler
      architecture: amd64
      domain:
...
```
> NOTE: The VM must be restarted for the new scheduler to kick in.

Verify VM is using the secondary-scheduler:
```
oc describe pod/virt-launcher-centos-stream9-secondary-scheduler-vm1-r275d -n blm-project
```

Example output:
```
...
Events:
  Type    Reason                 Age   From                 Message
  ----    ------                 ----  ----                 -------
  Normal  Scheduled              105s  secondary-scheduler  Successfully assigned blm-project/virt-launcher-centos-stream9-secondary-scheduler-vm1-r275d to kni-worker2
...
```

## Helpful Links

[How to bring your own scheduler into OpenShift](https://www.redhat.com/en/blog/how-to-bring-your-own-scheduler-into-openshift-with-the-secondary-scheduler-operator)


[Trimaran Scheduler GitHub repo](https://github.com/kubernetes-sigs/scheduler-plugins/blob/master/pkg/trimaran/README.md)



