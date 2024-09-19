# Kubernetes HA for VMs

Hypervisors

* Baremetal - executes on the host's hardware as a lyaer of a lightweight operating system.  
* Hosted Hypervisor - executes on the host's operating system; allowing you to use the host for purposes other than virtualization.

Kubernetes Features for VMs

* Load balancing using the service/route features
* Readiness/Liveness Probes
* Sticky sessions
* Watchdog devices - monitor OS and do not detect application features
* Machine health checks - only available for clusters that are installed as baremetal IPI
* Live migration
* VM run strategies - .spec.running or .spec.runStrategy
* Fencing nodes - reboots/deletes `Machine` CRDs to solve problems with automatically provisioned nodes

## Load Balance

Service resource definition:

```
apiVersion: v1
kind: Service
metadata:
  name: backend 
  namespace: prod2 
spec:
  type: ClusterIP
  selector:
    tier: backend 
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
```

Route resource definition:

```hl_lines="7 9 14"
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: helloworld
  namespace: ha-loadbalance
  annotations:
    router.openshift.io/cookie_name: "hello"
spec:
  host: hello-ha-loadbalance.apps.ocp4.example.com
  port:
    targetPort: 80
  to:
    kind: Service
    name: helloworld
```
> Note: Annotating a route with the router.openshift.io/cookie_name="cookie_name" annotation creates a custom-named cookie for the route.

## Update Run Strategy

Edit the VM resource to modify the behavior.

`RerunOnFailure` restarts the VM when it fails; VM not restarted when OS is gracefully shutdown.  
`Always` ensures the VM is always running.  
`Halted` ensure the VM is not running.  
`Manual` performs no automatic actions.   

> Note: the .spec.runing or .spec.runStrategy are mutually exclusive.

## Configure Health Probes

* HTTP Get - check is successful if response code is 200-399
* TCP Socket - establishes a connection to the app's TCP port

Readiness Probe using HTTP Get:

```hl_lines="13-21"
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
...output omitted...
spec:
  ...output omitted...
  template:
    metadata:
    ...output omitted...
    spec:
      domain:
        ...output omitted...
      readinessProbe:
        httpGet:
          path: /health
          port: 8080
        initialDelaySeconds: 120 
        periodSeconds: 20
        timeoutSeconds: 10
        failureThreshold: 3
        successThreshold: 3
      evictionStrategy: LiveMigrate
...
```

Liveness Probe using TCP Socket:

```hl_lines="13-17"
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
...output omitted...
spec:
  ...output omitted...
  template:
    metadata:
    ...output omitted...
    spec:
      domain:
        ...output omitted...
      livenessProbe:
        tcpSocket: 1
          port: 3306
        initialDelaySeconds: 120
        periodSeconds: 20
      evictionStrategy: LiveMigrate
...
```

## Configure Watchdog Device

Add the emulated watchdog device to your VM resource to activate watchdog monitoring. Use the oc edit command to add a new device section to the VM resource:

```hl_lines="26-29"
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
...output omitted...
spec:
  ...output omitted...
  template:
    metadata:
    ...output omitted...
    spec:
      domain:
        cpu:
          ...output omitted...
        devices:
          disks:
          - bootOrder: 1
            disk:
              bus: virtio
            name: mariadb-server
          interfaces:
          - macAddress: "02:48:09:00:00:00"
            masquerade: {}
            name: default
          networkInterfaceMultiqueue: true
          rng: {}
          watchdog:
            i6300esb:
              action: poweroff
            name: mywatchdog
...
```
> Note:   
> 1. OCPv can only emulate the Intel 6300ESB chipset.  
2. The `action` parameter can be set to poweroff, reset, or shutdown.

## Machine Health Checks

Limitations:

* Control plane hosts are not currently supported and are not remediated if they are unhealthy.
* Only hosts owned by a machine set are remediated by a machine health check.
* If the node for a host is removed from the cluster, the machine health check identifies that the host is unhealthy and remediates it immediately.
* A host with the master role cannot be remediated.
* A host is remediated immediately if the Machine resource enters a Failed status.
* For cloud environments, a machine health check relies on cloud provider integration for the machine to forcibly reboot, reprovision, and rejoin the cluster.

