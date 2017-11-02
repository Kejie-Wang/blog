---
title: Shadowsocks服务器搭建以及不同终端的客户端配置
tags: 'shadowsocks, vps'
category: shadowsocks
date: 2017-06-12 22:28:58
---

众所周知，中国建立了强大的防火长城已禁止网民访问敏感网站，为此衍生出了大批的翻墙软件，诸如：Shadowsocks, GoAgent, Lantern......本文主要讲介绍目前使用最多的Shadowsocks在服务器端的安装以及在不同终端的客户端的配置
<!-- more -->
# Shadowsocks服务器搭建

## vps获取

对于翻墙的前提就是需要一台在国外的服务器。对于专业的用户可以自行购买vps进行配置，目前作者采用的是[***搬瓦工***](https://bwh1.net/)的vps，这个价格相对比较便宜，有和中国电信直连的线路，而且支持支付宝付款。对于购买搬瓦工的用户，推荐购买最新推出的KVM架构的服务器，这样可以安装锐速进行加速。另一个作者使用过的为[***vultr***](https://www.vultr.com/)，价格相对较贵，但是部署服务器较为灵活，按小时进行计费，速度也相对较快。如果对于linux不是很熟悉也可以直接购买第三方提供的Shadowsocks的账号。

## 服务器搭建

在安装完服务器系统后启动系统ssh连接到服务器，搬瓦工服务器同时也提供了一键安装Shadowsocks服务器的功能，这里主要介绍自行安装。

对于Shadowsocks服务端推荐安装[*shadowsocks-libev*](https://github.com/shadowsocks/shadowsocks-libev)，按照其提示安装完之后

建立一个shadowsocks-config.json文件，写入一下内容：

```shell
{
    "server":"server_ip",   # server ip
    "server_port":65432,     # server port
    "password":"password",   # password
    "timeout":60,            
    "method":"aes-256-cfb"
}
```

其中server_ip填入vps的ip地址，server_port自定义一个未被占用的端口，password自定义一个密码

然后通过ss-server命令启动：

```shell
ss-server -c shadowsocks-config.json -f /tmp/ss.pid
```



# 客户端搭建

## Mac & Windows

Shadowsocks都提供了[***Mac***](/uploads/shadowsocks/MaxOS/ShadowsocksX-2.6.3.dmg)和[***Windows***](/uploads/shadowsocks/Windows/Shadowsocks-3.3.5.zip)的GUI客户端

下载安装完毕之后，会出现一个纸飞机的小图标即是shadowsocks，单击出现以下:![](/images/shadowsocks/shadowsocks.png)

打开shadowsocks后选择模式为Auto Proxy Mode(这样不用翻墙的网站就可以不用走代理了)，

然后添加服务器，如下图所示，将服务器的地址，端口号，加密方式以及密码填入

![](/images/shadowsocks/server.png)

打开浏览器访问下***[google](http://www.google.com/)***试下是否配置成功

## Linux

Linux虽然也有对应的GUI客户端，但是比较简陋，这里介绍命令行配置的方法。

首先按照服务器的方法按照[*shadowsocks-libev*](https://github.com/shadowsocks/shadowsocks-libev)，创建shadowsocks-config.json文件

```shell
{
"server":"server_ip",
"server_port":65432,
"local_address": "127.0.0.1",
"local_port":1080,
"password":"password",
"timeout":600,
"method":"aes-256-cfb",
"fast_open": false,
"workers": 1
}
```

按照服务器的配置填入上面的各个设置，然后使用ss-local启动

```shell
nohup ss-local -c shadowsocks-config.json &
```

## iOS

iOS上的很多客户端都是收费的，这里推荐一个免费的兼容shadowsocks的客户端[Wingy](https://www.wingy.site/)，直接AppStore搜索安装即可。

一个比较方便的功能是wingy支持二维码扫描添加，二维码可以有WIndows或者Mac的客户端生成，当然也可以自行手动添加。

## Chrome

在安装完Shadowsocks后，作者在MAC上采用Safari可以直接翻墙，但是Chrome不可以，不太了解什么原因，后在Chrome安装了[SwitchyOmega](chrome-extension://padekgcemlokbadohgkifijomclgjgif/options.html#/about)才成功

安装完SwithyOmega之后打开进行设置，首先设置代理服务器

![](/images/shadowsocks/switchyomega_proxy.png)

设置协议为*SOCKS5*，服务器的地址就是上面填写的*local_address*和*端口号*，默认为localhost和1080

然后设置autoswitch，添加规则列表，规则列表可从https://raw.githubusercontent.com/gfwlist/gfwlist/master/gfwlist.txt 下载

![](/images/shadowsocks/switchyomega_autoswitch.png)

对于一些规则列表中没有的网站也可以手动添加

## 终端

很多的时候需要在终端上进行翻墙，可以采用***proxchain-ng***实现socks5的代理

### Mac

对于Mac用户可以通过brew直接安装

```shell
brew install proxychains-ng
```

### Linux

#### 下载源代码

```shell
git clone https://github.com/rofl0r/proxychains-ng.git
cd proxychains-ng
```

#### 编译安装

```shell
./configure --prefix=/usr --sysconfdir=/etc
make & make install
```

其中—prefix为安装目录，—sysconfdir为配置文件目录

#### 添加配置文件

修改配置文件**proxychains.conf**

最后加入

```shell
socks5  127.0.0.1 1080 //可以根据自己ss的配置进行更改
```

#### 测试和使用

```shell
proxychain4 curl ip.gs
```

如果显示的ip地址为vps服务器的地址，则配置成功。

以后在需要代理的命令前加上proxychain4即可。
