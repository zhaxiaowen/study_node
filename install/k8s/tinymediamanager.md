# tinymediamanager

## tinymediamanager.yaml

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tinymediamanager-server
  namespace: jellyfin
  labels:
    app: tinymediamanager-server
spec:
  selector:
    matchLabels:
      app: tinymediamanager-server
  template:
    metadata:
      labels:
        app: tinymediamanager-server
    spec:
      nodeSelector:
        node: node1
      containers:
      - name: tinymediamanager-server
        image: tinymediamanager:v2
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 5800
          name: http
        - containerPort: 5900
          name: tcp
        env:
        - name: GROUP_ID
          value: '0'
        - name: USER_ID
          value: '0'
        resources:
          limits:
            cpu: "200m"
            memory: 1Gi
          requests:
            cpu: "200m"
            memory: 1Gi
        volumeMounts:
        - name: tinymediamanager-data
          mountPath: /media
        - name: tinymediamanager-config
          mountPath: /config
        - name: localtime
          mountPath: /etc/localtime
          readOnly: true
      volumes:
      - name: localtime
        hostPath:
          path: /etc/localtime
      - name: tinymediamanager-data
        hostPath:
          path: /data/movie
          type: Directory
      - name: tinymediamanager-config
        hostPath:
          path: /data/tinymediamanager/config
          type: Directory

---
apiVersion: v1
kind: Service
metadata:
  name: tinymediamanager-server
  namespace: jellyfin
spec:
  ports:
  - name: http
    protocol: TCP
    port: 5800
    targetPort: http
  selector:
    app: tinymediamanager-server
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tinymediamanager-ingress
  namespace: jellyfin
  annotations:
    kubernetes.io/ingress.class: "nginx"
    prometheus.io/http_probe: "true"
spec:
  rules:
  - host: tinymediamanager.zhaoxw.work
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: tinymediamanager-server
            port:
              number: 5800

```

## Dockerfile

```
#为了解决不识别中文的问题
wget https://mirrors.aliyun.com/alpine/edge/testing/x86_64/font-wqy-zenhei-0.9.45-r2.apk 
apk add --allow-untrusted font-wqy-zenhei-0.9.45-r2.apk 
---
# Dockerfile
# Version: 0.0.1
FROM romancin/tinymediamanager
COPY font-wqy-zenhei-0.9.45-r2.apk /tmp/font-wqy-zenhei-0.9.45-r2.apk
RUN ["apk","add","--allow-untrusted","/tmp/font-wqy-zenhei-0.9.45-r2.apk"]

# 构建镜像
docker build -t tinymediamanager:v2 .
```
