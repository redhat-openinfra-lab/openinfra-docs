# Ceph Authorization

## Introduction

Ceph uses user accounts for internal communication between Ceph daemons, client applications accessing the cluster through librados, and for cluster administration.  Administrative and all client access have accounts that begin with the *client.* prefix.

The Ceph super user account is client.admin with capabilities that allow the account to access everything and to modify the cluster configuration.  This is the default account that is used unless you specify a user name with the --name or --id options.

> NOTE: When using the --name option, use the client. prefix.  But when using the --id option, the client. prefix is assumed.  Alternately you can also export CEPH_ARGA='--id cephuser".

The keyring files are maintained in the /etc/ceph directory.  If your keyring is not in this directory, you must pass it the --keyring option along with the file name that contains the secret key.

## Capabilities

* r grants read access
* w grants write access
* x grants authorization to execute extended object classes
* \* grants full access

## Profiles

* profile rbd - grants read/write access to any RBD pool
* profile rbd-read-only - grants read only access to any RBD pool


## Create Account

> NOTE: to save the key to a file use the -o /etc/ceph/ceph.client.*name*.keyring option

To create a client account to read and write to any RBD pool:

```
# ceph auth get-or-create client.myappuser mon 'allow r' osd 'allow rw'
```

Limit the user to a specific pool:
```
# ceph auth get-or-create client.myappuser mon 'allow r' osd 'allow rw pool=myapppool'
```

Limit the user to specific objects in a specific pool:
```
# ceph auth get-or-create client.myappuser mon 'allow r' osd 'allow rw pool=myapppool object_prefix my_prefix'
```

Limit the user to a specific namespace with a pool:
```
# ceph auth get-or-create client.myappuser mon 'allow r' osd 'allow rw pool=myapppool namespace=myappspace'
```

Limit the user read/write access to a specific path (used by CephFS):
```
# ceph fs authorize cephfs client.myappuser /myapppath rw
# ceph auth get client.myappuser
exported keyring for client.myappuser
[client.myappuser]
        key = ABE9asvIhDW7FJE#kii982qavejJKEJjEji==
        caps mds = 'allow rw path=/myapppath
        caps mon = 'allow r'
        caps osd = 'allow rw pool=cephfs_data'
```

Limit the user to a specific administraive commands against a mon:
```
# ceph auth get-or-create client.operator mon 'allow r allow command "auth get-or-create", allow command "auth list"' 
```

List Users:
```
# ceph auth list
```

Get current key and permissions of a user:
```
# ceph auth print-key client.myappuser
# ceph auth get client.myappuser
```

## Update Capabilities

Use the ceph auth caps command and the same options as creating an account.  Keeping in mind that any update to the user will overwrite the existing capabilities, therefore make sure you include any existing capabilites that need to be retained along with any additional.

## Delete Account

```
# ceph auth del client.myappuser
```

> NOTE: You should also remove the /etc/ceph/ceph.client.*name*.keyring file.