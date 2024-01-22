# Quotas and Permisstions

## RBAC 

Default Roles

| Roles | Description | 
| ----- | ----- | 
| admin | Can manage all project resources, including granting permissions |
| basic-user | Read access to the project | 
| cluster-admin | Superuser access to the cluster resources; full control on all projects |
| cluster-status | Get cluster status info | 
| edit | Can create, change, and delete common app resources from the project |
| self-provisioner | Can create new projects; cluster role, not project role | 
| view | Can view project resources, cannot modify the resources | 

Managing RBAC using CLI

Adding a cluster admin role to a user:
```
oc adm policy add -cluster-role-to user `cluster-role` `username`
```
> NOTE: A user with `cluster-admin` role can fully manage all storage pools, classes, and PVCs in all projects.

Adding a role to a user:
```
oc policy add-role-to-user `role-name` `user-name` -n project
```

Use the who-can command:
```
oc adm policy who-can create persistentvolumeclaims -n `namespace`
```

## Limits and Quotas

Two types of quotes, single project or for mulitple projects

| Resource Name | Description | 
| ---- | ---- | 
| requests.storage | Sum of storage requests across all PVCs in any state |
| persistentvolumeclaims | Total number of PVCs that can exist in the project | 
| `storageclass`.storageclass.storage.k8s.io/requests.storage | Same as requests.storage but for a given storage class | 
| `storageclass`.storageclass.storage.k8s.io/persistentvolumeclaims | Same as persistentvolumeclaims but for a given storage class | 
