# k8s 常用指令

## 获取集群 kubeconfig, 添加 lens 用

```
kubectl config view --minify --raw
```

## 通过证书请求 k8s apiserver

```
curl https://192.168.122.100:6443/api/ --cacert /etc/kubernetes/pki/ca.crt --cert /etc/kubernetes/pki/apiserver-kubelet-client.crt --key /etc/kubernetes/pki/apiserver-kubelet-client.key

curl https://192.168.122.100:6443/api/v1/nodes --cacert /etc/kubernetes/pki/ca.crt --cert /etc/kubernetes/pki/apiserver-kubelet-client.crt --key /etc/kubernetes/pki/apiserver-kubelet-client.key

curl -sX GET -H "Authorization:bearer `cat /root/dashboard/test/cluster.token`" -k https://192.168.50.100:6443/api/v1/nodes/node1/proxy/metrics/cadvisor
```

## DNS 解析

```
StatefulSet中每个Pod的DNS格式为`statefulSetName-{0..N-1}.serviceName.namespace.svc.cluster.local`
例:kubectl exec redis-cluster-0 -n wiseco -- hostname -f  # 查看pod的dns 
```

## 标签操作

```
kubectl get pod -l tier=frontend
kubectl get pod -l 'tier in (frontend),env in (production)'
kubectl get pod -l 'env notin (production)'
kubectl get pod --show-labels # 查看pod,并线上标签内容
kubectl get pod -L env,tier  #显示所有资源对象标签的值
kubectl get pod -l env,tier  #只显示符合键值对象的pod

kubectl label pod grafana abc=123 给grafana pod添加标签
kubectl label pod grafana abc=456  --overwrite 修改标签名
kubectl label pod grafana abc- 删除标签

kubectl label nodes node01 disk=ssd      #给节点node01添加disk标签
kubectl label nodes node01 disk=sss –overwrite    #修改节点node01的标签
kubectl label nodes node01 disk-         #删除节点node01的disk标签


```

## 重启 deployment

```
## 第一种方法
kubectl rollout restart deploy <deployment-name> -n <namespace>

## 第二种方法:加环境变量
kubectl patch deploy <deployment-name> 
  -p '{"spec":{"template":{"spec":{"containers":[{"name":"<container-name>","env":[{"name":"RESTART_TIME","value":"'$(date +%s)'"}]}]}}}}'
  
## 第三种:重新设置镜像
```

## rollout

```
# 查看deployment历史版本

 kubectl rollout history deployment frontend-perfchild -n wswangyueche-dev01
 
 # 查看滚动状态
 kubectl rollout status deployment frontend-perfchild -n wswangyueche-dev01
 
 # 查看当前版本:  Annotations:            deployment.kubernetes.io/revision: 9
 kubectl describe deployment frontend-perfchild -n wswangyueche-dev01
```

## rollout 回滚

```
 # deployment有回滚,pod没回滚功能
 kubectl set image deployment.v1.apps/nginx-deployment nginx=nginx:1.9.1  #更新镜像版本
 
 kubectl rollout history deployment nginx-deployment  #查询deployment
 
 kubectl rollout undo deployment nginx-deployment  #回滚到deployment上一个版本
 
 kubectl rollout undo deployment nginx-deployment --to-revision=2  #回滚到指定版本
 
 kubectl rollout pause deployment nginx-deployment #暂停回滚
 kubectl rollout resume deploy/nginx-deployment    #恢复
```

## 标签

```
kubectl label nodes node1 beta.kubernetes.io/fluentd-ds-ready=true   #添加
kubectl label node node1  beta.kubernetes.io/fluentd-ds-ready-   #删除
```

## 将本地端口 9200 转发到 es-pod 对应的端口

```
port-forward: 转发一个本地端口到容器端口
kubectl port-forward es-0 9200:9200 -n logging
curl http://localhost:9200/_cluster/state?pretty   # 在另一个端口测试
```

## 查看命令执行的日志

```
kubectl run --image=nginx --v=10
kubectl top pods -v 9
kubectl get --raw "/api/v1/nodes/node1/proxy/metrics/resource"
```

## cp

```
kubectl cp mysql-478535978-1dnm2:/tmp/message.log message.log  # 将容器内的文件copy到本地
kubectl cp message.log -n namespace -c mysql-478535978-1dnm2:/tmp/message.log  # 将本地文件copy到容器内
```

## node 管理

```
# 禁止pod调度到该节点
kubectl cordon node3
# 取消禁止调度
kubectl uncordon node3

# 驱逐该节点上的所有pod
kubectl drain node3
```

## 强制删除 pod

```
kubectl delete pod -n monitoring node-exporter-pvwwp --force --grace-period=0
```

## 查找 Pod 另一端的 veth 设备

> 需要在 pod 所在的 node 上执行命令

```
docker exec -it `docker ps |grep -v POD |grep grafana0 |awk '{print $1}'` ip addr |grep eth0@ |awk -F ': <' '{print $1}' |awk -F '@if' '{print $2}'
8
ip addr |grep ^8 |awk -F '@if' '{print $1}'
```

## 查看所有 api 资源

```
kubectl api-resources  #可以看到资源的对象名称
kubectl api-versions # 查看api的版本
```

## 修改 config view

```
export KUBE_APISERVER="https://192.168.50.201:6443"
kubectl config set-cluster kubernetes \
--certificate-authority=/etc/kubernetes/pki/ca.crt \
--embed-certs=true \
--server=${KUBE_APISERVER} \
--kubeconfig=/etc/kubernetes/admin.conf 
kubectl config set-credentials kubernetes-admin \
--client-certificate=/etc/kubernetes/pki/admin.crt \
--client-key=/etc/kubernetes/pki/admin.key \
--embed-certs=true \
--kubeconfig=/etc/kubernetes/admin.conf 
kubectl config set-context kubernetes-admin@kubernetes \
--cluster=kubernetes \
--user=kubernetes-admin \
--kubeconfig=/etc/kubernetes/admin.conf 
kubectl config use-context \
kubernetes-admin@kubernetes \
--kubeconfig=/etc/kubernetes/admin.conf

#操作完后,把admin.conf 拷贝到/root/.kube/config  即可
```

## pod 的 veth_pair 网卡对

```
在pod中,ethtool -S eth0:
[root@node2 ~]# ethtool -S eth0
NIC statistics:
     peer_ifindex: 8  #8就是另一端的设备号
     rx_queue_0_xdp_packets: 0
     rx_queue_0_xdp_bytes: 0
     rx_queue_0_xdp_drops: 0

ip a:
8: cali4dbb7a2e80c@if3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1480 qdisc noqueue state UP group default
    link/ether ee:ee:ee:ee:ee:ee brd ff:ff:ff:ff:ff:ff link-netnsid 4
    inet6 fe80::ecee:eeff:feee:eeee/64 scope link
       valid_lft forever preferred_lft forever
```

### docker

```
# 查看容器细节
docker inspect --format "{ { .NetworkSettings.IPAddress } }" containerid

# 进入容器的网络命令空间,使用宿主机的命令
nsenter
pid=$(docker inspect --format "{ { .State.Pid } }") <container>
nsenter -n -t <pid>
nsenter -t <pid> -n ip addr #进入某个namespace运行命令
lsns #查看当前系统的namespace
lsns -t <type>
ls -al /proc/<pid>/ns/  #查看某进程的namespace
nsenter -t <pid> -n <ip addr> #进入某namespace执行命令
# 不进容器执行命令
docker exec -it <container> /bin/bash -c "ls"

# cp文件到容器
docker cp file1 <container>:/root
```

## 镜像导入导出

```
docker save busybox > busybox.tar
docker load < busybox.tar
```

## 容器导入导出

```
docker export busybox > busybox.tar
cat busybox.tar | docker import - busybox:latest
```

## 删除容器

```
docker stop $(docker ps -q)  # 停用全部运行中的容器
docker rm $(docker ps -aq)   # 删除全部容器
docker stop $(docker ps -q) & docker rm $(docker ps -aq)
```

## 获取容器 pid + veth 名

```
kubectl get pod  # 拿到容器名
docker ps |grep $name # 拿到docker name
docker inspect --format "{ { .State.Pid } }" 6f8c58377aae

# 拿到veth名
if [ ! -d /var/run/netns ]; then sudo  mkdir -p /var/run/netns; fi
ln -sf /proc/${pid}/ns/net /var/run/netns/ns-${pid}
ip netns exec "ns-4012085" ip link show type veth | grep "eth0"
```

## nsenter

```
nsenter -t pid -n pwd		#在容器netns下执行命令
nsenter -n --target 910351   #进入容器netns
```

### 动态监控 pod 状况

```
watch kubectl top pods
```

## 设置 course-nfs-storage 为 storageclass 默认

```
kubectl patch storageclass course-nfs-storage -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

## [解决 pvc 一直处于 pending 状态](https://blog.csdn.net/Promise_410/article/details/117112550)

```
vim /etc/kubernetes/manifests/kube-apiserver.yaml

spec:
  containers:
  - command:
    - kube-apiserver
    - --feature-gates=RemoveSelfLink=false

kubectl apply -f /etc/kubernetes/manifests/kube-apiserver.yaml
```
