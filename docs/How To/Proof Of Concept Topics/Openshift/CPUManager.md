# Configure CPU Pinning

This is achieved within Openshift using CPU Manager.

## [Official Documentation](https://docs.openshift.com/container-platform/4.14/scalability_and_performance/using-cpu-manager.html)
[OCP Virtualization KB](https://access.redhat.com/solutions/6996014)

## Process

Edit the machine config pool

`oc edit machineconfigpool worker`

Add the custom-kubelet identifier for cpumanager:

```
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfigPool
metadata:
  creationTimestamp: "2023-05-06T13:32:03Z"
  generation: 2
  labels:
    custom-kubelet: cpumanager-enabled
```

Create a Kubelet Configuration for CPU Manager:

```
export KUBECONFIG=~/ocp-trial/kubeconfig
cat << EOF > cpumanager-kubeletconfig.yaml
apiVersion: machineconfiguration.openshift.io/v1
kind: KubeletConfig
metadata:
  name: cpumanager-enabled
spec:
  machineConfigPoolSelector:
    matchLabels:
      custom-kubelet: cpumanager-enabled
  kubeletConfig:
     cpuManagerPolicy: static
     cpuManagerReconcilePeriod: 5s
EOF
oc create -f ./cpumanager-kubeletconfig.yaml
```

Once this configuration is rolled out to the nodes, a Virtual Machine can utilize pinned CPU’s by editing the Virtual Machine YAML and adding the following:

```
spec:
  template:
    spec:
      domain:
        cpu:
          dedicatedCpuPlacement: true
          isolateEmulatorThread: true <---- this is optional, and will
                                             reserve an extra CPU to run
                                             qemu threads, likely best
                                             for I/O intensive workloads
```

Obviously remove the “<---- comment”. While you can configure `isolateEmulatorThreads` to `true`, you will lose the ability to **LiveMigrate**. This is being tracked in this issue: [https://issues.redhat.com/browse/CNV-28774](https://issues.redhat.com/browse/CNV-28774)
