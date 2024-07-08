# 第二十一章：网络故障排除

解决网络问题就像任何故障排除一样。了解你的网络，熟练使用基本工具，耐心和系统性十分重要。

在本章中，我们学习如何使用*ping*，FPing，Nmap，*httping*，*arping*和*mtr*测试连通性，映射网络，找到不良服务，测试网站性能，找到重复 IP 地址以及找到路由瓶颈。

# 诊断硬件

如果你发现自己被神秘的未标记以太网和电话线缠绕住了，那么请获取一台以太网/电话线缆测试和音频跟踪器。有很多款售价低于 100 美元。这些设备由两部分组成：一个发射器和一个接收器。两人合作使用会更快，一个在每条电缆的一端。当你找到电缆的两端时，请给它贴上标签并继续下一步。你也可以一个人完成，但两个人会更快。

万用表在很多工作中都很有用，比如找短路和断路、测试连通性和衰减、确定电线是否正确终端、测试电源插座以及测试计算机电源和主板。[Adafruit](https://adafruit.com)是一个非常好的网站，提供关于使用万用表和学习电子学的优秀教程。

如果可能的话，保留一些备用零件。有时更换网络接口、电缆或交换机可以更快地找出有问题的硬件。

# 21.1 使用 ping 测试连通性

## 问题

你的网络上的某些服务或主机无法访问或出现间歇性故障。你想弄清楚这是否是硬件问题，名称服务问题，路由问题还是其他问题。

## 解决方案

当你调试网络问题时，从近处开始，系统地逐步从近到远解决。这指的是物理距离和需要经过的路由器数量。首先从本地 LAN 段开始。然后到下一个 LAN 段，如果有多个，则跨越一个路由器。然后到下一个距离两个路由器的地方，依此类推。

首先使用传统的*ping*测试连通性。首先 ping *localhost*：

```
$ ping localhost
PING localhost (127.0.0.1) 56(84) bytes of data.
64 bytes from localhost (127.0.0.1): icmp_seq=1 ttl=64 time=0.065 ms
64 bytes from localhost (127.0.0.1): icmp_seq=2 ttl=64 time=0.035 ms
```

通过按 Ctrl-C 停止*ping*。首先 ping *localhost*确认你的网络接口已经启动并正常运行。如果看到“connect: Network is unreachable”，那么你的网络接口有问题。备有一些备用的 USB 网络接口可以快速确认是否有损坏的接口。

一旦网络接口设置好，就 ping 你的主机名以测试名称解析，并告诉*ping*在三次 ping 后停止：

```
$ ping -c 3 *client4*
PING client4 (192.168.1.97) 56(84) bytes of data.
64 bytes from client4 (192.168.1.97): icmp_seq=1 ttl=64 time=0.087 ms
64 bytes from client4 (192.168.1.97): icmp_seq=2 ttl=64 time=0.059 ms
64 bytes from client4 (192.168.1.97): icmp_seq=3 ttl=64 time=0.061 ms

--- client4 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2046ms
rtt min/avg/max/mdev = 0.059/0.069/0.087/0.012 ms
```

如果返回正确的 IP 地址，则你的名称解析已正确设置。如果返回一个类似于 127.0.1.1 的本地主机地址或者“Name or service not known”，则你的 DNS 配置有问题。

当你的本地 DNS 修复后，通过主机名 ping 你的网络主机之一。如果*ping*失败并显示“Destination Host Unreachable”，尝试 ping 它的 IP 地址。如果成功了，请检查你的 DNS。如果仍然失败并显示相同的消息，则你的主机名和地址可能不正确，或者主机已关闭。

如果您无法到达任何外部 IP 地址，您的网络接口可能正常，问题可能出在上游：您的以太网电缆、无线接入点或交换机上。“网络不可达”意味着您的机器未连接到网络。

当你追踪间歇性中断的源头时，设置 *ping* 运行一段时间，比如 500 次 ping，间隔 2 秒，这样你就不会超载主机或网络，并将结果输出到文本文件中。下面的示例追加了额外的信息到文件中，以便你可以停止和重新开始：

```
$ ping -c 500 -i 2 *server2* >> server2-ping.txt
```

或者，使用 *tee* 命令查看输出并将其记录到文件中：

```
$ ping -c 500 -i 2 *server2* | tee server2-ping.txt
```

在多宿主主机上，使用 *ping -i 接口名称* 来指定要使用的接口。

## 讨论

不要阻塞 *echo-request*、*echo-reply*、*time-exceeded* 或 *destination-unreachable* ping 消息。一些管理员在其防火墙上阻止所有 ping 消息，这是一个错误，因为许多网络功能至少需要这四种 ping 消息才能正常运行。

如果使用 *-a*（可听见）开关，则 *ping* 命令实际上会发出 ping，尽管您可能需要做一些设置才能使其工作。在早期，我们的 PC 机箱内置了电脑主板，直接连接到主板，自动加载了激活机箱扬声器的内核模块。你可能熟悉这个扬声器发出的低保真烦人的蜂鸣声，也许您甚至运行了一些黑客技术来让它播放音乐。

现在，在现代，桌面音箱大多数已经消失了，大多数笔记本电脑也不再有主板蜂鸣声了。但是大多数 PC 主板仍然支持它们，现代蜂鸣器是一个小东西（图 21-1）。你可能需要购买一个。

![计算机主板的蜂鸣器。](img/lcb2_2101.png)

###### 图 21-1\. 计算机主板的蜂鸣器

一旦安装了蜂鸣器，加载 *pcspkr* 内核模块，然后确认已加载：

```
$ sudo modprobe pcspkr
$ lsmod|grep pcspkr
pcspkr                 16384  0
```

现在试一试。切换到纯文本控制台使用 Ctrl-Alt-F2，或启动 X 终端，并使用 *echo* 命令播放 ASCII 响铃字符。所有示例都是同样的事情，ASCII 字符代码 7 的不同表示：

```
$ echo -e "\a"
$ tput bel
$ echo -e '\007'
```

或按下 Ctrl-G。

如果在图形终端中听不到任何声音，请检查其设置以启用声音。*xfce4-terminal* 和 *gnome-terminal* 都会播放 ASCII 响铃。*Konsole* 支持使用您选择的声音文件进行通知，但不支持蜂鸣器。

## 参见

+   *man 8 ping*

+   [ICMP 参数的 IANA 列表](https://oreil.ly/pWYWE)

# 21.2 使用 fping 和 nmap 对您的网络进行分析

## 问题

您希望生成网络上所有主机和 IP 地址的列表，并探测 MAC 地址和开放端口。

## 解决方案

使用 *fping* 和 *nmap* 探测您的 LAN，并记录结果。

*fping* 按顺序对一个范围内的所有地址进行 ping。本示例对一个子网进行一次 ping，报告哪些主机是活动的，查询主机名的 DNS，并打印汇总：

```
$ fping -c1 -gAds 192.168.1.0/24 2>1 | egrep -v "ICMP|xmt" >> *fping.txt*
client1.net (192.168.1.15)     : [0], 84 bytes, 3.12 ms (3.12 avg, 0% loss)
server2.net (192.168.1.91)     : [0], 84 bytes, 5.34 ms (5.34 avg, 0% loss)
client4.net (192.168.1.97)     : [0], 84 bytes, 0.03 ms (0.03 avg, 0% loss)

     254 targets
       3 alive
     251 unreachable
       0 unknown addresses

     251 timeouts (waiting for response)

 0.03 ms (min round trip time)
 2.83 ms (avg round trip time)
 5.34 ms (max round trip time)
        3.575 sec (elapsed real time)
```

要查看未经过滤的输出，请省略 *2>1 | egrep -v “ICMP|xmt”* 部分。任何离线的机器都不会被找到，所以你可以在不同时间运行这个命令来尝试捕获所有内容。*>> fping.txt* 将每次运行的新结果追加到文件中。

这个 *nmap* 示例执行了类似的任务，输出较少：

```
$ sudo nmap -sn 192.168.1.0/24 > *nmap.txt*
Starting Nmap 7.70 ( https://nmap.org ) at 2021-03-31 18:30 PDT
Nmap scan report for client1.net (192.168.1.15)
Host is up (0.0052s latency).
MAC Address: 44:A5:6E:D7:8F:B9 (Unknown)
Nmap scan report for BRW7440BBC7CA75.net (192.168.1.39)
Host is up (1.0s latency).
MAC Address: 74:40:BB:C7:CA:75 (Unknown)
Nmap scan report for client4.net (192.168.1.97)
Host is up (0.47s latency).
MAC Address: 9C:EF:D5:FE:8F:20 (Panda Wireless)
Nmap scan report for server2.net (192.168.1.91)
Host is up.
Nmap done: 256 IP addresses (6 hosts up) scanned in 15.19 seconds
```

这是一个相当难消化的块，所以在每个主机之前插入一个换行符，并将输出存储在一个新文件中：

```
$ awk '/Nmap/{print ""}1' *nmap.txt* > *nmap2.txt*
```

现在你有了良好的分组：

```
Nmap scan report for client1.net (192.168.1.15)
Host is up (0.0052s latency).
MAC Address: 44:A5:6E:D7:8F:B9 (Unknown)

Nmap scan report for BRW7440BBC7CA75.net (192.168.1.39)
Host is up (1.0s latency).
MAC Address: 74:40:BB:C7:CA:75 (Unknown)

Nmap scan report for client4.net (192.168.1.97)
Host is up (0.47s latency).
MAC Address: 9C:EF:D5:FE:8F:20 (Panda Wireless)

Nmap scan report for server2.net (192.168.1.91)
Host is up.

Nmap done: 256 IP addresses (6 hosts up) scanned in 15.19 seconds
```

探测你网络中主机的开放端口：

```
$ sudo nmap -sS  *192.168.1.**
Starting Nmap 7.70 ( https://nmap.org ) at 2021-03-31 19:36 PDT
Nmap scan report for client2.net (192.168.1.15)
Host is up (0.027s latency).
Not shown: 997 closed ports
PORT     STATE    SERVICE
53/tcp   open     domain
80/tcp   open     http
MAC Address: 44:A5:6E:D7:8F:B9 (Unknown)

Nmap scan report for 192.168.1.39
Host is up (0.074s latency).
Not shown: 994 closed ports
PORT     STATE SERVICE
25/tcp   open  smtp
80/tcp   open  http
443/tcp  open  https
515/tcp  open  printer
631/tcp  open  ipp
9100/tcp open  jetdirect
MAC Address: 74:40:BB:C7:CA:75 (Unknown)
[...]
```

client2.net 正在运行 DNS 和 Web 服务器。你可以从防火墙外运行相同的探测，看它们在你的网络之外是否可见。

第二项条目很有趣，因为它是一个运行多个服务的网络打印机。打印机文档说它们都有各自的目的。打印机支持通过 Web 控制面板进行远程管理，因此如果需要，它们可以被禁用。

收集主机及其 IP 地址的列表：

```
$ nmap -sn 192.168.43.0/24 | grep 'Nmap scan report for' |cut -d' ' -f5,6
server2 (192.168.43.15)
dns-server (192.168.43.74)
client4 (192.168.43.14)
```

## 讨论

*nmap* 具有多种选项用于探测网络。未经允许不要探测其他人的网络，因为这可能被视为一种敌对行为，探测漏洞。

运行端口扫描需要一些时间，但定期执行此操作以查看网络上正在发生的情况是一个好主意。运行只有必要服务并禁用所有其他服务是基本的安全措施。

## 参见

+   *man 1 nmap*

+   [*https://nmap.org*](https://nmap.org)

+   *man 8 fping*

+   [*https://fping.org*](https://fping.org)

# 21.3 使用 arping 查找重复的 IP 地址

## 问题

你想搜索你的网络中重复的 IP 地址。

## 解决方案

这个例子搜索你的网络中的 192.168.1.91 并发送四个 ping：

```
$ sudo arping -I wlan2 -c 4 192.168.1.91
ARPING 192.168.1.91
42 bytes from 9c:ef:d5:fe:01:7c (192.168.1.91): index=0 time=49.463 msec
42 bytes from 9c:ef:d5:fe:01:7c (192.168.1.91): index=1 time=458.306 msec
42 bytes from 9c:ef:d5:fe:01:7c (192.168.1.91): index=2 time=73.938 msec
42 bytes from 9c:ef:d5:fe:01:7c (192.168.1.91): index=3 time=504.482 msec

--- 192.168.1.91 statistics ---
4 packets transmitted, 4 packets received,   0% unanswered (0 extra)
rtt min/avg/max/std-dev = 49.463/271.547/504.482/210.659 ms
```

所有的 MAC 地址都相同，因此它没有找到重复的。这是 *arping* 找到重复 IP 地址的一个例子：

```
$ sudo arping -I wlan2 -c 4 192.168.1.91
ARPING 192.168.1.91
42 bytes from 9c:ef:d5:fe:01:7c (192.168.1.91): index=0 time=49.463 msec
42 bytes from 2F:EF:D5:FE:8F:20 (192.168.1.91): index=1 time=458.306 msec
42 bytes from 9c:ef:d5:fe:01:7c (192.168.1.91): index=2 time=73.938 msec
42 bytes from 2F:EF:D5:FE:8F:20 (192.168.1.91): index=3 time=504.482 msec
[...]

--- 192.168.1.91 statistics ---
4 packets transmitted, 4 packets received,   0% unanswered (0 extra)
rtt min/avg/max/std-dev = 49.463/271.547/504.482/210.659 ms
```

使用 *nmap* 来识别具有相同 IP 地址的两台机器：

```
$ nmap -sn *192.168.43.0/24* | grep 'Nmap scan report for' |cut -d' ' -f5,6
```

## 讨论

*arp* 是地址解析协议，将 IP 地址匹配到 MAC 地址。

使用 DHCP 动态分配 IP 地址的优势在于，比手动设置静态 IP 地址少了创建重复的风险。你可以使用 DHCP 分配静态地址；参见 第十六章。

*arping* 用于在 *ping* 找不到主机时查看主机是否在线。有些人喜欢阻止 *ping*，这是不好的，因为它对网络功能至关重要。*arping* 无法阻止而不禁用网络主机之间的通信能力。*arp*，地址解析协议，维护一个 MAC 地址表。当网络主机向另一个主机发送数据包时，*arp* 将 IP 地址与 MAC 地址匹配，然后可以传递数据包。

你可以通过像 *tcpdump* 这样的数据包嗅探器看到 *arp* 探测你的网络以更新其地址表时的情况。

```
$ sudo tcpdump -pi eth1 arp
listening on eth1, link-type EN1000MB (Ethernet), capture size 262144 bytes
21:19:36.921293 ARP, Request who-has client4.net tell m1login.net, length 28
21:19:36.921309 ARP, Reply client4.net is-at 9c:ef:d5:fe:8f:20
```

## 参见

+   第十六章

+   *man 8 arping*

# 21.4 使用 httping 测试 HTTP 吞吐量和延迟

## 问题

您想测试您托管的网站是否能在合理的时间内加载。

## 解决方案

*httping* 测量 HTTP 服务器的吞吐量和延迟。它最简单的调用测试延迟：

```
$ httping -c4 -l -g www.oreilly.com
PING www.oreilly.com:443 (/):
connected to 184.86.29.153:443 (453 bytes), seq=0 time=292.25 ms
connected to 184.86.29.153:443 (453 bytes), seq=1 time=726.35 ms
connected to 184.86.29.153:443 (452 bytes), seq=2 time=629.11 ms
connected to 184.86.29.153:443 (453 bytes), seq=3 time=529.95 ms
--- https://www.oreilly.com/ ping statistics ---
4 connects, 4 ok, 0.00% failed, time 6179ms
round-trip min/avg/max = 292.2/544.4/726.3 ms
```

这并不告诉您页面加载时间，只告诉您服务器响应头部的时间（以毫秒计），而不包含内容。GET（*-G*）请求会获取整个页面：

```
$ httping -c4 -l -Gg www.oreilly.com
PING www.oreilly.com:443 (/):
connected to 104.112.183.230:443 (453 bytes), seq=0 time=2125.72 ms
connected to 104.112.183.230:443 (453 bytes), seq=1 time=701.94 ms
connected to 104.112.183.230:443 (453 bytes), seq=2 time=470.66 ms
connected to 104.112.183.230:443 (453 bytes), seq=3 time=433.11 ms
--- https://www.oreilly.com/ ping statistics ---
4 connects, 4 ok, 0.00% failed, time 7733ms
round-trip min/avg/max = 433.1/932.9/2125.7 ms
```

添加 *-r* 开关以最小化 DNS 延迟，仅解析一次主机名：

```
$ httping -c4 -l -rGg www.oreilly.com
PING www.oreilly.com:443 (/):
connected to 23.10.2.218:443 (452 bytes), seq=0 time=961.29 ms
connected to 23.10.2.218:443 (452 bytes), seq=1 time=1091.16 ms
connected to 23.10.2.218:443 (452 bytes), seq=2 time=925.46 ms
connected to 23.10.2.218:443 (452 bytes), seq=3 time=913.26 ms
--- https://www.oreilly.com/ ping statistics ---
4 connects, 4 ok, 0.00% failed, time 7894ms
round-trip min/avg/max = 913.3/972.8/1091.2 ms
```

如果最小化 DNS 延迟有很大的差异，则需要查看您的域名服务器。

通过将其附加到 URL 来测试备用端口，例如 8080：

```
$ httping -c4 -l -rGg www.oreilly.com:8080
```

使用 *-s* 开关显示返回码，例如 200 OK，表示成功加载页面：

```
$ httping -c4 -l -srGg www.oreilly.com
PING www.oreilly.com:443 (/):
connected to 23.10.2.218:443 (452 bytes), seq=0 time=920.88 ms 200 OK
connected to 23.10.2.218:443 (452 bytes), seq=1 time=857.60 ms 200 OK
connected to 23.10.2.218:443 (452 bytes), seq=2 time=1246.69 ms 200 OK
connected to 23.10.2.218:443 (452 bytes), seq=3 time=1134.91 ms 200 OK
--- https://www.oreilly.com/ ping statistics ---
4 connects, 4 ok, 0.00% failed, time 8249ms
round-trip min/avg/max = 857.6/1040.0/1246.7 ms
```

## 讨论

在不同时间运行多次测试，收集具有代表性的用户体验数据。

*httping* 不是一个深入分析您站点瓶颈的超级复杂的测试工具。它是一个快速简单的工具，可帮助您了解整体站点性能，并告诉您是否需要深入诊断性能问题。

## 参见

+   [HTTP 返回码](https://oreil.ly/pMvFV)

+   *man 1 httping*

+   [httping](https://oreil.ly/2ts3n)

# 21.5 使用 *mtr* 查找有问题的路由器

## 问题

有一个您正在尝试访问的站点，但它非常缓慢或无法访问。

## 解决方案

使用 *mtr*（My Traceroute）查看数据包走错的位置。这在您控制的网络上效果更好，因为互联网辽阔，路由会发生变化，但当您无法访问站点时，它将提供有用信息。

让我们看看漫游的路径如何带我们到 *carlaschroder.com*：

```
$ mtr -wo LSRABW carlaschroder.com
Start: 2021-03-31T09:54:17-0700
HOST: client4                               Loss%   Snt   Rcv   Avg  Best  Wrst
  1.|-- m1login.net                            0.0%    10    10  55.5   1.2 199.6
  2.|-- 172.26.96.169                          0.0%    10    10  92.3  29.0 243.6
  3.|-- 172.18.84.60                           0.0%    10    10  84.5  29.3 220.3
  4.|-- 12.249.2.25                            0.0%    10    10  80.7  36.4 215.5
  5.|-- 12.122.146.97                          0.0%    10    10  65.6  34.8 156.6
  6.|-- 12.122.111.33                          0.0%    10    10  49.3  35.5  97.6
  7.|-- cr2.st6wa.ip.att.net                   0.0%    10    10  46.7  35.9  64.0
  8.|-- 12.122.111.109                         0.0%    10    10  57.9  31.4 215.4
  9.|-- 12.122.111.81                          0.0%    10    10  72.3  27.6 231.4
 10.|-- 12.249.133.242                         0.0%    10    10 101.2  31.7 263.1
 11.|-- ae6.cbs01.wb01.sea02.networklayer.com  0.0%    10    10  93.7  31.6 202.7
 12.|-- fc.11.6132.ip4.static.sl-reverse.com   0.0%    10    10 106.0  86.1 171.2
 13.|-- ae1.cbs02.eq01.dal03.networklayer.com 60.0%    10     4 102.0  86.5 115.8
 14.|-- ae0.dar01.dal13.networklayer.com       0.0%    10    10 103.7  80.3 230.8
 15.|-- 85.76.30a9.ip4.static.sl-reverse.com   0.0%    10    10 114.8  82.8 305.7
 16.|-- a1.76.30a9.ip4.static.sl-reverse.com   0.0%    10    10 122.7  83.7 278.4
 17.|-- hs17.name.tools                        0.0%    10    10 145.9  74.9 277.2

```

m1login.net 是我的网络的互联网网关路由器。之后就是广阔的互联网世界。第 13 跳可能是一个瓶颈，丢包率为 60%。第 13 跳可能是负载均衡集群的一部分；注意第 11 跳和第 14 跳具有相同的域名。如果它是集群的一部分，那么丢包并不重要。

Ping 最后一跳，hs17.name.tools。以下示例看起来一切正常：

```
$ ping -c 3 hs17.name.tools
PING hs17.name.tools (169.61.1.230) 56(84) bytes of data.
64 bytes from hs17.name.tools (169.61.1.230): icmp_seq=1 ttl=46 time=319 ms
64 bytes from hs17.name.tools (169.61.1.230): icmp_seq=2 ttl=46 time=168 ms
64 bytes from hs17.name.tools (169.61.1.230): icmp_seq=3 ttl=46 time=166 ms
[...]
```

如果 *mtr* 显示问题，请使用 *whois* 查询域名所有者及其联系信息：

```
$ whois -H networklayer.com
```

*whois* 也适用于 IP 地址。*-H* 开关可以禁用烦人的法律术语。

在每个条目末尾附上日期和时间，将 *mtr* 输出保存到文件中：

```
$ mtr -r -c25 oreilly.com >> mtr.txt && date >> mtr.txt
```

通过创建一个 cron 作业（Recipe 3.7）每小时运行前述的 *mtr* 并让其运行一天或两天来随时间收集数据。别忘了关闭它。

## 讨论

*mtr -wo LSRABW* 限制列数，以便示例更好地适应本页。*mtr -w* 是报告的宽格式。

如果需要报告问题，请保存您的记录；*whois* 示例显示如何找到联系人。

*mtr* 生成大量流量，因此请小心不要频繁运行它。

## 参见

+   *man 8 mtr*
