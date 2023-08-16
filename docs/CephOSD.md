# Ceph OSD

## OSD Configuration and Bluestore 

Use the .asok socket to pull information from the OSD daemons.  The bluefs stats command will show the 

```
# pwd
/var/run/ceph/8484edfa-8c65-11ed-a639-90e2babd60b8
# ceph --admin-daemon ./ceph-osd.5.asok config show
# ceph --admin-daemon ./ceph-osd.5.asok bluefs stats
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

## Ceph Pools

Create Pool:
```
# ceph osd pool create <name> <pg_num> <pgp_num> replicated <CRUSH_ruleSet>
```

Where pg_num is the total configured number of placement groups and pgp_num is the effective number of placement groups.  This should be the same.
For the CRUSH_ruleSet the osd_pool_default_crush_replicated_ruleset configuration parameter sets the default values.

Create erasure coded pool:
```
# ceph osd pool create <name> <pg_num> <pgp_num> erasure <erasureProfile> <CRUSH_ruleSet>
```

Where erasureProfile is the name of the profile created with the ceph osd erasure-code-profile set command.  The profile contains the k and m values and the erasure code plug in to use.

* plugin - optional; erasure coding algorithm
* directory - optional; location of plug in library (/usr/lib64/ceph/erasure-code) 
* crush-failure-domain - optional; controls chunk placement, default is hosts can be set to osd or rack
* crush-device-class - optional; select OSDs backed by devices of this class for the pool where class is hdd, ssd, name
* crush-root - optional; set the root node of the CRUSH rule set 

Auto-scaling:
```
# ceph osd pool get <name> pg_autoscale_mode
# ceph config get mon osd_pool_default_pg_autoscale_mode
```

Management:
```
# ceph osd lspools
# ceph osd pool ls detail
# ceph osd pool autoscale-status
```

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

