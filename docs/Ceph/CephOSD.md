# Ceph OSD

## CPU Requirements

NVMe - 10 cores per OSD  
Non-NVMe SSD - 2 cores per OSD  
HDD - .5 core per OSD  

HDD: sockets * cores * GHz * 100 = estimated 4k randrw performance  
SSD: sockets * cores * GHz * 1500 = estimated 4k randrw performance  
  
  
## Drive Recommendations

SSD Interface - NVMe  
SSD Technology - TLC   
DWPD - 3   

> DWPD - Drive Writes per Day: how many times an SSD's usable capacity can be overwritten in 24-hours w/in the specified lifetime.  
> TBW - TeraBytes Written: total amount of data written to the SSD over the specified lifetime.  

| Capability | QLC | TLC | MLC | SLC |
|-----|-------|-----|-----|------|
| Architecture | 4-bits | 3-bits | 2-bits | 1-bit |  
| Capacity | Highest | High | Medium | Lowest |  
| Endurance | Low | Medium | High | Highest |  
| Cost | $ | $$ | $$$ | $$$$ |  



| DWPD to TBW Conversion | TBW to DWPD Conversion |  
|-----|-------|
| TBW = DWPD * SSD Capacity(TB) * Lifetime(years) * 365 Days | DWPD = TBW / (SSD Capacity(TB) * Lifetime(years) * 365 Days) |  


## OSD Flags

Backfill - adding/removing OSDs to the cluster.  Backfill does a per object comparison.  No blocked IO/writes.  
Recovery - when OSDs crash/fail and come back online.  Recovery uses the pglogs to determine changed objects.  
Rebalance - optimize the allocation of PGs.  

[Minimize the Data Pause during OSD or node loss](https://access.redhat.com/articles/6005681)  
[Differences between backfill and recovery](https://access.redhat.com/solutions/3352091#whatispeering)


## OSD Configuration and Bluestore 

Use the .asok socket to pull information from the OSD daemons.  The bluefs stats command will show the 

```
# pwd
/var/run/ceph/8484edfa-8c65-11ed-a639-90e2babd60b8
# ceph --admin-daemon ./ceph-osd.5.asok config show
# ceph --admin-daemon ./ceph-osd.5.asok bluefs stats
```

## Add OSD to the Ceph Cluster

```
# ceph orch daemon add osd serverName:/dev/vda
Created osd(s) 9 on host ‘serverName’
```

Details of OSD:
```
# ceph osd find 9
{
   “Osd”: 9
   “Adds”:  {
        “Addrvec”: [
…
}
```

## Remove OSD from Ceph Cluster

Mark the OSD 'out'; this may already be the status.  
Mark the OSD 'down'  
Remove the OSD  
Once removed it will be in the DNE (does not exists status)  
Remove the OSD from the crush map  
Remove the authentication/authorization keys  

```
# ceph osd out osd.XX
# ceph osd down osd.XX
# ceph osd rm osd.XX
# ceph osd crush rm osd.XX
# ceph auth del osd.XX
```

## Replace OSD

```
ceph orch daemon rm osd.xx --zap --replace
```

Use the following command to ensure the device can be reused if necessary:
```
ceph orch device zap --force 'serverName' /dev/sdx
```

## Manage OSDs

Start|stop|restart OSD daemon:
```
# ceph orch daemon [start|stop|restart] osd.13
Scheduled to stop osd.13 on host ‘ServerName’ 
```

Verify the correct OSD service is stopped
```
# ceph orch ps serverName —daemon-type osd 
```

## Updated OSD Device Class:

```
for i in {0..59}
do 
    ceph osd crush rm-device-class osd.$i
    ceph osd crush set-device-class ssd osd.$i
done
```

## Encryption

LUKS keys to unencrypt the drives are stored in the MON.  The key to authenticate to the MON is a cephx key.
```
client.osd-lockbox.f7626ae7-a6df-4229-99c5-aaccb1aebf5c
	key: AQAm2bVjC/V4ABAA+DIzvkQh7MqoSsUreWnmAQ==
	caps: [mon] allow command "config-key get" with key="dm-crypt/osd/f7626ae7-a6df-4229-99c5-aaccb1aebf5c/luks"
```

## Deepscrub

```
ceph pg dump pgs_brief | grep deep
dumped pgs_brief
4.2d     active+clean+scrubbing+deep           [47,11,58]          47           [47,11,58]              47
4.27     active+clean+scrubbing+deep           [51,44,48]          51           [51,44,48]              51
3.5      active+clean+scrubbing+deep           [55,40,19]          55           [55,40,19]              55
5.1a     active+clean+scrubbing+deep           [46,57,43]          46           [46,57,43]              46
32.19    active+clean+scrubbing+deep            [41,15,1]          41            [41,15,1]              41
4.45     active+clean+scrubbing+deep             [18,4,2]          18             [18,4,2]              18
4.62     active+clean+scrubbing+deep            [38,25,9]          38            [38,25,9]              38
3.6f     active+clean+scrubbing+deep            [12,33,3]          12            [12,33,3]              12
```

## Backfill

```
ceph pg dump pgs_brief | grep backfill
```

## Ceph Pools

### Primary Affinity

Manually control the selection of an OSD as the primary for a PG by setting the primary_affinity.  Mitigate issues or bottlenecks by configuring the cluster to avoid using slow disks or controllers for a primary OSD.

```
# ceph osd primary-affinity <osd-number> <affinity>  
```
Where affinity is a real number between 0 and 1.  


### Create Pool

Replicated Pool:
```
# ceph osd pool create <name> <pg_num> <pgp_num> replicated <CRUSH_ruleSet>
```

Where pg_num is the total configured number of placement groups and pgp_num is the effective number of placement groups.  This should be the same.
For the CRUSH_ruleSet the osd_pool_default_crush_replicated_ruleset configuration parameter sets the default values.

Erasure Coded Pool:
```
# ceph osd pool create <name> <pg_num> <pgp_num> erasure <erasureProfile> <CRUSH_ruleSet>
```

Where erasureProfile is the name of the profile created with the ceph osd erasure-code-profile set command.  The profile contains the k and m values and the erasure code plug in to use.

* plugin - optional; erasure coding algorithm
* directory - optional; location of plug in library (/usr/lib64/ceph/erasure-code) 
* crush-failure-domain - optional; controls chunk placement, default is hosts can be set to osd or rack
* crush-device-class - optional; select OSDs backed by devices of this class for the pool where class is hdd, ssd, name
* crush-root - optional; set the root node of the CRUSH rule set 

### Auto-scaling

The pg_autoscale_mode property is enabled by default and allows Ceph to make recommendations and automatically adjust the pg_num and pgp_num parameters.  This can be changed on a per pool basis:


Status:
```
# ceph osd pool autoscale-status
```

Set autoscale on specific pool:
```
# ceph osd pool get <name> pg_autoscale_mode [off|on|warn]
```

Get the default autoscale mode:
```
# ceph config get mon osd_pool_default_pg_autoscale_mode
```

### Management

Set replicas for the pool:
```
# ceph osd pool set <name> size|min_size <int>
```

Set application type for the pool:
```
# ceph osd pool application enable <poolName> [rbd|rgw|cephfs]
```

List erasure-code-profiles
```
# ceph osd erasure-code-profile ls
```

Get a profile’s details:
```
# ceph osd erasure-code-profile get
```

Create erasure-code-profile
```
# ceph osd erasure-code-profile set <name> k=4 m=2 
```

Set quotas:
```
# ceph osd pool <name> set-quota
```

Rename pool:
```
# ceph osd pool rename <oldName> <newName>
```

Delete the pool:
```
# ceph config get mon mon_allow_pool_delete
# ceph tell mon.* config set mon_allow_pool_delete true
# ceph osd pool delete <name> <name> —yes-i-really-really-mean-it
```

Details of a placement group:
```
# ceph pg <pgID> query
```

To have ceph manage all devices automatically:
```
# ceph orch apply osd —all-available-devices
```
This creates a systemd service as osd.all-available-devices (ceph orch ls)


## NVMe Firmware Update

Grab the ISO file from Samsung Website.
Mount the ISO file:
```
mount ./Samsung_SSD_970_EVO_Plus_4B2QEXM7.iso /mnt/iso
```

Extract the ISO and run the fumagician command:
```
mkdir /tmp/fwupdate; cd /tmp/fwupdate
gzip -dc /mnt/iso/initrd | cpio -idv --no-absolute-filenames
cd root/fumagician
./fumagician
```

## Performance Testing

### FIO Utility

Random Write Testing (no caching):
```
fio --name=rbd-test-direct1 --ioengine=libaio --direct=1 --rw=randwrite --bs=4k --numjobs=1 --size=4g --iodepth=1 --runtime=60 --time_based --end_fsync=1
```

With caching:
```
fio --name=rbd-test-direct0 --ioengine=libaio --rw=randwrite --bs=4k --numjobs=1 --size=4g --iodepth=1 --runtime=60 --time_based --end_fsync=1
```

Using input file:

```
cat rbd.fio
[global]
ioengine=rbd
clientname=admin
pool=rbd
rbdname=myimage
rw=randwrite
bs=4k
[rbd_iodepth32]
iodepth=32

fio ./rbd.fio
```
### Rados Bench Tool

Random Write Testing (leave data) using 16 concurrent threads of 4194304 bytes for 10 seconds:
```
rados bench -p rbd 10 write --no-cleanup
```

### DD Utility

Write 1 Gib of data 4 times (no caching):
```
dd if=/dev/zero of=here bs=1G count=4 oflag=direct
```

Wipe an msdos label from device:
```
# dd if=/dev/zero of=/dev/vdb bs=512 count=1
```
