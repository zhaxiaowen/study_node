# gitbook

## TODO

* 通过jenkins从gitlab上拉数据,实现自动部署还未实现

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-gitbook
  namespace: gitbook
  labels:
    env: prod
    k8s-app: gitbook
    tier: backend
    type: app
spec:
  selector:
    matchLabels:
      env: prod
      k8s-app: gitbook
      tier: backend
      type: app
  replicas: 1
  revisionHistoryLimit: 5
  template:
    metadata:
      labels:
        env: prod
        k8s-app: gitbook
        tier: backend
        type: app
      annotations:
        prometheus.app/path: /prometheus/metrics
        prometheus.app/port: "4000"
        prometheus.app/scrape: "true"
    spec:
      nodeSelector:
        node: node3
#      imagePullSecrets:
#        - name: harbor-register
      containers:
        - name: gitbook
          image: fellah/gitbook
          imagePullPolicy: IfNotPresent
          env:
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: LANG
              value: en_US.UTF-8
            - name: TZ
              value: Asia/Shanghai
            - name: APP_NAME
              value: gitbook
          resources:
            limits:
              cpu: 1001m
              memory: 1Gi
            requests:
              cpu: 100m
              memory: 200Mi
          ports:
            - containerPort: 4000
          livenessProbe:
            tcpSocket:
              port: 4000
            initialDelaySeconds: 240
            timeoutSeconds: 20
          readinessProbe:
            tcpSocket:
              port: 4000
            initialDelaySeconds: 240
            periodSeconds: 10
          terminationMessagePath: /tmp/termination-log
          terminationMessagePolicy: File
#          volumeMounts:
#            - name: lxcfs-cpu
#              mountPath: /proc/cpuinfo
#            - name: lxcfs-stat
#              mountPath: /proc/stat
#            - name: lxcfs-mem
#              mountPath: /proc/meminfo
#            - mountPath: /data/wanshun/log
#              name: logdata
#          workingDir: /data/wanshun
          volumeMounts:
            - mountPath: /srv/gitbook
              name: word-dir
            - mountPath: /srv/html
              name: html
      restartPolicy: Always
      volumes:
        - name: word-dir
          hostPath:
            path: /root/study_node
            type: Directory
        - name: html
          hostPath:
            path: /root/study_node/html
            type: Directory
#      securityContext:
#        runAsUser: 2000
#        runAsGroup: 2000
      terminationGracePeriodSeconds: 30
---
apiVersion: v1
kind: Service
metadata:
  name: gitbook
  namespace: gitbook
  annotations:
    prometheus.io/probe: "true"
spec:
  selector:
    env: prod
    k8s-app: gitbook
    type: app
    tier: backend
  ports:
  - name: tcp
    port: 4000
    protocol: TCP
    targetPort: 4000

---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: gitbook
  namespace: gitbook
  annotations:
    prometheus.io/probe: "true"
    prometheus.io/http_probe: "true"
    kubernetes.io/ingress.class: "nginx"
spec:
  rules:
  - host: gitbook.zhaoxw.work
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: gitbook
            port:
              number: 4000
```



## Dockerfile

> git clone https://github.com/zhaxiaowen/study_node.git

```
FROM fellah/gitbook 
COPY gitbook /srv/gitbook
docker build -t gitbook:v2 .
```

## gitbook.yaml

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-gitbook
  namespace: gitbook
  labels:
    env: prod
    k8s-app: gitbook
    tier: backend
    type: app
spec:
  selector:
    matchLabels:
      env: prod
      k8s-app: gitbook
      tier: backend
      type: app
  replicas: 1
  revisionHistoryLimit: 5
  template:
    metadata:
      labels:
        env: prod
        k8s-app: gitbook
        tier: backend
        type: app
      annotations:
        prometheus.app/path: /prometheus/metrics
        prometheus.app/port: "4000"
        prometheus.app/scrape: "true"
    spec:
      nodeSelector:
        node: node3
#      imagePullSecrets:
#        - name: harbor-register
      containers:
        - name: gitbook
          image: gitbook:v2
          imagePullPolicy: IfNotPresent
          env:
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: LANG
              value: en_US.UTF-8
            - name: TZ
              value: Asia/Shanghai
            - name: APP_NAME
              value: gitbook
          resources:
            limits:
              cpu: 1001m
              memory: 1Gi
            requests:
              cpu: 100m
              memory: 200Mi
          ports:
            - containerPort: 4000
          livenessProbe:
            tcpSocket:
              port: 4000
            initialDelaySeconds: 240
            timeoutSeconds: 20
          readinessProbe:
            tcpSocket:
              port: 4000
            initialDelaySeconds: 240
            periodSeconds: 10
          terminationMessagePath: /tmp/termination-log
          terminationMessagePolicy: File
#          volumeMounts:
#            - name: lxcfs-cpu
#              mountPath: /proc/cpuinfo
#            - name: lxcfs-stat
#              mountPath: /proc/stat
#            - name: lxcfs-mem
#              mountPath: /proc/meminfo
#            - mountPath: /data/wanshun/log
#              name: logdata
#          workingDir: /data/wanshun
      restartPolicy: Always

#      securityContext:
#        runAsUser: 2000
#        runAsGroup: 2000
      terminationGracePeriodSeconds: 30
---
apiVersion: v1
kind: Service
metadata:
  name: gitbook
  namespace: gitbook
  annotations:
    prometheus.io/probe: "true"
spec:
  selector:
    env: prod
    k8s-app: gitbook
    type: app
    tier: backend
  ports:
  - name: tcp
    port: 4000
    protocol: TCP
    targetPort: 4000

---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: gitbook
  namespace: gitbook
  annotations:
    prometheus.io/probe: "true"
    prometheus.io/http_probe: "true"
    kubernetes.io/ingress.class: "nginx"
spec:
  rules:
  - host: gitbook.zhaoxw.work
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: gitbook
            port:
              number: 4000
```

