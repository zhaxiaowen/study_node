# k8s 故障排查思路

> 按照以下至上的顺序排查 : 先检查 pod, 再到 service 和 ingress

## 排查 Deployent 故障

* pod 启动错误

  - ImagePullBackoff
    - 镜像名称无效 : 拼写错误或者镜像不存在
    - 指定的 tag 不存在
    - 检索的镜像是私有仓库 , 且 k8s 没有访问它的凭据

  - ImageInspectError

  - ErrImagePull

  - ErrImageNeverPull

  - RegistryUnavailable

  - InvalidImageName

* 运行时错误

  - CrashLoopBackOff
    - 应用程序存在错误 , 导致无法启动
    - 配置错误
    - liveness 探测失败 (端口错了 ; 未配置 healthcheck 端口 ; 端口启动慢)

  - RunContainerError
    - 配置错误导致的

  - KillContainerError

  - VerifyNonRootError

  - RunInitContainerError

  - CreatePodSandboxError

  - ConfigPodSandboxError

  - KillPodSandboxError

  - SetupNetworkError

  - TeardownNetworkError

* pod 处于 pending 状态

  * 集群资源不够

  * 当前命名空间有资源限制

  * pod 与一个 pending 状态的 pvc 绑定

  * ```
    # 检查事件
    kubectl describe pod <pod name>

    kubectl get events 
    ```

* pod 不处于 Ready 状态
  * pod 正在运行但是不 Ready, 意味着 Readiness 探针出现故障 . 当 Readiness 探针出现故障时 ,Pod 无法附加到 Service 上 , 并且流量无法转发到实例上
  * Readiness 探针故障是特定于应用程序的错误 , 因此使用 `kubectl describe` 来检查事件部分

## 排查 Service 故障

> Pod 正在运行并且就绪 , 但是仍旧无法接受来自应用程序的响应

* 检查 service 对应的 endpoint, 如果 endpoint 为空
  * label 没有匹配到 pod
  * service 的 selector 标签有错别字
* endpoint 正常
  * 检查 service 的 `targetPort`

## 排查 ingress 故障

* 检查 ingress 使用的 `serviceName` 和 `servicePort` 的信息是否配置正确
  * `kubectl describe ingress <ingress-name>`
  * 如果 Backend 列是空的 , 配置肯定存在错误
  * Backend 正常
    * 将 ingress 暴露于公网的方式有问题
    * 将集群暴露于公网的方式有问题
    * `kubectl port-forward nginx-ingress-controller-6fc5bcc 3000:80 --namespace kube-system`
    * 将访问的端口 `3000` 转发到 nginx-ingress 的 80 端口 , 检查是否能正常访问
      * 如果工作正常 , 问题就处在基础设施 , 应该检查流量如何路由到集群
      * 如果不正常 , 问题就在 ingress controller, 检查日志

* ingress-nginx 有 kubectl 的官方插件
* `kubectl ingress-nginx lint`: 检查 nginx.conf
* `kubectl ingress-nginx backend`: 检查 Backend
* `kubectl ingress-nginx logs`: 检查日志
