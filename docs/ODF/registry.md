# Internal Registry

Storage options for the OCP Internal Registry:  


| Storage Type | ROM | RWM | Registry | Scaled Registry |  
|-----|-----|-----|-----|-----|  
| Block | yes | No | Configurable | Not Configurable |   
| File | yes | yes | Configurable | Configurable |   
| Object | yes | yes | **Recommended** | **Recommended** |  

> NOTE: Red Hat does not recommend using NFS server in production environments; tests have shown issues using NFS server on RHEL as the storage back end for OCP core services.

The `configs.imageregistry.operator.openshift.io` resource controls the configuration and desired status of the container image registry.

To configure the storage, use the storage parameter:
```
storage:
  s3:
    bucket: <bucketName>
    region: <regionName>
    regionEndpoint: <regionEndpointName>
```
> NOTE: These parameters only apply if the `spec.storage.managementState` is set to `Managed`

Create the `image-registry-private-configuration-user` secret in the `openshift-image-registry` namespace.

* REGISTRY_STORAGE_S3_ACCESSKEY
* REGISTRY_STORAGE_S3_SECRETKEY

```
oc create secret generic image-registry-private-configuration-user \
--from-literal=REGISTRY_STORAGE_S3_ACCESSKEY=myaccesskey \
--from-literal=REGISTRY_STORAGE_S3_SECRETKEY=mysecretkey \
--namespace openshift-image-registry
```

Patch the `configs.imageregistry.operator.openshift.io cluster` resource.
```
cat imageregistry-patch.yaml
---
apiVersion: imageregistry.operator.openshift.io/v1
kind: Config
metadata:
  name: cluster
spec:
  storage:
    managementState: Managed
    pvc: null
    s3:
      bucket: noobaa-review-f9911a42-3b8a-437d-a9f9-80898d97aa03
      region: us-east-1
      regionEndpoint: https://s3-openshift-storage.apps.ocp4.example.com
```


```
oc patch configs.imageregistry.operator.openshift.io cluster --type merge --patch-file imageregistry-patch.yaml
```