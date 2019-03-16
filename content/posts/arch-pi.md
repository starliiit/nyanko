---
title: "给 Pi 装 Arch 并配置网络"
date: 2019-03-16T23:10:05+08:00
draft: false
tags: [linux]
description: ""
---

我的树莓派吃灰好久了，整理房间的时候发现了它，决定用来玩一玩，重温一下 Arch。

主要参考了这篇文章：[Raspberry Pi Setup Guide](https://github.com/phortx/Raspberry-Pi-Setup-Guide)

安装超级简单，直接分区然后拷贝俩目录就完事儿了——简单粗暴到被 Debian/Ubuntu 日惯了的我难以置信。

## 连接无线网络

用键盘和显示器连上板子，要做的第一件事肯定是连接网络啦。

主要使用 iw, ip, wpa_supplicant 命令。（我决定不再用 ifconfig！

```
lsusb -v # 查看 USB 无线网卡是否连接
iw dev # 列出无线网卡
iw dev wlan0 link # 查看网卡 wlan0 连接状态
iw dev wlan0 scan | less # 扫描无线网络
wpa_passphase <ssid> <key> # 生成无线网络配置文件
wpa_supplicant -i wlan0 -c <config_file> # 连接到无线网络，`-B` 后台执行
dhcpcd wlan0 # DHCP 自动获取 IP
```

参考：[Wireless network configuration](https://wiki.archlinux.org/index.php/Wireless_network_configuration_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87))

家里的 Wi-Fi 不是我架设的，SSID 还是中文的，`iw dev wlan0 scan` 扫到的 SSID 	长这样：
```
\xe4\xb8\xad\xe5\x9b\xbd\xe8\x81\x94\xe4\xb8\x8d\xe9\x80\x9a
```

试图连接网络时显示 SSID too long，但是在命令行中并没有中文输入法，无法显示和输入中文。挠了一会儿头，用手机开了个热点，让 Pi 和笔记本都连上，然后 ssh 进入，用我能输入中文的 iTerm 直接 echo 到配置文件中去，成功连上。

产生了一个疑惑：是什么决定在当前的环境中是否能显示和输入中文呢？

## 没有能输入中文的环境怎么办

我用 `echo '中国联不通' > ssid` 命令创建了一个含有中文的文件，用 hexdump 看了一下文件的内容：

```
[root@ArchPi ~]# hexdump ssid
0000000 b8e4 e5ad bd9b 81e8 e494 8db8 80e9 0a9a
0000010
```

文件的内容实际上就是经过编码的字节流，因此我们使用 printf 命令，直接向文件输出字节流：

```
printf "%b" '\xe4\xb8\xad\xe5\x9b\xbd\xe8\x81\x94\xe4\xb8\x8d\xe9\x80\x9a' > ssid
```

然后再用 `wpa_passphase $(cat ssid) <password>` 就可以生成配置文件了。


## 自动连接网络

使用自带的 netctl 配置自动连接 Wi-Fi。

0. 安装 wpa_actiond：`pacman -S wpa_actiond`
1. 从 `/etc/netctl/examples` 中复制 `wireless-wpa` 到 `/etc/netctl` 目录，并重命名，一般用网卡接口名称，例如 `wlan0`
2. 修改文件中的接口名、SSID、密码。
3. `netctl enable wlan0`，配置开机启动，注意命令中的 `wlan0` 是位于 `/etc/netctl` 中的配置文件的名字。

参考：https://wiki.archlinux.org/index.php/Netctl

## 配置 static profile

在 DHCP 网络中想让机器的 IP 地址稳定一点，不想变来变去。可以通过配置 dhcpcd 让每次向 DHCP 服务器发送请求时加上想要的 IP 地址来实现，不过在该 IP 地址被占用时会失败。不过家庭网络机器不多，板子也不会频繁重启，所以还是值得一用。

仿照下面的方式修改 `/etc/dhcpcd.conf`：

```
interface eth0
static ip_address=192.168.0.10/24	
static routers=192.168.0.1
static domain_name_servers=192.168.0.1 8.8.8.8
```

参考：https://wiki.archlinux.org/index.php/dhcpcd

---

板子要用来做啥还没想好……