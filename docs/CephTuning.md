# Ceph Performance Tuning

## Primary OSD for a PG

Use the `ceph osd primary-affinity osd-number affinity` command to influence the  selection of an OSD as the primary OSD for a placement group.


## Recovery and Backfill

| Parameter | Definition | 
| --- | --- | 
| osd_recovery_op_priority | Priority for recovery operations |  
| osd_recovery_max_active | Max number of active recovery requests per OSD in parallel |  
| osd_recovery_threads | Number of threads for data recovery |  
| osd_max_backfills | Max number of back fills for an OSD |  
| osd_backfill_scan_min | Minimum number of objects for backfill scan |  
| osd_backfill_scan_max | Maximum number of objects for backfill scan |  
| osd_backfill_full_ratio | Threshold for backfill requests to an OSd |  
| osd_rbackfill_retry_interval | Seconds to wait before retrying backfill requests |  



### IOPS optimized

* Use two OSDs per NVMe device.  
* NVMe drives have data, the block database, and WAL collocated on the same storage device.  
* Assuming a 2 GHz CPU, use 10 cores per NVMe or 2 cores per SSD.  
* Allocate 16 GB RAM as a baseline, plus 5 GB per OSD.  
* Use 10 GbE NICs per 2 OSDs.  

### Throughput optimized

* Use one OSD per HDD.  
* Place the block database and WAL on SSDs or NVMes.  
* Use at least 7,200 RPM HDD drives.  
* Assuming a 2 GHz CPU, use one-half core per HDD.  
* Allocate 16 GB RAM as a baseline, plus 5 GB per OSD.  
* Use 10 GbE NICs per 12 OSDs.

### Capacity optimized

* Use one OSD per HDD.  
* HDDs have data, the block database, and WAL collocated on the same storage device.  
* Use at least 7,200 RPM HDD drives.  
* Assuming a 2 GHz CPU, use one-half core per HDD.  
* Allocate 16 GB RAM as a baseline, plus 5 GB per OSD.  
* Use 10 GbE NICs per 12 OSDs.  


## Performance Stress Test

RADOS bench command

```
# rados -p mypool bench seconds write|seq|rand -b objsize -t concurrency [--no-cleanup]
```
NOTE: Default object size is 4 MB and default concurrency is 16

RBD Bench command

```
rbd bench --io-type write testimage --pool=mypool  
```

Defaults:
* --io-size  4096 bytes  
* --io-threads  16  
* --io-total  1 GB  
* --io-pattern   seq


## Scrubbing

You can control scrubbing globally or at the pool level.  To control at the pool level use the `ceph osd pool set poolName parameterValue` command.

| Parameter | Description |   
| --- | --- |  
| noscrub | If set to true, Ceph doesn't light scrub the pool; default is false |  
| nodeep-scrub | If set to true, Ceph doesn't deep scrub the pool; default is false |  
| scrub_min_interval | Wait a minimum number of seconds between scrubs; default is 0 (uses global parameter) |  
| scrub_max_interval | Do not wait more than this number of seconds before performing a scrub; default is 0 |  
| deep_scrub_interval | Interval for deep scrubbing; default is 0 |  


### Light Scrubbing

Verifes an objects presence, checksum, and size.

| Parameter | Description |   
| --- | --- |  
| osd_scrub_end_hour = end_hour | Specifies the time to stop scrubbing; 0 - 23 |  
| osd_scrub_load_threshold | Perform scrub is system load is below the threshold; default is .5 |  
| osd_scrub_min_interval | Wait a minimum number of seconds between scrubs; default is 86,400 |  
| osd_scrub_internval_randomize_ratio | Add a random dealy to the value defined in the osd_scrub_min_interval parameter; default is .5 |  
| osd_scrub_max_interval | Do not wait more than this number of seconds before performing a scrub regardless of load; default 604,800 |  
| osd_scrub_priority | Set the priority for scrub operations; default is 5; relative to osd_client_op_priority (default 64) |  

### Deep Scrubbing

Reads the data and recalculates and verifies the objects checksum.

You can enable/disable deep scrubbing at the cluster level by using the `ceph osd set nodeep-scrub` and `ceph osd unset nodeep-scrub` commands.

| Parameter | Description |   
| --- | --- |  
| osd_deep_scrub_interval | Interval for deep scrubbing; default is 604,800 |  
| osd_scrub_sleep | Introduces a pause between deep scrub disk reads.  Increase the value to slow down scrub operations; default is 0 |  


## Logging

Ceph logging levels are on a scale of 1 (terse) to 20 (verbose).  A single value can be used for both log level and memory level or separate them with a slash for different values (i.e. 1/5 where 1 is for log level and 5 is for memory log level).

Get the logging config for daemons:
```
ceph --admin-daemon ./ceph-osd.11.asok config show | grep debug
```
> NOTE: The asok files are found in /var/run/ceph/*fsid*.



