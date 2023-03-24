# Jenkins

## Jenkins 部署 k8s 应用

- 编写代码
- 测试
- 编写 Dockerfile
- 构建打包 Docker 镜像
- 推送 Docker 镜像到仓库
- 编写 Kubernetes YAML 文件
- 更改 YAML 文件中 Docker 镜像 TAG
- 利用 kubectl 工具部署应用

## Jenkins 部署 java 项目

1. 输入任务名，构建 maven 项目
2. 填写源码管理，项目 git 地址
3. 构建设置：打包命令；服务器脚本存放路径，脚本执行命令
4. 立即构建
5. jenkins 服务器完成代码 clone, 编译，打包和构建
