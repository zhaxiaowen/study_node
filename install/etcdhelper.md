# etcdhelper

```
go get github.com/yamamoto-febc/kube-etcd-helper

/root/go/bin/kube-etcd-helper --endpoint https://172.16.50.30:2379 --cacert=/etc/kubernetes/pki/etcd/ca.crt  --cert=/etc/kubernetes/pki/etcd/server.crt --key=/etc/kubernetes/pki/etcd/server.key $@

# 查看k8s资源列表
./etcdheloper.sh ls  

# 查看指定pod的信息
./etcdheloper.sh get --pretty /registry/pods/kube-system/calico-kube-controllers-6d75fbc96d-4gsrx

# watch pod的创建过程
./etcdheloper.sh watch 2>/dev/null |grep "/registry/pods/default/ubuntu"

kubectl apply -f 1.yaml --v=9 (日志等级)
```
