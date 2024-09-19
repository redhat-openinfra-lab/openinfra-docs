# Templates

**Boot Source**

* PVC (creates PVC): Clone an existing PVC available in the cluster to create a new PVC. 
* Registry (creates PVC): Create a new PVC by importing content from a container registry.  
* URL (creates PVC): Create a new PVC by importing content from a URL with a HTTP or S3 endpoint.  
* PXE (network boot - adds network interface): Boots an operating system stored on a server on the network (a PXE bootable network attachment definition is required).  

**Flavor** 

This field indicates the size of your VM in terms of CPU and memory and includes the following options:  

* Tiny: Creates a VM with 1 CPU and 1 GiB memory; recommended for testing VM creation.  
* Small: Creates a VM with 1 CPU and 2 GiB memory; this is the default option for any preconfigured template.  
* Medium: Creates a VM with 1 CPU and 4 GiB memory; appropriate for code testing or to store basic application resources. 
* Large: Creates a VM with 2 CPU and 8 GiB memory; recommended for systems that require a significant consumption of resources. 
* Custom: Specify custom values of CPU and memory as needed for your VM.  

**Workload** 

This field indicates the workload type for your VM and includes the following options:  

* Desktop: A configuration for a desktop system that prioritizes VM density over guaranteed VM performance. VMs with this configuration are recommended for use in the OpenShift web console.  
* Server: The default option for any preconfigured template and compatible with various server workloads. This option balances performance and prioritizes VM density over VM performance.  
* High-Performance: Optimized for high-performance or high-consumption workloads. This option prioritizes guaranteed VM performance over VM density.  

**Storage**
A preconfigured Linux-based template has two partitions by default, cloud-init and root disk. However, it is possible to configure additional disks. Each field has several available options for customizing your template:  

* Source: You can create or import a disk from an existing or blank PVC, from an external source such as a container registry or URL, or by using a container in a registry that is accessible from the cluster.  
* Type: You can customize the storage type, such as a disk or CD-ROM, according to the needs of your VM.  
* Interface: You can select the communication interface of your disk based on compatibility standards and desired performance of your VM. The available options are *virtio, SATA*, or *SCSI*. 
* Storage Class: Select the storage class for the disk. The storage profile sets the optimized access mode and the volume mode for the storage class.  
* Access mode: You can customize the disk's access mode, overriding the default storage profile settings. The available access modes are **Single user (RWO), Shared access (RWX)**, or **Read only (ROX)**.  
* Volume mode: You can choose between **Filesystem** and **Block** storage volume modes for your VM, depending upon the selected storage class.  

> Note: VMs must have a PVC with a shared ReadWriteMany (RWX) access mode to enable live migration.

**Networking**

Red Hat provides templates that are set with a default network interface connected to the pod network. You can configure additional network interfaces for your VM. For each field, you have several options available to customize your template:  

* Interface model: You can choose either `virtio` or `e1000e`, based on your VM's needs and required performance. The `virtio` model is optimized for best performance and is supported by most Linux distributions. Additional drivers are needed for Windows VMs. The `e1000e` model is supported by most operating systems, including Windows, but offers slower performance compared to the `virtio` model. 
* Network: You can select from a list of available network attachment definitions to connect to additional networks. Additional networks must use the `bridge` binding method.  
* Type: You can select from a list of available binding methods. You must use the `masquerade` binding method on the default pod network. Additional networks must use the `bridge` binding method.  
* MAC Address: You can specify a custom MAC address for the network interface. MAC addresses are automatically assigned unless you specify a custom address.  

**Cloud-init**

Available on compatible systems, you can use the `cloud-init` service to configure the guest operating system post-installation. You can use this service to install tools and packages, configure users and passwords, and manage system applications.  

## CDI and Data Volumes

The Containerized Data Importer (CDI) is an add-on that manages persistent storage for VMs in OpenShift Virtualization. CDI provides the ability to import, upload, and clone existing PVCs for your VMs. CDI provides a custom resource definition (CRD) for DataVolume objects that orchestrate the import, clone, and upload operations associated with an underlying PVC.  

```
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: vm-clone-pvc
  labels:
    kubevirt.io/vm: vm-clone-pvc
  namespace: backup-vms	
spec:
  running: false
  template:
    metadata:
      labels:
        kubevirt.io/vm: vm-clone-pvc
    spec:
      domain:
	      cpu:
          cores: 1
        devices:
          disks:
          - disk:
              bus: virtio
            name: root-disk
        resources:
          requests:
            memory: 2Gi
      terminationGracePeriodSeconds: 0
      volumes:
        - dataVolume:
            name: new-pvc-from-clone
          name: root-disk
  dataVolumeTemplates:	
    - apiVersion: cdi.kubevirt.io/v1beta1
      metadata:
        name: new-pvc-from-clone
      spec:
        pvc:
          accessModes:
          - ReadWriteOnce
          resources:
            requests:
              storage: 10Gi	
        source:
          pvc:
            namespace: storage-vm
            name: database-storage-pvc
```

Red Hat Images:

https://access.redhat.com/downloads/content/479/ver=/rhel---9/9.3/x86_64/product-software

## Custom Template Creation

To make custom templates available to all projects, create the template in the `openshift` namespace. To view the list of available templates use the `oc get templates -n openshift` command.

| Parameter | Required | Description | 
| --- | --- | --- | 
| Name | Yes | Display name. | 
| Template Provider | Yes | Specifies the author. | 
| Template Support | No | Denotes support level | 
| Description | No | Descriptive description | 
| Operating System | Yes | List of available VM OS; selecting an OS automatically configures the default flavor and workload | 
| Boot Source | Yes | Manner of access to the OS image file i.e. URL | 
| Flavor | Yes | Selection of machine resource configuration for both memory and processing requirements | 
| Workload Type | Yes | Purpose for the VM; workstation or server | 


```
kind: Template
apiVersion: template.openshift.io/v1
metadata:
  name: vm-template-example 1
  namespace: default 2
  annotations:
    name.os.template.kubevirt.io/rhel8.5: Red Hat Enterprise Linux 8.0 or higher
    description: VM template example
    iconClass: icon-rhel
    template.kubevirt.ui/parent-support-level: Full 3
    template.kubevirt.ui/parent-provider: Red Hat 4
    template.kubevirt.ui/parent-provider-url: 'https://www.redhat.com' 5
  labels:
    os.template.kubevirt.io/rhel8.5: 'true' 6
    flavor.template.kubevirt.io/tiny: 'true' 7
    workload.template.kubevirt.io/server: 'true' 8
    vm.kubevirt.io/template: rhel8-server-tiny
    vm.kubevirt.io/template.namespace: openshift
    template.kubevirt.io/type: vm
parameters:
  - description: VM name
    from: rhel8-[a-z0-9]{16}
    generate: expression
    name: NAME 9
  - description: Name of the DataSource to clone
    name: DATA_SOURCE_NAME 10
    value: rhel8
  - description: Namespace of the DataSource
    name: DATA_SOURCE_NAMESPACE 11
    value: openshift-virtualization-os-images
  - description: Randomized password for the cloud-init user cloud-user
    from: '[a-z0-9]{4}-[a-z0-9]{4}-[a-z0-9]{4}'
    generate: expression
    name: CLOUD_USER_PASSWORD 12
objects:
  - apiVersion: kubevirt.io/v1
    kind: VirtualMachine
    metadata:
      annotations:
        vm.kubevirt.io/validations: |
          [
            {
              "name": "minimal-required-memory",
              "path": "jsonpath::.spec.domain.resources.requests.memory",
              "rule": "integer",
              "message": "This VM requires more memory.",
              "min": 1610612736
            }
          ]
      labels:
        app: '${NAME}'
        vm.kubevirt.io/template: rhel8-server-tiny
        vm.kubevirt.io/template.revision: '1'
        vm.kubevirt.io/template.version: v0.19.3
      name: '${NAME}'
    spec:
      running: false
      template:
        metadata:
          annotations:
            vm.kubevirt.io/flavor: tiny
            vm.kubevirt.io/os: rhel8
            vm.kubevirt.io/workload: server
          labels:
            kubevirt.io/domain: '${NAME}'
            kubevirt.io/size: tiny
        spec:
          domain:
            cpu:
              cores: 1
              sockets: 1
              threads: 1
            devices:
              disks: 13
                - name: containerdisk
                  bootOrder: 1
                  disk:
                    bus: virtio
                - disk:
                    bus: virtio
                  name: cloudinitdisk
              interfaces: 14
                - masquerade: {}
                  name: default
                  model: virtio
              networkInterfaceMultiqueue: true
              rng: {}
            machine:
              type: pc-q35-rhel8.4.0
            resources:
              requests:
                memory: 1.5Gi
          evictionStrategy: LiveMigrate
          networks:
            - name: default
              pod: {}
          terminationGracePeriodSeconds: 180
          volumes:
            - name: containerdisk
              containerDisk:
                image: registry.redhat.io/rhel8/rhel-guest-image
            - name: cloudinitdisk
              cloudInitNoCloud: 15
                userData: |
                  #cloud-config
                  user: cloud-user
                  password: 1n4u-4qfw-wxsg
                  chpasswd:
                    expire: false
          hostname: '${NAME}'
```


