# RHEL9 Upgrade 

Get a list of OSDs on the server to remove:
```
ceph osd tree | grep -A12 storage05 | grep ssd | awk '{print $1}'
```

Remove the OSDs
```
ceph orch osd rm 0 7 15 21 27 34 39 44 50 55 --zap -replace
```

Check the services running; move/redeploy them if necessary:
```
ceph orch apply mds cephfs --placement="2 cephstorage06 cephstorage04"
```

Relabel any hosts:
```
ceph orch host label add cephstorage06 mds
```

Drain the services from the host:
```
ceph orch host drain cephstorage05
```

Check status of the OSD removals:
```
ceph orch osd rm status
```

Remove the host from the cluster:
```
ceph orch host rm cephstorage05
```

After the OS is reloaded, copy the files from /etc/ceph on cephstorage01 to the rebuilt server:
```
scp podman-auth.json cephstorage05:/etc/ceph/
```

On the rebuilt server:
```
subscription-manager register
```

```
subscription-manager repos --disable=*
subscription-manager repos --enable=rhel-9-for-x86_64-baseos-rpms
subscription-manager repos --enable=rhel-9-for-x86_64-appstream-rpms
```

```
subscription-manager repos --enable=rhceph-5-tools-for-rhel-9-x86_64-rpms
```

Run the ansible playbook to install pre-requisites:
```
ansible-playbook -i hosts cephadm-preflight-rhel9.yml --limit cephstorage05
```

Add the host back to the cluster:
```
ceph orch host add cephstorage05 172.20.0.15
```

Redeploy the OSDs:
```
ceph orch apply -i colocated_osds.yaml
```

Correct the device class on the OSDs:
```
for i in 0 7 15 21 27 34 39 44 50 55; do ceph osd crush rm-device-class osd.$i; ceph osd crush set-device-class ssd osd.$i; done
```
