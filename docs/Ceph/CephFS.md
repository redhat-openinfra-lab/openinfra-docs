# CephFS 

## Introduction

A single CephFS filesystem can be created with multiple subvolumes defined within the CephFS for fine grained authorization and access control.


| Option | Description | 
| --- | --- | 
| r | Read access to the specified folder |
| w | Write access to the specified folder | 
| p | Required to use layouts and quotas | 
| s | Required to create snapshots | 

By default the Fuse client mounts the root directory; use the `ceph-fuse -r /directory` command to mount a subdirectory.


## Configuration

Create the CephFS volume; this creates the pools, the volumes, and starts the MDS service on the hosts.
```
# ceph fs volume create fs-name --placement="2 ceph01,ceph02"
```

Create CephFS Manually:
```
# ceph osd pool create cephfs_data [erasure]
# ceph osd pool create cephfs_metadata 
# ceph osd pool application enable cephfs_data cephfs
# ceph osd pool application enable cephfs_metadata cephfs
# ceph fs new fs-name cephfs_metadata cephfs_data
```

Add erasure coded pool to your CephFS:
```
# ceph fs add_data_pool fs-name data-pool
```

Deploy the CephFS service and verify the service is up:
```
# ceph orch apply mds fs-name --placement="2 ceph01 ceph02"
# ceph mds stat
```

### Authorization

All clients must be authorized to access the CephFS filesystem.

Authorize a client to use extended attributes and snapshots for files in the /mydirectory:
```
# ceph fs authorize mycephfs client.user / r /mydirectory rwps
```
> NOTE: When authorizing both extended attributes and snapshots, they must be in alpa order.
> NOTE: Prevent the deletion of files from UID/GID=0 by using the `root_squash` option.


### Mounting with kernel client

Use the kernel client with LINX kernel versions of 4.0 and newer.  You can mount the root filesystem or subdirectories.  

* Verify the ceph-common package is installed.

Mount:
```
# mount -t ceph mon1,mon2,etc:/path mount-point -o name=client.user1,fs=,mycephfs
```
> NOTE: Subdirectories in the CephFS can be mounted as well.
> NOTE: You can also leave off the `mon1,mon2,etc` and Ceph will use the monitors in the /etc/ceph/ceph.conf file.


| Option Name | Description |
| --- | --- |
| name=name | Cephx client ID to use; default is guest | 
| fs=fs-name | Name of CephFS filesystem to mount | 
| secret=secret-value | Value of the secret key for the client (no longer needed) | 
| secretfile=filename | Path to file that contains client secret (no longer needed) | 
| rsize=bytes | Maximum read size in bytes | 
| wsize=bytes | Maximum write size in bytes; default none |

Persistent Mount /etc/fstab Entry:
```
mon1,mon2:/ /mnt/cephfs ceph name=user1,_netdev 0 0
```


### Mounting with Fuse client

Use the FUSE client when the LINUX kernel version is 4.0 or earlier.  Limitation is that you have to mount the root filesystem.  
> NOTE: This was stated in the online course.  The documentation indicates otherwise.  Use the -r option to instruct the client to treat the path as its root.

* Verify the ceph-common and ceph-fuse packages are installed.
* Verify the /etc/ceph/ceph.conf exists.
* When using the FUSE client as a non-root user, add user_allow_other in the /etc/fuse.conf file.


Authorize the client; providing read/write access and ability to create snapshots:
```
# ceph fs authorize mycephfs client.fsuser / r /fsuser-directory rws
```

Mount:
```
# ceph-fuse /mnt/mycephfs -n client.fsuser --client-fs=mycephfs
```

Persistent Mount /etc/fstab Entry:
```
mon-hostname:/ /mnt/cephfuse fuse.ceph ceph.id=myuser,_netdev 0 0
```
NOTE:  If you don't use the default name and location of the user's keyring file, then use the --keyring option to specify the path to the file.



### Remove CephFS Filesystem

Mark CephFS down, then remove it:
```
# ceph fs set fs-name down true
# ceph fs rm fs-name --yes-i-really-mean-it
```
> NOTE: the mon_allow_pool_delete option must be set to true for the above to succeed.  


## MDS Autoscaler

CephFS requires at least one active MDS server and one standby for HA.  The MDS Autoscaler ensures the availability of enough MDS daemons.  The modules monitors the number of ranks and number of standby daemons and adjusts the number of MDS daemons that the orchestrator spawns.

To enable:
```
ceph mgr module enable mds_autoscaler
```

To set the max number of active MDS daemons:  
```
ceph fs set mycephfs max_mds 2  
```

To set the max number of standby MDS daemons:  
```
ceph fs set mycephfs standby_count_wanted 2  
```

## MDF Failover

When the active MDS becomes unresponsive, a MON daemon waits for the number of seconds specified by the mds_beacon_grace parameter.  If unresponsive after this number of seconds, the MON marks the MDS daemon as laggy and one of the standby daemons becomes active.

```
# ceph config get mds mds_beacon_grace
15.000000
```


## CephFS Mirror

To enable and deploy the feature:
```
# ceph mgr module enable mirroring
# ceph orch apply cephfs-mirror node-name
```

For each CephFS peer, you must create a user on the target cluster:
```
# ceph fs authorize fs-name client_ / rwps
```

Enable mirroring on the source cluster:
```
# ceph fs snapshot mirror enable fs-name
```

Prepare the target peer:
```
# ceph fs snapshot mirror peer_bootstrap create fs-name peer-name site-name
```

Import the bootstrap token on the source cluster:
```
# ceph fs snapshot mirror peer_bootstrap import fs-name boostrap-token
```

Configure a directory for snapshot mirroring on the source clsuter:
```
ceph fs snapshot mirror add fs-name path
```

## File Layout

| Attribute Name | Description |  
| --- | ---|  
| ceph.[file\|dir].layout.pool | The pool where Ceph stores the file's or directory's data objects |  
| ceph.[file\|dir].layout.stripe_unit | The size in bytes of a block of data that is used in RAID 0 distribution of a file/directory |  
| ceph.[file\|dir].layout.stripe_count | The number of consecutive stripe units that constitute a RAID 0 stripe |  
| ceph.[file\|dir].layout.object_size | File or directory data is split into RADOS objects of this size in bytes; 4MiB by default |  
| ceph.[file\|dir].layout.pool_namespace | The namespace in the pool that is used; if exists |  

Use the `getfattr` and `setfattr` commands to retrieve and set the attributes.  Keeping in mind, a file's attributes cannot be changed once the file is written.  A file's attributes are inherited by the parent.

To change the pool from the replicated pool to an erasure coded pool, create the new pool, update allow_ec_overwrites to true, add it to the cephfs, and then set the layout on a new directory.  

```
# ceph fs add_data_pool mycephfs cephfs_data_ec
# setfattr -n ceph.dir.layout.pool -v cephfs_data_ec /mnt/mycephfs/mynewdirectory

