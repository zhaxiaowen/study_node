# keepalived+nginx

#### keepalived.conf

```
global_defs {
    script_user root
}
vrrp_script check_nginx {
    script "/etc/keepalived/check_nginx.sh"
    interval 10
}

vrrp_instance VI_1 {
    state MASTER
    interface ens33
    virtual_router_id 70
    priority 100
    unicast_src_ip 192.168.137.15
    unicast_peer {
        192.168.137.25
    }
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass bzl111
    }
    virtual_ipaddress {
        192.168.137.100
    }
    track_script {
        check_nginx
    }
}

```

#### check_nginx.sh

```
#!/bin/sh
d=`date --date today +%Y%m%d_%H:%M:%S`
n=`ps -C nginx --no-heading|wc -l`
if [ $n -eq "0" ]; then
  /usr/local/nginx/sbin/nginx
  n2=`ps -C nginx --no-heading|wc -l`
  if [ $n2 -eq "0"  ]; then
          echo "$d nginx down,keepalived will stop" >> /var/log/check_ng.log
          systemctl stop keepalived
  fi
fi

```

#### 采坑

刚开始配置 keepalived 时 , 一直无法拉起 nginx

1. check_nginx.sh: 没有赋权限
2. 一直报脚本返回状态不对 , 返回 15, 返回 1
3. 最大的问题 : 没有指定执行脚本的用户 , 导致脚本一直执行有问题 , 添加 :script_user root
4. 添加执行用户后 , 报 Disabling track script check_nginx due to insecure, 删除 enable_script_security
