# os 安装

## Centos 部分

#### 挂载磁盘

```
vim /etc/fstab
/dev/sda /data ext4 defaults 0 0
```

#### /dev/null:Permission denied 问题处理

```
rm -f /dev/null; mknod -m 666 /dev/null c 1 3
```

#### U 盘安装 Centos8, 找不到 local media

```
https://blog.csdn.net/weixin_42836409/article/details/112652877
```

#### 禁用显卡

```
# https://halo.sherlocky.com/archives/centos-install-by-usb

# 查看u盘盘符
linuxefi /images/pxeboot/vmlinuz linux dd nomodeset quiet

# 禁用显卡驱动
linuxefi /images/pxeboot/vmlinuz inst.stage2=hd:/dev/sdd4 nomodeset nouveau.modeset=0 quiet
```

#### centos8 更换国内源

```
# 备份旧的配置文件
mv /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.backup

# 下载新的文件
wget -O /etc/yum.repos.d/CentOS-Base.repo https://mirrors.aliyun.com/repo/Centos-8.repo
# 替换部分字段(非阿里云机器要做)
sed -i -e '/mirrors.cloud.aliyuncs.com/d' -e '/mirrors.aliyuncs.com/d' /etc/yum.repos.d/CentOS-Base.repo

yum makecache
```

#### centos7 更换国内源

```
mv /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.backup
wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
curl -o /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
yum makecache
```

#### 修改分辨率

```
# 查看支持的分辨率
xrandr

# 增加分辨率
xrandr --newmode "1920x1080_60.00" 173.00 1920 2048 2248 2576 1080 1083 1088 1120 -hsync +vsync
# 增加分辨率
xrandr --addmode default 1920x1080_60.00
# 设置分辨率
xrandr --output default --mode 1920x1080_60.00
```

#### Centos 安装软件桌面快捷方式

```
# 以typora为例
cd /usr/share/applications
vim typora.desktop

[Desktop Entry]
Name=Typora
Exec=/home/soft/typora/Typora %U
Terminal=false
Type=Application
Icon=/home/soft/typora/resources/assets/icon/icon_128x128.png  #图标,不知道位置可以在软件目录下搜索下
```

## Ubuntu

#### 安装搜狗输入法

```
sudo apt install fcitx-bin
sudo apt-get install fcitx-table

sudo dpkg -i sougou的文件名.deb

sudo apt-get remove ibus
sudo apt-get purge ibus

# 解决无法切换中英文
sudo apt install libqt5qml5 libqt5quick5 libqt5quickwidgets5 qml-module-qtquick2
sudo apt install libgsettings-qt1
```
