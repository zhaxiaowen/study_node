# jenkins



## 生成访问harbor的凭证

```
kubectl create secret generic jenkins-harbor-register -n jenkins \
    --from-file=.dockerconfigjson=/root/.docker/config.json> \
    --type=kubernetes.io/dockerconfigjson
```

