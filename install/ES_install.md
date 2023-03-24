```
1.vi /etc/sysctl.conf
	vm.max_map_count=655360

2.vi  /etc/security/limits.conf
  es soft nofile 65536
  es hard nofile 65536
  es soft nproc 4096
  es hard nproc 4096

3.sudo sysctl -p /etc/sysctl.conf
4.adduser es  #es只能用非root用户启动
5.关闭防火墙
	systemctl stop firewalld
  service  iptables stop
  systemctl disable firewalld
  chkconfig iptables off
6.修改elasticsearch.yml

每次修改配置文件，记得删除data目录和logs目录下的东西，否则会出问题
#node1
cluster.name: my-application
node.name: node1
path.data: /home/elk/data
path.logs: /home/elk/logs
node.master: true
node.data: false
network.host: 0.0.0.0
bootstrap.memory_lock: false
http.port: 9200
transport.tcp.port: 9300
discovery.zen.minimum_master_nodes: 1
discovery.seed_hosts: ["192.168.50.100:9300","192.168.50.101:9300","192.168.50.102:9300"]
cluster.initial_master_nodes: ["node1"]


#node2
cluster.name: my-application
node.name: node2
path.data: /home/elk/data
path.logs: /home/elk/logs
node.master: true
node.data: true
network.host: 0.0.0.0
bootstrap.memory_lock: false
http.port: 9200
transport.tcp.port: 9300
discovery.zen.ping.unicast.hosts: ["192.168.50.100","192.168.50.101","192.168.50.102"]
discovery.zen.minimum_master_nodes: 1
cluster.initial_master_nodes: ["node1"]

#node3
cluster.name: my-application
node.name: node3
path.data: /home/elk/data
path.logs: /home/elk/logs
node.master: true
node.data: true
network.host: 0.0.0.0
bootstrap.memory_lock: false
http.port: 9200
transport.tcp.port: 9300
discovery.zen.ping.unicast.hosts: ["192.168.50.100","192.168.50.101","192.168.50.102"]
discovery.zen.minimum_master_nodes: 1
cluster.initial_master_nodes: ["node1"]


7.修改jvm.options
	-Xms512m
  -Xms512m
  
8.启动：./elasticsearch -d 
```
