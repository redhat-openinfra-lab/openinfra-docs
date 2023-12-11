# Configure Affinity / Anti-Affinity Rules

## [Official Documentation](https://docs.openshift.com/container-platform/4.14/virt/virtual_machines/advanced_vm_management/virt-specifying-nodes-for-vms.html)

## Process

For the some applications, a customer may want to ensure that the specific Virtual Machines are never executed on the same underlying host as other Virtual Machines.  To accomplish this, an anti-affinity rule may be created.

### Label the Virtual Machine (Pod)

Each Virtual Machine needed a label applied to the pod.  To do this, edit the Virtual Machine and add a label to it.  In our example here, we will have two different identifiers, AS and MS nodes.  For this rule, we do not want any AS nor MS Virtual Machine to run on the same host as any other AS or MS machine:

```
spec:
  template:
    metadata:
      labels:
        customer_node_type: AS
```

The label won’t appear on the pod until the Virtual Machine is restarted. All associated Virtual Machines should be labelled first.

### Apply Affinity Rules

Add the Anti-Affinity rule to the Virtual Machine:

```
spec:
  template:
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution: 
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: customer_node_type
                  operator: In
                  values:
                  - AS
                  - MS
              topologyKey: kubernetes.io/hostname
```

Restart the Virtual Machine. This will have the label take effect as well as the Anti-Affinity Rule.
