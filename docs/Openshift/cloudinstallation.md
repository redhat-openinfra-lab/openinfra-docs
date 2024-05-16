# AWS OCP 4 Cloud Installation

This page will guide you through the IPI installation of OCP 4 on AWS.

## Pre-requisites

Download the OpenShift installer from the [Install on Bare Metal with user-provisioned infrastructure](https://cloud.redhat.com/openshift/install/metal/user-provisioned) web-site.  Select the appropriate OS and architecture.  Once you have the file locally, decompress and extract the tar file. Place the ocp-installer command in a location that is in your PATH for easy access.

When running the ocp-install command, the secret json data is used when connecting to AWS.  Save this data in the user’s home directory so it can be copied and pasted when prompted.  To pull your secret json structure, click the `Copy pull secret` link.  Paste the contents into the pull-secret.txt file and save the contents in the user’s home directory.

Download the `oc` CLI binary to use after the installation is complete.  Once you have the file locally, decompress and extract the tar file. Place the `oc` command in a location that is in your PATH for easy access.


## Demo Environment

Request the AWS Blank Open Environment from the Red Hat Demo Platform.  Once the environment is built, you will receive all the pertinent information needed to configure the AWS CLI and access the AWS Console.

Once the demo environment has been provisioned, configure the AWS environment.

```
aws configure
AWS Access Key ID []: <accessKey from demo.redhat.com>
AWS Secret Access Key []: <secretKey from demo.redhat.com>
Default region name [us-east-1]: <region from demo.redhat.com>
Default output format [json]: 
```

## Create install-config.yaml 

Use the `openshift-install` command to create a default yaml file.  Use the up/down arrow keys to select an option when prompted.  When prompted for `Pull Secret`, copy and paste your pull secret json data that was obtained in the pre-requesite section.  The `install-config.yaml` file will be created in the `ocp_install` directory, or the directory specified with the –dir option.  You can modify this file if you need to change the architecture type, number of replicas, or networking information.  

```
./openshift-install --dir ocp_install --log-level debug create install-config
```

## Install OCP 4

Ensure that you are not in the `ocp_install` directory and that the `openshift-install` command is in your path.  Run the installation using the `openshift-install` command.  You can specify the log-level as info, warn, debug, or error.

```
openshift-install --dir ocp_install --log-level debug create cluster
```

Once the installation is complete, generally around 35-40 minutes, the link to the OpenShift console is provided along with the KUBECONFIG file to export when using the `oc` command.

```
...
DEBUG Still waiting for the cluster to initialize: Cluster operators authentication, console, image-registry, monitoring, openshift-samples are not available 
DEBUG Still waiting for the cluster to initialize: Cluster operators authentication, console, monitoring, openshift-samples are not available 
DEBUG Still waiting for the cluster to initialize: Cluster operators authentication, console, monitoring are not available 
DEBUG Still waiting for the cluster to initialize: Cluster operators authentication, console, monitoring are not available 
DEBUG Still waiting for the cluster to initialize: Cluster operators authentication, console are not available 
DEBUG Still waiting for the cluster to initialize: Cluster operators authentication, console are not available 
DEBUG Still waiting for the cluster to initialize: Cluster operator authentication is not available 
DEBUG Cluster is initialized                       
INFO Checking to see if there is a route at openshift-console/console... 
DEBUG Route found in openshift-console namespace: console 
DEBUG OpenShift console route is admitted          
INFO Install complete!                            
INFO To access the cluster as the system:admin user when using 'oc', run 'export KUBECONFIG=/Users/bmclaren/ocp.0124/ocp_install/auth/kubeconfig' 
INFO Access the OpenShift web-console here: https://console-openshift-console.apps.blm-aws.sandbox2258.opentlc.com 
INFO Login to the console with user: "kubeadmin", and password: "gqYnD-yXtEL-yiq6f-xJofS" 
DEBUG Time elapsed per stage:                      
DEBUG            cluster: 4m52s                    
DEBUG          bootstrap: 38s                      
DEBUG Bootstrap Complete: 10m57s                   
DEBUG                API: 1m48s                    
DEBUG  Bootstrap Destroy: 2m31s                    
DEBUG  Cluster Operators: 10m17s                   
INFO Time elapsed: 29m27s                         
```

## Create Dedicated ODF Nodes

### Create Machine Sets for ODF

Login to the ODF Console GUI.  Navigate to Compute, Machine Sets.  Download each of the existing machine sets.  These will be used as templates for the new machine sets.  Click the machine name, select YAML from the top menu bar and then click the Download link in the lower right corner.  Alternatively, you can use the `oc get machineset/<machineSetName> -n openshift-machine-api -o yaml` command and save the output to a file.  

For each machine YAML files, make the following edits:

* Delete the highlighted line:
```hl_lines="4-9 10 15 16"
apiVersion: machine.openshift.io/v1beta1
kind: MachineSet
metadata:
  annotations:
    capacity.cluster-autoscaler.kubernetes.io/labels: kubernetes.io/arch=amd64
    machine.openshift.io/GPU: "0"
    machine.openshift.io/memoryMb: "16384"
    machine.openshift.io/vCPU: "4"
  creationTimestamp: "2024-01-11T18:40:17Z"
  generation: 1
  labels:
    machine.openshift.io/cluster-api-cluster: blm-aws-grv6f
  name: blm-aws-grv6f-worker-us-east-2a
  namespace: openshift-machine-api
  resourceVersion: "25011"
  uid: 3ba3aa6e-f751-414c-bc8a-5c2e44b04bac

```

* Modify the replicas from 1 to 0
```
spec:
  replicas: 0
```

* Modify the name from worker to storage:
```hl_lines="1 7 12"
name: blm-aws-grv6f-storage-us-east-2a
spec:
  replicas: 0
  selector:
    matchLabels:
      machine.openshift.io/cluster-api-cluster: blm-aws-grv6f
      machine.openshift.io/cluster-api-machineset: blm-aws-grv6f-storage-us-east-2a
  template:
    metadata:
      labels:
        machine.openshift.io/cluster-api-cluster: blm-aws-grv6f
        machine.openshift.io/cluster-api-machineset: blm-aws-grv6f-storage-us-east-2a
```

* Update the metadata section to add the labels and the taints:
```hl_lines="12-17"
spec:
  replicas: 0
  selector:
    matchLabels:
      machine.openshift.io/cluster-api-cluster: blm-aws-grv6f
      machine.openshift.io/cluster-api-machineset: blm-aws-grv6f-storage-us-east-2a
  template:
    metadata:
      labels:
        machine.openshift.io/cluster-api-cluster: blm-aws-grv6f
        machine.openshift.io/cluster-api-machineset: blm-aws-grv6f-storage-us-east-2a
        node-role.kubernetes.io/infra: ''
        cluster.ocs.openshift.io/openshift-storage: ''
      taints:
        - effect: NoSchedule
          key: node.ocs.openshift.io/storage
          value: true
```

* Update the instanceType to m5a.4xlarge; this satisfies the minimum requirements for ODF and is the lowest cost EC2 instance.
```hl_lines="3"
          iamInstanceProfile:
            id: blm-aws-grv6f-worker-profile
          instanceType: m6i.4xlarge
```

* Remove the status section from the bottom of the file:
```
status:
  availableReplicas: 1
  fullyLabeledReplicas: 1
  observedGeneration: 1
  readyReplicas: 1
  replicas: 1
```

Once all the edits are complete to all downloaded yaml files, access the OpenShift console, navigate to *Compute*, *Machine Sets*, click the `Create MachineSet` icon.  Copy/paste the contents of the first file and click `Create`.  Rinse and repeat for each of the yaml files.

After all three new machine sets are created, (one per AWS zone), modify the *Machine Set Count* to 1 to create the new EC2 instance in each zone.  Click the elipses on the far right of the machine set, select *Edit Machine Count*.  In the dialog box, set the count to 1, click the `Save` icon.
> NOTE: Monitor the *Compute* , *Machines* section to get the status of the builds.  Once they are complete, the status will be *Provisioned as node*.

## Install ODF and Create Storage Cluster

To install the ODF Operator, select *Operator Hub, Operators* from the left pane.  In the search text box, input ODF.  Select *OpenShift Data Foundation*.  On the pop-up Window, click `Install`.  On the install Operator window, scroll to the bottom and click the `Install` icon.

After a few minutes, the installation will complete and a message will be displayed with the `Create StoraegSystem` icon; click it to create the ODF Storage Cluster.

On the *Backing Storage* screen, accept the defaults and click `Next`.  

On the *Capacity and nodes* screen, you can modify the size of the volumes using the drop-down menu.  Select the nodes that are labeled as *Infra*.  There should be three.  Scroll to the bottom of the screen and click teh Taint Nodes checkbox.  Click the `Next` icon.

On the *Security and network* screen, take the defaults and click `Next`.

Review the final configuration on the *Review and create* screen.  Click `Create StorageSystem` ot create or `Back` to make modifications.
> NOTE: It will take time to create the StorageSystem.  Once it is complete, the status on the *Overview* page will show green for Storage.

## Post Install Configuration

### Default StorageClass

Update the default storage class to ocs-storagecluster-ceph-rbd.  Use the `oc patch` command or edit the StorageClass YAML in the GUI.

```
oc patch storageclass gp3 -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class": "false"}}}'
```

```
oc patch storageclass ocs-storagecluster-ceph-rbd -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class": "true"}}}'
```

```hl_lines="10"
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: ocs-storagecluster-ceph-rbd
  uid: ce6ff323-544b-44ae-b88b-fed208aa901e
  resourceVersion: '74999'
  creationTimestamp: '2024-01-11T20:32:29Z'
  annotations:
    description: 'Provides RWO Filesystem volumes, and RWO and RWX Block volumes'
    storageclass.kubernetes.io/is-default-class: 'true'
```

### Health Checks

Using the yaml file below, add health checks to the worker and storage nodes.  Update the `name` and the `matchLabels` parameters to match the node/machine name, execute the `oc apply` command to apply.  Rinse and repeat for each node/machine.

```hl_lines="4 9"
apiVersion: machine.openshift.io/v1beta1
kind: MachineHealthCheck
metadata:
  name: worker-2c 
  namespace: openshift-machine-api
spec:
  selector:
    matchLabels:
      machine.openshift.io/cluster-api-machineset: blm-aws-grv6f-worker-us-east-2c
  unhealthyConditions:
  - type:    "Ready"
    timeout: "300s" 
    status: "False"
  - type:    "Ready"
    timeout: "300s" 
    status: "Unknown"
  maxUnhealthy: "40%" 
```