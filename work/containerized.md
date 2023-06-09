# 容器化总结

## 变更前

* 变更方案明确,尽量不要修改
  * 配置文件
    * yaml统一生成放到git上
    * 我们前期是用运开的模板临时生成yaml,然后部署;后期改为先生成yaml到git,然后部署
    * nginx的配置文件是否允许动态加载
  * 容器
    * 前期用的是虚拟机node部署docker,然后部署pod
    * 后期改为tke超级节点
  * 部署
    * 前期说非核心服务部署完成后,核心服务先部署green集群,将流量打到green集群,然后再部署blue集群
    * 实际情况,由于green方案有问题,改为核心服务仍旧按之前方式部署
* 自动化脚本
  * 网段放通校验;mysql库访问权限
  * 配置信息从apollo读取
  * cmdb配置修改脚本:jvm参数;控制器绑定;腾讯云还是华为云
  * 重启线上所有服务的命令
  * 批量发包脚本

## 变更中

* 变更周期尽量缩短
  * 我们的变更时间长达半年左右,带来了很多麻烦
  * 生产环境会有一段时间处于容器和虚拟机并存,发版时,要区分对应服务目前是容器还是虚拟机,用不同的jenkins_job发布
  * uat1环境,虚拟机和容器用的不是同一个jenkins,导致测试常常无法分清到底用哪个jenkins
* 架构有变化,及时同步出来
  * 运开的发包逻辑修改,未及时同步出来,导致发包时报错
  * 运开发包逻辑变化,有些jenkins_job失效,未删除旧的,具体使用的人分不清该用哪个
  * 日志采集从daemonset变为filebeat
* 变更文档
  * 变更文档拉会审核,严格按照文档执行变更
  * 除了关注变更文档上的内容,还要考虑到一些特殊的情况,比如某些服务使用的是独立apollo,zk或者需要额外配置一些白名单等

## 变更完成

* 后续优化
  * 把网关从openresty改为apisix
  * 把org服务单独放到一个namespace
