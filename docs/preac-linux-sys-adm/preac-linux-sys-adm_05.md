# 第五章：连接到网络

独立的 Linux 系统功能强大，但其最终能力范围有限。只有一个操作员或管理员可以同时使用独立系统。虽然系统仍然可以运行多个工作负载，但其服务仅限于本地访问。

Linux 是一种多任务、多用户操作系统。它最突出的价值之一是能够与其他计算机联网，允许多用户同时运行各种工作负载。将 Linux 系统连接到网络使其能够成为本地网络、网格、云或全球互联网的一部分。

在本章中，您将学习如何为您的网络选择 IP 寻址方案以及静态和动态选项的一些优缺点。您还将了解将系统连接到网络的安全影响。您将学习如何通过实施良好的安全实践，如使用安全协议、关闭不必要的服务和守护程序，以及保持系统的补丁和更新，尽可能地防止安全漏洞。

# 连接到网络

将服务器系统连接到现有网络不需要太多技能。如今，一旦新系统上线，动态主机配置协议（DHCP）服务器将为其提供 IP 地址、子网掩码、网关、域名系统（DNS）服务器和一些基本的路由信息。关于 DHCP 和服务器有两种思想流派。一种认为所有服务器系统都应具有静态 IP 地址，另一种则认为所有系统都应使用 DHCP 进行 IP 地址分配和管理。我一直将服务器、打印机和网络设备配置为静态 IP 地址，这些 IP 地址接近 IP 地址池的下端。对我来说，这样做比依赖 DHCP 和 DNS 服务来维护用户非常依赖的服务的秩序更有组织性。

在接下来的章节中，我将描述两种 IP 寻址方案——静态和动态——以及各自的优缺点。每种情况下的示例网络具有以下 IP 寻址信息：

+   网络：192.168.1.0/24

+   网关：192.168.1.254（静态 IP 地址）

+   DNS：192.168.1.1, 192.168.1.2（静态 IP 地址）

此示例网络仅用于学习目的，仅适用于最小的公司或用户组。因为在这个网络上只有 251 个可用的 IP 地址（254 - 3 = 251，扣除网关地址和两个 DNS 服务器地址），所以这种方案只适用于小型公司。用户可能有多达五个消耗 IP 地址的设备，很容易看出这样一个小的地址池很快就会被用完。

分配静态 IP 地址在 Linux 发行版中有所不同。在这些示例中，我使用 Debian 和 CentOS 发行版。

## 静态 IP 地址分配

静态 IP 地址是嵌入到配置文件中的“硬编码” IP 地址。除非手动更改，否则该 IP 地址不会变化。静态 IP 地址不依赖于或使用 DHCP 服务。在为系统分配静态 IP 地址时，您应该决定一个保留的 IP 地址块，用于服务器、网络设备（交换机、路由器、WiFi 接入点）、打印机以及任何不可移动的 IP 可能系统。工作站、笔记本电脑、平板电脑、手机和其他移动设备应使用 DHCP。

例如，在前一节提到的网络参数中，我保留了 192.168.1.1 到 192.168.1.25 用于服务器、网络设备、打印机和其他固定系统。正如您所注意到的，地址 192.168.1.1 和 .2 已在此保留列表中使用。您可以在 DHCP 配置中排除一段 IP 地址范围，以使这些地址不成为可用 IP 地址池的一部分。

使用静态 IP 地址池的一个优点是网络服务稳定。如果在您的网络上设置了一些库存服务和监视，我强烈建议您，拥有这些静态寻址系统将为您节省一些麻烦。您将能够随时间比较库存、看到发生了什么变化、测量增长并计划变更。另一个优点是，即使 DNS 和 DHCP 失效，您也可以找到关键系统。静态 IP 地址为您提供了所需的稳定性，无论其他服务的状态如何。

一些网络服务要求服务器系统具有静态 IP 地址。FTP、Web、VPN、数据库、Active Directory 和 DNS 等服务需要静态 IP 地址或建议具有静态 IP 地址。静态 IP 地址分配的主要缺点和反对观点是管理。争论是管理静态 IP 地址是费时费力的，可能会在网络的其他地方引起冲突。对于大多数系统管理员来说，我不认为这是一个问题，因为您为静态 IP 地址分配的设备使用年限长，它们的位置是稳定的。您通常不会将服务器移动到公司内的不同位置；您不会像笔记本电脑、平板电脑和手机一样将它们移动到外部使用，然后归还。从 DHCP 池中排除的保留池可以防止与其他系统的潜在冲突。

## 动态 IP 地址分配

使用 DHCP 服务可以消除在 DHCP 之前困扰系统管理员的许多 IP 地址管理问题。为超过几个系统分配静态 IP 地址并跟踪它们变成了管理噩梦。在桌面系统唯一存在的时代，静态 IP 地址有点合理。但如今，有了笔记本电脑、平板电脑和手机，静态 IP 地址可能意味着用户只能在其企业网络上访问，如果他们连接到另一个网络，可能会遇到 IP 冲突，或者他们必须知道如何每次连接到不同网络时将其系统从静态分配 IP 地址改为动态分配 IP 地址。

例如，如果您的笔记本电脑在公司网络上有一个静态 IP 地址 192.168.1.50，而您带着笔记本电脑去家里或其他远程地点工作，那么您家里必须使用与您的办公室相同的 IP 地址方案。如果是这样，您的笔记本电脑可能与电视、某人的手机或其他支持 IP 的设备的动态分配 IP 地址发生冲突。在家里可能会解决这个问题，但是当您将同一台笔记本电脑带到酒店并尝试连接公司 VPN 时，很可能由于静态分配的 IP 地址而无法连接。酒店可能使用 10.0.1.0 IP 地址方案，因此您的笔记本电脑将永远无法连接。

设备的移动性是 DHCP 的一大优势。用户无论在笔记本电脑连接到另一个网络或互联网的任何地方，都无需重新配置任何内容。另一个优势是，一旦您配置了一个 DHCP 地址池，您就不需要做太多维护工作。您将需要确定哪种租约持续时间最适合您的用户。我见过的租约持续时间范围从 24 小时到 30 天不等。我没有特别偏爱，但如果您使用长租约（超过几天），您可能会偶尔需要通过删除重复、过期的租约等来进行一些清理工作。DHCP 应该在自身之后进行清理，但并非总是如此。

是否使用静态 IP 地址分配、DHCP 或两者混合取决于您个人对 IP 地址管理的偏好。您仍然可以使用纯 DHCP 方案，并通过输入 MAC 地址保留 IP 地址，使其静态分配。唯一的问题是如果您更改了网络接口卡（NIC），您必须记住使用新 NIC 的 MAC 地址更新 DHCP 预留列表。

###### Tip

如果更改网络接口卡（NIC 或适配器）设置，例如从 DHCP 更改为静态 IP 地址，则需要重新启动适配器才能使这些更改生效。通过发出以下命令重新启动适配器：

```
$ sudo ifdown *adapter_name*
$ sudo ifup *adapter_name*
```

适配器名称因系统而异，但一些示例是 `eth0` 和 `enp0s3`。

接下来的部分讨论了将系统放置在网络上的安全相关后果，以及如何在您的系统上应用最佳安全实践。

# 网络和安全

可以认为两种类型的 Linux 系统是安全的：一个是关闭电源的系统，另一个是开机但未连接到网络的系统。关闭电源的系统免受网络攻击，非网络化系统同样如此。关闭电源的系统唯一的安全漏洞是物理安全性。有人可以物理访问系统，窃取、拆解或损坏它。虽然非网络化系统在本章开头提到过，仍然有一定的用处，但除了单个操作员之外的用途有限。

一旦您将系统网络化，就暴露于网络攻击之下。恶意行为者不断扫描 IP 地址范围，寻找可利用的易受攻击系统。尽管许多服务器通过[非军事区（DMZ）](https://oreil.ly/wxezB)暴露在互联网上，或者位于企业或家庭防火墙保护的网络内部，它们仍然容易受到攻击。一旦进入您的网络，恶意行为者可以对所有连接的系统执行自动扫描，寻找漏洞。

此外，在您的系统上创建用户账户会降低安全性，因为密码弱、存在路径攻击的潜在风险，以及社会工程学攻击可能会向恶意行为者泄露其凭据。

由于这些原因，管理员必须采取以下措施：

+   仅授予用户必要的工作权限（最小特权原则）

+   强制执行强密码、密钥或多因素认证的安全策略

+   定期打补丁和更新系统

+   对所有系统、网络设备和访问企业资源的设备定期进行安全审计

## 为网络连接准备系统

当系统管理员为新系统提供配置并将其安装到机架中时，插入网络电缆或将虚拟机连接到虚拟网络是标准做法。我们经常依赖他人来审查系统的功能、目的和安全选项。然而，并非总是如此。新系统安装通常是通过从准备好的操作系统镜像中提供“标准构建”自动执行的，这些镜像可能已有几个月或几年之久。使用旧镜像是一种糟糕的安全实践。如果在系统完全更新和安全保护之前立即将其连接到网络，则在正式开始在网络上执行常规任务之前，它可能会容易受到攻击和妥协。

一个解决方案是在私有网络上为新系统提供配置，这样它们可以从内部仓库接收更新、补丁和安全配置，然后再放入生产网络中。

## 修剪您的系统

精简意味着从系统中删除任何不必要的服务和守护程序。没有必要通过运行生产系统中没有人使用但可能使系统容易受攻击的多个服务来给自己制造问题。只安装您需要为用户或其他系统提供服务的内容。

至少，您需要在系统上运行 SSH 守护程序，以便远程登录和管理系统。如果发现某些用户需要的特殊服务仅偶尔使用，或者使系统处于不太安全状态，则在需要时打开服务，不再使用时关闭，或将服务放置在只能从受限制的系统访问的安全网络上。

## 安全的网络守护程序

确定安装和支持哪些网络守护程序通常很容易，因为每次构建系统时您都知道系统的预期用途。如果是 Web 服务器，您知道会安装诸如 Apache 或 NGINX 之类的 Web 服务。如果系统是数据库服务器，则会安装 MySQL、MariaDB 或其他一些数据库软件。问题在于，为了使系统有用，必须暴露其服务的相应 TCP 端口。这些网络守护程序容易受到攻击，因此必须加以保护。

保护网络守护程序和服务有多种方法，但安装所需提供的服务的安全版本是最简单的方法。例如，如果您的新系统是 DNS 服务器，请使用 DNSSEC。如果配置轻量级目录访问协议（LDAP）服务器，请使用 LDAPS。对于通过证书保护的 Web 服务器，请始终使用 HTTPS。表格 5-1 显示了安全服务的部分列表。

表格 5-1\. 安全服务示例

| Protocol | Port | 描述 |
| --- | --- | --- |
| https | 443/tcp | TLS/SSL 上的 HTTP 协议 |
| https | 443/udp | TLS/SSL 上的 HTTP 协议 |
| ldaps | 636/tcp | SSL 上的 LDAP |
| ldaps | 636/udp | SSL 上的 LDAP |
| imaps | 993/tcp | SSL 上的 IMAP |
| imaps | 993/udp | SSL 上的 IMAP |
| pop3s | 995/tcp | SSL 上的 POP-3 |
| pop3s | 995/udp | SSL 上的 POP-3 |

使用安全协议和加密并不能保证安全性，但比起使用没有加密的非安全协议要好得多。即使是安全的应用程序和协议也经常会出现漏洞。保持系统更新和打补丁有助于防止安全漏洞。安全补丁通常在造成广泛破坏之前就可用，但并非总是如此，因此您必须保持警惕以维护安全性。

## 安全外壳守护程序

几乎每个 Linux 系统上都有的最常见的网络守护程序是安全外壳（SSH）守护程序。SSH 通过网络为 Linux 系统提供安全（加密）连接。尽管 SSH 守护程序（SSHD）通过其加密通道具有内置安全性，但其通信仍然容易受到攻击。有多种方法可以增强 SSH 的安全性，以减少攻击者的成功几率。

### 限制从特定主机访问 SSHD

管理员可以使用两个文件来限制对任何守护程序的访问：*/etc/hosts.allow*和*/etc/hosts.deny*。它们不需要服务重启，因为它们不是配置文件，系统在每次客户端访问服务时检查它们。*/etc/hosts.allow*文件比*/etc/hosts.deny*文件更重要，因为其设置会覆盖后者的设置。

显示了一个允许从单个 IP 地址（192.168.1.50）启用 SSH 连接的*/etc/hosts.allow*文件条目：

```
sshd: 192.168.1.50
sshd: ALL: DENY
```

此条目仅接受来自 192.168.1.50 的 SSH 连接，并拒绝来自所有其他 IP 地址的连接。如果您发现在*/etc/hosts.allow*中设置这样的条目对您无效，则需要使用以下命令检查`sshd`中的`tcp_wrapper`集成：

```
$ sudo ldd /path/to/binary | grep libwrap
```

如果未收到响应，则您的`sshd`未启用`tcp_wrappers`，并且可能已被您的发行版淘汰。很遗憾，这在许多打包的`sshd`安装中是这样的。您希望看到的响应类似于以下内容：

```
$ sudo ldd /usr/sbin/sshd | grep libwrap
        libwrap.so.0 => /lib/x86_64-linux-gnu/libwrap.so.0 (0x00007fc6c2ab2000)
```

如果您的`sshd`不支持`tcp_wrappers`，您有三个选项：

+   使用其他方法保护`sshd`（`firewall`规则，`iptables`，`nftables`）。

+   使用启用了`tcp_wrappers`的`openssh_server`进行编译。

+   查找并替换您当前的`sshd`为已启用`tcp_wrappers`的`openssh_server`软件包。

为了为`firewalld`、`iptables`和`nftables`提供选项，请考虑执行以下命令，这些命令执行与将条目添加到*/etc/hosts.allow*和*/etc/hosts.deny*相似的功能。

### 实施`firewalld`规则

如果您使用`firewalld`，首先从`firewalld`的规则中删除`ssh`服务：

```
$ sudo firewall-cmd --permanent --remove-service=ssh
```

添加一个新的`zone`，而不是使用默认的`zone`：

```
$ sudo firewall-cmd --permanent --new-zone=SSH_zone
$ sudo firewall-cmd --permanent --zone=SSH_zone --add-source=192.168.1.50
$ sudo firewall-cmd --permanent --zone=SSH_zone --add-service=ssh
```

必须重新加载`firewall`以使新配置生效：

```
$ sudo firewall-cmd --reload
```

如果您使用`iptables`，可以使用单个命令将`ssh`访问限制为单个 IP 地址：

```
$ sudo iptables -A INPUT -p tcp -s 192.168.1.50 --dport 22 -j ACCEPT
```

对于 netfilter（`nft`），您可以添加一个新的`rule`：

```
$ sudo nft insert rule ip filter input ip saddr 192.168.1.50 tcp dport 22 accept
```

但是，如果您收到以下错误消息，则必须为规则创建新表和链，或使用现有链：

```
Error: Could not process rule: No such file or directory.
```

使用以下命令创建一个名为`input`的新表和链：

```
$ sudo nft add table ip filter # create table

# next, create chain
$ sudo nft add chain ip filter input { type filter hook input priority 0\; }

$ sudo nft insert rule ip filter input ip saddr 192.168.1.50 tcp dport 22 accept
```

注意链名区分大小写。例如，如果您使用了现有的输入链`INPUT`，您将不会收到错误：

```
$ sudo nft insert rule ip filter INPUT ip saddr 192.168.1.50 tcp dport 22 accept
```

在进行更改后重新启动`nftables`：

```
$ sudo systemctl restart nftables
```

`nftables` 系统取代了 `iptables`，并将 `iptables`、`ip6tables`、`arptables` 和 `ebtables` 的功能合并到一个单一的实用程序中。您可以在 [netfilter 主页](https://oreil.ly/1RXmC) 上了解更多关于 `nftables` 的信息。

您现在有多种方法可以限制 SSH 守护进程从随机主机的访问。如果不想单独指定特定 IP 地址，您可以使用 192.168.1.0/24 代替单个 IP 地址来隔离目标系统。

###### 注意

注意，如果从整个子网打开访问权限，仍可能使您的系统处于显著的危险中，如果入侵者渗入您的网络。理想情况下，您应该将访问限制在一到两台主机上，以便日志和监控系统更容易检测到入侵。

您还可以通过修改 */etc/ssh/sshd_config* 文件来配置 SSH 守护进程，限制对特定用户的访问。您必须阻止根用户使用 SSH。您将在下一节学习如何防止根用户通过 SSH 访问。

### 拒绝根用户通过 SSH 访问

安装后，您应该禁止根用户通过 SSH 访问。一些 Linux 系统默认禁止根用户通过 SSH 登录，而其他系统允许此操作。根用户绝不能通过 SSH 登录任何系统。普通用户应通过 SSH 登录，然后成为根用户或使用 `sudo` 命令以根用户身份执行任务。

您需要检查 */etc/ssh/sshd_config* 中的以下行：

```
PermitRootLogin yes
```

将 `yes` 改为 `no`，然后重新启动 SSH 服务：

```
$ sudo systemctl restart sshd
```

根用户不能通过 SSH 登录。根用户可以直接登录控制台。

### 使用密钥而不是密码进行身份验证

密码认证是验证用户身份最不安全的方法。使用密钥文件更安全和高效。您必须对 */etc/ssh/sshd_config* 文件进行以下更改，并重新启动 `sshd` 以使新配置生效：

```
PasswordAuthentication yes
```

将 `yes` 改为 `no`，然后查找以下两个条目，确保它们被取消注释并设置如下所示：

```
PubkeyAuthentication yes
AuthorizedKeysFile .ssh/authorized_keys
```

重新启动 `sshd` 以启用新的设置：

```
$ sudo systemctl restart sshd
```

在客户端（本地系统）上，用户需要执行以下操作来设置密钥对认证。本示例适用于用户 `tux`，远程目标系统为 192.168.1.99：

```
$ ssh-keygen -t rsa

Generating public/private rsa key pair.
Enter file in which to save the key (/home/tux/.ssh/id_rsa):
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in /home/tux/.ssh/id_rsa.
Your public key has been saved in /home/tux/.ssh/id_rsa.pub.
The key fingerprint is:
SHA256:NVweugZXvDitzl0JGypcWTJOww/F54vmHUbX7r4U2LQ tux@server1
The key's randomart image is:
+---[RSA 2048]----+
|         . +=    |
|         .B=+..  |
|        .o*@.+ ..|
|         +*o* * +|
|       .S.o+ B E |
|        o.o + * o|
|         + + + + |
|          o o o .|
|               oo|
+----[SHA256]-----+
```

将生成的密钥复制到远程主机：

```
$ ssh-copy-id tux@192.168.1.99
/usr/bin/ssh-copy-id: INFO: Source of key(s) to be installed: "/home/tux/...
/usr/bin/ssh-copy-id: INFO: attempting to log in with the new key(s), to f...
/usr/bin/ssh-copy-id: INFO: 1 key(s) remain to be installed -- if you are ...
tux@192.168.1.99's password:

Number of key(s) added:        1
```

现在尝试使用 `ssh tux@192.168.1.99` 登录目标系统，并确保只有您想要的密钥存在于目标系统上：

```
$ ssh tux@192.168.1.99

Last login: Sun Sep 26 13:48:19 2021 from 192.168.0.10
[tux@server1 ~]$
```

用户 `tux` 已成功使用安全密钥对而非密码登录到远程主机 `server1` (192.168.1.99)。

### 远程连接：客户端到服务器

当连接到安全协议服务时，您的客户端软件与服务守护进程在安全通道上通信。您无需让用户执行任何特殊操作来在客户端和服务守护进程之间协商安全通信链接。

让客户端软件保持更新就像让您作为管理员更新服务器软件一样重要。我建议您在每个用户系统上设置`cron`任务来自动下载和安装更新，并配置每个新系统以接收定期更新，无需用户交互。请确保将任何必需的重新启动安排在夜间或用户系统处于空闲状态时进行。

# 总结

在本章中，您学习了为您的网络选择 IP 地址方案以及静态和动态选项的一些优缺点。您现在还了解了将系统连接到您的网络的安全影响。您应该了解到安全漏洞的危险以及如何尽可能通过实施一些良好的安全实践来预防安全漏洞，例如使用安全协议、关闭不必要的服务和守护程序，并保持系统的补丁和更新。

在下一章中，您将学习如何通过软件包管理器安装软件，更新系统，并从源代码安装软件。