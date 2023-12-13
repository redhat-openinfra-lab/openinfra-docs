# Data Foundation Storage 

## Storage Classes
```
oc get storageclasses -o name
```

```
oc describe sc <storageClass>
```

### Persistent Volume Claim

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: production-application
spec:
  accessModes:
  - ReadWriteMany
  storageClassName: ocs-storagecluster-cephfs
  resources:
    requests:
      storage: 10Gi
```

### Logs

Get listing of ODF pods:
``` 
oc get pods -n openshift-storage
```

Monitor Logs
```
oc logs rook-ceph-mon-a-blah -n openshift-storage
```

OSD Logs
```
oc logs rook-ceph-osd-0-blah -n openshift-storage
```

Rook Operator Logs:
```
oc logs rook-ceph-operator-blah -n openshift-storage
```

RBD Logs:
```
oc logs csi-rbdplugin-blah -n openshift-storage -c csi-rbdplugin
```

CephFS Logs:
```
oc logs csi-cephfsplugin-blah -n openshift-storage
```

## Cluster Config Tool

Ceph CLI
```
oc exec -ti pod/rook-ceph-operator-blah -n openshift-storage -c rook-ceph-operator -- /bin/bash
```

Cluster Health
```
ceph -c /var/lib/rook/openshift-storage/openshift-storage.config health
```

Cluster Status
```
ceph -c /var/lib/rook/openshift-storage/openshift-storage.config -s
```

Storage Usage
```
ceph -c /var/lib/rook/openshift-storage/openshift-storage.config df
```

Tool - must-gather
```
oc adm must-gather --image=registry.redhat.io/ocs4/ocs-must-gather-rhel8:v4.7 --dest-dir=must-gather
```
## Monitoring Stack

By default the monitoring data storage is set to ephemeral.  All metrics are lost when the pods are restarted or recreated.  Block (recommended) or file storage can be configured for persistent storage for the monitoring stack.


```hl_lines="12 18 21 28  31"
apiVersion: v1
kind: ConfigMap
data:
  config.yaml: |
    prometheusOperator:
      baseImage: quay.io/coreos/prometheus-operator
      prometheusConfigReloaderBaseImage: quay.io/coreos/prometheus-config-reloader
      configReloaderBaseImage: quay.io/coreos/configmap-reload
      nodeSelector:
        node-role.kubernetes.io/infra: ""
    prometheusK8s:
      retention: 15d <-- 1
      baseImage: openshift/prometheus
      nodeSelector:
        node-role.kubernetes.io/infra: ""
      volumeClaimTemplate:
        spec:
          storageClassName: ocs-storagecluster-ceph-rbd <-- 2
          resources:
            requests:
              storage: 2000Gi <-- 3
    alertmanagerMain:
      baseImage: openshift/prometheus-alertmanager
      nodeSelector:
        node-role.kubernetes.io/infra: ""
      volumeClaimTemplate:
        spec:
          storageClassName: ocs-storagecluster-ceph-rbd <-- 2
          resources:
            requests:
              storage: 20Gi <-- 3
    nodeExporter:
      baseImage: openshift/prometheus-node-exporter
    kubeRbacProxy:
      baseImage: quay.io/coreos/kube-rbac-proxy
    kubeStateMetrics:
      baseImage: quay.io/coreos/kube-state-metrics
      nodeSelector:
        node-role.kubernetes.io/infra: ""
    grafana:
      baseImage: grafana/grafana
      nodeSelector:
        node-role.kubernetes.io/infra: ""
    auth:
      baseImage: openshift/oauth-proxy
    k8sPrometheusAdapter:
      nodeSelector:
        node-role.kubernetes.io/infra: ""
metadata:
  name: cluster-monitoring-config
namespace: openshift-monitoring

```

1. Time units are (s)econds, (m)inutes, (h)ours, or (d)ays
2. The object storage class provided by ODF is `ocs-storagecluster-ceph-rbd`
3. Storage values, E,P,T,G,M,K or Ei,Pi,Ti,Gi,Mi,Ki

Check the current configuration of `prometheus-k8s` or `alertmanager-main`:
```
#!/bin/bash
set -veuo pipefail

# Prometheus container volume mount spec:
oc get statefulset/prometheus-k8s \
  -n openshift-monitoring \
-o jsonpath='{.spec.template.spec.containers}' | \
jq '.[] | select(.name == "prometheus") | .volumeMounts[] | select(.name == "prometheus-k8s-db")'

# Prometheus container volume mount spec:
oc get statefulset/prometheus-k8s \
  -n openshift-monitoring \
-o jsonpath='{.spec.template.spec.volumes}' | \
jq '.[] | select(.name == "prometheus-k8s-db")'

```

Create PVC Config Map:
```
prometheusK8s:
  retention: 7d
  volumeClaimTemplate:
    spec:
      storageClassName: ocs-storagecluster-ceph-rbd
      resources:
        requests:
          storage: 40Gi
alertmanagerMain:
  volumeClaimTemplate:
    spec:
      storageClassName: ocs-storagecluster-ceph-rbd
      resources:
        requests:
          storage: 20Gi
```

```
oc create -n openshift-monitoring configmap cluster-monitoring-config --from-file config.yaml=metrics-storage.yml
```
