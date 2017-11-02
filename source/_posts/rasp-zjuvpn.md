---
title: 树莓派（Raspberry Pi）-浙大(ZJU)VPN连接
tags: 'raspberry pi, 浙大, vpn'
category: EmbededSystem
date: 2016-03-09 23:32:00
---

连接上树莓派后，系统自动默认进入的是命令行模式，默认的用户名为：pi，密码：raspberry。
可以在刚刚启动的界面中进行修改为图形界面模式，然后sudo reboot后就进入可桌面模式。由于树莓派默认的为UK标准的键盘，会发现和外接键盘不匹配，可以将其修改成US的键盘。
<!-- more -->
# 连接内网

关于树莓派的VPN联网，现在玉泉的有线需要申请固定的ip的，网络ip地址申请完成后。
首先需要将树莓派的网卡的Mac地址改成申请的地址，或者使用其地址申请（ifconfig命令查看）
修改的方法为如下图所示
![](/images/rasp-zju-vpn/addr-change.png)
然后设置VPN的一些参数：
设置ip地址
将网线插上后，执行下列命令
```shell
$ sudo ip addr flush dev eth0
$ sudo ip addr add dev eth0 $I/$U
```
其中$I用你的ip静态ip地址代替，$U用你的子网掩码的前导1的个数代替（如果数不清楚的话，把你的子网掩码转换成二进制数，然后从最高位开始数1的个数）
```shell
$ sudo ip route replace default via $D
```
$D用你的默认网关代替
```shell
$ sudo nano /etc/resolv.conf
$ nameserver $DNS
```
$DNS用DNS服务器代替
设置完成之后就可以尝试ping下cc98
![](/images/rasp-zju-vpn/ping-cc98.png)


# 连接外网

上面已经将树莓派连接上了内网，如果需要访问外网需要配置vpn。下载安装包，使用U盘将其拷贝到Raspberry Pi中去，解压后执行以下的命令
```shell
$ sudo dpkg –i libpcap0.8_1.3.0-1_armhf.deb
$ sudo dpkg –i ppp_2.4.5-5.1_armhf.deb
$ sudo dpkg –i xl2tpd_1.3.1+dfsg-1_armhf.deb
$ tar –zxvf zjuvpn-8.2.tar.gz –C /
$ sudo zjuvpn -c
```
然后输入用户名和密码就可以上网了，ping下度娘测试下外网有没有用吧

对于/ect/resolv.conf文件会被重写的问题可以执行以下命令
```shell
$ aptitude purge dhcpcd5
```
