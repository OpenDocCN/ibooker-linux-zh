# 第十七章：使用 ntpd、chrony 和 timesyncd 进行时间保持

通过 NTP，网络时间协议，在你的计算机和网络上所有主机上保持精确的时间是容易且自动的。在 Linux 上，NTP 被实现为 *ntpd*，即 NTP 守护进程，*chrony*，即 *ntpd* 的现代替代品，以及 systemd 的 *timesyncd*。没错，朋友们，有三种（至少），数一数，三种方式可以自动管理你的 Linux 计算机上的时间。

*ntpd* 和 *chrony* 可以作为局域网时间服务器，而 *timesyncd* 则是一个更简单的轻量级客户端，没有服务器功能。*ntpd* 和 *chrony* 是完整的 NTP 实现，而 *timesyncd* 使用简单网络时间协议 SNTP。

大多数 Linux 发行版提供一个默认配置，指向它们维护的时间服务器。这些服务器的名称如 *2.fedora.pool.ntp.org* 和 *0.ubuntu.pool.ntp.org*。你不必做任何事情，只需确保在安装过程中不要禁用它们。在本章中，您将学习如何检查当前设置，如何更改它们以及如何设置局域网时间服务器。

有一个全球网络的时间服务器供每个人免费使用，并且它们按照 *strata* 组织，从第 0 级开始。第 0 级是所有时间保持的源头，是一个原子钟、调谐到原子钟的无线接收器，以及使用 GPS 卫星广播信号的 GPS 接收器的网络。

接下来是第 1 级，那里是主要的时间服务器所在。第 1 级的主要时间服务器直接连接到第 0 级的来源。

第 2 级包含成千上万个公共服务器，并且它们与第 1 级同步。与第 2 级服务器连接是良好的礼仪，以防止第 1 级服务器不堪重负，并且不要没有充分理由地使用第 1 级服务器。

层次结构沿着这条线继续，例如有第 4、5 和 6 级公共服务器，以及与它们同步的私人局域网服务器。实际情况并不那么有条理；你可以将你的私人局域网 NTP 服务器指定为第 10 级，并且它不必连接到第 9 级服务器，而是任何它能够到达的服务器。你不必担心选择正确的服务器，因为你将使用 *池* 服务器，这些服务器是 NTP 服务器的集群，而不是单个服务器。

当你深入研究计算机上的时间保持时，它会变得令人困惑和不知所措，或许取决于你想要多么迷恋这个话题。访问[NTP 池项目](https://ntppool.org)和[NTP：网络时间协议](http://ntp.org)来学习这些技术和如何运行自己的公共时间服务器。

你的 Linux 系统上至少有两个时间管理器。一个是你主板上的硬件时钟，也称为实时时钟（RTC）。另一个是由 Linux 内核管理的系统时间。即使你的机器关闭了，RTC 也会始终有电，这得益于主板上的电池或电容器。当你的 Linux 计算机启动时，你选择的 NTP 客户端从 RTC 获取时间。然后，在网络可用后，它会根据其上游时间服务器纠正时间。

你的 RTC 时间由 BIOS/UEFI 设置，并且使用本章中将学到的某些命令。它应该始终设置为协调世界时（UTC），然后 Linux 内核从 UTC 计算你的时区时间。UTC 类似于格林尼治平均时间（GMT），尽管它们不完全相同。UTC 是一个时间标准，而 GMT 是一个时间区域。UTC 和 GMT 都不会因夏令时改变。

时区数据来自[IETF.org 时区](https://oreil.ly/gUnet)。这是一个动态的目标，因为各国改变其夏令时日期、选择退出夏令时或重新加入。大多数 Linux 系统将这些信息存储在*/usr/share/zoneinfo/*中。互联网工程任务组（IETF）跟踪这些变化并免费提供他们的数据库。

# 17.1 确定你的 Linux 系统上使用了哪个 NTP 客户端

## 问题

你已经阅读了章节介绍，现在你知道在 Linux 上，时间同步由*ntpd*、*chrony*或*timesyncd*管理，你需要知道你的 Linux 系统使用的是哪一个。

## 解决方案

使用*ps*命令查看系统上是否有这三个时间同步守护进程之一正在运行： 

```
$ ps ax|grep -w ntp
$ ps ax|grep -w chrony
$ ps ax|grep -w timesyncd

```

如果有任何一个正在运行，请跳到本章的相关小节，学习如何管理你的时间守护进程。

如果没有任何一个在运行，请查看你的系统是否在使用*timedatectl*，它是 systemd 的一部分：

```
$ timedatectl status
                      Local time: Sun 2020-10-04 10:59:48 PDT
                  Universal time: Sun 2020-10-04 17:59:48 UTC
                        RTC time: Sun 2020-10-04 17:59:48
                       Time zone: America/Los_Angeles (PDT, -0700)
       System clock synchronized: no
systemd-timesyncd.service active: no
                 RTC in local TZ: no

```

这个输出显示*timedatectl*在运行，没有任何时间守护进程，显示*systemd-timesyncd.service active: no*。通过查询*systemd-timesyncd*的状态再次确认：

```
$ systemctl status systemd-timesyncd
● systemd-timesyncd.service - Network Time Synchronization
   Loaded: loaded (/lib/systemd/system/systemd-timesyncd.service; disabled;
vendor preset: enabled)
   Active: inactive (dead)
     Docs: man:systemd-timesyncd.service(8)

```

这表明*systemd-timesyncd*没有运行，这意味着你的系统没有时间同步，而是从你系统的实时时钟（RTC）获取时间。在这种情况下，你需要设置*ntpd*、*chrony*或*timesyncd*。

## 讨论

一些 Linux 发行版不使用 systemd；查看第 4.1 节了解如何知道你的 Linux 是否安装了它。如果你运行的是没有 systemd 的 Linux 系统，则你的 NTP 选择是*ntpd*或*chrony*。

如果你发现系统上同时运行*ntpd*和*chrony*，请删除*ntpd*，因为*chrony*更新更快、更可靠。同时使用两者会造成冲突。

*timedatectl*的输出包含大量有用信息。示例显示 RTC 已正确设置为协调世界时（UTC）协议，并且系统时区为太平洋夏令时，PDT。*systemd-timesyncd.service*未运行，并且系统未同步。

## 参见

+   [timedatectl：控制系统时间和日期](https://oreil.ly/QddJ7)

+   *man 1 ps*

# 17.2 使用 timesyncd 进行简单时间同步

## 问题

您想知道如何设置最简单的 NTP 客户端以保持计算机上的正确时间。

## 解决方案

使用*systemd-timesyncd*守护程序启用与公共 NTP 服务器的同步，该守护程序需要 systemd。检查*systemd-timesyncd*的状态：

```
$ systemctl status systemd-timesyncd
● systemd-timesyncd.service - Network Time Synchronization
     Loaded: loaded (/usr/lib/systemd/system/systemd-timesyncd.service;
       disabled; vendor preset: enabled)
     Active: inactive (dead)
       Docs: man:systemd-timesyncd.service(8)

```

使用*timedatectl*启用它，并验证*systemd-timesyncd*已启动：

```
$ timedatectl set-ntp true
$ systemctl status systemd-timesyncd
● systemd-timesyncd.service - Network Time Synchronization
   Loaded: loaded (/lib/systemd/system/systemd-timesyncd.service; enabled;
vendor preset: enabled)
   Active: active (running) since Sun 2020-10-04 18:17:51 PDT; 16min ago
     Docs: man:systemd-timesyncd.service(8)
 Main PID: 3990 (systemd-timesyn)
   Status: "Synchronized to time server 91.189.89.198:123 (ntp.ubuntu.com)."
    Tasks: 2 (limit: 4915)
   CGroup: /system.slice/systemd-timesyncd.service
           └─3990 /lib/systemd/systemd-timesyncd

Oct 04 18:17:51 pc systemd[1]: Starting Network Time Synchronization...
Oct 04 18:17:51 pc systemd[1]: Started Network Time Synchronization.
Oct 04 18:33:01 pc systemd-timesyncd[3990]: Synchronized to time server
91.189.89.198:123 (ntp.ubuntu.com).

```

如果*systemd-timesyncd*未启动，请启动它：

```
$ sudo systemctl start systemd-timesyncd

```

现在查看*timedatectl*报告的内容：

```
$ timedatectl status
                      Local time: Sun 2020-10-04 18:35:56 PDT
                  Universal time: Mon 2020-10-05 01:35:56 UTC
                        RTC time: Mon 2020-10-05 01:35:56
                       Time zone: America/Los_Angeles (PDT, -0700)
       System clock synchronized: yes
systemd-timesyncd.service active: yes
                 RTC in local TZ: no

```

一切看起来都正确。您的系统已同步，所有时间都正确，并且*systemd-timesyncd.service*处于活动状态。

配置多个公共时间服务器以增强冗余性是一种良好的实践。编辑*/etc/systemd/timesyncd.conf*以通过取消注释`NTP`行并输入空格分隔的公共服务器池列表来添加更多 NTP 服务器：

```
[Time]
NTP=0.north-america.pool.ntp.org 1.north-america.pool.ntp.org
2.north-america.pool.ntp.org
#FallbackNTP=ntp.ubuntu.com
#RootDistanceMaxSec=5
#PollIntervalMinSec=32
#PollIntervalMaxSec=2048

```

## 讨论

在您原始的*/etc/systemd/timesyncd.conf*文件中，注释选项记录了默认配置。

由于它们是单个池中的多个服务器，而不是单个服务器，因此池服务器非常可靠。为了获得最佳性能，请使用适合您地区的池服务器，可以通过单击大陆池链接或国家池找到它们。

您的 Linux 发行版可能会配置多个自己的服务器池，例如：

```
0.opensuse.pool.ntp.org 1.opensuse.pool.ntp.org 2.opensuse.pool.ntp.org
```

这是很好的，您不需要更改它，但通常更多样化的配置通常更可靠。

## 参见

+   第四章

+   [NTP Pool 项目](https://ntppool.org)

+   *man 5 timesyncd.conf*

# 17.3 使用 timedatectl 手动设置时间

## 问题

你希望手动设置系统和 RTC 时间。

## 解决方案

使用*timedatectl*。它通过单个命令设置日期、系统时间和 RTC 时间：

```
$ timedatectl set-time "2020-10-04 19:30:00"
Failed to set time: Automatic time synchronization is enabled

```

当*systemd-timesyncd*正在运行时，您无法执行此操作，因此必须停止它：

```
$ sudo systemctl stop systemd-timesyncd
```

然后按照示例中显示的格式 YYYY-MM-DD HH:MM:SS 输入新的设置，并验证其是否有效：

```
$ timedatectl set-ntp false
$ timedatectl set-time "2020-10-04 19:30:00"
$ timedatectl status
                      Local time: Sun 2020-10-04 19:30:06 PDT
                  Universal time: Mon 2020-10-05 02:30:06 UTC
                        RTC time: Mon 2020-10-05 02:30:06
                       Time zone: America/Los_Angeles (PDT, -0700)
       System clock synchronized: no
systemd-timesyncd.service active: no
                 RTC in local TZ: no

```

如果重新启动*systemd-timesyncd*，它将覆盖您的手动设置。

## 讨论

*timedatectl*有一组少量的命令。如果您习惯于使用*date*命令设置时间以及其他时间和日期操作，那么*timedatectl*可能会显得轻量级。它设计简单，您仍然可以使用*date*及其众多选项执行复杂任务。

## 参见

+   *man 5 timesyncd.conf*

# 17.4 使用 chrony 作为您的 NTP 客户端

## 问题

您希望拥有功能齐全的 NTP 客户端/服务器，并且想知道如何将*chrony*设置为您的 NTP 客户端。

## 解决方案

首先，检查是否安装了 *ntpd*。如果安装了，请将其删除。如果您有 *systemd-timesyncd*，请将其禁用：

```
$ sudo systemctl disable systemd-timesyncd
$ sudo systemctl stop systemd-timesyncd

```

然后安装 *chrony*。大多数 Linux 发行版的软件包名称是 *chrony*。安装后，使用 *chronyc* 命令来检查其状态：

```
$ chronyc activity
200 OK
8 sources online
0 sources offline
0 sources doing burst (return to online)
0 sources doing burst (return to offline)
0 sources with unknown address

```

成功！它已经在工作中，如 *8 sources online* 所示。（如果未启动，请参阅讨论。）找到您的 *chrony.conf*，可能是 */etc/chrony.conf*（Fedora）或 */etc/chrony/chrony.conf*（Ubuntu），查看设置。如果需要，您不需要做太多更改就可以用它作为客户端。检查 NTP 服务器列表，您将看到 *server* 选项或 *pool*。在 Ubuntu 系统上的以下示例包括默认的 Ubuntu NTP 服务器池和本地 LAN 服务器：

```
pool 0.ubuntu.pool.ntp.org iburst
pool 1.ubuntu.pool.ntp.org iburst
pool 1.ubuntu.pool.ntp.org iburst
server ntp.domain.lan iburst prefer
```

您可以使用一些公共服务器池替换一些 Ubuntu 服务器池，以提高可靠性和多样性：

```
pool 0.ubuntu.pool.ntp.org iburst
pool 1.ubuntu.pool.ntp.org iburst
pool 0.north-america.pool.ntp.org iburst
pool 1.north-america.pool.ntp.org iburst
server ntp.domain.lan iburst prefer
```

在更改配置文件后重新启动 *chronyd*。

## 讨论

*iburst* 意味着在网络中断后快速同步，*prefer* 意味着除非不可用，否则始终使用此服务器。

这确实是客户端设置所需的全部操作。*chrony* 是一个完整的 NTP 实现，并拥有许多选项；详细描述请参阅 *man 5 chrony.conf*。

管理 *chronyd* 就像使用以下命令管理任何其他服务一样：

+   *systemctl status chrony*

+   *sudo systemctl stop chrony*

+   *sudo systemctl start chrony*

+   *sudo systemctl restart chrony*

Chrony 在客户端方面比 *ntpd* 有几个优点。主要优点包括更好地处理网络连接中断和在恢复连接时更快地重新同步。

## 另请参阅

+   [chrony](https://oreil.ly/1S41c)

+   *man 5 chrony.conf*

+   *man 1 chronyc*

# 17.5 使用 chrony 作为 LAN 时间服务器

## 问题

您想将 *chrony* 设置为您的 LAN 时间服务器。

## 解决方案

就像在 Recipe 17.4 中一样，如果系统中存在 *systemd-timesyncd*，请禁用它并删除 *ntpd*。然后安装 *chrony* 软件包。

查找配置文件，可以是 */etc/chrony.conf*（Fedora、openSUSE）或 */etc/chrony/chrony.conf*（Ubuntu）。以下示例是一个基本的配置：

```
*pool 0.north-america.pool.ntp.org iburst
pool 1.north-america.pool.ntp.org iburst
pool 2.north-america.pool.ntp.org iburst

local stratum 10
allow 192.168.0.0/16
allow 2001:db8::/56*

driftfile /var/lib/chrony/chrony.drift
maxupdateskew 100.0
rtcsync
logdir /var/log/chrony
log measurements statistics tracking
leapsectz right/UTC
makestep 1 3
```

然后，您的客户端需要将您的服务器名称添加到其 *chrony.conf* 文件中：

```
server *ntp.domain.lan* iburst prefer
```

*prefer* 选项意味着只要该服务器可用，始终使用此服务器。保留本地时间服务器的原因之一是减轻公共时间服务器的负载。使用 *prefer* 选项，您可以将一些公共服务器配置为备用服务器，以防本地服务器不可用，并且无需担心给它们带来负担，如下所示：

```
server *ntp.domain.lan* iburst prefer
pool 1.north-america.pool.ntp.org iburst
pool 2.north-america.pool.ntp.org iburst
```

## 讨论

*local stratum 10* 配置 *chrony* 以继续充当您的本地 NTP 服务器，即使您的互联网连接中断，*stratum 10* 将您的服务器安全地放在层次结构中的较低位置，以便它低于您使用的任何外部 NTP 服务器。允许的值为 1–15。请使用除 10 以外的数字，以防此设置使 *stratum 10* 大受欢迎。

*allow* 选项定义允许使用你的 NTP 服务器的网络。

*rtcsync* 告诉 *chrony* 保持 RTC 与系统时间同步。

*log* 启用日志记录并定义需要记录的事件。

你可以在 *man 5 chrony.conf* 或默认的 *chrony.conf* 中查找其他选项，通常这些文件都有详细的注释。

## 参见

+   [chrony](https://oreil.ly/1S41c)

+   *man 5 chrony.conf*

+   *man 1 chronyc*

# 17.6 查看 chrony 统计信息

## 问题

你想要调用一些实时的 *chrony* 活动和统计信息，如上游 NTP 服务器、偏移、偏差、当前与之同步的服务器以及其他信息。

## 解决方案

使用 *chronyc* 命令。*tracking* 子命令显示已应用的校正量、RTC 时间、偏差等信息：

```
$ chronyc tracking
Reference ID    : A29FC87B (time.cloudflare.com)
Stratum         : 4
Ref time (UTC)  : Tue Oct 06 02:20:23 2020
System time     : 0.002051390 seconds fast of NTP time
Last offset     : +0.002320110 seconds
RMS offset      : 0.017948814 seconds
Frequency       : 28.890 ppm fast
Residual freq   : +0.252 ppm
Skew            : 1.250 ppm
Root delay      : 0.069674924 seconds
Root dispersion : 0.003726898 seconds
Update interval : 838.2 seconds
Leap status     : Normal

```

列出当前源服务器：

```
$ chronyc sources
chronyc sources
210 Number of sources = 19
MS Name/IP address         Stratum Poll Reach LastRx Last sample
===============================================================================
^- golem.canonical.com           2   9     0   37m    +55ms[  +58ms] +/-  209ms
^- alphyn.canonical.com          2   9     0   34m    +23ms[  +25ms] +/-  158ms
^- pugot.canonical.com           2   9     0   44m    +92ms[  +80ms] +/-  229ms
^- chilipepper.canonical.com     2   9    11    31    +48ms[  +48ms] +/-  181ms
[...]

```

列出当前源服务器及其描述：

```
$ chronyc sources -v
210 Number of sources = 19

  .-- Source mode  '^' = server, '=' = peer, '#' = local clock.
 / .- Source state '*' = current synced, '+' = combined , '-' = not combined,
| /   '?' = unreachable, 'x' = time may be in error, '~' = time too variable.
||                                                 .- xxxx [ yyyy ] +/- zzzz
||      Reachability register (octal) -.           |  xxxx = adjusted offset,
||      Log2(Polling interval) --.      |          |  yyyy = measured offset,
||                                \     |          |  zzzz = estimated error.
||                                 |    |           \
MS Name/IP address         Stratum Poll Reach LastRx Last sample
===============================================================================
^- golem.canonical.com           2   9     0   46m    +67ms[  +58ms] +/-  209ms
^- alphyn.canonical.com          2   9     0   44m    +35ms[  +25ms] +/-  158ms
^* pugot.canonical.com           2   9     1   54m   +104ms[  +80ms] +/-  229ms
^- chilipepper.canonical.com     2   9    11   587    +60ms[  +48ms] +/-  181ms
^- ntp.wdc1.us.leaseweb.net      2   7     4   327    +26ms[  +15ms] +/-  198ms
^- 216.126.233.109               2   9     1   459   +106ms[  +95ms] +/-  171ms
^- 157.245.170.163               3   9     1   476  +1191us[  -10ms] +/-  145ms

```

星号显示你的系统当前正在与哪个服务器同步。

## 讨论

*chrony* 调整网络延迟、间歇连接、以及客户端机器的睡眠和休眠模式。*chrony* 时钟从不停止，即使外部名称服务器不可用也能保持网络同步。

## 参见

+   *man 1 chronyc*

+   [chrony.tuxfamily.org](https://oreil.ly/1S41c)

# 17.7 使用 ntpd 作为你的 NTP 客户端

## 问题

是的，你已经了解 *chrony* 和 *timesyncd*，以及它们的优点，但你仍然希望将 *ntpd* 作为你的 NTP 客户端。

## 解决方案

没问题，因为 *ntpd* 正在积极维护，并且能很好地完成工作。首先确保 *ntpd* 是系统上唯一的 NTP 客户端（参见 Recipe 17.1）。在大多数 Linux 发行版中，查找 *ntp* 包进行安装。

在大多数 Linux 发行版中，*ntpd* 配置有用的配置，并且在安装后启动。用 *ps* 命令检查：

```
$ ps ax | grep -w ntpd
3754 ?        Ssl    0:00 /usr/sbin/ntpd -u ntp:ntp -g

```

如果不会自动启动，请手动启动：

```
$ systemctl start ntpd

```

当 *ntpd* 运行时，请查看你的配置文件，通常是 */etc/ntp.conf*。Linux 发行版的默认配置应该能正常工作，不需要更改。如果你的网络有自己的 LAN 服务器，以下配置将本地服务器设置为主服务器，Fedora Linux 服务器池作为备用：

```
server *ntp.domain.lan* iburst prefer
pool 2.fedora.pool.ntp.org iburst
```

大多数情况下，你可以保持默认配置，这对大多数情况都适用。Linux 发行版通常会维护自己的 NTP 服务器池，并在默认配置中提供这些信息。如果你希望替换或添加一些外部公共服务器，请查看[continental pools](https://oreil.ly/W70Ba)以获取大洲 NTP 服务器池的列表，或者使用你国家的池，你可以通过点击大洲池链接找到它们。

当更改 */etc/ntp.conf* 时，重新启动 *ntpd*：

```
$ systemctl restart ntpd

```

```
$ sudo /etc/init.d/ntp restart

```

使用 *ntpq* 检查其是否工作。星号显示该机器正在与 LAN NTP 服务器同步：

```
$ ntpq -p

     remote           refid      st t when poll reach   delay   offset  jitter
==============================================================================
 2.fedora.pool.n .POOL.          16 p    -   64    0    0.000   +0.000   0.000
**ntp.domain.lan*. 172.16.16.3      2 u   34  256  203   80.324  -49.772  54.508
+138.68.46.177 ( 80.153.195.191   2 u   92  256  123   90.932  -15.534  39.947
+vps6.ctyme.com  216.218.254.202  2 u  453  256   46   69.927  -29.296  84.811
+ec2-3-217-79-24 132.163.97.6     2 u  426  256  202  165.888  -51.442  93.224

```

## 讨论

*iburst* 告诉 *ntpd* 在系统启动时快速同步。

*prefer* 意味着使用这个服务器，并且只在其不可用时使用其他服务器。

## 参见

+   *man 5 ntp.conf*

+   *man 8 ntpd*

+   *man 8 ntpq*

+   [NTP 文档](https://oreil.ly/lpDgk)

# 17.8 使用 ntpd 作为您的 NTP 服务器

## 问题

您想知道如何为您的 LAN 运行 *ntpd* 服务器。

## 解决方案

在 LAN 时间服务器上使用 *ntpd* 与作为 NTP 客户端使用它类似。配置几乎相同，只需添加一些访问控制。以下示例是完整的 */etc/ntp.conf* 配置：

```
driftfile /var/lib/ntp/drift

restrict default nomodify notrap nopeer noquery
restrict -6 default nomodify notrap nopeer noquery
restrict 127.0.0.1
restrict ::1

pool 0.north-america.pool.ntp.org
pool 1.north-america.pool.ntp.org
pool 2.north-america.pool.ntp.org

leapfile /usr/share/zoneinfo/leap-seconds.list

statistics clockstats loopstats peerstats
filegen loopstats file loopstats type day enable
filegen peerstats file peerstats type day enable
filegen clockstats file clockstats type day enable
statsdir /var/log/ntpstats/
```

## 讨论

driftfile 是 *ntpd* 跟踪由主板上石英晶体振荡频率波动引起的时间漂移的地方。您有以下选项：

+   *restrict default* 拒绝所有，只允许显式允许的内容，并设置默认值。

+   *nomodify* 不允许其他时间服务器对您的系统进行任何更改。允许查询。

+   *notrap* 禁用远程记录。

+   *nopeer* 不允许对等。对等服务器彼此同步，因此唯一允许提供时间服务的服务器由 *server* 或 *pool* 指令指定。

+   *noquery* 禁止远程查询和远程记录。

+   *restrict 127.0.0.1* 和 *restrict ::1* 表示信任本地主机。

*statistics* 部分将您选择的统计信息记录到 */var/log/ntpstats/* 中。这并非必需，但跟踪哪些上游 NTP 服务器性能最佳可能很有趣。

## 参见

+   *man 5 ntp.conf*

+   *man 8 ntpd*

+   *man 8 ntpq*

+   [NTP 文档](https://oreil.ly/lpDgk)

# 17.9 使用 timedatectl 管理时区

## 问题

您想列出所有时区，查看当前时区并更改时区。

## 解决方案

使用 *timedatectl*。查看当前时区：

```
$ timedatectl | grep -i "time zone"
     Time zone: America/Los_Angeles (PDT, -0700)

```

列出所有时区：

```
$ timedatectl list-timezones
Africa/Abidjan
Africa/Accra
Africa/Addis_Ababa
Africa/Algiers
[...]

```

列表超过 400 行。当您知道您要查找什么时，请使用 *grep* 命令。例如，列出主要城市，如柏林：

```
$ timedatectl list-timezones | grep -i berlin
Europe/Berlin

```

将此设置为您的时区：

```
$ sudo timedatectl set-timezone Europe/Berlin

```

变更立即生效。再次运行 *timedatectl* 验证。

## 讨论

当您与不同时区的人们合作时，请使用 UTC 协调会议时间。有几个在线时区转换器，例如 [时区转换器](https://oreil.ly/NyLj7)。

您必须使用区域/城市格式，如 *timedatectl list-timezones* 所示，来设置您的时区。这由 ISO 8601 标准定义，该标准定义了一种清晰表达时区、时间和日期的标准。该标准使用“降序表示法”，即从最大值到最小值的顺序。例如，对于时区，顺序是大洲/国家/城市。美国习惯将日期写为年-日-月不符合此标准。年-月-日 是标准格式，年份为四位数，YYYY-MM-DD。时间为 HH:MM:SS，采用 24 小时制。

官方发布的 ISO 8601 标准需付费，但通过一些网页搜索应该可以免费找到信息。

## 参见

+   *man 1 timedatectl*

# 17.10 不使用 timedatectl 管理时区

## 问题

您的 Linux 系统没有 systemd，需要了解如何使用哪些命令来管理时区。

## 解决方案

使用 *date* 命令查看当前时区：

```
$ date
Wed Oct  7 08:32:40 PDT 2020

```

或查看 */etc/localtime* 链接到的内容：

```
$ ls -l /etc/localtime
lrwxrwxrwx 1 root root 41 Oct  7 08:06 /etc/localtime ->
../usr/share/zoneinfo/America/Los_Angeles

```

*/usr/share/zoneinfo* 目录包含所有时区：

```
$ ls /usr/share/zoneinfo
total 324
drwxr-xr-x  2 root root  4096 May 21 23:02 Africa
drwxr-xr-x  6 root root 20480 May 21 23:02 America
drwxr-xr-x  2 root root  4096 May 21 23:02 Antarctica
drwxr-xr-x  2 root root  4096 May 21 23:02 Arctic
[...]

```

查看子目录以找到与您所在城市接近的城市，例如，马德里：

```
$ ls /usr/share/zoneinfo/Europe
[...]
-rw-r--r-- 1 root root 2637 May  7 17:01 Madrid
-rw-r--r-- 1 root root 2629 May  7 17:01 Malta
lrwxrwxrwx 1 root root    8 May  7 17:01 Mariehamn
-rw-r--r-- 1 root root 1370 May  7 17:01 Minsk
[...]

```

通过更改链接到 */etc/localtime* 来更改您的时区：

```
$ sudo ln -sf /usr/share/zoneinfo/Europe/Madrid/etc/localtime

```

更改立即生效。

## 讨论

这个很酷的命令按字母顺序列出所有时区：

```
$ php -r 'print_r(timezone_identifiers_list());'
Array
(
    [0] => Africa/Abidjan
    [1] => Africa/Accra
    [2] => Africa/Addis_Ababa
    [3] => Africa/Algiers
    [4] => Africa/Asmara
[...]
```

*php* 命令位于 *php-cli* 包中。

您的图形桌面应该有一个很好的图形实用程序来管理时间、日期和时区。如果桌面上显示了时钟，请尝试右键单击它以打开属性或设置面板。

## 参见

+   *man 1 date*

+   *man 1 ln*
