# 服务器部署

## linux 免密

```
ssh-keygen -t rsa -f /root/.ssh/id_rsa -N ''
ssh-copy-id root@192.168.50.100
```

## 网卡设置

### 静态 ip 设置

```
TYPE="Ethernet"
PROXY_METHOD="none"
BROWSER_ONLY="no"
DEFROUTE="yes"
IPV4_FAILURE_FATAL="no"
NAME=eth0
DEVICE=eth0
IPADDR=192.168.50.201
NETMASK=255.255.255.0
#NETWORK=192.168.122.0
GATEWAY=192.168.50.1
BROADCAST=192.168.50.255
ONBOOT=yes
BOOTPROTO=static
```

### 网卡加载成 eth0

```
vi /etc/default/grub

GRUB_TIMEOUT=5
GRUB_DISTRIBUTOR="$(sed 's, release .*$,,g' /etc/system-release)"
GRUB_DEFAULT=saved
GRUB_DISABLE_SUBMENU=true
GRUB_TERMINAL_OUTPUT="console"
GRUB_CMDLINE_LINUX="crashkernel=auto rd.lvm.lv=centos/root "net.ifnames=0 biosdevname=0" rd.lvm.lv=centos/swap rhgb quiet"
GRUB_DISABLE_RECOVERY="true"

grub2-mkconfig  -o /boot/grub2/grub.cfg 
move /etc/sysconfig/network-scripts/ifcfg-ens99 /etc/sysconfig/network-scripts/ifcfg-eth0
reboot
```

## linux 磁盘格式

### GPT:

```
parted /dev/sdc
mklabel gpt 
mkpart primary 0% 
100%
mkfs.ext4 /dev/sdc1
echo "/dev/sdc1 /test ext4 defaults 0 0" >> /etc/fstab
```

```
fdisk /dev/sdb

mkfs -t ext4 /dev/sdb1
```

## 修改 IBMC 风扇速度

```
# 查看当前风扇模式
ipmcget -d faninfo

# 风扇转速更改为手动设置
ipmcset -d fanmode -v 1 100000000

# 更改风扇转速
ipmcset -d fanlevel -v 30
```

## windows 服务器挂载 NFS

```
# 临时挂载
mount -t cifs -o username=username,password=password //192.168.50.52/F  /mnt
#开机挂载设置
1.鼠标邮件"我的电脑"
2.选择映射网络驱动器
3.驱动器选择一个盘符,文件夹: \\192.168.50.52\data\movie
4.确认完成
```

## esxi 定时快照

```
https://blog.csdn.net/zz960226/article/details/118363126
```
