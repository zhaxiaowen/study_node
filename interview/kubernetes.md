## Kubernetes

### k8s 有哪些组件 , 具体的功能是什么

* master 节点
  * kubectl: 客户端命令行工具 , 整个 k8s 集群的操作入口
  * api server: 资源操作的唯一入口 , 提供认证、授权、访问控制、API 注册和发现等机制
  * controller manager: 负责维护集群的状态 , 故障检测、自动扩展、滚动更新
  * scheduler: 负责资源的调度 , 按照预定的调度策略将 pod 调度到响应的 node 节点上
  * etcd: 担任数据中心 , 保存了整个群集的状态
* node 节点 :
  * kubelet: 负责维护 pod 的生命周期 , 同时也负责 Volume 和网络的管理 , 运行在所有的节点上 , 当 scheduler 确定某个 node 上运行 pod 后 , 将 pod 的具体信息发送给节点的 kubelet,kubelet 根据信息创建和运行容器 , 并向 master 返回运行状态 ; 容器状态不对时 ,kubelet 会杀死 pod, 重新创建 pod
  * kube-proxy: 运行在所有节点上 , 为 service 提供 cluster 内部的服务发现和负载均衡 (外界服务 ,service 接收到请求后就是通过 kube-proxy 来转发到 pod 上的)
  * container-runtime: 负责管理运行容器的软件
  * pod: 集群里最小的单位 , 每个 pod 里面可以运行一个或多个 container

### k8s pod 创建流程

1、用户提交创建 POD 请求。
2、API Server 处理用户请求，存储 Pod 数据到 Etcd。
3、Schedule 通过和 API Server 的监听机制，查看到新的 pod，尝试为 Pod 绑定 Node。
4、过滤主机：调度器用一组规则过滤掉不符合要求的主机，比如 Pod 指定了所需要的资源，那么就要过滤掉资源不够的主机。
5、主机打分：对第一步筛选出的符合要求的主机进行打分，在此阶段，调度器会考虑一些整体优化策略，比如把一个 Replication Controller 的副本分布到不同的主机上，使用最低负载的主机等。
6、选择主机：选择得分最高的主机，进行 binding 操作，结果存储到 Etcd 中。
7、kubelet 根据调度结果执行 Pod 创建操作：绑定成功后，会启动 container, Docker run, scheduler 会调用 API Server 的 API 在 etcd 中创建一个 bound pod 对象，描述在一个工作节点上绑定运行的所有 pod 信息。运行在每个工作节点上的 kubelet 也会定期与 etcd 同步 bound pod 信息，一旦发现应该在该工作节点上运行的 bound pod 对象没有更新，则调用 Docker API 创建并启动 pod 内的容器。
8、kube-proxy 为新创建的 pod 注册动态 DNS 到 CoreOS，然后给 pod 的 service 添加对应的 iptables 规则，用于服务发现和负载均衡。

### 镜像下载策略

* Always: 镜像标签为 latest 时 , 总是从指定的仓库中获取镜像
* Never: 禁止从仓库下载镜像 , 只能使用本地已有的镜像
* IfNotPresent: 仅本地没有对应镜像时 , 才下载
* 默认的镜像策略 : 当镜像标签是 latest 时 , 默认策略是 Always; 当镜像标签是自定义时 , 默认策略是 IfNotPresent

### pod 有哪些状态

* pending: 处在这个状态的 pod 可能正在写 etcd, 调度或者 pull 镜像或者启动容器
* running
* succeeded: 所有的容器已经正常的执行后退出 , 并且不会重启
* failed: 至少有一个容器因为失败而终止 , 返回状态码非 0
* unknown:api server 无法正常获取 pod 状态信息 , 可能是无法与 pod 所在的工作节点的 kubelet 通信导致的

### pod 优雅关闭的过程

1. 用户发出删除 pod 命令
2. K8S 会给旧 POD 发送 SIGTERM 信号；将 pod 标记为“Terminating”状态；pod 被视为“dead”状态，此时将不会有新的请求到达旧的 pod；
3. 并且等待宽限期（terminationGracePeriodSeconds 参数定义，默认情况下 30 秒）这么长的时间
4. 第三步同时运行，监控到 pod 对象为“Terminating”状态的同时启动 pod 关闭过程
5. 第三步同时进行，endpoints 控制器监控到 pod 对象关闭，将 pod 与 service 匹配的 endpoints 列表中删除
6. 如果 pod 中定义了 preStop 处理程序，则 pod 被标记为“Terminating”状态时以同步的方式启动执行；若宽限期结束后，preStop 仍未执行结束，第二步会重新执行并额外获得一个 2 秒的小宽限期 (最后的宽限期，所以定义 prestop 注意时间 , 和 terminationGracePeriodSeconds 参数配合使用),
7. Pod 内对象的容器收到 TERM 信号
8. 宽限期结束之后，若存在任何一个运行的进程，pod 会收到 SIGKILL 信号
9. Kubelet 请求 API Server 将此 Pod 资源宽限期设置为 0 从而完成删除操作

* 发起删除一个 Pod 命令 (发送 **TERM** 信号给 pod) 后系统默认给 30s 的宽限期 ,API-server 标记这个 pod 对象为 **Terminating** 状态
* kubectl 发下 pod 状态为 **Terminating** 则尝试关闭 pod, 如果有定义的 **preStop** 钩子 , 可多给 2s 宽限期
* Controller Manager 将 Pod 从 svc 的 endpoint 移除
* 宽限期到则发送 **TERM** 信号 ,API server 删除 pod 的 API 对象 , 同时告诉 kubectl 删除 pod 资源对象
* pod 在宽限期还未关闭 , 则再发送 SIGKILL 强制关闭
* 执行强制删除后 ,API server 不再等待来自 kubelet 终止 Pod 的确认信号

### 强制删除 StatefulSet 的 Pod, 可能会出现什么问题

* 强制删除不会等待 kubelet 对 Pod 已终止的确认消息 , 无论强制删除是否成功杀死 pod, 都会立即从 API server 中释放该 pod 名字
* 从而可能导致正在运行 pod 的重复

### 静态 Pod

> 正常情况 Pod 是由 Master 统一管理 , 指定 , 分配的 . 静态 Pod 就是不接收 Master 的管理 , 在指定 node 上当 **kuelet** 启动时 , 会自动启动所定义的静态 Pod
>
> 静态 Pod 由特定节点上的 kubelet 进程管理 , 不通过 **apiserver**, 无法与常用的 **deployment** 或者 **Daemonset** 关联

* 为什么能看到静态 Pod:
  * **kubelet** 会为每个它管理的静态 Pod, 调用 api-server 在 k8s 的 apiserver 上创建一个镜像 Pod, 所以可查询 , 进入 pod, 但是不能通过 apiserver 控制 pod(例如不能删除)
* 普通 pod 失败自愈和静态 pod 有什么区别
  * 普通 pod 用工作负载资源来创建和管理多个 pod. 资源的控制器能处理副本的管理、上线 , 并在 pod 失效时提供自愈能力
  * 静态 pod 在特定的节点运行 , 完全由 kubelet 进行监督自愈
* 删除静态 pod: 只能再配置目录下删除 yaml 文件 , 如果用 kubectl 删除 , 静态 pod 会进入 pending, 然后被 kubelet 重启
* 静态 pod 的作用
  * 可以预防误删除 , 可以用来部署一些核心组件 , 保障应用服务总是运行稳定数量和提供稳定服务
  * **kube-scheduler** **kube-apiserver** **kube-controller-manager** **etcd** 都是用静态 pod 部署的

### RC 和 RS

#### 作用 :

* 用来控制副本数量 , 保证 pod 以我们指定的副本数运行
* 确保 pod 健康 : 当 pod 不健康或无法提供服务时 ,RC 会杀死不健康的 pod, 重新创建
* 弹性伸缩 : 通过 RC 动态扩缩容
* 滚动升级

#### RC 和 RS 的区别 :

* RS 支持集合式的 selector,RC 只支持等式
* RS 比 RC 有更强大的筛选能力

#### 注意事项

* 要确保 RS 标签选择器的唯一性 ; 如果多个 RS 的标签选择规则重复 , 可能导致 RS 无法判断 pod 是否为自己创建 , 造成同一个 pod 被多个 RS 接管
* .spec.template.metadata.labels 的值必须与 spec.selector 值相匹配 , 否则会被 API 拒绝

#### 标签 Pod 和可识别标签副本集 ReplicaSet 先后创建顺序不同 , 会造成什么影响

* 无论 RS 何时创建 , 一旦创建 , 会将自己标签能识别的所有 Pod 纳入管理 , 遵循 RS 规约定义的副本数 , 开启平衡机制

#### RS 缩容算法策略

> 缩容时 ,rs 会对所有可用的 pod 进行一次权重排序 , 剔除最不利于系统高可用、稳定运行的 Pod

1. 优先剔除 peding 且不可调度的 pod
2. 如果设置了 **controller.kubernetes.io/pod-deletion-cost** 注解 , 则注解值较小的优先被剔除
3. 所处节点上副本个数较多的 pod 优先于所处节点上副本较少者被剔除
4. 如果 Pod 创建时间不同 , 最近创建的 pod 优先于早前创建的 pod 被剔除

### Ingress

#### Ingress --(待完善)

* ingress 和 ingress controller 结合实现了一个完整的 ingress 负载均衡器
* ingress controller 基于 ingress 规则将客户端请求直接转发到 service 对应的后端 endpoint 上 , 从而跳过了 kube-proxy 的转发功能
* ingress controller+ingress 规则--->service

#### nginx ingress 的原理本质是什么*(原理还不知道)*

* ingress controller 通过和 api server 交互 , 动态的去感知集群中 ingress 规则变化
* 然后按照自定义的规则 , 生成一段 nginx 配置
* 再写到 nginx-ingress-controller 的 pod 里 , 这个 pod 里运行着一个 nginx 服务 , 控制器会把生成的 nginx 配置写入 /etc/nginx.conf 中
* 然后 reload 一下使配置生效 , 以此达到域名分配和动态更新的问题

### 健康检查

#### 健康检查--资源探针 :

* LivenessProbe: 存活探针 , 失败的话杀掉 pod, 并根据容器的重启策略做出相应的处理

* ReadinessProbe: 可读性探针 ,ready 检测 , 失败的话从 service 的 endpoint 列表中删除 pod 的 ip

* startupProbe: 为了防止服务因初始化时间较长 , 被上面 2 种探针 kill, 用来定义初始化时间的

* 设置控制时间

  * initialDelaySeconds: 初始第一次探测间隔时间 , 防止应用还没起来就被健康检查失败
  * periodSeconds: 检查间隔 , 多久执行 probe 检查 , 默认 10s
  * timeoutSeconds: 检查超时时间 , 探测应用 timeout 后为失败
  * successThreshold: 成功探测阈值 , 默认探测一次为健康正常

* 探测方法

  * ExecAction: 在容器中执行一条命令 , 状态码为 0 则健康

  * TcpSocketAction: 与容器的某个 pod 建立连接

  * HttpGetAction: 通过向容器 IP 地址的某个指定端口的指定 path 发送 GET 请求 , 响应码为 2xx 或 3xx 即健康

* 探测结果 :
  * Success: 服务正常
  * Failure: 探测到服务不正常
  * Unknown: 通常是没有定义探针检测 , 默认成功

### 网络

#### kube-proxy iptables 原理

* iptables 模式下的 kube-proxy 不再起到 Proxy 的作用 , 作用是 : 通过 API Server 的 Watch 接口实时跟踪 Service 与 Endpoint 的变更信息 , 并更新到 iptalbes 规则 ,Client 的请求流量通过 iptables 的 NAT 机制 " 直接路由 " 到目标 pod

#### kube-proxy ipvs 原理

* IPVS 用于高性能负载均衡 , 使用更高效的 Hash 表 , 允许几乎无限的规模扩张 , 被 kube-proxy 采纳为最新模式
* IPVS 使用 iptables 的扩展 ipset, 而不是直接调用 iptables 来生产规则链 ,iptable 规则链是一个线性的数据结构 ,ipset 是带索引的数据结构 , 因此当规则多时 , 可以高效的查找和匹配

#### ipvs 为啥比 iptables 效率高

* IPVS 和 iptables 同样基于 Netfilter
* IPVS 采用 hash 表 ;iptables 采用一条条的规则列表
* iptables 又是为防火墙设计的 , 集群数量越多 ,iptables 规则就越多 , 而 iptables 的规则是从上到下匹配的 , 所以效率就越低
* 因此当 service 数量达到一定规模时 ,hash 表的速度优势就显现出来了 , 从而提高了 service 的服务性能
* 优势 :
  * 为大集群提供了更好的可扩展性和性能
  * 支持比 iptables 更复杂的负载均衡算法 (最小负载 , 最少连接 , 加权等)
  * 支持服务器健康检查和连接重试等功能
  * 可以统统修改 ipset 的集合

#### calico 和 flannel 的区别--(待完善)

* flannel: 简单 , 使用居多 , 基于 Vxlan 技术 (叠加网络+二层隧道), 不支持网络策略
* Calico: 较复杂 , 使用率低于 flannel: 也可以支持隧道网络 , 但是是三层隧道 (IPIP), 支持网络策略
* Calico 项目既能够独立的为 k8s 集群提供网络解决方案和网络策略 , 也能与 flannel 结合在一起 , 由 flannel 提供网络解决方案 , 而 calico 仅用于提供网络策略

#### 不同 node 上的 pod 之间的通信流程*--(待完善)*

* pod 的 ip 由 flannel 统一分配 , 通信也走 flannal 网桥
* 每个 node 上都有个 flannal0 虚拟网卡 , 用于跨 node 通信 ,
* 跨节点通信时 , 发送端数据会从 docker0 路由到 flannel0 虚拟网卡 , 接收到数据会从 flannel0 路由到 docker0

#### k8s 集群外流量怎么访问 pod --(待完善)

* NodePort: 会在所有节点上监听同一个端口 , 比如 30000, 访问节点的流量都会被重定向到对应的 service 上

#### 为什么 NetworkPolicy 不用限制 serviceIP 却又能生效？

* 防火墙策略重来不会遇到 clusterIP, 因为在到达防火墙策略前 ,clusterIP 都已经被转成 podIP 了
  * 在 pod 中使用 clusterIP 访问另一个 pod 时，防火墙策略的应用是在所在主机的 FORWARD 点，而把 clusterIP 转成 podIP 是在之前的 PREROUTING 点就完成了
  * 在主机中使用 clusterIP 访问一个 pod 时，防火墙策略的应用是在主机的 OUTPUT 点，而把 clusterIP 转成 podIP 也是在 OUTPUT 点

### storageclass,pv,pvc

* PVC: 定义一个持久化属性 , 比如存储的大小 , 读写权限等
* PV: 具体的 Volume
* storageclass: 充当 PV 的模板 , 不用再手动创建 PV 了
* 流程 :pod-->pvc-->storageclass(provisioner)-->pv,pvc 绑定 pv

### PV 的生命周期

* Available: 可用状态 , 还未绑定 PVC
* Bound: 已绑定 PVC
* Released: 绑定的 PVC 已经删除 , 资源已释放 , 但还没被集群回收
* Failed: 资源回收失败

### k8s 数据持久化

* EmptyDir:yaml 没有指定要挂载宿主机的哪个目录 , 直接由 pod 内部映射到宿主机上 . 同个 pod 里的不同 contianer 共享同一个持久化目录 ,
* HostPath: 将宿主机的目录挂载到容器内部 , 增加了 pod 与节点的耦合度
* PV

### Scheduler

* 调度算法
  * 预选 : 输入所有节点 , 输出满足预选条件的节点 , 过滤掉不符合的 node. 节点的资源不足或者不满足预选策略则无法通过预选
  * 优选 : 根据优先策略为通过预选的 Node 进行打分排名 , 选择得分最高的 Node: 例如 : 资源越服务员 , 负载越小的 node 可能具有越高的排名

### k8s 集群节点需要关机维护 , 怎么操作

* pod 驱逐 :kubectl drain <node_name>
* 确认 node 上已无 pod, 并且被驱逐的 pod 已经正常运行在其他 node 上
* 关机维护
* 维护完成开机
* 解除 node 节点不可调度 :kubectl uncordon node
* 使用节点标签测试 node 是否可以被正常调度

### 更新策略

* Recreate Deployment: 在创建出新的 pod 前杀掉所有已存在的 pod

* Rolling Update Deployment:

  * MaxSurge: 用来指定升级过程中可以超过期望 pod 数量的最大个数 , 可以是一个绝对值 (5), 或者 Pod 数量的百分比 (10%), 默认值是 1; 启动更新时 , 会立即扩容 10%, 待新 Pod ready 后 , 旧的 pod 缩容

  * MaxUnavailable: 指定升级过程中不可用 Pod 的最大数量 , 可以是一个绝对值 (5), 或者 Pod 数量的百分比 (10%), 计算百分比的绝对值向下取整 , 为 0 时 , 默认为 1; 启动更新时 , 会先缩容到 90%, 新的 Pod ready 后 , 旧的副本再缩容 , 确保在升级时所有时刻可以用的 Pod 数量至少是 90%

### 灰度

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

### 为什么 Kubernetes 放弃 DNS 轮询，而依赖代理模式将入站流量转发到后端呢

1. DNS 不遵守记录 TTL, 在 TTL 值到期后 , 依然对结果进行缓存
2. 有些应用程序仅执行一次 DNS 查找 , 但是却会无限期的缓存结果
3. 即时应用和库进行了适当的重新解析 ,DNS 记录上的 TTL 值低或为 0 也可能会给 DNS 带来高负载

## Linux 系统

### time_wait

* 作用 :

  > 帮助被动关闭连接的一方关闭连接

  * 最后的 ACK 是主动关闭端发出的 , 如果这个 ACK 丢失 , 服务器将重发 FIN 报文 . 因此客户端必须维护状态允许它重发最终的 ACK

  * 确保老的数据包在网络中消逝 , 每个数据包存活的时间为 MSL,time_wait 为 2MSL, 确保旧的数据包 , 不会影响到新建立的请求

* 大量 time_wait 造成的影响 :

  * 在搞并发短连接的 TCP 服务器上 , 当服务器处理完请求立刻主动正常关闭连接 , 这个场景下会出现大量 sokcet 处于 time_wait 状态 ,
  * 端口最多 65535 个 , 高并发场景可能占用大量端口

* 如何处理 time_wait 过多

  * net.ipv4.tcp_tw.reuse=1, 表示开启重用 . 允许将 time_wait sockets 重新用于新的 TCP 连接
  * nets.ipv4.tcp_tw.recycle=1, 表示开启 TCP 连接中 time_wait sockets 的快速回收
  * net.ipv4.tcp_fin_timeout: 修改系统默认的 timeout 时间

* 为什么 time_wait 等待时间是 2MSL

### 内存管理

* 虚拟内存
  * 虚拟内存是一种内存分配方案，是一项可以用来辅助内存分配的机制。我们知道，应用程序是按页装载进内存中的。但并不是所有的页都会装载到内存中，计算机中的硬件和软件会将数据从 RAM 临时传输到磁盘中来弥补内存的不足。如果没有虚拟内存的话，一旦你将计算机内存填满后，计算机会对你说：对不起，您无法再加载任何应用程序，请关闭另一个应用程序以加载新的应用程序。对于虚拟内存，计算机可以执行操作是查看内存中最近未使用过的区域，然后将其复制到硬盘上。虚拟内存通过复制技术实现了 妹子，你快来看哥哥能装这么多程序 的资本。复制是自动进行的，你无法感知到它的存在。

* 按需分页
  * 在操作系统中，进程是以页为单位加载到内存中的，按需分页是一种虚拟内存的管理方式。在使用请求分页的系统中，只有在尝试访问页面所在的磁盘并且该页面尚未在内存中时，也就发生了缺页异常，操作系统才会将磁盘页面复制到内存中。

### 进程

* 什么是进程和进程表
  * 进程就是正在执行程序的实例
  * 进程表 : 操作系统为了跟踪每个进程的活动状态 , 维护了一个`进程表`, 在进程表里 , 列出了每个进程的状态以及每个进程使用的资源等
* 进程、线程的区别
* 进程间的通信方式
* 进程间状态模型
* 什么是僵尸进程

### 调度算法

* 影响调度程序的指标有哪些

### 页面置换算法

### 死锁

## other

### nginx reload 流程

1. 向 master 进程发送 HUP 信号 (reload 命令)
2. master 进程校验配置语法是否正确
3. master 进程打开新的监听端口
4. master 进程用新配置启动新的 worker 子进程
5. master 进程向老 worker 进程发送 QUIT 信号
6. 老 worker 进程关闭监听句柄 , 处理完当前连接后结束进程

### 浏览器输入网址到显示 , 期间发生了什么
