## Monitoring Stack

By default the monitoring data storage is set to ephemeral.  All metrics are lost when the pods are restarted or recreated.  Block (recommended) or file storage can be configured for persistent storage for the monitoring stack.

To update the configuration to use persistent storage, create the PVC configMap:

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
> NOTE: Prometheus metrics will be retained for 7 days in this example.

```
oc create -n openshift-monitoring configmap cluster-monitoring-config --from-file config.yaml=metrics-storage.yml
```



```hl_lines="13 19 22 29 32"
oc get cm cluster-monitoring-config -n openshift-monitoring -o yaml
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
      retention: 7d <-- 1
      baseImage: openshift/prometheus
      nodeSelector:
        node-role.kubernetes.io/infra: ""
      volumeClaimTemplate:
        spec:
          storageClassName: ocs-storagecluster-ceph-rbd <-- 2
          resources:
            requests:
              storage: 40Gi <-- 3
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

