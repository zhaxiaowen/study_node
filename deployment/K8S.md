# K8S

### 测试环境迁移到 k8s

1. mysql 问题 :

   mysql 内 token 字段由 100 改为 256, 但是 DBA 操作错误 , 导致登录测试时 , 一直失败

2. 域名解析问题

3. apollo 配置 :

   早期 apollo 账号经过 base64 加密后 , 长度超过限制 , 因此 mysql 账号少写几位 , 迁移 apollo 后 , 新环境的配置 , 是最新的 apollo 账号 , 但是旧 apollo 用户密码未改 , 导致连接失败 ,pod 健康检查失败

### 访问域名报`服务器异常或没网络`

1. F12, 发现请求 imp-test5 报 405 错误
2. 手动访问 `https://imp-test5.wsecar.com/accountWebImpl`, 请求失败 , 不存在路径
3. 查看服务对应 Service, 配置正确
4. 查看 imp-test5 对应 Ingress, 发现更新转发配置 , 只有 `/api/accountWebImpl/` 转发 , 手动添加 `/accountWebImpl`
5. 请求域名 , 报 404, 请求资源不存在 . 查看 imp-test5 的 ingress 配置文件 , 少了后半截 , 手动添加 `rewrite ^/api/accountWebImpl/(.*) /$1 break;rewrite ^/accountWebImpl/(.*) /$1 break;`
6. 请求 , 回复正常

### 华为云容器服务

* 监听器个数不够 , 导致整个容器集群不可用

### k8s 生产故障

[k8s 网络诊断之我的流量去哪了](https://developer.aliyun.com/article/780661)

### k8s 遇到的问题

![k8s 注册问题](../picture/k8s注册问题.JPG)

### 测试环境电商服务调用网约车服务不通

* 问题现象 :
  * 电商虚拟机环境服务调用网约车 k8s 环境一个服务 ,1 个通 1 个不通 , 网约车这边两个服务都是正常的
* 问题排查 :
  * telnet 不通有 2 种原因 ,1 个是端口未启动 , 另一个是安全组问题
* 问题原因 :
  * 虚拟机调用容器是通过 entbus 转发的 ,entbus 的 svc 绑定多个服务端口 , 转发给 entbus 的 pod, 由 entbus 的 pod 转发给具体的后端服务 , 具体的转发规则通过 configmap 挂载到 pod 内
  * 一开始 fat06 环境是没有挂载 entbus-configmap, 但是 18930 端口是通的 ,18933 端口不通 , 以为不需要 entbus-configmap, 排查方向就一直在安全组
  * 后面发现 ,entbus-pod 内部默认有一些规则的 ,18930 刚好有 , 所以即时没挂载 configmap, 也是通的

#### org 服务容器化 ,apollo 地址不对 , 导致注册到 wyc 的 zk 了

* 问题现象 :
  * org 服务容器化后 , 测试时发现 , 点击某个前端页面时 , 会报程序异常 , 但是过个 1s 又会加载出数据
* 问题排查 :
  * F12, 看接口调用 , 发现调用某个网约车的服务故障 , 但这个服务不在本次容器化的服务清单里
  * 最后定位到 nginx, 发现对应接口请求 , 某个 ip 一直响应报错 , 最后发现这个 ip 是虚拟机的 ip, 并且不是对应的网约车服务 ,
  * 再看网约车 zk, 发现 org 的服务确实是注册到了网约车的 zk 里
  * 登录 org 的虚拟机服务器 , 发现虚拟机的 apollo 地址和容器的 apollo 地址不同
* 问题原因 :
  * org 服务的背景 :
    * org 服务其实是拿了网约车 10 多个服务的 jar 包 , 把服务名前面加了 org
    * 然后为 org 单独建了 apollo 和 zk 使用
