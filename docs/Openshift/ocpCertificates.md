# OCP Certificates

## Service Certificates

Use the `service-ca` controller to generate certificates for internal traffic.  To create a cert/key pair, use the `oc annotate service` command.

```
# oc annotate service serviceName service.beta.openshift.io/serving-cert-secret-name=serviceName-secret
```

Mount the secret in the app deployment.  The location is app dependent.

Sample yaml specification:
```
spec:
  template:
    spec:
      containers:
        - name: deploymentName
          volumeMounts:
            - name: volumeName
              mountPath: /location/to/mount
      volumes:
        - name: volumeName
          secret:
            defaultMode: 420
            secretName: serviceName-secret
            items:
            - key: tls.crt
              path: server.crt
            - key: tls.key
              path: private/server.key
```

To rotate a certifcate from the `service-ca`, delete the existing secret and the `service-ca` will automatically generate a new one.

```
# oc delete secret/serviceName-secret
```


## CA Bundle Certs

The `service-ca` will inject the CA certs when you apply the service.beta.openshift.io/inject-cabundle=true annotation using a configmap.

```
# oc annotate configmap ca-bundle service.beta.openshift.io/inject-bundle=true
```

> NOTE: The CA certificate is valide for 26 months by default and is automatically rotated after 13 months.  After a rotation, there is a 13 month grace period where the original CA certificate is valid to give each pod using the CA cert time to be restarted.  The new CA certificate is injected at restart.

To manually rotate the CA certificate, use the `oc delete secret` command.  This will automatically generate a new CA cert.

``` 
# oc delete secret/signing-key -n openshift-service-ca 
```

> WARNING: This process immediately invalidates the former service CA certificate.  All pods must be restarted that use it for TLS to function.

Create a standard configmap of the service-ca CA certificate; sample to mount the config map to a deployment:

```
# oc create configmap service-ca
# oc annotate configmap/service-ca service.beta.openshift.io/inject-cabundle=true
```

Update the deployment yaml to include the following:  

```
spec:
  containers: 
    ...
    volumeMounts:
      - name: trusted-ca
        path: /etc/pki/ca-trust/extracted/pem
  volumes:
    - name: trusted-ca
      configMap:
        defaultMode: 420
        name: service-ca
        items:
          key: service-ca.crt
          path: tls-ca-bundle.pem
```


