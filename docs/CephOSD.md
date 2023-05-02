# Ceph OSD

## OSD Configuration and Bluestore 

Use the .asok socket to pull information from the OSD daemons.  The bluefs stats command will show the 

```
# pwd
/var/run/ceph/8484edfa-8c65-11ed-a639-90e2babd60b8
# ceph --admin-daemon ./ceph-osd.5.asok config show
# ceph --admin-daemon ./ceph-osd.5.asok bluefs stats
```
