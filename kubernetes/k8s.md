# k8s

## k8s 中的资源对象

* apiVersion: 创建该对象所使用的 kubernetes API 版本
* kind: 想要创建的对象类型
* metadata: 帮助识别对象唯一性的数据 , 包括 `name` `UID` `namespace` 字段

* `spec` 字段 : 必须提供 , 用来描述该对象的期望状态 , 以及关于对象的基本信息 

* Annotation: 可以将 kubernetes 资源对象关联到任意的非标识性元数据

## metadata.label 和 template.metadata.label:

* metadata.label 是作用在 kind 对象上的 , 比如说 deployment, 那这个标签是打在 deployment 上的
* template.metadata.label 是打在 pod 上的 ,

### 一 . service

> 为什么 service ip 不能 ping?

* 来源 :
  * serviceIP 是 serviceController 生成的 , 参数--service-cluster-ip-range string 会配置在 controller-manager 上 ,serviceController 会在这个参数指定的 cidr 范围内取一个 ip
* 原因 :
  * serviceIP 是虚拟的地址 , 没有分配给任何网络接口 , 当数据包传输时不会把这个 IP 作为数据包的源 IP 和目的 IP
  * kube-proxy 在 iptables 模式下 , 这个 IP 没有被设置在任何的网络设备上 ,ping 这个 IP 的时候 , 没有任何的网络协议栈会回应这个 IP
  * 在 iptables 模式时 ,clusterIP 会在 iptables 的 PREROUTING 链里面用于 nat 转换规则中 , 而在 ipvs 模式下 , 会使用 ipvs 模式的 ipvs 规则转换中
  * 在 ipvs 模式下 , 所有的 clusterIP 会被设置在 node 上的 kube-ipvs0 的虚拟网卡上 , 所以是 ping 通

1. endpoint

   * 用来记录一个 service 对应的所有 pod 的访问地址 , 存储在 etcd 中 , 就是 service 关联的 pod 的 ip 地址和端口
   * service 配置了 selector,endpoint controller 才会自动创建对应的 endpoint 对象 , 否则不会生成 endpoint 对象

2. 没有 selector 的 Service

   * 使用 k8s 集群外部的数据库
   * 希望服务执行另一个 namespace 中或其他集群中的服务
   * 正在将工作负载迁移到 k8s 集群

3. ExternalName

   * 没有 selector, 也没有定义 port 和 endpoint, 对于运行在集群外的服务 , 通过返回该外部服务的别名这种方式来提供服务

     ```
     kind: Service
     apiVersion: v1
     metadata:
       name: my-service
       namespace: prod
     spec:
       type: ExternalName
       externalName: my.database.example.com

     当请求 my-service.prod.svc.cluster 时 , 集群的 DNS 服务将返回一个 my.database.example.com 的 CNAME 记录
     ```

4. Headless service

   * 不需要或不想要负载均衡 , 指定 spec.ClusterIP: None 来创建 Headless Service
   * 对这类 service 不会分配 ClusterIP,kube-proxy 不会处理它们 , 不会为它们进行负载均衡和路由 ;DNS 如何实现自动配置 , 依赖于 Service 是否定义了 selector
   * 配置 selector 的 ,endpoint 控制器在 API 中创建了 endpoints 记录 , 并且修改 DNS 配置返回 A 记录 , 通过这个地址直达后端 Pod

5. externalIPs

   * *my-service*可以在 80.11.12.10:80 上被客户端访问

     ```
     kind: Service
     apiVersion: v1
     metadata:
       name: my-service
     spec:
       selector:
         app: MyApp
       ports:
         - name: http
           protocol: TCP
           port: 80
           targetPort: 9376
       externalIPs: 
         - 80.11.12.10
     ```

### [二 . configMap 使用](https://www.bbsmax.com/A/kvJ3NoVwzg/)

1. items 字段使用 :

* 不想以 key 名作为配置文件名可以引入​​items​​​ 字段，在其中逐个指定要用相对路径​​path​​替换的 key
* 只有 items 下的 key 对应的文件会被挂载到容器中

```
  volumes:
    - name: config-volume
      configMap:
        name: cm-demo1
        items:
        - key: mysql.conf
          path: path/to/msyql.conf
```

2. valueFrom: 映射一个 key 值 , 与 configMapKeyRef 搭配使用
3. envFrom: 把 ConfigMap 的所有键值对都映射到 Pod 的环境变量中去 , 与 configMapRef 搭配使用

### [三 . HPA](https://zhuanlan.zhihu.com/p/368865741)

* autoscaling/v1: 只支持 CPU 一个指标的弹性伸缩
* autoscaling/v2beta1: 支持自定义指标
* autoscaling/v2beat2: 支持外部指标

1. metrics 指标
   * averageUtilization: 当整体资源超过这个百分比 , 会扩容
   * averageValue: 指标的平均值超过这个值 , 扩容
   * Value: 当指标的值超过这个 value 时 , 扩容
2. HPA 算法
   * 期望副本数 = 当前副本数 * (当前指标 / 期望指标)
   * 当前指标 200m, 目标设定 100m,200/100=2, 副本数翻倍
   * 当前指标 50m, 目标设定 100m,50/100=0.5, 副本数减半
   * 如果比例接近 1.0(根据--horizontal-pod-autoscaler-tolerance 参数全局配置的容忍值 , 默认 0.1), 不扩容

3. 冷却 / 延迟支持
   * 防抖动功能
   * --horizontal-pod-autoscaler-downscale-stabilization: 设置缩容冷却时间窗口长度 . 水平 pod 扩缩容能够记住过去建议的负载规模 , 并仅对此事件窗口内的最大规模执行操作 . 默认 5min
4. 扩所策略 : 平滑的操作
   * behavior 字段可以指定一个或多个扩缩策略

5. HPA 存在的问题 : 基于指标的弹性有滞后效应 , 因为弹性控制器操作的链路过长
   * 应用指标数据已经超出阈值
   * HPA 定期执行指标收集滞后效应
   * HPA 控制 Deployment 进行扩容的时间
   * Pod 调度 , 运行时启动挂载存储和网络的时间
   * 应用启动到服务就绪的时间
   * 很可能在突发流量出现时 , 还没完成弹性扩容 , 既有的服务实例已经被流量击垮

```
apiVersion: autoscaling/v2beta2
  kind: HorizontalPodAutoscaler
  metadata:
    name: web
    namespace: default
  spec:
    behavior: 
      scaleDown:   # 缩容速度策略
        policies: 
          periodSeconds:  15  # 每15s最多缩减currentReplicas * 100%个副本
          type: Percent
          value: 100
        stabilizationWindowSeconds: 300  # 且缩容后的最终副本不得低于过去300s内计算的历史副本数的最大值
      scaleUp: 	# 扩容速度策略
        policies:
        - type: Percent # 每15s翻倍
          value: 100
          periodSeconds: 15
        - type: Pods	# 每15s新增4个副本
          value: 4
          periodSeconds: 15
        selectPolicy: Max   # 上面2种情况取最大值:max(2* currentReplicas,4)
        stabilizationWindowSeconds: 0
    maxReplicas: 10   # 最大多少副本
    minReplicas: 1		# 最少副本数
    scaleTargetRef: 	# 需要动态伸缩的目标
      apiVersion: apps/v1
      kind: Deployment
      name: d1
    metrics: 
    - type: Resource 
      resource:  #伸缩对象Pod的指标.target.type只支持Utilization和AveragevALUE类型
        name: cpu
        target:
          type: Utilization
          averageUtilization: 50
    - type: ContainerResource
      containerResource:	# 伸缩对象下的container的cpu和memory指标,只支持Utilization和AverageValue
        container: C1
        name: memory
        target:
          type: AverageValue
          averageValue: 300Mi
    - type: Pods
      pods:		# 指的是伸缩对象pod的指标,数据需要第三方的adapter指标,只允许AverageValue类型的阈值
        metric:
          name: Pods_second 
          selector: 
          - matchExpressions: 
            - key: zone
              operator: In
              values: 
              - foo
              - bar
        target:
          type: AverageValue
          averageValue: 1k
    - type: External
      external:	# k8s外部的指标,第三方adapter提供,只支持value和AverageValue类型的阈值
        metric:
          name: External_second
          selector: "queue=worker_tasks"
        target:
          type: Value
          value: 20
    - type: Object
      object:
        describedObject:
          apiVersion: networking.k8s.io/v1beta1
          kind: Ingress
          name: main-route
        metric:
          name: ingress_test
        target:
          type: Value
          value: 2k
```

### 四 . VPA

* 纵向扩容 pod, 但是要重建 pod
* 社区有说法是可以修改 requests, 但是一直没合进去

### 五 . 网络原理

## [docker](https://blog.csdn.net/qq_41688840/article/details/108708415)

* 默认 bridge 桥接网络 : docker 自身生产一个 veth pair(虚拟网卡对) 一端放在 docker0 网桥上 , 一端放在容器内部
* 共享宿主机 Host 的网络 : 容器直接使用宿主机的的网络栈以及端口 port 范围
* container 共享模式 : 指定容器与另外某个已存在的容器共享它的网络 .k8s 里的 pause 容器 , 其他容器通过共享 pause 容器的网络栈 , 实现与外部 Pod 进行通信 , 通过 localhost 进行 Pod 内部的 container 的通信
* none 模式 : 此模式只给容器分配隔离的 network namespace, 不会分配网卡 ,ip 地址等 .k8s 网络插件是基于此做的网络分配 , 插件分配 ip 地址、网卡给 pause 容器

## 总结

* k8s 网络插件负责管理 ip 地址分配以及 veth pair(虚拟网卡对) 以及网桥连接工作 ,pause 容器采用 docker 的 none 网络模式 , 网络插件再将 ip 和 veth 虚拟网卡分配给 pod 的 pause 容器 , 其他的容器再采用共享 container 的网络模式来共享这个 pause 的网络即可与外界进行通信

## [pod 内部的容器如何通信](https://blog.csdn.net/weixin_41947378/article/details/110749413?ops_request_misc=%257B%2522request%255Fid%2522%253A%2522166065926816780366590319%2522%252C%2522scm%2522%253A%252220140713.130102334.pc%255Fblog.%2522%257D&request_id=166065926816780366590319&biz_id=0&utm_medium=distribute.pc_search_result.none-task-blog-2~blog~first_rank_ecpm_v1~rank_v31_ecpm-8-110749413-null-null.nonecase&utm_term=k8s&spm=1018.2226.3001.4450)

## 同节点的 pod 是如何通信的

* pod 通过 pause 容器的 veth 连接到宿主机的 docker0 虚拟网桥上 , 同节点的 pod 就是通过 docker0 这个虚拟网桥通信的
* flannel 插件有 2 种网卡 ,**cni0** 负责本节点的 ,**flannel.1** 负责跨节点

## 不同节点的 pod 是如何通信的

* pod 的 ip 由 flannel 统一分配 , 通信也走 flannal 网桥
* 每个 node 上都有个 flannal0 虚拟网卡 , 用于跨 node 通信 ,
* 跨节点通信时 , 发送端数据会从 docker0 路由到 flannel0 虚拟网卡 , 接收到数据会从 flannel0 路由到 docker0

### 路由路径

1. Kubectl get pod -n monitor -o wide : 查看两个通信的 Pod 的 ip
2. **netstat -rn** 查看路由表
3. node 间通信是通过 flannel 网桥处理的 ,flannel 会在 etcd 中查找 pod 所在 Node 节点的 ip
4. 目的节点的 flannel 收到数据包后 , 去除 flanneld 加上的头部 , 将原始的数据包发送到宿主机的网络栈
5. **route -n**, 根据路由表将包转发给 docker0 网桥上 ,docker0 网桥再将数据包转给对应 pod

## pod 如何对外提供服务

* 将物理机的端口和 pod 做映射 , 访问物理机的 ip+端口 , 转发到 pod, 可以使用 iptabels 的配置规则实现数据包转发

* 共享 pause 容器的网络栈 , 通过 veth0 虚拟网卡通信 , 直接通过 localhost 相互访问

## 为什么 NetworkPolicy 不用限制 serviceIP 却又能生效？

* 防火墙策略重来不会遇到 clusterIP, 因为在到达防火墙策略前 ,clusterIP 都已经被转成 podIP 了
  * 在 pod 中使用 clusterIP 访问另一个 pod 时，防火墙策略的应用是在所在主机的 FORWARD 点，而把 clusterIP 转成 podIP 是在之前的 PREROUTING 点就完成了
  * 在主机中使用 clusterIP 访问一个 pod 时，防火墙策略的应用是在主机的 OUTPUT 点，而把 clusterIP 转成 podIP 也是在 OUTPUT 点

## 误解

1. 相对于直接访问 podIP，使用 clusterIP 来访问因为多一次转发，会慢一些；
   - 其实只是在发送的过程中修改了数据包的目标地址，两者走的转发路径是一模一样的，没有因为使用 clusterIP 而多一跳，当然因为要做 nat 会有一些些影响，但影响不大
2. 使用 nodeport 因为比 clusterIP 又多一次转发，所以更慢；
   - 没有，nodeport 是一次直接就转成了 podIP，并没有先转成 clusterIP 再转成 podIP。

## DNS 解析方式

```
servicename.namespace.svc.cluster.local
```

## Pod 名称格式

```
${deployment-name}-${template-hash}-${random-suffix}
```

## StatefulSet

* StatefulSet 中每个 Pod 的 DNS 格式为 `statefulSetName-{0..N-1}.serviceName.namespace.svc.cluster.local`
* 例 :kubectl exec redis-cluster-0 -n wiseco -- hostname -f  # 查看 pod 的 dns 

### [k8s 部署应用 , 故障排查思路](https://www.cnblogs.com/rancherlabs/p/12330916.html)

[k8s 故障诊断流程](https://cloud.tencent.com/developer/article/1899950)

1. Deploymenr: 创建名为 Pods 的应用程序副本的方法
2. Service: 内部负载局衡器 , 将流量路由到 Pods
3. Ingress: 将流量从集群外部流向 Service

## 故障排查思路

> Pod 是否正在运行
>
> Service 是否将流量路由到 Pod
>
> 检查 Ingress 是否正确配置

## 0. 退出状态码

> kubectl describe pod  ; 查看 State 字段 ,ExitCode 即程序退出的状态码 , 正常退出为 0

* 退出状态码必须在 0-255 之间
* 外界中断将程序退出的时候状态码在 129-255 区间 (操作系统给程序发送中断信号 , 例 :kill-9 等)
* 程序自身原因导致的异常退出 , 状态码在 1-128 区间

## 1. 常见 Pod 错误

> 启动错误

```
ImagePullBackoff
ImageInspectError
ErrImagePull
ErrImageNeverPull
RegistryUnavailable
InvalidImageName
```

> 运行错误

```
CrashLoopBackOff: 可能是应用程序存在错误,导致无法启动;错误配置了容器;liveness探针失败次数太多
RunContainerError
KillContainerError
VerifyNonRootError
RunInitContainerError: 通常是错误配置导致,比如安装一个不存在的volume;将只读volume安装为可读写
CreatePodSandboxError
ConfigPodSandboxError
KillPodSandboxError
SetupNetworkError
TeardownNetworkErro
```

* 例 :CrashLoopBackOff
  1. 获取 describe pod, 主要看 event 事件 :`kubectl describe pod es-0 -n logging` 
  2. 观察 pod 日志 :`kubectl logs es-0 -n logging`
  3. 查看 liveness 探针 , 可能是 pod 因 `liveness` 探测器未返回成功而崩溃 , 再看 `describe pod `

## 2. 排查 Service 故障

1. 主要查看 service 是否与 pod 绑定 :` kubectl describe svc redis-exporter -n wiseco|grep "Endpoints" `
2. 测试端口 :`kubectl port-forward es-0 9200:9200 -n logging`

## 3.k8s - Annotations

### TODO: 可以只在 service 上加 , 因为 service-endpoints 最终也会落到 pod 上

> kubernetes-pods

* prometheus.io/scrape，为 true 则会将 pod 作为监控目标
* prometheus.io/path，默认为 /metrics
* prometheus.io/port , 端口

> kubernetes-service-endpoints

- prometheus.io/scrape，为 true 则会将 pod 作为监控目标
- prometheus.io/path，默认为 /metrics
- prometheus.io/port , 端口
- prometheus.io/scheme 默认 http，如果为了安全设置了 https，此处需要改为 https

## 4. 抓包方法

> https://zhuanlan.zhihu.com/p/372567807
>
> https://blog.csdn.net/chongdang2813/article/details/100863010

### [容器中获取 Pod 信息](https://blog.csdn.net/lsx_3/article/details/124399768)(https://www.cnblogs.com/cocowool/p/kubernetes_get_metadata.html)

* 环境变量 : 将 pod 或容器信息设置为容器的环境变量
* volume 挂载 : 将 pod 或容器信息以文件的形式挂载到容器内部

```
通过fieldRef设定的元数据如下:
metadata.name：Pod名称
metadata.namespace： Pod所在的命名空间名称
metadata.uid：Pod的UID （Kubernetes 1.8.0 +）
metadata.labels[‘<KEY>’]：Pod某个Label的值，通过KEY进行引用
metadata.annotations[‘<KEY>’]：Pod某个Annotation的值，通过KEY进行引用

Pod元数据信息可以设置为容器内的环境变量:
status.podIP：Pod的IP地址
spec.serviceAccountName：Pod使用的ServiceAccount名称
spec.nodeName：Pod所在Node的名称 （Kubernetes 1.4.0 +）
status.hostIP：Pod所在Node的IP地址 （Kubernetes 1.7.0 +）

```

### 调度策略

* Predicate 算法 : 筛选符合条件的 node

| GeneralPredicates           | 包含 3 项基本检查 : 节点 , 端口 , 规则                               |
| --------------------------- | ------------------------------------------------------------ |
| **NoDiskConflict**          | 检查 Node 是否可以满足 Pod 对硬盘的需求                          |
| **NoVolumeZoneConflict**    | 单集群跨 AZ 部署时，检查 node 所在的 zone 是否能满足 Pod 对硬盘的需求 |
| **PodToleratesNodeTaints**  | 检查 Pod 是否能够容忍 node 上所有的 taints                        |
| **CheckNodeMemoryPressure** | 当 Pod QoS 为 besteffort 时，检查 node 剩余内存量，排除内存压力过大的 node |
| **MatchInterPodAffinity**   | 检查 node 是否满足 pod 的亲和性、反亲和性需求                    |

* Priority 算法 : 给剩余的 node 评分 , 挑选最优的节点

| **LeastRequestedPriority**     | 按 node 计算资源 (CPU/MEM) 剩余量排序，挑选最空闲的 node          |
| ------------------------------ | ------------------------------------------------------------ |
| **BalancedResourceAllocation** | 补充 LeastRequestedPriority，在 cpu 和 mem 的剩余量取平衡         |
| SelectorSpreadPriority         | 同一个 Service/RC 下的 Pod 尽可能的分散在集群中。Node 上运行的同个 Service/RC 下的 Pod 数目越少，分数越高。 |
| **NodeAffinityPriority**       | 按 soft(preferred) NodeAffinity 规则匹配情况排序，规则命中越多，分数越高 |
| **TaintTolerationPriority**    | 按 pod tolerations 与 node taints 的匹配情况排序，越多的 taints 不匹配，分数越低 |
| **InterPodAffinityPriority**   | 按 soft(preferred) Pod Affinity/Anti-Affinity 规则匹配情况排序，规则命中越多，分数越高 |

## 限流 :[NetworkPolicy 网络插件](https://blog.csdn.net/xixihahalelehehe/article/details/108422856)

* 基于源 IP 的访问控制 :
  * 限制 Pod 的进 / 出流量
  * 白名单
* Pod 网络隔离的一层抽象
  * lable selector
  * namespace selector
  * port
  * CIDR

### 常用的标签分类

* 版本类标签（release）：stable（稳定版）、canary（[金丝雀](https://www.zhihu.com/search?q=金丝雀&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra={"sourceType"%3A"answer"%2C"sourceId"%3A2307372651})版本，可以将其称之为测试版中的测试版）、beta（测试版）；
* 环境类标签（environment）：dev（开发）、qa（测试）、production（生产）、op（运维）；
* 应用类（app）：ui、as、pc、sc；
* 架构类（tier）：frontend（前端）、backend（后端）、cache（缓存）；
* 分区标签（partition）：customerA（客户 A）、customerB（客户 B）；
* 品控级别（Track）：daily（每天）、weekly（每周）。

### [灰度](https://cloud.tencent.com/document/product/457/48877)

#### Nginx ingress 实现金丝雀发布

> 金丝雀发布场景主要取决于业务流量切分的策略 ,Ningx Ingress 支持基于 Header,Cookie, 和服务权重 3 种流量切分的策略 , 都需要部署 2 个版本的 service 和 deployment

##### 基于 Header 的流量切分

```
 # 仅将带有名为Region且值为cd或sz的请求头的请求转发给当前的Canary Ingress
 # curl -H "Host: canary.example.com" -H "Region: cd" http://EXTERNAL-IP
 annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-by-header: "Region"
    nginx.ingress.kubernetes.io/canary-by-header-pattern: "cd|sz"
  name: nginx-canary
```

##### 基于 Cookie 的流量切分

```
 # 仅将带有名为"user_from_cd"的Cookie的请求转发给当前Canary Ingress
 # curl -s -H "Host: canary.example.com" --cookie "user_from_cd=always" http://EXTERNAL-IP
 annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-by-cookie: "user_from_cd"
```

##### 基于服务权重的流量切分

```
 # for i in {1..10}; do curl -H "Host: canary.example.com" http://EXTERNAL-IP; done;
 annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "10"
```

#### 蓝绿发布和灰度发布

> 蓝绿发布 : 给 2 个集群的 deployment 设置不同的 labels 标签 (version=v1 / version=v2), 通过 service 选择对应的 labels, 切换流量到对应的集群
>
> 灰度发布 :2 个集群部署不同的 deployment,service 同时选中 2 个版本的 deployment, 然后通过控制 deployment 的副本数 (类似副本多的权重就搞), 来控制流量
