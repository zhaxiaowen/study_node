# 部署 K8s1.21

配置时间同步 :https://blog.csdn.net/qq_31086997/article/details/118424324

安装 k8s:https://blog.csdn.net/weixin_44859605/article/details/123551327

安装 docker:https://blog.csdn.net/qq_39591230/article/details/122700133

k8s 初始化和加入集群报错 :

```bash
rm /etc/containerd/config.toml
systemctl restart containerd
```

https://www.1024sky.cn/blog/article/55327

k8s-dashboard:  https://www.cnblogs.com/syjr/p/14341935.html

## 卸载 k8s

```
kubeadm reset -f
yum remove -y kubelet kubeadm kubectl
modprobe -r ipip
lsmod
rm -rf ~/.kube/
rm -rf /etc/kubernetes/
rm -rf /etc/systemd/system/kubelet.service.d
rm -rf /etc/systemd/system/kubelet.service
rm -rf /usr/bin/kube*
rm -rf /etc/cni
rm -rf /opt/cni
rm -rf /var/lib/etcd
rm -rf /var/etcd
```

## 1. 添加 k8s 源

```
# 添加kubernetes源
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
```

## 2. 刷新 yum

```
yum clean all
yum -y makecache
```

## 3. 查看可以下载的 k8s 版本

```
yum list kubelet --showduplicates | sort -r
```

## 4. 安装

```
# master节点
yum install -y kubectl-1.21.2-0 kubeadm-1.21.2-0 kubelet-1.21.2-0

# node节点
yum install -y kubeadm-1.21.2-0 kubelet-1.21.2-0

systemctl enable kubelet && systemctl start kubelet

```

## 5. 初始化

```
<<<<<<< HEAD
kubeadm init --image-repository registry.aliyuncs.com/google_containers --kubernetes-version v1.21.2 --apiserver-advertise-address 192.168.122.100 --pod-network-cidr=10.244.0.0/16 --token-ttl 0
=======
kubeadm init --image-repository registry.aliyuncs.com/google_containers --kubernetes-version v1.21.2 --apiserver-advertise-address 172.16.50.30 --pod-network-cidr=10.244.0.0/16 --token-ttl 0
>>>>>>> 556d332e398c0fb1a7ab5f46f7d60c18f712cd05
```

## 6. 部署 flannel 插件

```
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

## 7.[解决 pvc 一直处于 pending 状态](https://blog.csdn.net/Promise_410/article/details/117112550)

```
vim /etc/kubernetes/manifests/kube-apiserver.yaml

spec:
  containers:
  - command:
    - kube-apiserver
    - --feature-gates=RemoveSelfLink=false

kubectl apply -f /etc/kubernetes/manifests/kube-apiserver.yaml
```

## 8. 设置 course-nfs-storage 为 storageclass 默认

```
kubectl patch storageclass course-nfs-storage -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'

```

## 9.kubectl 命令自动补全

```
yum install -y bash-completion 
source /usr/share/bash-completion/bash_completion

source <(kubectl completion bash)
echo "source <(kubectl completion bash)" >> ~/.bashrc
```
