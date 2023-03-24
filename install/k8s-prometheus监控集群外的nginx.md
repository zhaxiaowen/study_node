# k8s-prometheus 监控集群外的 nginx

> ## nginx 服务需要安装 nginx-module-vts-master 模块

参考

[Kube-Prometheus Stack 监控 mysql](https://blog.csdn.net/weixin_44932410/article/details/125029852)

[Prometheus Operator 使用 ServiceMonitor 管理监控配置-1](https://blog.csdn.net/qq_34556414/article/details/122316980)

[prometheus 监控 nginx 的两种方式](https://blog.csdn.net/lvan_test/article/details/123579531)

修改 nginx.conf

http 中添加如下

```
vhost_traffic_status_zone;
```

server 下添加如下

```
location /status {
        vhost_traffic_status_display;
        vhost_traffic_status_display_format html;
        }
```

#### 原理

通过 nginx-vts-exporter 获取虚拟机上 nginx 的数据 , 再通过 servicemonitor 暴露

nginx-exporter.yaml

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-exporter
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-exporter
  template:
    metadata:
      labels:
        app: nginx-exporter
    spec:
      containers:
        - name: nginx-exporter
          image: sophos/nginx-vts-exporter
          command: ["/bin/sh", "-c", "--"]
          args: ["nginx-vts-exporter -nginx.scrape_uri=http://192.168.122.100/status/format/json"]
```

nginx-exporter-service.yaml

```
apiVersion: v1
kind: Service
metadata:
  name: nginx-exporter
  namespace: monitoring
  labels:
    app: nginx-exporter
spec:
  type: ClusterIP
  selector:
    app: nginx-exporter
  ports:
  - protocol: TCP
    port: 9913
    targetPort: 9913
    name: ng
```

service_monitor_nginx.yaml

```
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  labels:
    app: exporter-vts0422
  name: nginx-exporter
  namespace: monitoring
spec:
  jobLabel: app
  selector:
    matchLabels:
      app: nginx-exporter
  namespaceSelector:
    matchNames:
    - monitoring
  endpoints:
  - port: ng
    interval: 30s
    honorLabels: true
```

```bash
kubectl get servicemonitors.monitoring.coreos.com -n monitoring #查看servicemonitor服务
```
