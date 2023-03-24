# 常用指令

## [awk](https://blog.csdn.net/xumneg111/article/details/120400497?spm=1001.2101.3001.6650.1&utm_medium=distribute.pc_relevant.none-task-blog-2%7Edefault%7ECTRLIST%7Edefault-1-120400497-blog-122926825.pc_relevant_default&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2%7Edefault%7ECTRLIST%7Edefault-1-120400497-blog-122926825.pc_relevant_default&utm_relevant_index=2)

| 参数  | 注释                                                         |
| ----- | ------------------------------------------------------------ |
| -v    | 赋值一个用户定义变量                                         |
| -f    | 从脚本中读取 awk 命令                                          |
| -W    | 在兼容模式下运行 awk。所以 gawk 的行为和标准的 awk 完全一样，所有的 awk 扩展都被忽略。 |
| ’ ’   | 引用代码块                                                   |
| //    | 匹配代码块，可以是字符串或正则表达式                         |
| {}    | 命令代码块，包含一条或多条命令                               |
| ;     | 多条命令使用分号分隔                                         |
| BEGIN | 在 awk 程序一开始，未读取任何数据之前执行。BEGIN 后的动作只在程序开始时执行一次 |
| END   | 在 awk 程序处理完所有数据，即将结束时执行 ?END 后的动作只在程序结束时执行一次 |

| 参数                    | 注释                                               |
| ----------------------- | -------------------------------------------------- |
| $0                      | 表示整个当前行                                     |
| $1 ~ $n                 | 当前记录的第 N 个字段                                |
| FS                      | 输入字段分隔符（默认是空格)                        |
| RS                      | 输入记录分割符，默认换行符 (即文本是按一行一行输入) |
| NF                      | 字段个数就是列                                     |
| NR                      | 记录数，就是行号，默认从 1 开始                      |
| FNR                     | 与 NR 类似，不过多文件记录不递增，每个文件都从 1 开始  |
| OFS                     | 输出字段分隔符，默认空格                           |
| ORS                     | 输出记录分割符，默认换行符                         |
| \n                      | 换行符                                             |
| ~                       | 匹配正则表达式                                     |
| !~                      | 不匹配正则表达式                                   |
| = += -= *= /= %= ^= **= | 赋值                                               |
| &&                      | 逻辑与                                             |
| < <= > >= != ==         | 关系运算符                                         |
| * / %                   | 乘，除与求余                                       |
| $                       | 字段引用                                           |

## curl

```plain
# 抓取请求的状态返回码,可以用来测试网站是否正常
curl -o /dev/null -s -w %{http_code} www.baidu.com

# 发送post请求:注册consul服务
curl http://172.17.0.4:8500/v1/agent/service/register -X PUT -i -H "Content-Type:application/json" -d '{
  "ID": "userServiceId",  
  "Name": "userService",
  "Tags": [
    "primary",
    "v1"
  ],
  "Address": "127.0.0.1",
  "Port": 8000,
  "EnableTagOverride": false,
  "Check": {
    "DeregisterCriticalServiceAfter": "90m",
    "HTTP": "http://www.baidu.com",
    "Interval": "10s"
  }
}'

# 带账号请求
curl -u elastic 192.168.50.100:9200
# 带账号密码请求
curl -u username:passwor 192.168.50.100:9200

# 查看证书过期时间
curl -Ivs https://www.xxx.xxx --connect-timeout 1
```

## dig

```yaml
 dig +trace +nodnssec time.geekbang.org   # +trace:开启跟踪查询     # +nodnessec:禁止DNS安全扩展
```

## date

```
date -d "30 days ago" +%Y.%m.%d   # 获取30天前的日期,并指定日期格式
```

## docker

```
docker stop $(docker ps -q)  # 停用全部运行中的容器
docker rm $(docker ps -aq)   # 删除全部容器
docker stop $(docker ps -q) & docker rm $(docker ps -aq)
```

## find

```yaml
find -name
find -iname  #不区分字母大小写
find -inum n
find -regex "PATTERN"
find -user  #按属主查找
find -group #按属组查找
find -type:  #按文件类型查找
    f:普通文件
    d:目录文件
    -a:与
    -o:或
    -not:非
find -size  [+|-] UNIT #k,M,G,c
find -atime [+|-]
find:
	-print: #
  -ok command {} \; #find /tmp -ctime +3 -user joe -ok rm {} \;
  -exec command {} \; #find ~ -perm -002  -exec chmod o-w {} \;
```

## gdb

```plain
gdb -p 87746 :打印所有的线程信息
```

## grep

|      |                                   |                                                |
| ---- | --------------------------------- | ---------------------------------------------- |
| -c   | 仅列出包含字段的行数              | grep -c "$ip"                                  |
| -i   | 忽略大小写                        |                                                |
| -I   | 列出带有字段的整行                |                                                |
| -n   | 列出所有匹配行 , 并显示行号         |                                                |
| -v   | 匹配没有字段的行                  |                                                |
| -w   | 匹配完整的字符 , 忽略那些部分匹配的 |                                                |
| -r   | 递归搜索                          |                                                |
| --   | 表示命令行参数终止解析标志        | iptables-save \| grep -- '-A KUBE-POSTROUTING' |

## iostat

```plain
iostat -d -k 1 10     #查看TPS和吞吐量信息(磁盘读写速度单位为KB)
iostat -d -m 2        #查看TPS和吞吐量信息(磁盘读写速度单位为MB)
iostat -d -x -k 1 10  #查看设备使⽤率（%util）、响应时间（await）
iostat -c 1 10        #查看cpu状态实例分析


iostat -d -k 2 2
Linux 3.10.0-1160.49.1.el7.x86_64 (node1)       02/27/2022      _x86_64_        (4 CPU)

Device:            tps    kB_read/s    kB_wrtn/s    kB_read    kB_wrtn
sda               9.82        12.01        71.15    1852990   10979111
dm-0              9.91        11.77        71.13    1816856   10974958
dm-1              0.00         0.01         0.00       2072          0
dm-2              0.00         0.02         0.01       3485       2068

Device:            tps    kB_read/s    kB_wrtn/s    kB_read    kB_wrtn
sda               4.50         0.00        21.00          0         42
dm-0              4.50         0.00        21.00          0         42
dm-1              0.00         0.00         0.00          0          0
dm-2              0.00         0.00         0.00          0          0


-d:显示设备名称
-k:某些使用block为单位的列强制使用Kilobytes为单位
-x:用于显示和io相关的扩展数据
-c:获取cpu的部分状态值

tps:设备每秒的传输次数,一次传输代表一次I/O请求
await:每个IO请求处理的平均时间,(IO响应时间,单位微秒毫秒),一般要低于5ms
svctm:平均每次设备IO操作的服务时间(毫秒),svctm的值与await越相近,表示几乎没有I/O等待,磁盘性能很好,如果await值远高一svctm的值,表示I/O队列等待太长,系统上运行的应用程序变慢
%util:设备的繁忙程度
```

## ipvs

> ipvsadm 只有 2 个核心 ,service 和 server,service 是负载均衡实例 ,server 是后端 member
>
> ipvs 的负载是运行在内核态的 , 因此不会出现监听端口

```
-A -t 172.17.57.38:31517 -s rr  #-t:指定监听的为TCP端口,-s rr指定算法为轮询算法rr

#-t:service实例  -r:server地址  -w:权重  -m:转发模式(masquerading),即NAT; -g:为gatewaying,即直连路由模式; -i为ipip,即IPIP隧道
-a -t 172.17.57.38:31517 -r 172.17.53.118:9000 -m -w 1  


```

## [iptables](https://blog.csdn.net/a1809032425/article/details/120742889?spm=1001.2101.3001.6661.1&utm_medium=distribute.pc_relevant_t0.none-task-blog-2%7Edefault%7ECTRLIST%7EPayColumn-1-120742889-blog-121248469.experiment_layer_sa&depth_1-utm_source=distribute.pc_relevant_t0.none-task-blog-2%7Edefault%7ECTRLIST%7EPayColumn-1-120742889-blog-121248469.experiment_layer_sa&utm_relevant_index=1)

iptables [-t table] command [chain] CRETIRIA -j ACTION

* -t table, 操作的表 ,filter,nat,mangle,raw, 默认使用 filter
* command, 子命令 , 定义对规则的管理
* chain, 指明链路
* CRETIRIA, 匹配的条件或标准
* ACTION, 操作动作

```
iptables -t nat -nvL OUTPUT
iptables -t nat -nvL KUBE-SERVICES
iptables -t nat -nvL KUBE-SVC-G3OM5DSD2HHDMN6U

iptables -t nat -nvL POSTROUTING
```

例 :

| iptables | table     | command | chain | parameter | target    |
| -------- | --------- | ------- | ----- | --------- | --------- |
| iptables | -t filter | -A      | input | -p        | -j ACCEPT |

### command

* -A: 增加 , 追加新规则于指定链的尾部
* -D: 删除 , 根据规则编号删除规则
* -I: 插入 , 插入新规则于指定链的指定位置 , 默认为首部
* -L: 列出规则
* -v: 详细信息
* -n: 数字格式显示主机地址和端口号
* -S: 显示指定链的所有规则

### paramter

* -p: 协议类型 , -p tcp  ; -p udp; -p icmp
* -s: 来源地址
* -d: 目标地址
* -i: 流入
* -o: 流出
* -sport: 来源端口
* -dport: 目标端口

### target

* -j: 跳转
* -j ACCEPT, 接受
* -j DROP, 丢弃
* -j REJECT: 拒绝 , 给对方一个回应
* -j SNAT, 源地址转换
* -j DNAT, 目标地址转换
* -j MASQUERADE: 伪装 ip 地址

* iptables -t filter -A INPUT -s 0.0.0.0/0.0.0.0 -d x.x.x.x -j DROP #无论从哪里来 , 想要访问 x.x.x.x, 全部拒绝
* iptables -A INPUT -s 10.8.0.0/16 -d 172.12.12.12 -p tcp --dport 80 -j DROP  # 禁止 10.8.0.0/16 访问 172.12.12.12 的 80 端口
* iptables -I INPUT -s 0.0.0.0/0.0.0.0 -d x.x.x.x -p tcp --dport 22 -j ACCEPT  # 放开 22 端口
* iptables -A INPUT -s  0.0.0.0/0.0.0.0 -d x.x.x.x -p tcp --dport 80 -j ACCEPT  # 放开 80 端口

## journalctl #查看系统日志

```
journalctl -r	# 倒序输出日志(从新到旧)
journalctl -f	# 实时打印日志
journalctl --since "2022-01-09 20:00:00" --until "2022-08-04 20:15:00"	# 查看指定时间段的日志
journalctl -k; journalctl --dmesg	#查看系统的内核日志
journalctl -u docker.serivce	# 查看指定服务的日志
journalctl -p 3 -u docker.service	#查看指定级别日志(0-7,emerg-debug)
```

## lsof

```yaml
lsof -p $PID #通过查看进程，能看到打开了哪些文件
```

## mpstat

```plain
mpstat -P ALL 5:显示所有CPU指标，5s刷新一次
```

## netstat

```plain
netstat -s :网络状态统计，按照各个协议进行统计输出

# TCP连接状态统计
netstat -n | awk '/^tcp/ {++state[$NF]} END {for(key in state) print key,"\t",state[key]}'
```

## nmap

```yaml
主要功能：探测主机是否在线、扫描主机开放端口和嗅探网络服务，用于网络探测和安全扫描。

nmap -n -p T:21-25,80,110-sS 192.168.50.101  # 探测指定服务器是否启有特定端口的服务

nmap -sS 192.168.0.0.24  # 探测局域网段中各主机开启了哪些网段

nmap –sP –n 192.168.0.0/24 172.16.0.0/16   # 探测192.168.0.0和172.16.0.0/16网段中有哪些主机在运行

nmap -F 192.168.50.101  # 快速扫描主机开发端口
```

## nslookup

```yaml
查询对应域名的A记录
nslookup time.geekbang.org

域名不通,通过debug模式查找问题
 nslookup -debug time.geekbang.org
```

## openssl: 生成证书

```
openssl req -x509 -nodes -days 3650 -newkey rsa:2048 -keyout /root/tmp/zhaoxw.work.key -out /root/tmp/zhaoxw.work.crt -subj "/CN=timebye.github.io"
```

## pidstat

```plain
pidstat -u 5: 5s输出一组数据，-u：指CPU
```

## ps

```yaml
ps -eo pid,lstart,etime | egrep "pid" :查看进程启动时间
ps mp pid -o THREAD,tid # 查看主进程下对应的子线程
```

## pstack

```plain
pstack pid  :命令可以显示每个线程的栈跟踪信息（函数调用过程)
```

## strace

```yaml
strace -p pid

 strace -f -T -tt -p 9085  # -f:表示跟踪子进程和子线程  -T:表示显示系统调用的市场  -tt:表示显示跟踪时间
```

## stress

```plain
-c,--cpu:产生n个进程，每个进程都反复计算随机数的平方根
-i,--io:产生n个进程，每个进程反复调用sync()将内存的内容刷盘
-m,--vm:产生n个进程，每个进程不断分配和释放内存
-t,--timeout
模拟CPU:stress --cpu --timeout 300
模拟io:stress -i 1 --timeout 300
模拟多进程:stress -c 8 --timeout 300
```

## stat

```
stat filename :显示文件详细信息
```

## ss

```yaml
ss -antp:查看系统当前网络连接
```

## systemctl

```yaml
systemctl list-unit-files|grep enabled :查看已启动的服务列表
systemctl is-enabled nginx.service :查看服务是否开机启动

# 在开机时启用服务
systemctl enable postfix.service

# 在开机时禁用服务
systemctl disable postfix.service

# 查看服务是否开机启动
systemctl is-enabled postfix.service

# 查看开机自启动的服务列表
systemctl list-unit-files | grep enabled

# 查看启动失败的服务列表
systemctl --failed

# 启动服务
systemctl start postfix.service

# 关闭服务
systemctl stop postfix.service

# 重启服务
systemctl restart postfix.service

# 显示服务的状态
systemctl status postfix.service

==========================================================================
### 开机自启配置
[Unit]
Description=kibana
After=network.target
 
[Service]
User=elk
Group=elk
ExecStart=/home/elk/kibana/bin/kibana
ExecStop=/usr/bin/kill -15 $MAINPID
ExecReload=/usr/bin/kill -HUP $MAINPID
Type=simple
RemainAfterExit=yes
PrivateTmp=true
# file size
LimitFSIZE=infinity
# cpu time
LimitCPU=infinity
# virtual memory size
LimitAS=infinity
# open files
LimitNOFILE=65535
# processes/threads
LimitNPROC=64000
# locked memory
LimitMEMLOCK=infinity
# total threads (user+kernel)
TasksMax=infinity
TasksAccounting=false
 
[Install]
WantedBy=multi-user.target
==========================================================================
```

## sysctl

```
sysctl -a :查看当前系统中生效的所有参数

# 修改内核参数
vim /etc/sysctl.conf
sysctl -p # 使配置生效

# 临时修改
sysctl -w net.core.netdev_budget=600
```

## traceroute

```plain
traceroute hostname: 可以定位从源主机到目标主机之间经过了哪些路由器，以及到达各个路由器的耗时
```

## tcpdump

```yaml
tcpdump -i eth0 -c 10 test   # 截获eth0网卡10次收发所有数据包并将抓包结果保存到test文件,再读取test抓包结果文件
tcpdump -r test

tcpdump port 80   # 截获来访问80端口的所有数据包（指定端口范围portrange 1-1024）

tcpdump host 192.168.50.101  # 截获所有来自主机192.168.50.101的进出所有数据包

<<<<<<< HEAD
tcpdump src 192.168.50.101  截获ip包中源地址是192.168.50.101的
tcpdump dst 192.168.50.101  截获ip包中目的地址是192.168.50.101的
=======
>>>>>>> 556d332e398c0fb1a7ab5f46f7d60c18f712cd05

tcpdump host 114.254.151.51 and 114.254.151.52 # 截获主机114.254.151.51和主机114.254.151.52的通信

tcpdump tcp and src 114.254.151.51 and port 80  # 截获tcp协议并且源地址114.254.151.51来访问80的端口
```
## date

```
date -d "1 month"  +%Y.%m.%d # 下个月
date -d "1 days"
date -d "1 years"
date -d "1 week"
```

## vim

```plain
set paste ：vi/vim就不会启动自动缩进，而只是纯拷贝粘贴
```

## virsh

```
virsh net-start --network default   #激活默认网卡
virsh net-list --all  
virsh domifaddr node1 #获取虚拟机ip
```

## 文件描述符

```yaml
cat /proc/sys/fs/file-nr 
1216    0       197787 #1216:为已分配FD数量 0:已分配但未使用FD数量  197787:系统可用的最大FD数量
ulimit -n #系统默认描述符数量
lsof |wc -l  #整个系统打开的文件数量
```
