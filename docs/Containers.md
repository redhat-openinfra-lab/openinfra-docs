# OCP

## Download Link

[RHOCP Installers](https://developers.redhat.com/products/openshift/download)
[Product Documentation]( https://access.redhat.com/documentation/en-us/openshift_container_platform/4.12)

## Control Plane Components

| Component | Description | 
| --- | --- | 
| etcd | Distributed key-value database that stores cluster config details | 
| kube-apiserver | Front-end server that exposes the Kubernates API | 
| kube-scheduled | Watcher service that determins an available compute node for new pod requests | 

## Compute Plane Components

| Component | Description | 
| --- | --- | 
| kubelet | Main agent on each cluster compute node, responsible for executing pod requests that come from API and scheduler |
| kube-proxy | Provides network configuration adn communication for pods on a node. | 
| cri-o | CRI-O Engine, represents a small OCI-compliant runtime engine. |  
> NOTE: CRI (Container Runtime Interface) is a plug-in interface that provides configurable communication betwween kubelet and pod config requests | 

## OCP CLI

Login
```
# oc login -u user -p password https://api.ocp4.example.com:6443
```

Retrieve the Web-Console Link:
```
# oc whoami --show-console
```

Get Kubernates control plane URL:
```
# oc cluster-info
```

Get nodes in the cluster:
```
# oc get nodes
```

List of installed operators:
```
# oc get clusteroperator
```

Get resources:
```
# oc get [all|*resource_type*] [ -n namespace ]
```

Describe a resource:
```
# oc describe *resouce_type* *resource_name*
```

Get API-Resources:
```
# oc api-resources [ --api-group='' ]
```

Deployments:
```
# oc get deploy
# oc get deploy *name* -o wide
# oc describe deployment *name*
```

Describe a POD:
```
# oc get pods
NAME                    READY   STATUS      REASTARTS   AGE
myapp-77fb5cd997-xr889  1/1     Running     0           13m
# oc describe pod myapp-77fb5cd997-xr889 
Name:               myapp-77fb5cd997-xr889 
Namespace:          cli-resources
Priority:           0
Service Account:    default
...
```
> NOTE: Use the oc get pod with the -o yaml to format the output in YAML format; this yields more details about a resource than the `describe` option.


Explain the fields of an API resource:
```
# oc explain .pods.spec.containers.resources [--recursive]
```

Delete a resource:
```
# oc delete pod quotes-ui
```
> NOTE: When deleting managed resources, such as pods, results in the automatic creation of new instances of those resources.  When a project is deleted, it deletes all the resources and applications within it.





## Containers

| Options | Description | 
| --- | --- |
| -t | --tty meaning pseudo-tty | 
| -i | --interactive; standard input is kep open into the container | 
| -d | --detach; run the container in the background | 
| -e | environment variables | 



List images that have been downloaded locally:
```
# podman images
```

Run the container in the background:
```
# podman run -d -p 8080 registry.redhat.io/rhel8/httpd-24
```



## Appendix

| Acronym | Description | 
| --- | --- |
| S2I | Source-to-image feature uses a BuildConfig to build a container image from application source code that is stored in a Git Repo. | 


### JSON Formatting


Similar to JSON styled queries, use the -o custom-columns option:
```
$ oc get pods \
-o custom-columns=PodName:".metadata.name",\
ContainerName:"spec.containers[].name",\
Phase:"status.phase",\
IP:"status.podIP",\
Ports:"spec.containers[].ports[].containerPort"
PodName                  ContainerName   Phase     IP          Ports
myapp-77fb5cd997-xplhz   myapp           Running   10.8.0.60   <none>
```

JSONPath expression:
```
$ oc get pods  \
-o jsonpath='{range .items[]}{"Pod Name: "}{.metadata.name}
{"Container Names:"}{.spec.containers[].name}
{"Phase: "}{.status.phase}
{"IP: "}{.status.podIP}
{"Ports: "}{.spec.containers[].ports[].containerPort}
{"Pod Start Time: "}{.status.startTime}{"\n"}{end}'
Pod Name: myapp-77fb5cd997-xplhz
Container Names:myapp
Phase: Running
IP: 10.8.0.60
Ports:
Pod Start Time: 2023-03-15T18:45:40Z
```
