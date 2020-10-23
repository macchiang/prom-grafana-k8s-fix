# Update Prometheus and Kube State Metrics for K8S 1.6
## Prometheus
### Change labels from `pod_name` to `pod`

prometheus.yaml
```
  - "job_name": "cadvisor"
    "kubernetes_sd_configs":
      - "role": "node"
    "metric_relabel_configs":
      - "action": "drop"
        "regex": "^$"
        "source_labels":
          - "namespace"
      - "action": "drop"
        "regex": "^$"
        "source_labels":
          - "pod" # Here
```

rules.yaml
```
  - "name": "k8s.rules"
    "rules":
      - "expr": |
          sum(rate(container_cpu_usage_seconds_total{job="cadvisor", image!=""}[5m])) by (namespace)
        "record": "namespace:container_cpu_usage_seconds_total:sum_rate"
      - "expr": |
          sum(container_memory_usage_bytes{job="cadvisor", image!=""}) by (namespace)
        "record": "namespace:container_memory_usage_bytes:sum"
      - "expr": |
          sum by (namespace, label_name) (
             sum(rate(container_cpu_usage_seconds_total{job="cadvisor", image!=""}[5m])) by (namespace, pod)
           * on (namespace, pod) group_left(label_name)
             label_replace(kube_pod_labels{job="kube-state-metrics"}, "pod", "$1", "pod", "(.*)")
          )
        "record": "namespace_name:container_cpu_usage_seconds_total:sum_rate"
      - "expr": |
          sum by (namespace, label_name) (
            sum(container_memory_usage_bytes{job="cadvisor",image!=""}) by (pod, namespace)
          * on (namespace, pod) group_left(label_name)
            label_replace(kube_pod_labels{job="kube-state-metrics"}, "pod", "$1", "pod", "(.*)")
          )
        "record": "namespace_name:container_memory_usage_bytes:sum"
      - "expr": |
          sum by (namespace, label_name) (
            sum(kube_pod_container_resource_requests_memory_bytes{job="kube-state-metrics"}) by (namespace, pod)
          * on (namespace, pod) group_left(label_name)
            label_replace(kube_pod_labels{job="kube-state-metrics"}, "pod", "$1", "pod", "(.*)")
          )
        "record": "namespace_name:kube_pod_container_resource_requests_memory_bytes:sum"
      - "expr": |
          sum by (namespace, label_name) (
            sum(kube_pod_container_resource_requests_cpu_cores{job="kube-state-metrics"} and on(pod) kube_pod_status_scheduled{condition="true"}) by (namespace, pod)
          * on (namespace, pod) group_left(label_name)
            label_replace(kube_pod_labels{job="kube-state-metrics"}, "pod", "$1", "pod", "(.*)")
          )
        "record": "namespace_name:kube_pod_container_resource_requests_cpu_cores:sum"
```
### Restart prometheus
`kubectl rollout restart sts prometheus-prometheus`

## kube state metrics
Please change the service account and cluster role before applying
`kucectl apply -f prometheus-kube-state-metrics-1.6.yaml`

### Update RBAC
https://github.com/kubernetes/kube-state-metrics/blob/master/examples/standard/cluster-role.yaml


### Upgrade addon-resizer to 1.8.7 and kubernetes to v1.9.7
https://github.com/kubernetes/autoscaler/tree/addon-resizer-release-1.8/addon-resizer#example-deployment-file

## Grafana
### Making changes to a provisioned dashboard
```
    # <bool> allow updating provisioned dashboards from the UI
    allowUiUpdates: false

```
https://grafana.com/docs/grafana/latest/administration/provisioning/#dashboards

`kubectl apply -f prometheus-grafana-dashboardproviders-1.6.yaml`

### Upgrade Grafana to 7.2.2
`kubectl set image sts/prometheus-grafana grafana=grafana/grafana:7.2.2`

### Update Grafana dashboards JSON models
You need change `pod_name` -> `pod`, `container_name` -> `container` in Grafana dashboards JSON models.

- K8s / Compute Resources / Pod
- K8s / Compute Resources / Namespace
- Deployments
- Pods


## Reference
https://stackoverflow.com/a/58572044
>Kubernetes 1.16 removes the labels pod_name and container_name from cAdvisor metrics, duplicates of pod and container.


