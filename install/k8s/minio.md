# minio

```
# minio部署后,prometheus监控minio,需要token,否则会报403
# 生成token步骤
1. 宿主机进入minio pod的命名空间
2.下载minio的客户端mc
wget https://dl.min.io/client/mc/release/linux-amd64/mc 
chmod a+x mc
3.设置别名
./mc alias set linux http://localhost:9000
minioadmin minioadmin
4. 查看新添加的别名信息
./mc alias list
5.生成prometheus配置
./mc admin prometheus generate linux
6.将token添加到prometheus的minio_job配置
bearer_token: {token}
```
