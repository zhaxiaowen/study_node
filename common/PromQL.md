# PromQL 语句

### PromQL

```yaml
#范围查询
http_requests_total{}[5m]  #选择最近5分钟内的所有样本数据

http_request_total{} offset 5m  #查询5分钟前的瞬时样本数据

sum(http_request_total{})  #查询系统所有http请求总量

avg(node_cpu) by (mode)  #按照mode计算主机CPU的平均使用时间

sum(sum(irate(node_cpu{mode!='idle'}[5m])) / sum(irate(node_cpu[5m]))) by (instance)  #按照主机查询各个主机的CPU使用率

#数学运算
(node_memory_bytes_total - node_memory_free_bytes_total) / node_memory_bytes_total > 0.95  #筛选出当前内存使用率超过95%的主机

#bool运算
http_request_total > bool 1000  #HTTP请求量大于1000，返回1（true），否则返回0（false）
```

#### 语句汇总

| **编号** | **指标**                         | **Metrics**                                                  | **Visualization** | **Desicrible**                                         |
| -------- | -------------------------------- | ------------------------------------------------------------ | ----------------- | ------------------------------------------------------ |
| 1        | 系统运行时间                     | time() - node_boot_time_seconds{instance=~"$node"}           | Singlestat        | time: 获取当前时间 node_boot_time_seconds: 虚拟机开机时间 |
| 2        | CPU 核数                          | count(count((node_cpu_seconds_total{instance=~"$node",mode='system'}) by (cpu)) | Singlestat        |                                                        |
| 3        | CPU 使用率（1m）                  | 100-(avg(irate(node_cpu_seconds_total{instance=~"$node",mode="idle"}[1m]))*100) | Gauge             |                                                        |
| 4        | CPU 使用率                        | avg(irate(node_cpu_seconds_total{instance=~"$node",mode="system"}[1m]))avg(irate(node_cpu_seconds_total{instance=~"$node",mode="user"}[1m]))avg(irate(node_cpu_seconds_total{instance=~"$node",mode="idle"}[1m]))avg(irate(node_cpu_seconds_total{instance=~"$node",mode="iowait"}[1m])) |                   | 各种 mode 下的 cpu 使用率                                  |
| 5        | 系统平均负载                     | node_load1{instance=~"$node"}node_load5{instance=~"$node"}node_load15{instance=~"$node"} | Gauge             |                                                        |
| 6        | 内存使用率                       | ((node_memory_MemTotal_bytes{instance=~"$node"} - node_memory_MemFree_bytes{instance=~"$node"} - node_memory_Buffers_bytes{instance=~"$node"} - node_memory_Cached_bytes{instance=~"$node"}) / (node_memory_MemTotal_bytes{instance=~"$node"} )) * 100 | Gauge             |                                                        |
| 7        | 磁盘最大分区使用率               | 100 - ((node_filesystem_avail_bytes{instance=~"$node",mountpoint="$maxmount",fstype=~"ext4\|xfs"} * 100) / node_filesystem_size_bytes {instance=~"$node",mountpoint="$maxmount",fstype=~"ext4\|xfs"}) | Gauge Table       |                                                        |
| 8        | 磁盘总大小                       | node_filesystem_size_bytes{instance=~'$node',fstype=~"ext4\|xfs"} | Table             |                                                        |
| 9        | 已用磁盘空间                     | node_filesystem_size_bytes{instance=~'$node',fstype=~"ext4\|xfs"}-node_filesystem_avail_bytes {instance=~'$node',fstype=~"ext4\|xfs"} | Table             |                                                        |
|          | 每秒磁盘读写 iops                 | irate(node_disk_reads_completed_total{instance=~"$node"}[$interval]) irate(node_disk_writes_completed_total{instance=~"$node"}[$interval]) | Graph             |                                                        |
|          | 每秒磁盘读吞吐量每秒磁盘写吞吐量 | rate(node_disk_read_bytes_total{instance=~"$node"}[$interval]) rate(node_disk_written_bytes_total{instance=~"$node"}[$interval]) | Graph             |                                                        |
|          | 磁盘使用率                       | (node_filesystem_size_bytes{instance=~'$node',fstype=~"ext4\|xfs"}-node_filesystem_free_bytes{instance=~'$node',fstype=~"ext4\|xfs"})*100 / node_filesystem_size_bytes{instance=~'$node',fstype=~"ext4\|xfs"} |                   |                                                        |
|          | IO 读耗时 IO 写耗时                 | irate(node_disk_read_time_seconds_total{instance=~"$node"}[$interval]) irate(node_disk_write_time_seconds_total{instance=~"$node"}[$interval]) |                   |                                                        |
|          | %util                            | irate(node_disk_io_time_seconds_total{instance=~"$node"}[$interval]) |                   |                                                        |
|          | 文件描述符                       | node_filefd_allocated{instance=~"$node"}                     | Graph             |                                                        |
|          | 网络进带宽                       | irate(node_network_transmit_bytes_total{instance=~'$node',device=~'eth0'}[5m])*8 |                   |                                                        |
|          | 网络出带宽                       | irate(node_network_transmit_bytes_total{instance=~'$node',device=~'eth0'}[5m])*8 |                   |                                                        |
|          | TCP 连接数                        | node_netstat_Tcp_CurrEstab{instance=~'$node'}                |                   |                                                        |
|          |                                  |                                                              |                   |                                                        |
