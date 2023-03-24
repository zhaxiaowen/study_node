# K8S 部署 redis-cluster

### 0.Dashboard 编号

### 1. 部署 prometheus+grafana

```yaml
git clone https://gitee.com/liugpwwwroot/k8s-prometheus-grafana.git
kubectl apply -f k8s-prometheus-grafana/prometheus
kubectl apply -f k8s-prometheus-grafana/grafana
```

##### TODO: 修改了 prometheus 和 grafana 版本，

```yaml
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  labels:
    name: prometheus-deployment
  name: prometheus
  namespace: kube-system
spec:
  serviceName: "prometheus"
  replicas: 1
  selector:
    matchLabels:
      app: prometheus
  template:
    metadata:
      labels:
        app: prometheus
    spec:
      containers:
      - image: prom/prometheus:latest
        name: prometheus
        command:
        - "/bin/prometheus"
        args:
        - "--config.file=/etc/prometheus/prometheus.yml"
        - "--storage.tsdb.path=/prometheus"
        - "--storage.tsdb.retention=24h"
        ports:
        - containerPort: 9090
          protocol: TCP
        volumeMounts:
        - mountPath: "/prometheus"
          name: monitor
        - mountPath: "/etc/prometheus"
          name: config-volume
        resources:
          requests:
            cpu: 100m
            memory: 100Mi
          limits:
            cpu: 500m
            memory: 2500Mi
      serviceAccountName: prometheus
      volumes:
      - name: config-volume
        configMap:
          name: prometheus-config
  volumeClaimTemplates:
  - metadata:
      name: monitor
      annotations:
        volume.beta.kubernetes.io/storage-class: "course-nfs-storage"
    spec:
      accessModes:
        - ReadWriteMany
      resources:
        requests:
          storage: 3Gi
kind: StatefulSet
apiVersion: apps/v1
metadata:
  labels:
    app: grafana
  name: grafana
  namespace: kube-system  
spec:
  serviceName: "grafana"
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: grafana
  template:
    metadata:
      labels:
        app: grafana
    spec:
      securityContext:
        runAsUser: 0
      containers:
        - name: grafana
          image: grafana/grafana:latest
          imagePullPolicy: IfNotPresent
          env:
            - name: GF_AUTH_BASIC_ENABLED
              value: "true"
            - name: GF_AUTH_ANONYMOUS_ENABLED
              value: "false"
          readinessProbe:
            httpGet:
              path: /login
              port: 3000
          volumeMounts:
            - mountPath: /var/lib/grafana
              name: gf
          ports:
            - containerPort: 3000
              protocol: TCP
  volumeClaimTemplates:
  - metadata:
      name: gf
      annotations:
        volume.beta.kubernetes.io/storage-class: "course-nfs-storage"
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 1Gi
```

### 2. 部署 redis-cluster+redis_exporter

#### nfs-client.yaml

```yaml
kind: Deployment
apiVersion: apps/v1
metadata:
  name: nfs-client-provisioner
  namespace: wiseco
spec:
  replicas: 1
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: nfs-client-provisioner
  template:
    metadata:
      labels:
        app: nfs-client-provisioner
    spec:
      serviceAccountName: nfs-client-provisioner
      containers:
        - name: nfs-client-provisioner
          image: quay.io/external_storage/nfs-client-provisioner:latest
          volumeMounts:
            - name: nfs-client-root
              mountPath: /persistentvolumes
          env:
            - name: PROVISIONER_NAME
              value: fuseim.pri/ifs
            - name: NFS_SERVER
              value: 192.168.50.100
            - name: NFS_PATH
              value: /data/k8s
      volumes:
        - name: nfs-client-root
          nfs:
            server: 192.168.50.100
            path: /data/k8s
```

#### nfs-client-sa.yaml

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nfs-client-provisioner
  namespace: wiseco
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: nfs-client-provisioner-runner
  namespace: wiseco
rules:
  - apiGroups: [""]
    resources: ["persistentvolumes"]
    verbs: ["get", "list", "watch", "create", "delete"]
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    verbs: ["get", "list", "watch", "update"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["list", "watch", "create", "update", "patch"]
  - apiGroups: [""]
    resources: ["endpoints"]
    verbs: ["create", "delete", "get", "list", "watch", "patch", "update"]

---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: run-nfs-client-provisioner
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner
    namespace: wiseco
roleRef:
  kind: ClusterRole
  name: nfs-client-provisioner-runner
  apiGroup: rbac.authorization.k8s.io
```

#### nfs-client-class.yaml

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: course-nfs-storage
  namespace: wiseco
provisioner: fuseim.pri/ifs # or choose another name, must match deployment's env PROVISIONER_NAME'
```

#### redis-configmap.yaml

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: redis-cluster
  namespace: wiseco
data:
  fix-ip.sh: |
    #!/bin/sh
    CLUSTER_CONFIG="/data/nodes.conf"
    if [ -f ${CLUSTER_CONFIG} ]; then
      if [ -z "${POD_IP}" ]; then
        echo "Unable to determine Pod IP address!"
        exit 1
      fi
      echo "Updating my IP to ${POD_IP} in ${CLUSTER_CONFIG}"
      sed -i.bak -e '/myself/ s/[0-9]\{1,3\}\.[0-9]\{1,3\}\.[0-9]\{1,3\}\.[0-9]\{1,3\}/'${POD_IP}'/' ${CLUSTER_CONFIG}
    fi
    exec "$@"
  redis.conf: |
    cluster-enabled yes
    cluster-config-file /data/nodes.conf
    cluster-node-timeout 10000
    protected-mode no
    daemonize no
    pidfile /var/run/redis.pid
    port 6379
    tcp-backlog 511
    bind 0.0.0.0
    timeout 3600
    tcp-keepalive 1
    loglevel verbose
    logfile /data/redis.log
    databases 16
    save 900 1
    save 300 10
    save 60 10000
    stop-writes-on-bgsave-error yes
    rdbcompression yes
    rdbchecksum yes
    dbfilename dump.rdb
    dir /data
    requirepass root
    masterauth root
    appendonly yes
    appendfilename "appendonly.aof"
    appendfsync everysec
    no-appendfsync-on-rewrite no
    auto-aof-rewrite-percentage 100
    auto-aof-rewrite-min-size 64mb
    lua-time-limit 20000
    slowlog-log-slower-than 10000
    slowlog-max-len 128
    #rename-command FLUSHALL  ""
    latency-monitor-threshold 0
    notify-keyspace-events ""
    hash-max-ziplist-entries 512
    hash-max-ziplist-value 64
    list-max-ziplist-entries 512
    list-max-ziplist-value 64
    set-max-intset-entries 512
    zset-max-ziplist-entries 128
    zset-max-ziplist-value 64
    hll-sparse-max-bytes 3000
    activerehashing yes
    client-output-buffer-limit normal 0 0 0
    client-output-buffer-limit slave 256mb 64mb 60
    client-output-buffer-limit pubsub 32mb 8mb 60
    hz 10
    aof-rewrite-incremental-fsync yes
```

#### redis_exporter_svc.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app: redis-exporter
  name: redis-exporter
  namespace: wiseco
spec:
  ports:
  - name: prom
    port: 9121
    targetPort: 9121
  - name: redis
    port: 6379
    targetPort: 6379
  selector:
    app: redis-cluster
```

#### redis_svc.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  namespace: wiseco
  name: redis-cluster
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "9121"
spec:
  ports:
  - port: 6379
    targetPort: 6379
    name: client
  - port: 16379
    targetPort: 16379
    name: gossip
  - port: 9121
    targetPort: 9121
    name: prom
  selector:
    app: redis-cluster
```

#### redis-cluster.yml

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  namespace: wiseco
  name: redis-cluster
spec:
  serviceName: redis-cluster
  podManagementPolicy: OrderedReady
  replicas: 6
  selector:
    matchLabels:
      app: redis-cluster
  template:
    metadata:
      labels:
        app: redis-cluster
    spec:
      containers:
      - name: redis
        image: redis
        ports:
        - containerPort: 6379
          name: client
        - containerPort: 16379
          name: gossip
        command: ["/etc/redis/fix-ip.sh", "redis-server", "/etc/redis/redis.conf"]
        env:
        - name: POD_IP
          valueFrom:
            fieldRef:
              fieldPath: status.podIP
        volumeMounts:
        - name: conf
          mountPath: /etc/redis/
          readOnly: false
        - name: data
          mountPath: /data
          readOnly: false
      - name: redis-exporter
        image:  oliver006/redis_exporter:latest
        ports:
          - containerPort: 9121
            name: http
      dnsPolicy: ClusterFirst
      volumes:
      - name: conf
        configMap:
          name: redis-cluster
          defaultMode: 0755
  volumeClaimTemplates:
  - metadata:
      name: data
      annotations:
        volume.beta.kubernetes.io/storage-class: "course-nfs-storage"
    spec:
      accessModes:
        - ReadWriteMany
      resources:
        requests:
          storage: 3Gi
```

### 初始化集群

> kubectl get pods -l app=redis-cluster -n wiseco -o jsonpath='{range.items[*]}{.status.podIP}:6379 '
>
> kubectl exec -it redis-cluster-0 -n wiseco -- redis-cli -a root --cluster create --cluster-replicas 1 $(kubectl get pods -l app=redis-cluster -n wiseco -o jsonpath='{range.items[*]}{.status.podIP}:6379 ')

#### 上面这条指令拿 ip 时 , 多拿了个 :6379, 导致命令不可用 , 可以先登到一个 redis pod 内 , 然后初始化

> kubectl exec -it redis-cluster-0 -n wiseco -- /bin/bash
>
> redis-cli -a root --cluster create --cluster-replicas 1 10.244.166.136:6379 10.244.166.129:6379 10.244.104.27:6379 10.244.166.130:6379 10.244.135.28:6379 10.244.166.139:6379

#### 验证

> kubectl exec -it redis-cluster-0 -n wiseco -- redis-cli -a root cluster info  
>
> for x in $(seq 0 5); do echo "redis-cluster-$x"; kubectl exec redis-cluster-$x -n wiseco -- redis-cli -a root role; echo; done
