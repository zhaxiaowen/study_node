# jellyfin

## jellyfin.yaml

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jellyfin
  namespace: jellyfin
spec:
  selector:
    matchLabels:
      app: jellyfin
  template:
    metadata:
      labels:
        app: jellyfin
    spec:
      nodeSelector:
        node: node1
      containers:
      - env:
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              apiVersion: v1
              fieldPath: spec.nodeName
        - name: POD_NAME
          valueFrom:
            fieldRef:
              apiVersion: v1
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              apiVersion: v1
              fieldPath: metadata.namespace
        image: jellyfin/jellyfin
        imagePullPolicy: IfNotPresent
        name: jellyfin
        ports:
        - containerPort: 8096
          protocol: TCP
        resources:
          limits:
            cpu: '2'
            memory: 2Gi
          requests:
            cpu: '2'
            memory: 2Gi
        volumeMounts:
        - mountPath: /media
          name: media
        - mountPath: /config
          name: jellyfin-config
        - mountPath: /cache
          name: jellyfin-cache
      restartPolicy: Always
      volumes:
      - name: jellyfin-config
        hostPath:
          path: /data/jellyfin/config
          type: Directory
      - name: jellyfin-cache
        hostPath:
          path: /data/jellyfin/cache
          type: Directory
      - name: media
        hostPath:
          path: /data/movie
          type: Directory
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: jellyfin
  name: jellyfin
  namespace: jellyfin
spec:
  ports:
  - name: web
    port: 8096
    protocol: TCP
    targetPort: 8096
  selector:
    app: jellyfin
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: jellyfin-ingress
  namespace: jellyfin
  annotations:
    kubernetes.io/ingress.class: "nginx"
    prometheus.io/http_probe: "true"
spec:
  rules:
  - host: jellyfin.zhaoxw.work
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: jellyfin
            port:
              number: 8096

```
