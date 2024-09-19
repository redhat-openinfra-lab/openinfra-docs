# Image Registry and VDDK Configuration

For OCPv, it is highly recommended to create the VDDK container.  The VMware Virtual Disk Development kit accelerates the transferring of virtual disk from vSphere to OpenShift.  Once the container image is built with the VDDK, it gets pushed to the image-registry.

> NOTE: Per our documentation, storing the VDDK image in a public registry might violate the VMware license terms.  

Configure an S3 bucket as the image store using Noobaa.  Manage the image-registry and configure the backing store in the cluster operator.

## Create the S3 Bucket 

Configure an S3 bucket to be used as the backing store to the internal image-registry.  

Create a bucket in the `openshift-image-registry` namespace.  Once you have the bucket created, create a secret with the credentials.

```
cat <<EOF | oc apply -f -
apiVersion: objectbucket.io/v1alpha1
kind: ObjectBucketClaim
metadata:
  name: openshift-image-registry
  namespace: openshift-image-registry
spec:
  storageClassName: openshift-storage.noobaa.io
  bucketName: image-registry-bucket
EOF
```

Set up the following variables.

```
BUCKET_NAME=$(oc get obc -n openshift-image-registry image-registry-bucket -o jsonpath='{.spec.bucketName}')

AWS_ACCESS_KEY_ID=$(oc get secret -n openshift-image-registry image-registry-bucket -o yaml | grep -w "AWS_ACCESS_KEY_ID:" | head -n1 | awk '{print $2}' | base64 --decode)

AWS_SECRET_ACCESS_KEY=$(oc get secret -n openshift-image-registry image-registry-bucket -o yaml | grep -w "AWS_SECRET_ACCESS_KEY:" | head -n1 | awk '{print $2}' | base64 --decode)

ROUTE_HOST=$(oc get route s3 -n openshift-storage -o=jsonpath='{.spec.host}')
```

Create the _image-registry-private-configuration-user_ secret in the _openshift-image-registry_ namespace.
```
oc create secret generic image-registry-private-configuration-user --from-literal=REGISTRY_STORAGE_S3_ACCESSKEY=${AWS_ACCESS_KEY_ID} --from-literal=REGISTRY_STORAGE_S3_SECRETKEY=${AWS_SECRET_ACCESS_KEY} --namespace openshift-image-registry
```

## Configure the image-registry

Manage the image-registry and configure the backing store in the cluster operator.

To enable https, extract the external route certificate.
```
oc extract secret/router-certs-default  -n openshift-ingress  --confirm
```
> NOTE: This will create the files tls.key and tls.crt in the current directory.

Create a configMap with the CA certificate using the tls.crt file.
```
oc create configmap image-registry-s3-bundle --from-file=ca-bundle.crt=./tls.crt  -n openshift-config
```

Update the cluster operator with the S3 bucket information and change the _spec.managementState_ from `Removed` to `Managed` and the _spec.replicas_ from `1` to `2`.  Make sure to use the full bucket name and the route contained in the $BUCKET_NAME and $ROUTE_HOST variables obtained from above.  The _spec.storage.s3.trustedCA.name_ the name of the _configMap_ created in the previous step.

> NOTE: Using an S3 bucket as the backing store allows us to configure multiple replicas and provide high availability to the image-registry service.

```
oc edit config.imageregistry.operator.openshift.io cluster
```

```hl_lines="4 8 16-23"
spec:
  httpSecret: 04c54e5a8e787ce951a7f5b899bf08d88983075c894d2a40cd1096fb75f852a033e7e1ab536544dd5996938a9d7e160eecbca89a71f004e89824b3428bbd46ff
  logLevel: Normal
  managementState: Managed
  observedConfig: null
  operatorLogLevel: Normal
  proxy: {}
  replicas: 2
  requests:
    read:
      maxWaitInQueue: 0s
    write:
      maxWaitInQueue: 0s
  rolloutStrategy: RollingUpdate
  storage:
    managementState: Unmanaged
    s3:
      bucket: image-registry-bucket-af047e2e-f030-4775-b255-89b9ab7f85b5
      region: us-east-1
      regionEndpoint: https://s3-openshift-storage.apps.blm-bmb.hexo.lab
      trustedCA:
        name: image-registry-s3-bundle
      virtualHostedStyle: false
  unsupportedConfigOverrides: null
```

> NOTE: If the image-registry was already in a `Managed` state, modify the _spec.rolloutStrategy_ from `RollingUpdate` to `Recreate`.

After a few minutes the image-registry should be in an `Available` state.
```
oc get co image-registry
```
```
NAME             VERSION   AVAILABLE   PROGRESSING   DEGRADED   SINCE   MESSAGE
image-registry   4.15.31   True        False         False      5s      
```

## Create the VDDK Container 

Pre-requisites:  

* Image Registry is configured  
* podman is installed  
* File system preserves symbolic links  


[Download the VDDK](https://developer.vmware.com/web/sdk/8.0/vddk) tar file and extract in temporary diretory.

> NOTE: You must have a valid login to Broadcom and the appropriate support or access level.  The customer should have this with their VMware support.

Make sure you are logged in to the OCP cluster or have exported the `KUBECONFIG` variable.

```
tar -xzf VMware--vix-disklib-8.0.1-21562716.x86_64.tar.gz
```
> NOTE: this will create the vmware-vix-disklib-distrib directory in your current directory.

Create a Dockerfile:
```
cat > Dockerfile <<EOF
FROM registry.access.redhat.com/ubi8/ubi-minimal
USER 1001
COPY vmware-vix-disklib-distrib /vmware-vix-disklib-distrib
RUN mkdir -p /opt
ENTRYPOINT ["cp", "-r", "/vmware-vix-disklib-distrib", "/opt"]
EOF

```

Build the VDDK image.  Make sure to use the URL of the image-registry route for your cluster.
```
podman build . -t default-route-openshift-image-registry.apps.blm-bmb.hexo.lab/openshift-mtv/vddk:8.0.1
```
> NOTE: You must include the namespace in the image path.

Verify your image:
```
podman image ls
```

```
REPOSITORY                                                                       TAG         IMAGE ID      CREATED        SIZE
default-route-openshift-image-registry.apps.blm-bmb.hexo.lab/openshift-mtv/vddk  8.0.1       55152a0e1b49  7 seconds ago  173 MB
registry.access.redhat.com/ubi8/ubi-minimal                                      latest      12c4198317ec  4 weeks ago    93.9 MB
```


Login to the registry.  Make sure to use the URL of the image-registry route for your cluster.
```
export TOKEN=$(oc create token builder -n openshift-mtv)
```

```
podman login --tls-verify=false -u builder -p $TOKEN default-route-openshift-image-registry.apps.blm-bmb.hexo.lab
```

Push the image to the registry using the URL of the image-registry route for your cluster and full path to the image.
```
podman push --tls-verify=false default-route-openshift-image-registry.apps.blm-bmb.hexo.lab/openshift-mtv/vddk:8.0.1
```

The image is only available to the openshift-mtv namespace but will be used by all namespaces the VMs are migrated to.  You can make the image available to all namespaces but creating a new roleBinding.

```
oc apply -f image-puller-rb.yaml
```

```
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  annotations:
    openshift.io/description: Allows all pods in all namespaces to pull images from
      this namespace.
  name: allow-image-pullers
  namespace: openshift-mtv
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:image-puller
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: Group
  name: system:authenticated
```

When you create your VMware provider, you can use the image-registry _service_ URL and port along with the full path to the VDDK image.

<img src="/images/vddk.png" alt="drawing" width="800"/>