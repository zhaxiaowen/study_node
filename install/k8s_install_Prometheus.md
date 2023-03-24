# K8s 部署 Prometheus

https://www.cnblogs.com/lfl17718347843/p/14115795.html

#### 1. 克隆 github 项目

```yaml
git clone https://gitee.com/liugpwwwroot/k8s-prometheus-grafana.git
```

#### 2. 安装 node-exporter

```yaml
kubectl apply -f k8s-prometheus-grafana/node-exporter.yaml
```

#### 3. 安装 prometheus 组件

```yaml
kubectl apply -f  k8s-prometheus-grafana/prometheus/rbac-setup.yaml
kubectl apply -f  k8s-prometheus-grafana/prometheus/configmap.yaml 
kubectl apply -f  k8s-prometheus-grafana/prometheus/prometheus.deploy.yml 
kubectl apply -f  k8s-prometheus-grafana/prometheus/prometheus.svc.yml
```

#### 4. 安装 grafana 组件

```yaml
kubectl apply -f k8s-prometheus-grafana/grafana/grafana-deploy.yaml
kubectl apply -f k8s-prometheus-grafana/grafana/grafana-svc.yaml
kubectl apply -f k8s-prometheus-grafana/grafana/grafana-ing.yaml
```

#### 5. 查看组件服务的映射端口

```yaml
kubectl get svc -n kube-system
```

#### 
