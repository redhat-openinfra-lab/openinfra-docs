# Ceph CRUSH Map

## Introduction

Ceph calculates which OSDs should hold the objects by using a placement algorithm called CRUSH, Controlled Replication Under Scalable Hashing.  Objects are assigned to placement groups (PGs) and CRUSH determines which OSDs those placement groups should use to store their objects.

Crush uses the concept of *buckets*.  Bucket types include root, region, datacenter, room, pod, pdu, row, rack chassis and host.  You can also had your own types.

## CRUSH Map Commands

Dump the crush map into a binary file and then convert to text file:
```
# ceph osd getcrushmap -o /tmp/map.bin
# crushtool -d /tmp/map.bin -o /tmp/map.txt
```

Compile a CRUSH map from text file, test it, and then import it:
```
# crushtool -c /tmp/map.txt -o /tmp/map.new.bin
# crushtool -i /tmp/map.new.bin --test
# ceph osd setcrushmap -i /tmp/map.new.bin
```
Sample CRUSH rule to select SSD for primary copy and HDDs for replicas:
> rule ssd-first {  
      id 5  
      type replicated  
      min_size 1  
      max_size 10  
      step take default class ssd  
      step chooseleaf firstn 1 type host  
      step emit  
      step take default class hdd  
      step chooseleaf firstn -1 type host  
      step emit  
  }   

Set the OSD Device Class:
```
# ceph osd crush set-device-class osd.1 ssd
```

Remove a device class:
```
# ceph osd crush rm-device-class hdd
```

List configured device classes:
```
# ceph osd crush class ls
```

## Customizing CRUSH Map

Create buckets:
```
# ceph osd crush add-bucket *name* *type* 
```

Organize the buckets
``` 
# ceph osd crush move *bucket* type=*parent*
```

Place the OSDs as leaves in the correct location:
```
# ceph osd crush set osd.1 1.0 root=default rack=rack1 host=hosta
```


## Adding CRUSH Rules

Create a replicated rule to start from root, replicate across buckets of *type*, using devices of *class*:
```
# ceph osd crush rule create-replicated *name* root *type* *class*
```

Create an erasure-coded pool to start from crush-root, using crush-failure-domain and crush-device-class:
```
# ceph osd erasure-code-profile set *myprofile* k=4 m=2 crush-root=root crush-failure-domain=rack crush-device-class=ssd
```

List EC profiles and CRUSH Rules:
```
# ceph osd erasure-code-profile ls
# ceph osd crush rule ls
```

## Calculating PGs

Single pool:
> totalPGs = (OSDs * 100 ) / numberOfReplicas

Red Hat recommends the use of the Ceph Placement Groups per Pool Calculator, https://access.redhat.com/labs/cephpgc/, from the Red Hat Customer Portal Labs.

## OSD Map Commands

```
# ceph osd dump
# ceph osd getmap -o osd.map.bin
# osdmaptool --print osd.map.bin
# osdmaptool --export-crush crush.bin osd.map.bin
# crushtool -d crush.bin crush.txt
# crushtool -c crush.txt -o crush.new.bin
# osdmaptool --import-crush crush.new.bin osd.map.bin
# osdmaptool --test-map-pgs-dump /tmp/osd.map.bin
```
