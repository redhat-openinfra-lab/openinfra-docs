# Ceph RADOS Block Devices

## Introduction

To manage the RBD images, use the `rbd` command to create, list, retrieve info, resize, and remvoe block devices in the Ceph Storage platform.

The rbd_default_pool parameter specifies the name of the default pool used to store RBD images.  Use the command `ceph config get osd rbd_default_pool` to get the default pool name.

Create an erasure coded RBD pool on SSDs and initialize the pool for RBD images:
```
# ceph osd erasure-code-profile set ecrbd k=4 m=2 crush-device-class=ssd
# ceph osd pool create customrbd 32 32 ecrbd
# ceph osd pool application enable ecrbd rbd
# rbd pool init
```

Create an RBD image:
```
# rbd create test_pool/image1 --size=128M
```
  
## Accessing the RBD image via Kernel Client (krbd)

Use the rbd map/unmap commands to map the RBD devices to /dev/rbd*x* devices
```
# rbd map rbd/imagename
# rbd unmap /dev/rbd0
```

Show mapped devices:
```
# rbd showmapped
```

Setting up Automount:
``` hl_lines="4 9"
# vi /etc/ceph/rbdmap
# RbdDevice		Parameters
#poolname/imagename	id=client,keyring=/etc/ceph/ceph.client.keyring
test_pool/image1	id=testpool,keyring=/etc/ceph/ceph.client.testpool.keyring
:wq
# vi /etc/fstab
UUID=e4f488bd-5775-4c5e-b376-eee627699f2b	/	xfs	defaults	0	0
UUID=7B77-95E7	/boot/efi	vfat	defaults,uid=0,gid=0,umask=077,shortname=winnt	0	2
/dev/rbd/testpool/image1    /mnt/image1    xfs   noauto 0 0 
:wq
# rbdmap map
# rbd showmapped
id  pool       namespace  image   snap  device   
0   test_pool             image1  -     /dev/rbd0
# systemctl enable rbdmap
```


## Snapshots and Cloning

| Name | Description |  
|---|---|
| layering | To enable cloning |   
| striping | v2 support for enhanced performance |   
| exclusive lock | Required when using RBD Mirroring |   
| object-map | Object map support (requires exclusive lock) |   
| fast-diff | Requires object map and exclusive lock |   
| deep-flatten | Flattens all snapshots of the image |   
| journaling | Required when using RBD Mirroring |   
| data-pool | EC data pool support |   
 
> NOTE: Must use fsfreeze --freeze to halt all I/O to the image before taking a snapshot.  Use the --unfreeze parameter to resume filesystem operations.

> NOTE: Deleting an RBD image fails if a snapshot exists.  Use `rbd snap purge` to delete the snapshots.

### Snapshots

Snapshots are read-only copies of an RBD image created at a specific point in time.

Creating a snapshot:
```
# rbd snap create pool/image@snapname
# rbd snap ls pool/image
```

Rollback to a previous snapshot:
```
# rbd snap rollback pool/image@snapname
```

Delete a snapshot:
```
# rbd snap rm pool/image@snapname
```

Get snapshot disk usage:
```
# rbd disk-usage pool/image
NAME           PROVISIONED  USED  
image1@mysnap        5 GiB  36 MiB
image1               5 GiB     0 B
<TOTAL>              5 GiB  36 MiB
```

### Cloning

Clones are read/write copies of an RBD image that use a protected RBD snapshot as a base.  Clones can be flatten, or converted to an independent image from the source.

Create the clone, protect the snap from deletion, create the clone from the protected snapshot:
```
# rbd snap create pool/image@snapname
# rbd snap protect pool/image@snapname
# rbd clone pool/image@snapname pool/cloneimage
```

To set the COR option one the specific image or globally:
```
# ceph config set client rbd_clone_copy_on_read true
# ceph config set global rbd_clone_copy_on_read true
```

Clone management commands:
```
# rbd children pool/image@snapname
# rbd clone pool/image@snapname pool/cloneimage
# rbd flatten pool/cloneimage
```

## Importing and Exporting Images

The `rbd export` command allows you to export an RBD image (or snapshot) to a file.  Use the `rbd import` command to import the file to another RBD image.  

Export:
```
# rbd export pool/image /tmp/img-exp.dat
```
> NOTE: Use the `--export-format 1|2` option to convert earlier format 1 images to the newer format 2.

Differential Export:
```
# rbd export-diff pool/image /tmp/exp-diff.dat
```

Import:
```
# rbd import /tmp/img-exp.dat pool/image
```
> NOTE: use the `export-format 1|2` option to specify the data format of the data to be imported.  Use the `--image-format 1|2` option  to specify the data format to import as along with the --stripe-count, --object-size, and --image-feature options.  
  
Differential Import:
```
# rbd import-diff /tmp/img-exp.dat pool/image
```

## Take out the trash

Use the `rbd trash mv` command to move an image from a pool to the trash.  Then delete the image from the trash using the `rbd trash rm` command.


## RBD Mirrors

RBD Mirroring supports both active/passive and active/active along with two modes, pool mode and image mode.  

* Pool mode - automatically enables mirroring for each RBD image created in a mirrored pool.  Ceph creates the image on the remote cluster.  
* Image mode - selectively enabled for individual RBD images.  

Data is asynchronously mirrored using either journal-based or snapshot based mirroring.  

* Journal-based - to ensure point-in-time and crash consistent replication.  Data is written to the associated journal before the actual image.  The remote cluster reads the journal and replays the updates to its local copy of the image.  
* Snapshot-based - uses scheduled or manually created snapshots to replicate crash-consistent images.  The remote cluster determines any data or metadata updates between two mirror snapshots and copies the deltas to the image's local copy.  The RBD fast-diff image feature enables quick determination of updated data blocks without having to scan the entire image.
> NOTE: The complete delta between two snapshots must be synced prior to use during a failover.  Any partial sync will be rolled back the moment of failover.

### Configuration 

To configure mirroring, ensure the pool is defined on both the local and remote cluster.  
```
# rbd mirror pool peer bootstrap create --site-name primary poolname > /tmp/primary.token
```
> NOTE: Copy the /tmp/primary.token to the secondary cluster.  

On secondary site:  
```
# ceph orch apply rbd-mirror --placement=nodename
# rbd mirror pool peer bootstrap import --site-name secondary --direction rx-only test_pool /tmp/bootstrap.token
```

Enable pool mirroring:  
```
# rbd mirror pool enable pool [pool|image]
```

Enable image mirroring:  
```
# rbd mirror image enable pool/image
```

Snapshot-based mirroring:  
```
# rbd mirror image enable pool/image snapshot
```
> NOTE: Journal mirror must not be enabled; use `rbd mirror image disable pool/image` if necessary.

### Failover Procedure

On the primary site:  
```
# rbd mirror image demote pool/image
```
On the secondary site:  
```
# rbd mirror image promote pool/image
```
> NOTE: Use the --force option during the promotion if you cannot demote the primary site.  