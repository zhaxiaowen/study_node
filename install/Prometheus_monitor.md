# Prometheus 监控 K8s

#### 1. 创建一个超级管理员

```yaml
adm_account="k8s-dash-admin"
kubectl create serviceaccount ${adm_account} -n kube-system
kubectl create clusterrolebinding ${adm_account} --clusterrole=cluster-admin --serviceaccount=kube-system:${adm_account}
kubectl -n kube-system describe secrets $(kubectl -n kube-system get secret | grep ${adm_account} | awk '{print $1}')
# 获取token值
kubectl -n kube-system get secret `kubectl -n kube-system get secret|grep "k8s-dash-admin"|awk '{print $1}'` -o jsonpath={.data.token} | base64 -d
```

#### 2. 部署 Blackbox_exporter

> [部署 Blackbox_exporter](https://www.cnblogs.com/xiao987334176/p/12022482.html)
>
> 监测 k8s 的 services 需要用到这个组件

```
blackbox_exporter是Prometheus 官方提供的 exporter 之一，可以提供 http、dns、tcp、icmp 的监控数据采集。
Blackbox_exporter 应用场景
    HTTP 测试
          定义 Request Header 信息
          判断 Http status / Http Respones Header / Http Body 内容
    TCP 测试
          业务组件端口状态监听
           应用层协议定义与监听
    ICMP 测试
           主机探活机制
    POST 测试
          接口联通性
    SSL 证书过期时间

```

```
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape] 
        action: keep
        regex: true
  #如果配置文件中没有设置:
    template:
    metadata:
      labels:
        app: prometheus
      annotations:
        prometheus.io/scrape: 'true'
prometheus的配置文件中如果写了这个source_labels,会把没有prometheus.io/scrape: 'true'的全部过滤keep掉,所以如果没有打prometheus.io/scrape: 标签,可以把prometheus里的source_labels:[__meta_kubernetes_pod_annotation_prometheus_io_scrape]删掉
```

#### 3.k8s - Annotations

##### TODO: 可以只在 service 上加 , 因为 service-endpoints 最终也会落到 pod 上

> kubernetes-pods

* prometheus.io/scrape，为 true 则会将 pod 作为监控目标
* prometheus.io/path，默认为 /metrics
* prometheus.io/port , 端口

> kubernetes-service-endpoints

- prometheus.io/scrape，为 true 则会将 pod 作为监控目标
- prometheus.io/path，默认为 /metrics
- prometheus.io/port , 端口
- prometheus.io/scheme 默认 http，如果为了安全设置了 https，此处需要改为 https

#### 正式版

```
  - job_name: 'kubernetes-apiservers'
    honor_timestamps: true
    scrape_interval: 30s
    scrape_timeout: 10s
    metrics_path: /metrics
    scheme: https
    kubernetes_sd_configs:
    - api_server: https://192.168.50.100:6443
      role: endpoints
      bearer_token_file: /root/dashboard/test/cluster.token
      tls_config:
        insecure_skip_verify: true
    bearer_token_file: /root/dashboard/test/cluster.token
    tls_config:
      insecure_skip_verify: true
    relabel_configs:
    - source_labels: [__meta_kubernetes_namespace, __meta_kubernetes_service_name, __meta_kubernetes_endpoint_port_name]
      action: keep
      regex: default;kubernetes;https

  - job_name: 'kubernetes-cadvisor'
    honor_timestamps: true
    scrape_interval: 30s
    scrape_timeout: 10s
    metrics_path: /metrics
    scheme: https
    kubernetes_sd_configs:
    - api_server: https://192.168.50.100:6443
      role: node
      bearer_token_file: /root/dashboard/test/cluster.token
      tls_config:
        insecure_skip_verify: true
    bearer_token_file: /root/dashboard/test/cluster.token
    tls_config:
      insecure_skip_verify: true
    relabel_configs:
    - action: labelmap
      regex: __meta_kubernetes_node_label_(.+)
    - target_label: __address__
      replacement: 192.168.50.100:6443
    - source_labels: [__meta_kubernetes_node_name]
      regex: (.+)
      target_label: __metrics_path__
      replacement: /api/v1/nodes/${1}/proxy/metrics/cadvisor
  # 这个是监控所有pod对应的访问地址和端口
  - job_name: 'kubernetes-service-endpoints'
    honor_timestamps: true
    scrape_interval: 30s
    scrape_timeout: 10s
    metrics_path: /metrics
    scheme: http
    kubernetes_sd_configs:
    - api_server: https://192.168.50.100:6443
      role: endpoints
      bearer_token_file: /root/dashboard/test/cluster.token
      tls_config:
        insecure_skip_verify: true
    bearer_token_file: /root/dashboard/test/cluster.token
    tls_config:
      insecure_skip_verify: true
    relabel_configs:
    - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scrape]
      action: keep
      regex: true
    - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scheme]
      action: replace
      target_label: __scheme__
      regex: (https?)
    - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_path]
      action: replace
      target_label: __metrics_path__
      regex: (.+)
    - source_labels: [__address__, __meta_kubernetes_service_annotation_prometheus_io_port]
      action: replace
      target_label: __address__
      regex: ([^:]+)(?::\d+)?;(\d+)
      replacement: $1:$2
    - action: labelmap
      regex: __meta_kubernetes_service_label_(.+)
    - source_labels: [__meta_kubernetes_namespace]
      action: replace
      target_label: kubernetes_namespace
    - source_labels: [__meta_kubernetes_service_name]
      action: replace
      target_label: kubernetes_name
  # 这个就是kubectl get svc --all-namespaces显示的svc
  - job_name: 'kubernetes-services'
    honor_timestamps: true
    scrape_interval: 30s
    scrape_timeout: 10s
    metrics_path: /probe
    kubernetes_sd_configs:
    - api_server: https://192.168.50.100:6443
      role: service
      bearer_token_file: /root/dashboard/test/cluster.token
      tls_config:
        insecure_skip_verify: true
    bearer_token_file: /root/dashboard/test/cluster.token
    tls_config:
      insecure_skip_verify: true
    params:
      module: [http_2xx]
    relabel_configs:
    - source_labels: [__address__]
      target_label: __param_target
    - target_label: __address__
      replacement: 192.168.50.100:9115
    - source_labels: [__param_target]
      target_label: instance
    - action: labelmap
      regex: __meta_kubernetes_service_label_(.+)
    - source_labels: [__meta_kubernetes_namespace]
      target_label: kubernetes_namespace
    - source_labels: [__meta_kubernetes_service_name]
      target_label: kubernetes_name
```

#### 监控 redis_exporter(完成版)

```
  - job_name: 'kubernetes-pods'
    honor_timestamps: true
    scrape_interval: 30s
    scrape_timeout: 10s
    metrics_path: /metrics
    kubernetes_sd_configs:
    - api_server: https://192.168.50.100:6443
      role: pod
      bearer_token_file: /root/dashboard/test/cluster.token
      tls_config:
        insecure_skip_verify: true
    bearer_token_file: /root/dashboard/test/cluster.token
    tls_config:
      insecure_skip_verify: true
    relabel_configs:
    - source_labels: [__meta_kubernetes_pod_container_name]
      action: keep
      regex: "redis-exporter"
    - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
      action: replace
      target_label: __metrics_path__
      regex: (.+)
    - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
      action: replace
      regex: ([^:]+)(?::\d+)?;(\d+)
      replacement: $1:$2
      target_label: __address__
    - action: labelmap
      regex: __meta_kubernetes_pod_label_(.+)
    - source_labels: [__meta_kubernetes_namespace]
      action: replace
      target_label: kubernetes_namespace
    - source_labels: [__meta_kubernetes_pod_name]
      action: replace
      target_label: kubernetes_pod_name


## TODO,下面的这个可以替换,第一个是过滤contianer_name,第二个是过滤端口
    - source_labels: [__meta_kubernetes_pod_container_name]
      action: keep
      regex: "redis-exporter"
---
    - source_labels: [__address__]
      regex: '(.*):(9121)'
      action: keep

```

#### Blackbox_exporter 检测服务在线状态 (还有待优化)

```
  - job_name: 'blackbox_tcp_pod_probe'
    honor_timestamps: true
    scrape_interval: 30s
    scrape_timeout: 10s
    metrics_path: /probe
    kubernetes_sd_configs:
    - api_server: https://192.168.50.100:6443
      role: service
      bearer_token_file: /root/dashboard/test/cluster.token
      tls_config:
        insecure_skip_verify: true
    bearer_token_file: /root/dashboard/test/cluster.token
    tls_config:
      insecure_skip_verify: true
    params:
      module: [http_2xx]
    relabel_configs:
    # 
    #- source_labels: [__meta_kubernetes_ingress_annotation_prometheus_io_http_probe]
    - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_http_probe]
      action: keep
      regex: true
    - source_labels: [__address__,__meta_kubernetes_service_cluster_ip]
      action: replace
      separator: " "
      regex: (?:.*?:)(\d+)\s(.*)
      replacement: $2:$1
      target_label: __address__
    - source_labels: [__address__]
      target_label: __param_target
    - target_label: __address__
      replacement: 192.168.50.100:9115
    - source_labels: [__param_target]
      target_label: instance
    - action: labelmap
      regex: __meta_kubernetes_service_label_(.+)
    - source_labels: [__meta_kubernetes_namespace]
      target_label: kubernetes_namespace
    - source_labels: [__meta_kubernetes_service_name]
      target_label: kubernetes_name

```
