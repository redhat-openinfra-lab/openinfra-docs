# Multus 

![Multus Cluster and Public Network](../images/multus1.png)

## Planning

Homeogeneous network interfaces across all nodes  

* All storage and worker nodes need public newtork interfaces (if applicable)  
* Only nodes hosting ODF need the cluster network interfaces (if applicable)  


## Configuring Multus

Define a Multus Network using a Newtork Attachment Definition (NAD)

```
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachment Definition
metadata:
  name: stg-cluster
  namespace: openshift-storage
spec:
  config: |
    {
       "cniVersion": "0.3.1",  
       "type": "macvlan",
       "master": "enp6s0",   
       "mode": "bridge",
       "ipam": {
          "type": "whereabouts",
          "range": "10.40.0.0/24"
       }
    }
```
> NOTE: Create a second NAD if you want to isolate the frontend storage traffic (stg-public)


* namespace can be default (unspecified) or openshift-storage
* master - parent interface or interface/bond on host
* range - CIDR, unique for each network
* type - macvlan (recommended) or ipvlan
* ipam - IP Address Management; *whereabouts*, uses OCP/Kubernetes leases, or *dhcp*
  > NOTE: If there is a DNCP server, ensure Multus won't give out the same range so that multiple MAC addresses on the network can't have the same IP.


Driver Types:  

  * macvlan  
      * each connection gets a sub-interface of the parent interface with its own MAC addr  
      * each interface is isolated from the host network  
      * uses less CPU and privdes better throughput than LINUX bridge or ipvlan  
      * almost always want bridge mode  
      * near-host performance when NIC supports virtual ports/VLANs in hardware  
  * ipvlan  
      * each connection gets its own IP address and shares the same MAC addr  
      * L2 mode is analogous to macvlan bridge mode  
      * L3 mode is analogous to a router existing on the parent interface  
      * L3 mode is useful for BGP, otherwise use macvlan for reduced CPU and better throughput  
      * if NIC doesn't support VLANs in hardware, might be better than macvlan (but unlikely)  


## Pre-flight Check

[ODF Multus Validation Tool](https://access.redhat.com/articles/7014721)

Create the config file and then edit the file to add the name(s) of the NADs for the public and private networks.  Modify any other setting if necessary.

```
./rook multus validation config converged > rook.config
```

```
./rook multus validation run -c rook.config
2024/04/25 18:51:45 maxprocs: Leaving GOMAXPROCS=6: CPU quota undefined
2024-04-25 18:51:45.983978 I | rookcmd: created kube client interface from default CLI parameters
2024-04-25 18:51:45.984538 I | multus-validation: starting multus validation test with the following config:
namespace: openshift-storage
publicNetwork: stg-public
clusterNetwork: stg-cluster
resourceTimeout: 3m0s
flakyThreshold: 30s
nginxImage: quay.io/nginx/nginx-unprivileged:stable-alpine
nodeTypes:
  shared-storage-and-worker-nodes:
    osdsPerNode: 1
    otherDaemonsPerNode: 16
    placement:
      nodeSelector: {}
      tolerations: []
2024-04-25 18:51:46.078379 I | multus-validation: continuing: expected number of image pull pods not yet ready: image pull daemonset for node type "shared-storage-and-worker-nodes" expects zero scheduled pods
2024-04-25 18:51:48.084005 I | multus-validation: waiting to ensure num expected image pull pods per node type to stabilize at map[shared-storage-and-worker-nodes:3]
...
2024-04-25 18:53:22.524068 I | multus-validation: continuing: number of 'Ready' clients [41] is not the number expected [51]
2024-04-25 18:53:24.552008 I | multus-validation: continuing: number of 'Ready' clients [48] is not the number expected [51]
2024-04-25 18:53:26.582430 I | multus-validation: all 51 clients are 'Ready'

RESULT: multus validation test succeeded!

cleaning up multus validation test resources in namespace "openshift-storage"
multus validation test resources were successfully cleaned up
```

## Installing ODF

  When installing ODF Operator, on the *Security and network* screen, select the NAD definition created for Multus for the Public Network Interface or the Cluster Network Interface (or both).

## Troubleshooting

  * If pods aren't starting
    * problem with NAD definition
    * `oc pod describe` usually contains the error as an even
    * NIC may not support enough virtual ports/VLANs for the additional MAC addresses
  * If pods are starting but not communicating
      * Network design/configuration may contain errors
      * Newtork switch may be blocking sub-interface MAC addresses/IPs
      * Switch firewalling
      * System or NAD LINCU networking configuration may be blocking traffic
      * SOS report will have good info to look at next.
          * `ip_netns_exec_*_address_*` and `ip_netns_exec_*_route_*` from container namespace
    
    > NOTE: Some NICs support a limited number of virtual ports/VLANs (64 per physical) via hardware/driver.  Also, switches with port security can block unknown MAC addresses, they must support promiscuous (promisc) traffic.
