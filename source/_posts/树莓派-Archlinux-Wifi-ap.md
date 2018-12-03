---
title: 树莓派-Archlinux-Wifi-ap
date: 2017-10-29 05:31:48
categories:
  - 硬件
  - 树莓派
tags:
  - 树莓派
  - 折腾
---

## 前言

在树莓派安装好Archlinux系统并进行简要设置之后，我又将其配置成了一个无线ap，以给手机电脑上网所用。经过两个月的使用，发现树莓派工作还算稳定，由此总结整理一下折腾的过程以方便他人和自己。

<!-- more -->

## 设置hostapd

安装hostapd:
>sudo pacman -S hostapd

复制基本配置文件到/etc/hostapd/
>cp /usr/share/doc/hostapd/hostapd.conf /etc/hostapd/

修改hostapd文件:
    
    interface=wlan0
    logger_stdout=-1
    logger_stdout_level=2
    ssid=Excited@pi
    hw_mode=g
    channel=13
    max_num_sta=5
    macaddr_acl=0
    #auth_algs=3
    auth_algs=1
    #ignore_broadcast_ssid=0
    wpa=2
    wpa_passphrase=YourPassword
    wpa_key_mgmt=WPA-PSK WPA-EAP
    wpa_pairwise=TKIP
    rsn_pairwise=CCMP

开启hostapd服务
>sudo systemctl start hostapd
>
>sudo systemctl enable hostapd

## 为无线网卡分配IP地址
>ifconfig wlan 192.168.0.1/24

## 设置wlan0
建立`/etc/systemd/system/network-wireless@.service`
    
    [Unit]
    Description=Wireless network connectivity (%i)
    Wants=network.target
    Before=network.target
    BindsTo=sys-subsystem-net-devices-%i.device
    After=sys-subsystem-net-devices-%i.device

    [Service]
    Type=oneshot
    RemainAfterExit=yes
    EnvironmentFile=/etc/conf.d/network-wireless@%i

    ExecStart=/usr/bin/ip link set dev %i up
    ExecStart=/usr/bin/ip addr add ${address}/${netmask} broadcast ${broadcast} dev %i

    ExecStop=/usr/bin/ip addr flush dev %i
    ExecStop=/usr/bin/ip link set dev %i down

    [Install]
    WantedBy=multi-user.target

建立编辑`/etc/conf.d/network-wireless@wlan0`:

    address=192.168.1.1
    netmask=24
    broadcast=192.168.1.255

启动wlan0服务
>sudo systemctl enable network-wireless@wlan0.service

>sudo systemctl start network-wireless@wlan0.service

## 设置dnsmasq
安装dnsmasq
>sudo pacman -S dnsmasq

编辑`/etc/dnsmasq.conf`

    interface=wlan0
    dhcp-range=192.168.1.2,192.168.1.10,255.255.255.0,12h
后来发现以上设置还不够，dnsmasq服务不能开机启动
{% asset_img dnsmasq_log2.JPG dnsmasq日志 %}
检查日志发现dnsmasq服务在启动的时候找不到`/etc/resolv.conf`文件。查找资料发现，默认情况下dnsmasq读取`/etc/resolv.conf`，指定从哪里获取上游DNS Server。
解决办法：建立一个`/etc/resolv.dnsmasq`文件，内容如下
    
    nameserver 127.0.0.1
    # Tsinghua nameservers
    nameserver 166.111.8.28
    nameserver 166.111.8.29

    # Google nameservers
    nameserver 8.8.8.8
    nameserver 8.8.4.4
同时修改`/etc/dnsmasq.conf`文件
将`resolv-file=/etc/resolv.conf`改为`resolv-file=/etc/resolv.dnsmasq`

启动dnsmasq服务
>sudo systemctl enable dnsmasq

>sudo systemctl start dnsmasq

## 设置端口转发
编辑/etc/sysctl.d/30-ipforward.conf

    net.ipv4.ip_forward=1
    net.ipv6.conf.default.forwarding=1
    net.ipv6.conf.all.forwarding=1

设置 iptables
>sudo iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE

>sudo iptables -A FORWARD -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT

>sudo iptables -A FORWARD -i wlan0 -o eth0 -j ACCEPT

>sudo sh -c "iptables-save > /etc/iptables/iptables.rules"

启动iptables
>sudo systemctl enable iptables

>sudo systemctl start iptables

## 参考
1. [Raspberry-pi Archlinux wifi ap](http://blog.carlcarl.me/1399/raspberry_pi_archlinux_wifi_ap/)
2. [ubuntu14.04 dnsmasq搭建本地名字服务器](http://www.cnblogs.com/asnjudy/p/4687193.html)
3. [dnsmasq](https://wiki.archlinux.org/index.php/Dnsmasq)