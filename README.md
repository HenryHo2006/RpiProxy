# RpiProxy
Make a Raspberry PI as a proxy route, work with shadowsocks server, provide clean dns/proxy service
> 前言：  
> 　　书籍是人类进步的阶梯，墙是阻碍人类进步的陷阱   --- 高尔基·马克西姆·方·滨兴
>  
> 　　翻墙的最佳姿势，当然是不在电脑/手机/pad上安装任何软件，不更改任何设置，就能访问到任何网址，墙变得透明了。
> 实现这个目标主要有两种做法：
1. 刷路由器固件，用OpenWrt加上其它软件。这个方式适合爱折腾的玩家，也有上网不稳定的风险存在。
2. 用一台专用电脑充当网关/DNS服务器，安装设置上专用软件。这种方式优点很多，还可以出差时随时携带。要说低成本电脑，
当然要首推树莓派了。本文就是专门介绍如何让树莓派做为透明翻墙网关。已经在Raspbian(Jiessie)和OSMC系统下面测试通过了。

> 说明：工作目录在/home/pi下面  
> 说明：用su取得root权限，我下面的命令不再加sudo前缀  
> 说明：我的安装配置都是按systemd系统的，较老的系统不适用  

## 1.1、下载/编译/打包/安装shadowsocks-libev

    apt-get update  
    apt-get install git build-essential autoconf libtool libssl-dev \  
        gawk debhelper dh-systemd init-system-helpers pkg-config  
    git clone https://github.com/shadowsocks/shadowsocks-libev.git  
    cd shadowsocks-libev  
    dpkg-buildpackage -us -uc -i  
    cd ..  
    //下面这行版本号可能不同  
    dpkg -i  shadowsocks-libev_2.4.6-1_armhf.deb  

    这些步骤若有问题请参阅https://github.com/shadowsocks/shadowsocks-libev

## 1.2、禁用shadowsocks-libev-server服务，这个服务默认是打开的

    systemctl stop shadowsocks-libev.service
    systemctl disable shadowsocks-libev.service

## 1.3、配置启动shadowsocks-libev-redir服务

    vi /etc/shadowsocks-libev/config.json
    格式如下：
    {
        "server":"境外shadowsocks服务器ip地址",
        "server_port":境外shadowsocks服务器端口,
        "local_address": "0.0.0.0",
        "local_port":1080,
        "password":"境外shadowsocks服务密码",
        "timeout":3000,
        "method":"aes-256-cfb"
    }

    mv /lib/systemd/system/shadowsocks-libev-redir@.service /lib/systemd/system/shadowsocks-libev-redir.service
    vi /lib/systemd/system/shadowsocks-libev-redir.service
    注意这行：ExecStart=/usr/bin/ss-redir -c /etc/shadowsocks-libev/config.json
    systemctl daemon-reload
    systemctl enable shadowsocks-libev-redir
    systemctl start shadowsocks-libev-redir
    用下面这句检查运行状态
    systemctl status shadowsocks-libev-redir

## 1.4、配置启动shadowsocks-libev-tunnel服务

    cp /etc/shadowsocks-libev/config.json /etc/shadowsocks-libev/config-tunnel.json
    vi /etc/shadowsocks-libev/config-tunnel.json
    {
        "server":"境外shadowsocks服务器ip地址",
        "server_port":境外shadowsocks服务器端口,
        "local_address": "127.0.0.1",
        "local_port":25353,
        "password":"境外shadowsocks服务密码",
        "timeout":3000,
        "method":"aes-256-cfb"
    }
    mv /lib/systemd/system/shadowsocks-libev-tunnel@.service /lib/systemd/system/shadowsocks-libev-tunnel.service
    vi /lib/systemd/system/shadowsocks-libev-tunnel.service
    注意这行：ExecStart=/usr/bin/ss-tunnel -c /etc/shadowsocks-libev/config-tunnel.json -L 8.8.4.4:53 -u
    systemctl daemon-reload
    systemctl enable shadowsocks-libev-tunnel
    systemctl start shadowsocks-libev-tunnel
    用下面这句检查运行状态
    systemctl status shadowsocks-libev-tunnel

## 2.1、下载/编译/安装chinadns

    wget https://github.com/shadowsocks/ChinaDNS/releases/download/1.3.2/chinadns-1.3.2.tar.gz
    tar -zxf chinadns-1.3.2.tar.gz
    cd chinadns-1.3.2
    ./configure
    make
    make install

## 2.2、配置启动chinadns服务

    vi /lib/systemd/system/chinadns.service
    格式如下：
    [Unit]
    Description=chinaDNS service
    
    [Service]
    ExecStart=/usr/local/bin/chinadns -v -m -c /usr/local/share/chnroute.txt -l /usr/local/share/iplist.txt -p 53 -s "114.114.114.114,223.5.5.5,127.0.0.1:25353"
    Restart=on-failure
    RestartSec=42s
    
    [Install]
    WantedBy=multi-user.target

    systemctl daemon-reload
    systemctl enable chinadns
    systemctl start chinadns
    用下面这句检查运行状态
    systemctl status chinadns

## 3.1 设置nat转发规则

    vi /etc/sysctl.conf
    注意这句：net.ipv4.ip_forward = 1
    sysctl -p /etc/sysctl.conf
    下载本项目文件/etc/shadowsocks-libev/iptables.up.rules
    修改你自己的Shadowsocks服务器ip
    vi /etc/network/if-pre-up.d/iptables
        #!/bin/bash
        /sbin/iptables-restore < /etc/shadowsocks-libev/iptables.up.rules
    chmod +x /etc/network/if-pre-up.d/iptables

    **注意：raspberry PI的发行版本OSMC中，不支持网络接口链接/断开脚本，做为临时办法，放到rc.local中执行

## 4.1、chinadns让出53端口(可选)

    systemctl stop chinadns
    vi /lib/systemd/system/chinadns.service
    注意下面这句，监听端口变为15353，因为53端口(dns服务默认端口)让给了dnsmasq
     ExecStart=/usr/local/bin/chinadns -v -m -c /usr/local/share/chnroute.txt -l /usr/local/share/iplist.txt -p 15353 -s "114.114.114.114,223.5.5.5,127.0.0.1:25353"
    systemctl daemon-reload
    systemctl start chinadns
    用下面这句检查运行状态
    systemctl status chinadns

## 4.2、安装设置dnsmasq(可选)

    apt-get install dnsmasq
    vi /etc/dnsmasq.conf
        no-resolv
        server=127.0.0.1#15353
    systemctl restart dnsmasq
    用下面这句检查运行状态
    systemctl status dnsmasq

另外，发现dnsmasq有个bug，默认选项local-service的本意是只提供服务给同一子网的客户机，但我发现他有时不能正确分辨，需要禁用  
     vi /etc/init.d/dnsmasq  
    注释掉这一行  
    DNSMASQ_OPTS="$DNSMASQ_OPTS --local-service"  

