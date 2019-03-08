---
title: "tcpreplay"
date: 2019-03-08T20:29:21+08:00
draft: false
---

流量重放工具。

每次重放都需要三步：

1. tcpprep
2. tcprewrite
3. tcpreplay

## 1. tcpprep

对 pcap 做预处理，将 pcap 中的双向流量分为 client 和 server，会创建一个 cache 文件。

这个 cache 文件被用于 tcprewrite 和 tcpreplay。

- 在 tcprewrite 在改写 IP 地址的时候，会使用这个 cache 中的 client 和 server 信息作为参考。
- 在 tcpreplay 发包的时候，会将 client 和 server 两方向的包使用两个网卡接口分别发送。

有多种方式来确定 pcap 中哪些包属于 client 或者 server，可以选择其中一种。

### 分类方式

- `--auto <option>`

	扫描数据包试图自动分类。

  - bridge: weighed according to the server/client ratio
  - router: like bridge mode, after weighing is done, systems which are undetermined are considered a server if they fall inside a network known to contain other servers. 
  - client: like bridge mode, except that unclassified systems are treated as clients.
  - server: ike bridge mode, except that unclassified systems are treated as servers.
  - first:  looking at the first time each IP is seen in the SRC and DST fields in the IP header.

- `--cidr <string>`

	指定 CIDR 地址列表匹配 src IP，匹配的认为是 server。

- `--regex <regex_string>`

	匹配 src ip，匹配的认为是 server。

- `--port <port>`

	匹配 dst port，匹配的认为是 server。

- `--mac <mac list>`

	指定MAC 列表（comma delimited）进行匹配。匹配的认为是 server。

### 其他选项

- `--reverse`

	将所有分类模式的分类结果取反，之前被认为是 server 的现在被认为是 client。例如如果使用 `--port 9999 --reverse`，则数据包中 dst port 是 9999 的认为是  client。

- `--include <rule>`

	"Override default of processing all packets stored in the capture file and only send/edit packets which match the provided rule"

- `--exclude <rule>` 

	不包含匹配的 packet。

- `-i, -o`

	指定输入 pcap、输出 cache 文件路径。

## 2. tcprewrite

根据 tcpprep 生成的 cache 文件修改 pcap 包中的内容。

- `--portmap=<list of map>`

	如 `--portmap=80:8000,8080:999`，将 80 端口改写为 8000 端口，将 8080 改写为 9999。

- `--seed=<number>`

	使用种子将 client/server 的 IP 地址伪随机化。使用相同的种子可以生成相同的 IP 地址。

- `--pnat=<list of CIDR pair>`

	对每个 CIDR pair，使用后者 IP 替换前者 IP。用于把内网 IP 转换成外网 IP。如：
	```
	--pnat=192.168.0.0/16:10.77.0.0/16, 172.16.0.0/12:10.1.0.0/24
	```

- `--srcipmap=<list of CIDR pair>`

- `--dstipmap=<list of CIDR pair>`

	工作方式同 `--pnat`，但只影响 src 或者 dst ip。

- `--endpoints=<src:dst>`

	修改所有 client/server 的地址为 src/dst。

- `--skipbroadcast`

	默认情况下 `--seed, --pnat, --endpoints` 会修改广播和多播包的 IP，使用这个选项防止修改非单播包。

- `--fixcsum`

	重新计算 checksum，默认开启。

- `--enet-dmac=<list of mac pair>`

	如 `--enet-dmac=00:12:13:14:15:16,00:22:33:44:55:66` 替换目的 MAC 地址。

- `--enet-smac=<list of mac pair>`

	修改源 MAC 地址。

- `--enet-vlan*`

	修改 VLAN 相关选项。

- `-i, -o, --cachefile`

	指定输入 pcap、输出 pcap、使用的 tcpprep cache 文件路径。

## 3. tcpreplay

重放 pcap 文件。

可以设定多种发包速度。

不显式指定发包速度时，默认的发包速度应该与 pcap 中捕获的速度相等。

- `-K, --enable-file-cache`

	使用循环多次发送 pcap 时，将 pcap 文件缓存到内存中，使后面的循环不用进行磁盘 I/O，提高发包效率。

- `--cache-file`

	指定 tcpprep 产生的 cache 文件路径

- `--intf1`

	客户端－>服务器的数据包通过这个接口发送。

- `--intf2`

	服务器－>客户端的数据包通过这个接口发送。

- `-L, --limit`

	设置发送的包的个数

- `-x, --multiplier=N`

	设置发包速度为捕获的 pcap 中速度的 N 倍

- `-p, --pps=N`

	设置发包速度为 N packets/sec

- `-M, --mbps=N`

	设置发包速度为 N Mbps

- `-t, --topspeed`

	设置发包速度为最快

- `-o, --oneatatime`

	用户每输入一次发一个包

- `--pps-multi=N`

	"When trying to send packets at very high rates, the time between each packet can be so short that it is impossible to accurately sleep for the required period of time. This option allows you to send multiple packets at a time, thus allowing for longer sleep times which can be more accurately implemented."

Reference: http://tcpreplay.synfin.net/tcpreplay.html