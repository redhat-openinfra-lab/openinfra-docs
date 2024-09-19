# OCP General Info  

## Terminology

| Term | Definition | 
| -- | -- | 
|Operator | cluster component that simplifies the management of another application for function |  
| Resource | any configureable or consumable compenent managed by the OCP Cluster |  
| Control Plane |  cluster layer, responsible for constainer lifecycle mgmt through API |  
| Data Plane | cluster layer, responsible for providing resources required to run containers (stg, CPU, RAM) |  
| Pod | group of running containers that provide a single application, service or function |  
| Container | small executable image that defines the libraries and dependencies for an application |   

## Subscription Guide

[ODF Subscription Guide KCS](https://access.redhat.com/articles/6932811)

## What's New Slide Deck

[ODF 4.13](https://docs.google.com/presentation/d/1hvmVJl1UulEs4556j7TDray6MmXHBVIQ-anUGd06LmU/edit#slide=id.g24d38371479_0_870)

## Operator Hub (operatorhub.io)

[Operator Hub](https://operatorhub.io)

## ODF DR Offerings

[OpenShift Data Foundation Disaster Recovery Offerings](https://access.redhat.com/articles/7007419)

## Request an evaluation subscription

[Sales Assistence Product Trials](https://docs.google.com/document/d/1HnehIvH2TjOBUp86hHec1Y7d3Zff0C8LhvrsI9RmLBo/edit)

## RHEL CoreOS Versions 

[KCS Article](https://access.redhat.com/articles/6907891)

[Internal Support Streams Spreadsheet](https://docs.google.com/spreadsheets/d/1VO00pWkWf8Fr30PHl8mZFTK9ZnJO51BGXH4FT6efwp4/edit#gid=1551125754)


## Taints and Tolerations

Allow the node to control which pods should or should not be schedule on them. 

A *taint* allows a node to reguse a pod to be scheduled unless that pod has a matching *toleration*.

Taints are applied to nodes (NodeSpec) and tolerations are applied to a pod (PodSpec)

Taint:
```
apiVersion: v1
kind: Node
metadata:
  name: my-node
#...
spec:
  taints:
  - effect: NoExecute
    key: key1
    value: value1
#...
```

Toleration:
```
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
#...
spec:
  tolerations:
  - key: "key1"
    operator: "Equal"
    value: "value1"
    effect: "NoExecute"
    tolerationSeconds: 3600
#...
```

## The `virtctl` Tool

To download the virtctl tool:

```
oc get ConsoleCLIDownload virtctl-clidownloads-kubevirt-hyperconverged -o yaml
```

Use the appropriate link to the get the latest version.
```
curl https://hyperconverged-cluster-cli-download-openshift-cnv.apps.blm-ocp.hexo.lab/amd64/linux/virtctl.tar.gz --insecure --output virtctl.tar.gz
```



Access the serial or VNC consoles of a VM with the virtctl CLI.  First login to the OCP cluster with the `oc login` command.  Select the VMs project with `oc project vm-project-name` command.  Access the serial console with the `virtctl console vm-name` command.  

> Note: The escape character for the serial console is **Ctlr-]**

Access the VNC or graphical interface of a VM using `virtctl vnc vm-name`.


## Commands

### Login:
```
oc login -u user -p password https://api.blah.example.com:6443
```

### Get the console:
```
oc whoami --show-console
```

### Status of all VMs in the cluster:
```
oc get vm -A
```

### Restrict to specific namespace
```
oc get vm -n namespace
```

### Get details on VM
```
oc describe vm vm-name -n namespace
```

### List of filesystems in the VMI
```
virtctl fslist vmi-name
```

### View guest info avou the VMI OS
```
virtctl guestosinfo vmi-name
```

### SSH to master or worker nodes:
```
ssh -i ./id_rsa core@kni-master1
```

### Extract Secrets

```
oc extract secret/<secretName> -n <nameSpName>
```%                                           