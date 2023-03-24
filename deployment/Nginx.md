# Nginx

### Nginx 报 499,5xx

1. 查看 nginx**请求流量状态，没有增加，反而减少；不是流量突增导致故障**
2. **查看 nginx 响应时间监控（rpcdfe.latency）, 响应时间变长**

3. **查看 nginx upstream 响应时间；响应时间边长，猜测后端 upstream 响应时间拖住 nginx, 导致 nginx 出现请求流量异常**
4. **top 查看 cpu，发现 nginx worker cpu 比较高；** 主要开销在 free,malloc,json 解析上面  

### CLB 替换 , 需要加载 nginx

1. nginx 的域名解析直接依赖宿主机 /etc/resolv.conf 配置声明的 DNS 服务器
2. nginx 无视 TTL 并缓存 DNS 记录 , 直到下一次重启或配置重载 , 至于域名解析 ,nginx 启动时就对 `img.ffutop.com` 进行了解析
3. 临时解决方法 :nginx -s reload
4. NGINX 对 DNS TTL 的非标实现，对 IP 频繁发生变更的服务是无法接受的 ;NGINX 提供了标准实现，通过提供 `resolver` 指令声明 DNS 服务器地址，NGINX 将在 DNS 记录 TTL 到期后，重新解析域名。

​	

### [修改 nginx 属主和属组导致故障](nginx 关于 client_max_body_size client_body_buffer_size 配置)

#### 背景 : 业务容器化过程中 , 有同事不小心 rm -rf 了所有 upstream 配置 , 为了防止再次出现这种情况 , 用 ansible 备份 nginx 的所有 conf 文件

#### 操作 : 因为 ansible 只免密了 carapp 用户 , 而 nginx 是用 nginx 用户启动的 , 然后修改了所有 nginx 目录的属主和属组 , 导致线上故障

#### 故障原因 :client_body_buffer_size

* nginx 分配给请求数据的 buffer 大小 , 如果请求的数据小于 client_body_buffer_size, 直接将数据先存储在内存中 ; 如果请求的值大于 client_body_buffer_size 小于 client_max_body_size, 就会将数据先存储到临时文件中 , 存储在 client_body_temp 指定的路径中
* 所以配置的 client_body_temp 路径 , 一定让执行的 nginx 用户组有读写权限 , 否则 , 当传输的数据大于 client_body_buffer_size, 写进临时文件失败会报错
