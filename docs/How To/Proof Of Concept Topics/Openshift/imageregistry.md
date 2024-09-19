# Image Registry and VDDK Configuration

Configure an S3 bucket as the image store using Noobaa.  Manage the image-registry and configure the backing store in the cluster operator.

## Create the S3 Bucket 

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
  bucketName: openshift-image-registry
EOF
```

Set up the following variables.

```
BUCKET_NAME=$(oc get obc -n openshift-image-registry openshift-image-registry -o jsonpath='{.spec.bucketName}')

AWS_ACCESS_KEY_ID=$(oc get secret -n openshift-image-registry openshift-image-registry -o yaml | grep -w "AWS_ACCESS_KEY_ID:" | head -n1 | awk '{print $2}' | base64 --decode)

AWS_SECRET_ACCESS_KEY=$(oc get secret -n openshift-image-registry openshift-image-registry -o yaml | grep -w "AWS_SECRET_ACCESS_KEY:" | head -n1 | awk '{print $2}' | base64 --decode)

ROUTE_HOST=$(oc get route s3 -n openshift-storage -o=jsonpath='{.spec.host}')
```

Create the secret.
```
oc create secret generic image-registry-private-configuration-user --from-literal=REGISTRY_STORAGE_S3_ACCESSKEY=${AWS_ACCESS_KEY_ID} --from-literal=REGISTRY_STORAGE_S3_SECRETKEY=${AWS_SECRET_ACCESS_KEY} --namespace openshift-image-registry
```

## Configure the image-registry

Extract the certificates.
```
oc extract secret/router-certs-default  -n openshift-ingress  --confirm
```
> NOTE: This will create the files tls.key and tls.crt in the current directory.

Create a configMap with the CA certificate.
```
oc create configmap image-registry-s3-bundle --from-file=ca-bundle.crt=./tls.crt  -n openshift-config
```

Update the cluster operator with the S3 bucket information and change the spec.managementState from `Removed` to `Managed`.

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
  rolloutStrategy: Recreate
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

After a few minutes the image-registry should be in an `Available` state.
```
oc get co image-registry
```
```
NAME             VERSION   AVAILABLE   PROGRESSING   DEGRADED   SINCE   MESSAGE
image-registry   4.15.31   True        False         False      5s      
```

## Download the VDDK bundle 

Pre-requisites:  

* Image Registry is configured  
* podman is installed  
* File system preserves symbolic links  


Download the VDDK tar file and extract in temporary diretory.

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

Build the VDDK image:
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


Login to the registry:
```
export TOKEN=$(oc create token builder -n openshift-mtv)
```

```
podman login --tls-verify=false -u builder -p $TOKEN default-route-openshift-image-registry.apps.blm-bmb.hexo.lab
```

Push the image to the registry.
```
podman push --tls-verify=false default-route-openshift-image-registry.apps.blm-bmb.hexo.lab/openshift-mtv/vddk:8.0.1
```

Make the image available to all namespaces using the roleBinding below.

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
