# Migration Toolkit for Virtualization


## Cold vs. Warm Migration

Cold  
* Shutdown, conversion and then disk transfers
* virt-v2v (el9)
* Disks are transferred sequentially

Warm  
* Pre-copy, shutdown, disk transfer, and then conversion
* Containerized Data Importer (CDI) + virt-v2v (el8)
* Disks are transferred in parallel by different pods
