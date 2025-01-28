# The must-gather Tool 

Link to our [documentation]<https://docs.openshift.com/container-platform/4.17/support/gathering-cluster-data.html#gathering-data-specific-features_gathering-cluster-data>


When opening a support case for OSV, run the following must-gathers:

```
oc adm must-gather
```

```
oc adm must-gather --image=registry.redhat.io/container-native-virtualization/cnv-must-gather-rhel9:v4.17.3
```

[MTV must-gather Documentation]<https://docs.redhat.com/en/documentation/migration_toolkit_for_virtualization/2.7/html/installing_and_using_the_migration_toolkit_for_virtualization/troubleshooting_mtv#using-must-gather_mtv>

Check the link for instructions to filter the data based on namespace, migration, or VM.

```
oc adm must-gather --image=registry.redhat.io/migration-toolkit-virtualization/mtv-must-gather-rhel8:2.7.8
```

