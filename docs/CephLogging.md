# Ceph Logging

By default, Ceph daemons logs ot stderr and logs are captured by the container runtime environment.  These logs are sent to journald and are accessible through the `journalctl` command.  

Get list of Ceph daemons, view logs:  
```
# systemctl list-units "ceph*"
# journalctl -eu ceph-dd6eede2-e3a6-11ed-9d9d-fa163ee4fccc@mgr.ceph1.exaerd.service
```

To enable logging to files instead of journald, set `log_to_file` to true.  The will create the Ceph log files in /var/log/ceph/fsid.  

```
# ceph config set global log_to_file true 
# ceph config set log_to_stderr false
```
> NOTE: Make sure to set log_to_stderr false to avoid double logging.


## Configuration Checks

Cephadm periodically scans each of the hosts in the storage cluster, to understand the state of the OS, disks, and NICs . These facts are analyzed for consistency across the hosts in the storage cluster to identify any configuration anomalies. The configuration checks are an optional feature.  

| Config Check | Description |  
| --- | --- | 
| CEPHADM_CHECK_KERNEL_LSM | Check to ensure all hosts are running SELINUX | 
| CEPHADM_CHECK_SUBSCRIPTION | Verifies valid subscriptions |  
| CEPHADM_CHECK_PUBLIC_MEMBERSHIP | At least one NIC is configured on public network |  
| CEPHADM_CHECK_MTU | Ensure all OSD nodes are running the same MTU size |  
| CEPHADM_CHECK_LINKSPEED | Ensure all nodes are running the same link speeds |  
| CEPHADM_CHECK_NETWORK_MISSING | Ensure all nodes are running configured for the same subnets |  
| CEPHADM_CHECK_CEPH_RELEASE | Ensure all nodes are running the same Ceph version |  
| CEPHADM_CHECK_KERNEL_VERSION | Ensure the OS kernel version is consistent across the nodes |  


```
# ceph config set mgr mgr/cephadm/config_checks_enabled

# ceph cephadm config-check status
# ceph cephadm config-check ls
```

