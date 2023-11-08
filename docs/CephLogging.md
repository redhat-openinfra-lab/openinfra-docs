# Ceph Logging

For container-based deployments, by default, Ceph daemons logs to stderr and logs are captured by the container runtime environment.  These logs are sent to journald and are accessible through the `journalctl` command.  

Ceph logging levels operate on a scale of 1 (terse) to 20 (verbose).  Use a single value for the log level and memory level to set them both to the same value or use different values for output log level and memory level separating the values with a forward slash (i.e. debug_mon = 1/5 sets log level to 1 and memory level to 5).  


Get list of Ceph daemons, view logs:   
```
# systemctl list-units "ceph*"
# journalctl -eu ceph-dd6eede2-e3a6-11ed-9d9d-fa163ee4fccc@mgr.ceph1.exaerd.service
```

To enable logging to files instead of journald, set `log_to_file` to true.  The will create the Ceph log files in /var/log/ceph/fsid.  

By default, Cephadm sets up log rotation on each host to rotate these files. You can configure the logging retention schedule by modifying /etc/logrotate.d/ceph.CLUSTER_FSID.

```
# ceph config set global log_to_file true 
# ceph config set log_to_stderr false
```
> NOTE: Make sure to set log_to_stderr false to avoid double logging.

## Running Configuration vs. Configuration Database vs. Configuration File

Ceph manages the configuration database of options which centralizes management by storing options in key value pairs.  But there are a handful of Ceph options that can be defined in the local Ceph configuration file, `/etc/ceph/ceph.conf`.  These few config options control how other Ceph components connect to the monitors to authenticate and fetch the configuration information from the database.  

When the same option exists in the config database and the config file, the config database option has a lower priority; the config file takes precedence.  

To get the current running configuration use the `ceph config show` command.  

```
# ceph config show mon.osp-blm-controller-1  
```

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

## Monitor cephadm Log Messages

```
ceph config set mgr mgr/cephadm/log_to_cluster_level [debug|info]
ceph -W cephadm --watch-debug
ceph -W cephadm --verbose
```

## Logging to `stderr`

```
journalctl -u ceph-<fsid>@<hostName>
```

## View boot log

```
journalctl -b 
```

## Data Location

Cephadm daemon data and logs are located in:

* /var/log/ceph/<fsid> - storage cluster logs (usually not present when logging to stderr)
* /var/lib/ceph/<fsid> - cluster daemon data besides logs
* /var/lib/ceph/<fsid>/<daemonName> - data for specific daemon
* /var/lib/ceph/<fsid>/crash - crash reports for the storage cluster
* /var/lib/ceph/<fsid>/removed - old daemon data that have been removed by cephadm


