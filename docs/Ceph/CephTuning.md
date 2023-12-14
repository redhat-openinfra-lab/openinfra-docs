# Performance Tuning

* IOPS - HDDs are in the range of 50-200; SSDs are thousands to hundreds of thousands; NVMe are some hundreds of thousands. 
* Throughput - HDDs around 150Mb/s; SSDs are ~500Mb/s; NVMe are ~2,000Mb/s.  


* Tune the BlueStore back end used by OSDs
* Adjust the schedule for automatic data scrubbing and deep scrubbing
* Adjust the schedule of asynchronous snapshot trimming (snapshot cleanup)
* Control rate of backfill and recovery operations with OSD failures/additions

## Blog Links

[CPU Scaling](https://ceph.io/en/news/blog/2022/ceph-osd-cpu-scaling/)  
[BlueStore (Default vs. Tuned) Performance Comparison](https://ceph.io/en/news/blog/2019/bluestore-default-vs-tuned-performance-comparison/)  
[Ceph Block Storage Performance on All-Flash Cluster with BlueStore backend](https://ceph.com/community/ceph-block-storage-performance-on-all-flash-cluster-with-bluestore-backend/)  
[RHCS Bluestore performance Scalability ( 3 vs 5 nodes )](https://ceph.com/community/part-3-rhcs-bluestore-performance-scalability-3-vs-5-nodes/)  
[RHCS 3.2 Bluestore Advanced Performance Investigation](https://ceph.io/en/news/blog/2019/part-4-rhcs-3-2-bluestore-advanced-performance-investigation/)  
  

## Recovery and Backfill

* Recovery occurs when OSD becomes inaccessible and then back online.  OSD has to recover the latest copy of the data.  Default is to allow 3 simultaneous recovery operations for HDDs and 10 for SSDs.

* Backfill occurs with new OSDs and when and OSD dies and Ceph reassigns its PGs to other OSDs.  Default is to allow 1 PG backfill to/from an OSD at a time.

| Parameter | Definition | 
| --- | --- | 
| osd_recovery_op_priority | range of 1-63; default is 3 |  
| osd_recovery_max_active | concurrent recovery operations per OSD in parallel |  
| osd_recovery_threads | Number of threads for data recovery |  
| osd_max_backfills | concurrent backfill operations per OSD |  
| osd_backfill_scan_min | Minimum number of objects for backfill scan |  
| osd_backfill_scan_max | Maximum number of objects for backfill scan |  
| osd_backfill_full_ratio | Threshold for backfill requests to an OSd |  
| osd_rbackfill_retry_interval | Seconds to wait before retrying backfill requests |  



### IOPS optimized

Workloads on block devices.  Typical deployments require high-performance SAS drives for storage and journals place on SSDs or NVMe.  

* Use two OSDs per NVMe device.  
* NVMe drives have data, the block database, and WAL collocated on the same storage device.  
* Assuming a 2 GHz CPU, use 10 cores per NVMe or 2 cores per SSD.  
* Allocate 16 GB RAM as a baseline, plus 5 GB per OSD.  
* Use 10 GbE NICs per 2 OSDs.  

### Throughput optimized

Workloads on RGW.  

  > NOTE: Workloads on RGW are often throughput-intensive and are backed on HDDs.  However, the bucket index pool is typically I/O-intensive so store it on SSD/NVMe.  One index/bucket that is stored in one RADOS object.  When a bucket stores more than 100K objects, the index performance degrades.  Use sharding for buckets by setting the `rgw_override_bucket_index_max_shards` parameter.  Recommended value is number of objects/100,000.  As the index grows, Ceph must reshard the bucket.  Enable automatic resharding with `rgw_dynamic_resharding`.  

* Use one OSD per HDD.  
* Place the block database and WAL on SSDs or NVMes.  
* Use at least 7,200 RPM HDD drives.  
* Assuming a 2 GHz CPU, use one-half core per HDD.  
* Allocate 16 GB RAM as a baseline, plus 5 GB per OSD.  
* Use 10 GbE NICs per 12 OSDs.


### Capacity optimized

Workloads that require a large amount of data as inexpensively as possible; usually trading performance for price using SATA drives.

* Use one OSD per HDD.  
* HDDs have data, the block database, and WAL collocated on the same storage device.  
* Use at least 7,200 RPM HDD drives.  
* Assuming a 2 GHz CPU, use one-half core per HDD.  
* Allocate 16 GB RAM as a baseline, plus 5 GB per OSD.  
* Use 10 GbE NICs per 12 OSDs.  


## Performance Stress Test

### RADOS Bench Command

```
# rados -p <mypool> bench <seconds> --io-type [write|seq|rand] -b <objsize> -t concurrency [--no-cleanup]
```
  > Defaults:  
       --io-size  4096 bytes  
       --io-threads  16  
       --io-total  1 GB  
       --io-pattern   seq



## Fragmentation

Check fragmentation on the OSDs:

```
# ceph tell <osd.ID> bluestore allocator score block
```

Value 0 to .7 is considered acceptable.  
Value .7 to .9 is considered safe fragmentation.  
Value .9 and above indicates severe fragmentation that is causing performance issues.  

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

```
ceph pg scrub <pg-id>
```

### Deep Scrubbing

Reads the data and recalculates and verifies the objects checksum.

You can enable/disable deep scrubbing at the cluster level by using the `ceph osd set nodeep-scrub` and `ceph osd unset nodeep-scrub` commands.

| Parameter | Description |   
| --- | --- |  
| osd_deep_scrub_interval | Interval for deep scrubbing; default is 604,800 |  
| osd_scrub_sleep | Introduces a pause between deep scrub disk reads.  Increase the value to slow down scrub operations; default is 0 |  

```
ceph pg deep-scrub <pg-id>
```

### Snapshot Trimming

Ceph schedules the removal of the snapshot data as an asynchronous operation when a snapshot is removed. To reduce the impact, configure a pause after the deletion with the osd_snap_trim_sleep parameter.  This adds a delay before allowing the next snapshot trim operation. Default is 0.

Control the snapshot trimming process using the osd_snap_trim_priority parameter.  Default 5

## Logging

Ceph logging levels are on a scale of 1 (terse) to 20 (verbose).  A single value can be used for both log level and memory level or separate them with a slash for different values (i.e. 1/5 where 1 is for log level and 5 is for memory log level).

Get the logging config for daemons:
```
ceph --admin-daemon ./ceph-osd.11.asok config show | grep debug
```
> NOTE: The asok files are found in /var/run/ceph/*fsid*.



