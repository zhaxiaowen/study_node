# kubernetes-dashboard

## kubernetes-dashboard.yaml

```
# ref: https://raw.githubusercontent.com/kubernetes/dashboard/v2.3.1/aio/deploy/recommended.yaml

# Copyright 2017 The Kubernetes Authors.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

apiVersion: v1
kind: Namespace
metadata:
  name: kubernetes-dashboard

---

apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard

---

kind: Service
apiVersion: v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard
spec:
  type: LoadBalancer
  loadBalancerIP: 192.168.50.201
  ports:
    - port: 443
      targetPort: 8443
      nodePort: 30001
  selector:
    k8s-app: kubernetes-dashboard
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: dashboard
  namespace: kubernetes-dashboard
  annotations:
    kubernetes.io/ingress.class: "nginx"
    prometheus.io/http_probe: "true"
    # 开启use-regex，启用path的正则匹配
    nginx.ingress.kubernetes.io/use-regex: "true"
    nginx.ingress.kubernetes.io/rewrite-target: /
    # 默认为 true，启用 TLS 时，http请求会 308 重定向到https
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    # 默认为 http，开启后端服务使用 proxy_pass https://协议
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
spec:
  rules:
  - host: dashboard.zhaoxw.work
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: kubernetes-dashboard
            port:
              number: 443
  tls:
    - secretName: kubernetes-dashboard-key-holder
---
apiVersion: v1
kind: Secret
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard-certs
  namespace: kubernetes-dashboard
type: Opaque
data:
  dashboard.key: LS0tLS1CRUdJTiBQUklWQVRFIEtFWS0tLS0tCk1JSUV2d0lCQURBTkJna3Foa2lHOXcwQkFRRUZBQVNDQktrd2dnU2xBZ0VBQW9JQkFRQ20vUEpZWG5vY3d2eXgKVnhCdGVTdEJQZ052S1FYOENLd2V6Skg2eWlRN2dDR241VWZVdGpjZXlaeG1KSFlLcC9KWEQ0TDQ5M2JYSktHNApRdGlLUVlpUFFwa21uVmwrajN0dmJIM3FZL1IxQXhTNTNhQllaVTFRZ2NPQ1ZMemg3MktzbkJ5bzFMNndseTQ3CkUxa1d1V2VTZEV2N21acitrcnBtcnBsek5GUVFmNkI2dm1hdWIyVm1hN3dqTmVnejlwWkVsQ2d0dTg5OEVPNEkKT25KSEhlRGFSZ0ZTdFpJY0Q0UmJqYkhFVnNoNTRYSlN0L09heFZNRTJiWTVNSmhIaUVnZXp6ZzFCbW1Pazk5QgpsbVhWRXl0NHUyMVc4L3hUTmJGaUdBWDgwRStQMGh6Mk0xVFNwYitOYU1DVHlPRFRLZ2drZjl3USt4cGgxR2tvCmxjSHZKTmdaQWdNQkFBRUNnZ0VCQUloRHF1TFBuYWZ3dVZGaGNZZFR0Q2RXR21sUU9aRHo1cmh2U01RMHhhSkUKS2JLZkY2R05XNmRrNzVvdU1LRDdjWGIzc25IRlJoWER6Ni9UNUczVmtrRU5JSHB4TmtGZmhtTmpUZERCNWc3QwpCOXl2N0pPVmZxU3ViMExnTVEzUlVWejNPeS9PQXhtSkZIR2lsVFZFOEM2RGRpbUdyQU1HNnRLMXNZUmY5Q1ZOCkhpMU1xc04vajN2MFdSS3ZyUXdaL3o5cWZublR4UFJUdDU2a0s2Z1ViQ3FReitQd3U0SGFtYnlhUmJYZ0Y0cEMKVEZYcUE1a1BUV3QwZ1NhS3JEOFFrRG00RmxvZ1dxNUtkbmJaSTIwNlVBb2dqVkFsdG9xaUpSMUpoVXFzYnpvUQpldXpjbHdSOUFMTnI1MlVyRklDWUFQVFMwYks5cG84MlZZR291OUthVnowQ2dZRUEyOWFoNnpJc1NoUGlwa20yCm1qaTlkUy9mVVd3RHFFWmZ1V3p1WTlqamhMNld5RVRaRTQ1S3A1d0pHWWh0SENic0R3cDZLN256dHZaQmVNWm8KRE0vZDcxTTlFbEQ5QWF4VndYbUFHSlRZWC9zWjlTR2ZPTitzRXlsamtjb0JSaWJZNFNsTHFFRWU0YS8wcTMyVAp0NmFXaUJHNDBLQ2VZZy9iTjNiaTNnQ0tHU01DZ1lFQXduVExFaUVRbXRDZWYwUmhoc1BScnAwckpMdjBhRUVICmV3TFI5YWx0Y0oyS0hNaWpybFJMOFJUK1JhUEVIaUxtQ2NCVmhFV241aGo1SFd4eVF1L3NhT3FNNVh0T0ZIZnYKRktDajFLZ3JYL3Frckxqck5WNXhibEVST3pnQS9TSk5BVDJGRExWdHZUc0ZSVXpoczg0WUhWay9OcEk3bVFzSQpISXpob3gyUEE1TUNnWUVBcVhBMFBHTGZYL2tUcDdjSTFyVUUwVjJrY2MwZXhJUDVJNkdoMjdNL0tRRDhsajc2ClVPaExBZ1J4dnd3M2pJc3pSaVI5SlZhZFVWZGIvd3B0Qi9MdXk1Y01heUdnMzdsRUgycldJQndZNldGUUVHOXAKbVJ4TU5EaWlWYXVzYjdWaFU2blFkazQ2enhnZkxFNE5uRzc1ZHNheCs1clFlQ1JnZ2M5UDdHdmVCS0VDZ1lFQQpqczAyVkJuMEY3MGNxRm1QUldpSWs3TFgvQ0lMV29SbStlOFlRVkFyRG9paTVJQnpzNUkwTXRjMzQreGdHY0dICkxhSVJLeEg4T3Y0YjgzK3dhWGZJSlVRYU5HeFk2cThvNC8wVVV4Y3N3MDlObjRvdE1RUXFTTmsvemoxU2ZKS3oKK2pVemdDRzhkVHJpcEFIUnZqbWJlL0lPZWdUcHYzcGFlcHo3RnM2ZU9BRUNnWUExMi9OUisyRnREU0lnNHJkUApMTS9od3FkVXg0UUk2MnowZUtWQWhUYzc5UXJUZFlCamkwbWZBZFZ4SzE0V2EyTFNsQ0xTOEdDWVBTWHZVWkd0Cnd5ZlUyejNNTm9QYmxLa0VZeUcxU1pTTS9mRHpNcTAvemgxNWdOaHVQeHZONDNrTHQ5UkhBN0g0OWJMaDVYY1AKRU1BMGtWcXVMa1BFaDREL3NiT3hsbmlhR2c9PQotLS0tLUVORCBQUklWQVRFIEtFWS0tLS0tCg==
  dashboard.crt: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSURDekNDQWZPZ0F3SUJBZ0lKQUl1OVF0T2oxSVNsTUEwR0NTcUdTSWIzRFFFQkN3VUFNQnd4R2pBWUJnTlYKQkFNTUVYUnBiV1ZpZVdVdVoybDBhSFZpTG1sdk1CNFhEVEl5TVRFeE56QXhNemN6TkZvWERUTXlNVEV4TkRBeApNemN6TkZvd0hERWFNQmdHQTFVRUF3d1JkR2x0WldKNVpTNW5hWFJvZFdJdWFXOHdnZ0VpTUEwR0NTcUdTSWIzCkRRRUJBUVVBQTRJQkR3QXdnZ0VLQW9JQkFRQ20vUEpZWG5vY3d2eXhWeEJ0ZVN0QlBnTnZLUVg4Q0t3ZXpKSDYKeWlRN2dDR241VWZVdGpjZXlaeG1KSFlLcC9KWEQ0TDQ5M2JYSktHNFF0aUtRWWlQUXBrbW5WbCtqM3R2YkgzcQpZL1IxQXhTNTNhQllaVTFRZ2NPQ1ZMemg3MktzbkJ5bzFMNndseTQ3RTFrV3VXZVNkRXY3bVpyK2tycG1ycGx6Ck5GUVFmNkI2dm1hdWIyVm1hN3dqTmVnejlwWkVsQ2d0dTg5OEVPNElPbkpISGVEYVJnRlN0WkljRDRSYmpiSEUKVnNoNTRYSlN0L09heFZNRTJiWTVNSmhIaUVnZXp6ZzFCbW1Pazk5QmxtWFZFeXQ0dTIxVzgveFROYkZpR0FYOAowRStQMGh6Mk0xVFNwYitOYU1DVHlPRFRLZ2drZjl3USt4cGgxR2tvbGNIdkpOZ1pBZ01CQUFHalVEQk9NQjBHCkExVWREZ1FXQkJSOEZ3Q2g5Q05WVXQ4akQ3bFJsK21Wd2x5Und6QWZCZ05WSFNNRUdEQVdnQlI4RndDaDlDTlYKVXQ4akQ3bFJsK21Wd2x5Und6QU1CZ05WSFJNRUJUQURBUUgvTUEwR0NTcUdTSWIzRFFFQkN3VUFBNElCQVFBegpNVjVFL25tZng4TWNoQXVRbUdxTGR4aGdMdGtPZUNhaiszSkNjS1pwZVVjZGVtbytGTTdFKzlJOXJGRGpJdXArCjZaVjFpZkoyQ0QySUxpR1pJYy9VTXhDLzZVZGdQS3dQQnoxZjVlS0lDbTB6bmpKQm9nN0E0bVpzZnRBLzNoTlQKTmZQbDdESGZDQmhpeVdJRE5DbGxLbjljYUhUc1pubkJ0S1NNemQyY0dXRU0rcHlHc1c1ZCtzcnNJMWxJQTR0Zwp4ejlRcW5YWnNUMHIrQ00rTlpOMGtKS3BQSFVWS3VWZStsOHN2L0xSM1RTQUFHaENRQUdPQkd6eWM4SitpL0VHCldtb0xUamZWZGJWTmVCSTVsSFRBK0Z6V3J6OEE4MVlTSzJZdGlLeTdtcS9hZVJCVTR4Q0JEUk1hbXNoMWZUb2IKc0MyS3NmZUtlazVCSDdmWnJ5ZG8KLS0tLS1FTkQgQ0VSVElGSUNBVEUtLS0tLQo=

---

apiVersion: v1
kind: Secret
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard-csrf
  namespace: kubernetes-dashboard
type: Opaque
data:
  csrf: ""

---

apiVersion: v1
kind: Secret
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard-key-holder
  namespace: kubernetes-dashboard
type: Opaque

---

kind: ConfigMap
apiVersion: v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard-settings
  namespace: kubernetes-dashboard

---

kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard
rules:
  # Allow Dashboard to get, update and delete Dashboard exclusive secrets.
  - apiGroups: [""]
    resources: ["secrets"]
    resourceNames: ["kubernetes-dashboard-key-holder", "kubernetes-dashboard-certs", "kubernetes-dashboard-csrf"]
    verbs: ["get", "update", "delete"]
    # Allow Dashboard to get and update 'kubernetes-dashboard-settings' config map.
  - apiGroups: [""]
    resources: ["configmaps"]
    resourceNames: ["kubernetes-dashboard-settings"]
    verbs: ["get", "update"]
    # Allow Dashboard to get metrics.
  - apiGroups: [""]
    resources: ["services"]
    resourceNames: ["heapster", "dashboard-metrics-scraper"]
    verbs: ["proxy"]
  - apiGroups: [""]
    resources: ["services/proxy"]
    resourceNames: ["heapster", "http:heapster:", "https:heapster:", "dashboard-metrics-scraper", "http:dashboard-metrics-scraper"]
    verbs: ["get"]

---

kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
rules:
  # Allow Metrics Scraper to get metrics from the Metrics server
  - apiGroups: ["metrics.k8s.io"]
    resources: ["pods", "nodes"]
    verbs: ["get", "list", "watch"]

---

apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: kubernetes-dashboard
subjects:
  - kind: ServiceAccount
    name: kubernetes-dashboard
    namespace: kubernetes-dashboard

---

apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: kubernetes-dashboard
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: kubernetes-dashboard
subjects:
  - kind: ServiceAccount
    name: kubernetes-dashboard
    namespace: kubernetes-dashboard

---

kind: Deployment
apiVersion: apps/v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard
spec:
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      k8s-app: kubernetes-dashboard
  template:
    metadata:
      labels:
        k8s-app: kubernetes-dashboard
    spec:
      containers:
        - name: kubernetes-dashboard
          image: registry.aliyuncs.com/kubeadm-ha/kubernetesui_dashboard:v2.3.1
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 8443
              protocol: TCP
          args:
            - --auto-generate-certificates
            - --namespace=kubernetes-dashboard
            # Uncomment the following line to manually specify Kubernetes API server Host
            # If not specified, Dashboard will attempt to auto discover the API server and connect
            # to it. Uncomment only if the default does not work.
            # - --apiserver-host=http://my-address:port
          volumeMounts:
            - name: kubernetes-dashboard-certs
              mountPath: /certs
              # Create on-disk volume to store exec logs
            - mountPath: /tmp
              name: tmp-volume
          livenessProbe:
            httpGet:
              scheme: HTTPS
              path: /
              port: 8443
            initialDelaySeconds: 30
            timeoutSeconds: 30
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            runAsUser: 1001
            runAsGroup: 2001
      volumes:
        - name: kubernetes-dashboard-certs
          secret:
            secretName: kubernetes-dashboard-certs
        - name: tmp-volume
          emptyDir: {}
      serviceAccountName: kubernetes-dashboard
      nodeSelector:
        "kubernetes.io/os": linux
      # Comment the following tolerations if Dashboard must not be deployed on master
      tolerations:
        - key: node-role.kubernetes.io/master
          effect: NoSchedule

---

kind: Service
apiVersion: v1
metadata:
  labels:
    k8s-app: dashboard-metrics-scraper
  name: dashboard-metrics-scraper
  namespace: kubernetes-dashboard
spec:
  ports:
    - port: 8000
      targetPort: 8000
  selector:
    k8s-app: dashboard-metrics-scraper

---

kind: Deployment
apiVersion: apps/v1
metadata:
  labels:
    k8s-app: dashboard-metrics-scraper
  name: dashboard-metrics-scraper
  namespace: kubernetes-dashboard
spec:
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      k8s-app: dashboard-metrics-scraper
  template:
    metadata:
      labels:
        k8s-app: dashboard-metrics-scraper
      annotations:
        seccomp.security.alpha.kubernetes.io/pod: 'runtime/default'
    spec:
      containers:
        - name: dashboard-metrics-scraper
          image: registry.aliyuncs.com/kubeadm-ha/kubernetesui_metrics-scraper:v1.0.6
          ports:
            - containerPort: 8000
              protocol: TCP
          livenessProbe:
            httpGet:
              scheme: HTTP
              path: /
              port: 8000
            initialDelaySeconds: 30
            timeoutSeconds: 30
          volumeMounts:
          - mountPath: /tmp
            name: tmp-volume
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            runAsUser: 1001
            runAsGroup: 2001
      serviceAccountName: kubernetes-dashboard
      nodeSelector:
        "kubernetes.io/os": linux
      # Comment the following tolerations if Dashboard must not be deployed on master
      tolerations:
        - key: node-role.kubernetes.io/master
          effect: NoSchedule
      volumes:
        - name: tmp-volume
          emptyDir: {}
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kubernetes-dashboard
```

## 获取 token

```
kubectl -n kubernetes-dashboard describe secret $(kubectl -n kubernetes-dashboard get secret | grep admin-user | awk '{print $1}')
```
