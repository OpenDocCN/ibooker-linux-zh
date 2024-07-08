# 第十三章：安全远程访问与 OpenVPN

Open Virtual Private Network（OpenVPN）在不同物理位置的两个不同网络之间创建 TLS/SSL 加密连接，例如将分支办公室连接到总部，或远程工作者从家中登录公司网络。这种连接称为加密 *隧道*，是一种安全的传输方式，保护您免受互联网的威胁。OpenVPN 依赖于 OpenSSL，因此具备 OpenSSL 知识将会有所帮助。

###### 注意

如果您已经熟悉 OpenVPN，可以直接跳到配方 13.5、13.6 和 13.7 查看如何创建您的加密证书以及客户端和服务器配置。如果您对 VPN 还不熟悉，请按顺序尝试每个配方。耐心点，VPN 很复杂也很挑剔。在投入生产系统之前进行大量测试。

# OpenVPN 概述

VPN 是网络的安全扩展，使远程工作者可以访问与本地用户相同的所有服务，因此远程用户的体验与实际位于您位置的用户相同。他们可以访问您的本地 Web 服务器、电子邮件、文件共享、聊天服务器、视频会议应用程序、内部维基等所有对外界隔离的服务，这些服务仅供内部网络用户使用。VPN 不同于连接个人计算机的 SSH。VPN 连接的是网络和个别主机。

在本章中，您将学习如何设置 OpenVPN 服务器，配置客户端，并创建和管理适当的公钥基础设施（PKI）进行身份验证和加密。您的服务器将验证和保护各种客户端：Linux、macOS、Windows PC、Android 和 iOS 设备。

OpenVPN 是一个开源项目，提供免费下载和商业选项。免费的服务器和客户端是 *openvpn* 软件包，适用于所有 Linux 发行版，并可以从 [OpenVPN 社区下载](https://oreil.ly/vwEAs)。商业选项包括 OpenVPN Access Server，这是一个带有额外管理工具和云选项的本地服务器。托管的个人计划只需安装客户端，即可访问全球的 OpenVPN 服务器网络。

真正的 VPN 很强大，因为它不信任任何人，并要求经过身份验证的端点，在这里服务器和客户端相互验证。大多数商业 TLS/SSL VPN 并不这样做，而是信任所有客户端，就像购物网站一样。这样做更加灵活，允许用户从任何地方使用任何设备登录。这样做省去了安装和配置客户端软件以及复制加密密钥的麻烦。但是对于您的内部网络来说，这种做法是短视的——最后一件您需要的事情是用户从带有键盘记录器和间谍软件的随机 PC 或智能手机登录，然后被热情地欢迎进入您的局域网。

## 证书颁发机构

证书颁发机构（CA）是运行 OpenVPN 服务器的最重要部分。CA 颁发数字证书并认证公钥的所有权。在网页浏览器中点击小锁图标可以查看网站的公共证书及其由哪个 CA 签署。CA 是一个受信任的机构，这就是为什么很多网站使用商业 CA 的原因。像我们在本章创建的自签名证书可以在你的组织内部使用。面向客户的站点应使用商业 CA。使用 CA 可以避免在 OpenVPN 服务器上保存客户端证书的麻烦；服务器只需知道客户端证书由你的 CA 认证即可。

## SSL 与 TLS

安全套接字层（SSL）和传输层安全性（TLS）是加密协议。TLS 是从 SSL 进化而来的。所有版本的 SSL 都已被弃用，包括 TLS 1.0 和 TLS 1.1。请使用 TLS 1.2 或 1.3，并禁用所有其他版本（配方 13.10）。由于安全漏洞，旧版本已被弃用，请不要让任何人说服你支持弃用的版本。

## TUN/TAP

*TUN*和*TAP*设备是虚拟网络接口。这些已经集成到 Linux 内核中，你无需进行任何操作即可使用。*TUN*设备用于路由网络，*TAP*设备用于桥接网络。你的服务器和客户端配置文件会指定使用哪种设备。

# 好的安全措施需要工作。

良好的安全性需要持续学习和维护。本章旨在向你展示如何设置一个强大且相对用户友好的 VPN。还有许多其他方法可以使你的 VPN 更加安全，如使用短寿命的客户端证书、额外的认证、硬件设备、SELinux、chroot 监狱、短密码超时等等。如果你需要超高安全性，请咨询专业的专家。

# 13.1 安装 OpenVPN，服务器和客户端

## 问题

你需要知道如何安装 openVPN。

## 解决方案

[OpenVPN 网站](https://openvpn.net)提供了社区开源的 OpenVPN 和商业 OpenVPN Access Server。社区版 OpenVPN 免费且开源。本章介绍了社区版 OpenVPN。

在 Linux 上安装*openvpn*软件包。（像往常一样，请验证你特定 Linux 发行版上的软件包名称。）尽可能获取最新版本，从 2.4.5 版本开始。这提供了服务器和客户端。源代码 tarball 和 Windows 安装程序可以从[OpenVPN 社区下载](https://oreil.ly/vwEAs)获取。

对于你的客户端，你可以尝试免费的[OpenVPN Access Clients](https://oreil.ly/vQugl)，适用于 Linux、macOS、Android、iOS 和 Windows。这些客户端设计用于商业 OpenVPN Access Server，也与社区版 OpenVPN 服务器兼容。

你也可以在 Google Play 商店找到安卓的 OpenVPN 社区客户端。

参见 Recipe 13.9 了解如何使用 *.ovpn* 内联文件格式来更轻松地配置客户端。

## 讨论

在 Linux 上，OpenVPN 必须安装在您的 OpenVPN 服务器和所有客户端上。OpenVPN 软件包提供客户端和服务器功能。

Ubuntu、Fedora 和 openSUSE 包含额外的软件包，提供与 NetworkManager 的集成，这使得管理、连接和断开 VPN 变得非常简便。

*NetworkManager-openvpn* (Fedora、openSUSE) 和 *network-manager-openvpn* (Ubuntu) 将 OpenVPN 与 Network Manager 集成。如果您使用 GNOME 环境（如 GNOME、Xfce、Cinnamon 或 Mate），您还需要 *NetworkManager-openvpn-gnome* (openSUSE、Fedora)，或者 *network-manager-openvpn* (Ubuntu)。

OpenVPN Access Server 是免费下载的，您可以同时连接最多两个客户端而无需购买许可证。它带有额外的功能，如 Web 管理界面和与免费 OpenVPN Access Client 的自动配置。如果您从社区版 OpenVPN 开始，然后决定迁移到 OpenVPN Access Server，那么您在社区版 OpenVPN 服务器上学到的所有内容也适用于 Access Server。

## 参见

+   [EasyRSA](https://oreil.ly/eKbsg)

+   [OpenVPN 文档](https://oreil.ly/Ah124)

+   *man 8 openvpn*

+   [OpenSSL Cookbook](https://oreil.ly/Ctm0X)

# 13.2 设置一个简单的连接测试

## 问题

您希望运行最简单的 OpenVPN 连接测试，以了解其工作原理并验证连接性。

## 解决方案

以下简单测试创建了两台位于同一网络上的 Linux 计算机之间的未加密隧道。这两台计算机都必须安装 OpenVPN。首先确保 OpenVPN 守护进程在任一主机上都没有运行，如果运行了，请停止它：

```
$ systemctl status *openvpn@.openvpn1.service*
● openvpn.service - OpenVPN service
     Loaded: loaded (/lib/systemd/system/openvpn.service; enabled; vendor prese>
     Active: active (exited) since Sun 2021-01-10 13:43:18 PST; 33min ago
[...]
$ sudo systemctl stop *openvpn@.openvpn1.service*
```

# 使用不同的子网用于您的 VPN

使用不同的子网用于您的 OpenVPN 隧道；例如，*host1* 和 *host2* 在 192.168.43.0/24 上，因此示例使用 10.0.0.0/24 私有地址空间作为 VPN 隧道。

在以下示例中，两台计算机分别命名为 *host1* 和 *host2*。第一个示例创建了从 *host1* 到 *host2* 的 VPN 隧道：

```
[madmax@host1 ~]$ sudo openvpn  --remote *host2* --dev tun0 --ifconfig *10.0.0.1 \
10.0.0.2*
Sat Jan  9 14:40:34 2021 disabling NCP mode (--ncp-disable) because not in P2MP
  client or server mode
Sat Jan  9 14:40:34 2021 OpenVPN 2.4.8 x86_64-redhat-linux-gnu [SSL (OpenSSL)]
  [LZO] [LZ4] [EPOLL] [PKCS11] [MH/PKTINFO] [AEAD] built on Jan 29 2020
Sat Jan  9 14:40:34 2021 library versions: OpenSSL 1.1.1d FIPS  10 Sep 2019,
  LZO 2.10
Sat Jan  9 14:40:34 2021 ******* WARNING *******: All encryption and
  authentication features disabled -- All data will be tunnelled as clear text
  and will not be protected against man-in-the-middle changes. PLEASE DO
  RECONSIDER THIS CONFIGURATION!
Sat Jan  9 14:40:34 2021 TUN/TAP device tun0 opened
Sat Jan  9 14:40:34 2021 /sbin/ip link set dev tun0 up mtu 1500
Sat Jan  9 14:40:34 2021 /sbin/ip addr add dev tun0 local 10.0.0.1 peer 10.0.0.2
Sat Jan  9 14:40:34 2021 TCP/UDP: Preserving recently used remote address:
  [AF_INET]192.168.122.239:1194
Sat Jan  9 14:40:34 2021 UDP link local (bound): [AF_INET][undef]:1194
Sat Jan  9 14:40:34 2021 UDP link remote: [AF_INET]192.168.122.239:1194
```

这个示例创建了从 *host2* 到 *host1* 的链接：

```
[stash@host2 ~]$ sudo openvpn --remote *host1* --dev tun0 --ifconfig *10.0.0.2 \
10.0.0.1*
Sat Jan  9 14:50:53 2021 disabling NCP mode (--ncp-disable) because not in P2MP
  client or server mode
Sat Jan  9 14:50:53 2021 OpenVPN 2.4.7 x86_64-pc-linux-gnu [SSL (OpenSSL)] [LZO]
  [LZ4] [EPOLL] [PKCS11] [MH/PKTINFO] [AEAD] built on Sep  5 2019
Sat Jan  9 14:50:53 2021 library versions: OpenSSL 1.1.1f  31 Mar 2020, LZO 2.10
Sat Jan  9 14:50:53 2021 ******* WARNING *******: All encryption and
  authentication features disabled -- All data will be tunnelled as clear text
  and will not be protected against man-in-the-middle changes. PLEASE DO
  RECONSIDER THIS CONFIGURATION!
Sat Jan  9 14:50:53 2021 TUN/TAP device tun0 opened
Sat Jan  9 14:50:53 2021 /sbin/ip link set dev tun0 up mtu 1500
Sat Jan  9 14:50:53 2021 /sbin/ip addr add dev tun0 local 10.0.0.2 peer 10.0.0.1
Sat Jan  9 14:50:53 2021 TCP/UDP: Preserving recently used remote address:
  [AF_INET]192.168.122.52:1194
Sat Jan  9 14:50:53 2021 UDP link local (bound): [AF_INET][undef]:1194
Sat Jan  9 14:50:53 2021 UDP link remote: [AF_INET]192.168.122.52:1194
Sat Jan  9 14:51:03 2021 Peer Connection Initiated with
  [AF_INET]192.168.122.52:1194
Sat Jan  9 14:51:04 2021 WARNING: this configuration may cache passwords in
  memory -- use the auth-nocache option to prevent this
Sat Jan  9 14:51:04 2021 Initialization Sequence Completed
```

当两台主机显示“Initialization Sequence Completed”消息时，您就建立了成功的连接。通过在两台主机上的 *tun0* 接口进行 ping 测试您的连接：

```
[madmax@host1 ~]$ ping -I tun0 *10.0.0.2*
PING 10.0.0.2 (10.0.0.2) from 10.0.0.1 tun0: 56(84) bytes of data.
64 bytes from 10.0.0.2: icmp_seq=1 ttl=64 time=0.515 ms
64 bytes from 10.0.0.2: icmp_seq=2 ttl=64 time=0.436 ms

[stash@host2 ~]$ ping -I tun0 *10.0.0.1*
PING 10.0.0.1 (10.0.0.1) from 10.0.0.2 tun0: 56(84) bytes of data.
64 bytes from 10.0.0.1: icmp_seq=1 ttl=64 time=0.592 ms
64 bytes from 10.0.0.1: icmp_seq=2 ttl=64 time=0.534 ms

```

在每台主机上按 Ctrl-C 停止 ping，并再次关闭隧道。

## 讨论

这个简单测试展示了 OpenVPN 的工作原理。它在两台主机上创建了虚拟网络接口 *tun0*，然后通过该接口路由网络流量。这个简单测试不会创建加密连接，您可以从命令输出中的“******* 警告 *******: 所有加密和认证功能已禁用”消息中看到。

## 参见

+   [EasyRSA](https://oreil.ly/eKbsg)

+   [OpenVPN 文档](https://oreil.ly/Ah124)

+   [systemd.unit](https://oreil.ly/2AAEe)

+   *man 8 openvpn*

+   [OpenSSL Cookbook](https://oreil.ly/Ctm0X)

# 13.3 设置 Easy Encryption with Static Keys

## 问题

您希望为 OpenVPN 创建和管理加密的简便方法。

## 解决方案

最简单的方法是使用共享的静态密钥。共享的静态密钥对于测试很有用，但不适合生产系统。（有关它们的缺点，请参阅讨论。）在这个示例中，您将学习如何创建和共享静态密钥，以及如何创建简单的服务器和客户端配置文件。

按照以下步骤操作：

1.  创建并在两台主机之间分发共享的静态密钥。

1.  创建服务器和客户端配置文件。

1.  在两台主机上启动 OpenVPN，引用其配置文件。

在以下示例中，OpenVPN 服务器在*server1*上，客户端在*client1*上，新密钥为*myvpn.key*。您可以按照自己的喜好命名您的密钥。

在 OpenVPN 服务器上创建一个新目录来存储密钥，然后创建一个新的静态密钥：

```
$ sudo mkdir */etc/openvpn/keys*
$ sudo openvpn --genkey --secret *myvpn.key*
```

将密钥复制到客户端机器：

```
$ scp *myvpn.key* *client1:/etc/openvpn/keys/*
Password:
myvpn.key                         100%  636   142.7KB/s   00:00

```

创建服务器配置文件。示例为*/etc/openvpn/server1.conf*，您可以根据需要命名。为您的 OpenVPN 隧道使用不同的子网；例如，*server1*和*client1*使用 192.168.43.0/24，因此示例使用 10.0.0.0/24 私有地址空间用于 VPN 隧道。服务器的*tun*地址为 10.0.0.1：

```
# server1.conf
dev tun
ifconfig *10.0.0.1 10.0.0.2*
secret */etc/openvpn/keys/myvpn.key*
local *192.168.43.184*
```

*local*是服务器的局域网 IP 地址。

在客户端机器上创建客户端配置文件。客户端的*tun*地址为 10.0.0.2：

```
# client1.conf
dev tun
ifconfig *10.0.0.2 10.0.0.1*
secret */etc/openvpn/keys/myvpn.key*
remote 192.168.43.184
```

确保 OpenVPN 守护程序在服务器或客户端上未运行：

```
$ sudo systemctl stop openvpn
```

在服务器和客户端上启动 OpenVPN：

```
[server1 ~] $ sudo openvpn */etc/openvpn/server1.conf*

```

```
[client1 ~] $ sudo openvpn */etc/openvpn/client1.conf*

```

当您在两台主机上看到“Initialization Sequence Completed”时，表示已建立连接。通过*tun*虚拟网络接口对每个主机进行 ping 测试：

```
[server1 ~] $ ping -I tun0 *10.0.0.1*
[client1 ~] $ ping -I tun0 *10.0.0.2*
```

在两台主机上按 Ctrl-C 关闭连接。

## 讨论

如果您在命令输出中看到“WARNING: INSECURE cipher with block size less than 128 bit (64 bit). This allows attacks like SWEET32\. Mitigate by using a cipher with a larger block size (e.g. AES-256-CBC)”，请在服务器和客户端配置文件中进行以下条目更正：

```
cipher AES-256-CBC
```

使用静态密钥的最大问题是您失去了完美的前向保密性，因为您的静态密钥从不更改。如果攻击者找到方法来嗅探和捕获您的网络流量，然后捕获并破解您的加密密钥，攻击者可以解密他们捕获的所有内容，包括过去和未来的内容。OpenVPN 的 PKI 使用复杂的过程生成会话密钥，这些密钥不是持久的，而是定期更改。因此，最多，成功的攻击者一次只能解密一个会话的流量，然后必须重新开始。

另一个缺点是每个客户端需要不同的密钥，并在服务器上复制每个客户端的密钥。通过正确的 PKI，管理多个客户端的工作量更少且更安全。

## 参见

+   [EasyRSA](https://oreil.ly/eKbsg)

+   [OpenVPN 文档](https://oreil.ly/Ah124)

+   [systemd.unit](https://oreil.ly/2AAEe)

+   *man 8 openvpn*

+   [OpenSSL Cookbook](https://oreil.ly/Ctm0X)

# 13.4 安装 EasyRSA 来管理你的 PKI

## 问题

你将使用 EasyRSA 来创建和管理你的公钥基础设施（PKI），并希望正确安装和设置它。

## 解决方案

你的 PKI 可以放在任何地方，不一定要在 OpenVPN 服务器上。你将在 PKI 上创建服务器和客户端证书，然后将它们复制到各自的主机上。

你可以安装 *easy-rsa* 软件包，或者从 [EasyRSA 发布页面](https://oreil.ly/LtAKu) 获取最新版本。

Fedora 和 Ubuntu 将所有 EasyRSA 文件都放在 */usr/share/*。这不是一个好的工作目录，并且会被系统更新覆盖。创建一个你控制且不需要 root 权限的新目录，比如我们优秀的示例用户 Duchess 创建的 */home/duchess/mypki*：

```
~$ mkdir *mypki*
```

在 Fedora 和 Ubuntu Linux 上，复制 */usr/share/easy-rsa* 目录到你的新目录：

```
~$ sudo cp -r /usr/share/easy-rsa *mypki*
```

这将创建 *mypki/easyrsa*。检查你的权限；你应该是目录中所有内容的所有者和组所有者。

openSUSE 通过在 */etc/easy-rsa* 中放置配置文件，将 *easyrsa* 命令放在 */usr/bin*，并在 */usr/share/* 中放置文档和许可文件来进行适当的安装。你无需移动任何文件或担心权限问题。

## 讨论

创建和管理你的 PKI 不需要 root 权限。你可以把它放在任何你想要的地方，应与你的 OpenVPN 配置分开，可以是在单独的目录或另一台机器上。OpenVPN 的开发者建议将其放在一个不会暴露于互联网的、受保护良好的机器上。

## 参见

+   [EasyRSA](https://oreil.ly/eKbsg)

+   [OpenVPN 文档](https://oreil.ly/Ah124)

+   [systemd.unit](https://oreil.ly/2AAEe)

+   *man 8 openvpn*

+   [OpenSSL Cookbook](https://oreil.ly/Ctm0X)

# 13.5 创建 PKI

## 问题

你已经安装了 EasyRSA（13.4 节），现在你想知道如何设置一个合适的公钥基础设施（PKI）。

## 解决方案

在安全地运行 OpenVPN 服务器时，一个合适的 PKI 至关重要。在这个示例中，我们将使用 EasyRSA 来创建 PKI，相比使用 *openssl* 命令，这大大简化了整个过程。创建 PKI 包括以下步骤：

1.  创建你自己的证书颁发机构（CA）证书来签署服务器和客户端证书。这应该放在与你的 OpenVPN 服务器配置分开的目录中，或者在另一台机器上。

1.  创建并签署 OpenVPN 服务器证书。

1.  创建并签署客户端证书。

1.  将服务器证书和客户端证书复制到各自机器上的 */etc/openvpn/keys* 目录。（你可以创建一个不同于 *keys* 的目录。）

在以下示例中，所有命令均从 */home/duchess/mypki/* 目录运行。

切换到你的 PKI 目录，并运行命令来初始化一个新的 PKI：

```
~$ cd *mypki*
~/mypki $ easyrsa init-pki

init-pki complete; you may now create a CA or requests.
Your newly created PKI dir is: /home/duchess/mypki/pki

```

这将为您的新 PKI 创建一个空结构。接下来，构建您的新 CA。CA 创建并签署服务器和客户端证书。用强密码保护它，并创建您想要的新 CA 的通用名称：

```
~/mypki $ easyrsa build-ca
[...]
Enter New CA Key Passphrase:*passphrase*
Re-Enter New CA Key Passphrase:*passphrase*
[...]
Common Name (eg: your user, host, or server name) [Easy-RSA CA]: *vpnserver1*
[...]
CA creation complete and you may now import and sign cert requests.
Your new CA certificate file for publishing is at:
/home/duchess/mypki/pki/ca.crt
```

###### 提示

如果您看到“RAND_load_file:Cannot open file:crypto/rand/randfile.c:98:Filename=/mypki/pki/.rnd”消息，请忽略它，因为它毫无意义。您可以通过找到 *openssl-easyrsa.cnf* 并在开头注释掉 *RANDFILE* 行来解决这个问题。

为您的 OpenVPN 服务器生成密钥对和证书签名请求。习惯上，不在服务器的私钥上设置密码。如果您希望可以省略 *nopass* 选项来保护您的服务器，请用密码保护。密码提供了强大的保护，但这意味着每次重启服务器时都需要输入密码：

```
~/mypki $ easyrsa gen-req *vpnserver1* nopass

Using SSL: openssl OpenSSL 1.1.1d  10 Sep 2019
Generating a RSA private key
.............................+++++
................................................................++++++
writing new private key to '/home/duchess/mypki/pki/private/vpnserver1.key.NYjr5y
c9kj'
[...]
Common Name (eg: your user, host, or server name) [*vpnserver1*]:

Keypair and certificate request completed. Your files are:
req: /home/duchess/mypki/pki/reqs/vpnserver1.req
key: /home/duchess/mypki/pki/private/vpnserver1.key
```

为客户端生成密钥对和证书签名请求。尤其是移动客户端的客户端私钥应该有密码保护：

```
~/mypki $ easyrsa gen-req *vpnclient1*

Using SSL: openssl OpenSSL 1.1.1d  10 Sep 2019
Generating a RSA private key
................+++++
....................................................................+++++
writing new private key to '/home/duchess/mypki/pki/private/vpnclient1.key.bicpOc
EC5S'
Enter PEM pass phrase:*passphrase*
Verifying - Enter PEM pass phrase:*passphrase*
[...]
Common Name (eg: your user, host, or server name) [*vpnclient1*]:

Keypair and certificate request completed. Your files are:
req: /home/duchess/mypki/pki/reqs/vpnclient1.req
key: /home/duchess/mypki/pki/private/vpnclient1.key

```

使用它们的通用名称签署请求。仅使用它们的名称；如果输入它们的路径会导致错误：

```
~/mypki $ easyrsa sign-req server *vpnserver1*
Using SSL: openssl OpenSSL 1.1.1d  10 Sep 2019

You are about to sign the following certificate.
Please check over the details shown below for accuracy. Note that this request
has not been cryptographically verified. Please be sure it came from a trusted
source or that you have verified the request checksum with the sender.

Request subject, to be signed as a server certificate for 1080 days:

subject=
    commonName                = vpnserver1

Type the word 'yes' to continue, or any other input to abort.
  Confirm request details: *yes*
Using configuration from /home/duchess/mypki/pki/safessl-easyrsa.cnf
Enter pass phrase for /home/duchess/mypki/pki/private/ca.key:
Check that the request matches the signature
Signature ok
The Subject's Distinguished Name is as follows
commonName            :ASN.1 12:'vpnserver1'
Certificate is to be certified until Jan 27 20:09:12 2024 GMT (1080 days)

Write out database with 1 new entries
Data Base Updated

Certificate created at: /home/duchess/mypki/pki/issued/vpnserver1.crt

mypki $ easyrsa sign-req client *vpnclient1*
[...]
Certificate created at: /home/duchess/mypki/pki/issued/vpnclient1.crt
```

为服务器生成 Diffie-Hellman 参数；这需要一到两分钟。此命令必须在您的 OpenVPN 服务器上运行：

```
$ easyrsa gen-dh
Using SSL: openssl OpenSSL 1.1.1d  10 Sep 2019
Generating DH parameters, 2048 bit long safe prime, generator 2
This is going to take a long time
...........................................+...........
..........+............................................
[...]
DH parameters of size 2048 created at /home/duchess/mypki/pki/dh.pem

```

在您的服务器上也创建一个基于哈希的消息认证码（HMAC）密钥：

```
$ openvpn --genkey --secret ta.key
```

将 *vpnclient1.key*、*vpnclient1.crt*、*ca.crt* 和 *ta.key* 复制到 *client1* 上的 */etc/openvpn/keys*。

将 *vpnserver1.key*、*vpnserver1.crt*、*ca.crt*、*dh.pem* 和 *ta.key* 复制到 *server1* 上的 */etc/openvpn/keys*。

当您签署了证书签名请求后，您可以删除所有的 **.req* 文件。

表 13-1 应该帮助您记住文件的存放位置。

表 13-1\. 服务器和客户端密钥位置

| 名称 | 位置 | 公共 | 私有 |
| --- | --- | --- | --- |
| ca.crt | 服务器和客户端 | X |  |
| ca.key | PKI 机器 |  | X |
| ta.key | 服务器和客户端 |  | X |
| dh.pem | 服务器 | X |  |
| server.crt | server | X |  |
| server.key | server |  | X |
| client1.crt | client1 | X |  |
| client1.key | client1 |  | X |
| client2.crt | client2 | X |  |
| client2.key | client2 |  | X |

## 讨论

什么是 Diffie-Hellman ？它是一种加密机制，允许两台主机创建和共享一个秘密密钥。一旦 OpenVPN 客户端和服务器相互验证，将生成额外的发送和接收密钥以加密会话。

HMAC 计算消息认证码。HMAC 验证消息的完整性和真实性。

*easyrsa init-pki* 创建一个新的 PKI，您还可以使用它来干净地删除和重建现有的 PKI。

您可以在任何地方设置您的 PKI，而 OpenVPN 的专家建议将其放在不直接连接到互联网且受到良好保护的机器上，防止任何不应该操作您的 PKI 的人员进行入侵。如果您的 CA 受到攻击，攻击者可以轻松渗透到您的网络中。显然，您必须有一种安全的方法来分发这些文件：USB 棒、*scp* 命令、从安全服务器下载的加密压缩包或通过电子邮件发送给用户。

查看您的 PKI 目录，了解所有这些项目的组织方式。

```
~/mypki $ ls */*
pki/ca.crt               pki/index.txt            pki/index.txt.old
pki/serial               pki/dh.pem               pki/index.txt.attr
pki/openssl-easyrsa.cnf  pki/serial.old           pki/extensions.temp
pki/index.txt.attr.old   pki/safessl-easyrsa.cnf  pki/ta.key

pki/certs_by_serial:
4954C26DB44106B20F1B9DA17CE515E5.pem  DA68CBE53E30923C9BCC3B9F1C5C9011.pem

pki/issued:
vpnclient1.crt  vpnserver1.crt

pki/private:
ca.key  vpnclient1.key  vpnserver1.key

pki/renewed:
certs_by_serial  private_by_serial  reqs_by_serial

pki/reqs:
vpnclient1.req  vpnserver1.req

pki/revoked:
certs_by_serial  private_by_serial  reqs_by_serial
```

签名请求文件扩展名为 *.req*，公钥为 *.crt*，私钥为 *.key*。密钥始终成对出现，公钥和私钥。

# 公钥加密，私钥解密

公钥加密，私钥解密。私钥必须受到保护，绝不能共享。公钥则是供共享使用。

单击文件管理器中的任何签名证书，以查看类似 Figure 13-1 的内容。这提供了大量信息：签署它的 CA、到期日期、序列号、指纹、签名等。

![查看签名证书](img/lcb2_1301.png)

###### 图 13-1\. 查看签名证书

或使用 *openssl* 命令读取它：

```
$ openssl x509 -noout -text -in *vpnserver1.crt*
```

证书是由 CA 签署的请求，其中包含公钥和 CA 的数字签名。请求包含公钥和对应私钥的数字签名。通过比较它们，您可以看到所有这些内容。

EasyRSA 最初是 OpenVPN 的一部分，后来作为独立项目分离出来。如果您习惯使用 OpenSSL 管理 PKI，您会欣赏 EasyRSA 如何简化该过程。

## 参见

+   [EasyRSA](https://oreil.ly/eKbsg)

+   [OpenVPN 文档](https://oreil.ly/Ah124)

+   *man 8 openvpn*

+   [OpenSSL Cookbook](https://oreil.ly/Ctm0X)

# 13.6 自定义 EasyRSA 默认选项

## 问题

EasyRSA 的默认设置并非您想要的，您想知道如何更改它们。

## 解决方案

查找您的 *vars.example* 文件，这是 EasyRSA 的一部分。将此文件的副本另存为 *vars*，保存在本章示例中的 PKI 目录中（*/home/duchess/mypki/pki/*）。*vars* 文件定义了创建和签署证书的默认设置。

该文件有很好的注释。在 `# DO YOUR EDITS BELOW THIS POINT` 行下进行编辑。所有以 *set_var* 开头的内容均可编辑。取消注释对您更改的所有内容。

例如，默认配置仅使用通用名称，而不是完整的 *org* 配置。以下示例创建了传统的 *org* 配置：

```
set_var EASYRSA_DN      "org"

set_var EASYRSA_REQ_COUNTRY    "*US*"
set_var EASYRSA_REQ_PROVINCE   "*Oregon*"
set_var EASYRSA_REQ_CITY       "*Walla Walla*"
set_var EASYRSA_REQ_ORG        "*MyCo*"
set_var EASYRSA_REQ_EMAIL      "*me@example.com*"
set_var EASYRSA_REQ_OU         "*MyOU*"
```

当您使用 *org* 配置时，请记住在运行 *easyrsa build-ca* 时输入您的通用名称，否则您将被困在默认的 *Easy-RSA CA* 中。

```
Common Name (eg: your user, host, or server name) [Easy-RSA CA]:*myCN*
```

## 讨论

根据您自己的政策和偏好使用 *cn* 或 *org*；这对服务器操作没有影响。

查看 Recipe 13.10 了解如何加固您的服务器。

## 参见

+   [EasyRSA](https://oreil.ly/eKbsg)

+   [OpenVPN 文档](https://oreil.ly/Ah124)

+   *man 8 openvpn*

+   [OpenSSL Cookbook](https://oreil.ly/Ctm0X)

# 13.7 创建和测试服务器和客户端配置

## 问题

现在您已经有了一个完整的 PKI，您想知道如何配置您的 OpenVPN 服务器和客户端。

## 解决方案

在这个示例中，我们将在同一子网上的两台主机 *server1* 和 *client1* 上设置一个简单的测试实例。这是测试服务器配置的一个简单方法，而不必处理路由和越过互联网网关的麻烦。

下面的示例是一个简单的 OpenVPN 服务器配置。请注意，只要在配置文件中正确引用，您可以将服务器密钥存储在服务器的任何位置：

```
# *vpnserver1.conf*
port 1194
proto udp
dev tun
user nobody
group nobody

ca /etc/openvpn/keys/ca.crt
cert /etc/openvpn/keys/*vpnserver1.crt*
key /etc/openvpn/keys/*vpnserver1.key*
dh /etc/openvpn/keys/dh.pem
tls-auth /etc/openvpn/keys/ta.key 0

server *10.10.0.0 255.255.255.0*
ifconfig-pool-persist ipp.txt
keepalive 10 120
persist-key
persist-tun
tls-server
remote-cert-tls client

status openvpn-status.log
verb 4
mute 20
explicit-exit-notify 1
```

一个客户端配置示例：

```
# *vpnclient1.conf*
client
dev tun
proto udp
remote *server1* 1194

persist-key
persist-tun
resolv-retry infinite
nobind

user nobody
group nobody
tls-client
remote-cert-tls server
verb 4

ca /etc/openvpn/keys/ca.crt
cert /etc/openvpn/keys/*vpnclient1.crt*
key /etc/openvpn/keys/*vpnclient1.key*
tls-auth /etc/openvpn/keys/ta.key 1

```

如果服务器正在运行，请停止您的 OpenVPN 服务器：

```
$ sudo systemctl stop *openvpn@.openvpn1.service*
```

使用 *openvpn* 命令在两台主机上启动 OpenVPN：

```
$ sudo openvpn */etc/openvpn/vpnserver1.conf*
Tue Feb 16 16:50:49 2021 us=265445 Current Parameter Settings:
Tue Feb 16 16:50:49 2021 us=265481   config = '/etc/openvpn/vpnserver1.conf'
[...]
Tue Feb 16 16:50:49 2021 us=270212 Initialization Sequence Completed

```

```
$ sudo openvpn */etc/openvpn/vpnclient1.conf*
Tue Feb 16 16:56:22 2021 OpenVPN 2.4.3 x86_64-suse-linux-gnu [SSL (OpenSSL)]
[LZO] [LZ4] [EPOLL] [PKCS11] [MH/PKTINFO] [AEAD] built on Jun 20 2017
Tue Feb 16 16:56:22 2021 library versions: OpenSSL 1.1.1d  10 Sep 2019, LZO 2.10
Enter Private Key Password: *******
[...]
Tue Feb 16 16:56:26 2021 Initialization Sequence Completed

```

以上配置都是正确的，您已经成功建立了连接。在两台主机上按下 Ctrl-C 停止连接。

## 讨论

OpenVPN 安装时会附带一批示例配置文件，存放在 */usr/share/doc/openvpn/* 目录下。这些配置文件有详细注释，是很好的参考资料。虽然有数十个选项可供选择，但在实际应用中只会用到其中几个。在本文的示例中，有几点需要注意。

`port 1194` 是默认端口，`proto udp` 优先于 `proto tcp`。UDP 更安全，可以一定程度上防止端口扫描和拒绝服务攻击，同时提供更高的吞吐量和更低的延迟。TCP 在远程用户使用像酒店和咖啡店这样有限制的公共网络时很有用。

*tls-auth /etc/openvpn/keys/ta.key* 在服务器上必须始终为 0，在客户端上必须为 1。*tls-auth* 强制使用 TLS 连接。

*verb 4* 是日志记录级别。1 是最低级别，9 是最详细的。在确认一切设置正确之前，请将其保持在 4-6。当您从命令行启动 OpenVPN 时，将会看到大量消息。

有很多过时的指南建议使用 *comp_lzo* 选项来启用压缩。不必费心，因为它并没有提供太多好处。大多数流量要么已经被压缩，要么已经加密且无法被压缩。压缩还存在至少一种由于压缩引起的漏洞，VORACLE。

## 另请参阅

+   安装中的示例配置文件

+   [EasyRSA](https://oreil.ly/eKbsg)

+   [OpenVPN 文档](https://oreil.ly/Ah124)

+   *man 8 openvpn*

+   [OpenSSL Cookbook](https://oreil.ly/Ctm0X)

# 13.8 使用 systemctl 控制 OpenVPN

## 问题

您希望像管理其他守护程序一样管理 OpenVPN 守护程序，使用 *systemctl*，但您看不到 OpenVPN 单元文件。或者，您看到一个看起来奇怪的单元文件，如 *openvpn-server@.service*，当您尝试启动它时，它会抛出错误消息。

## 解决方案

双引号符@创建了*参数化*单元文件。这意味着您可以通过调用不同的配置文件轻松创建同一服务的多个单元文件。例如，假设您的服务器配置文件是*/etc/openvpn/austin.conf*。您的单元文件是*openvpn@austin.service*，使用*systemctl*创建：

```
$ sudo systemctl enable openvpn@austin
Created symlink /etc/systemd/system/multi-user.target.wants/openvpn@austin.service
→ /usr/lib/systemd/system/openvpn@.service.
Created symlink /etc/systemd/system/openvpn.target.wants/openvpn@austin.service
→ /usr/lib/systemd/system/openvpn@.service.
```

请注意，不要在 OpenVPN *.conf*文件的文件扩展名中输入文件扩展名。现在您可以像任何其他服务一样使用*systemctl*来控制您的 OpenVPN 守护程序。

## 讨论

这是一种相当巧妙的方法，使您能够在无需编写多个单元文件的情况下创建多个配置。您可以“参数化”任何 systemd 单元文件。

您可以在同一台计算机上同时运行多个隧道。每个配置需要一个不同的*tun*设备，例如*tun0*、*tun1*、*tun2*，每个隧道需要一个不同的子网和一个不同的 UDP 端口。使用不同的配置文件及其对应的参数化单元文件控制所有这些隧道。

## 参见

+   [EasyRSA](https://oreil.ly/eKbsg)

+   [OpenVPN 文档](https://oreil.ly/Ah124)

+   [systemd.unit](https://oreil.ly/2AAEe)

+   *man 8 openvpn*

+   [OpenSSL Cookbook](https://oreil.ly/Ctm0X)

# 13.9 通过.ovpn 文件更轻松地分发客户端配置

## 问题

设置客户端需要相当多的工作，您想知道是否有更快的方法，使用户可以自行完成而无需太多帮助。

## 解决方案

将客户端配置和密钥捆绑到扩展名为.ovpn 的单个文件中。所有客户端，包括 Linux、Windows、macOS、iOS 和 Android，都可以导入这些文件。

首先创建用户的证书，然后按照此模板创建他们的.ovpn 文件。此示例基于 Recipe 13.7 中的示例。而不是链接到所有他们的证书，请将它们复制到此文件中。所有证书都是明文的，因此您只需将 BEGIN/END 部分复制到.ovpn 文件中：

```
#*vpnclient1.ovpn*
client
dev tun
proto udp
remote server2 1194

persist-key
persist-tun
resolv-retry infinite
nobind

user nobody
group nobody
tls-client
remote-cert-tls server
verb 4

# ca.crt
<ca>
-----BEGIN CERTIFICATE-----
MIIDSDCCAjCgAwIBAgIUD2UxdEwgvhhr0zq5fAxIDIueB2EwDQYJKoZIhvcNAQEL
BQAwFTETMBEGA1UEAwwKdnBuc2VydmVyMTAeFw0yMTAyMjExODU1MjNaFw0zMTAy
MTkxODU1MjNaMBUxEzARBgNVBAMMCnZwbnNlcnZlcjEwggEiMA0GCSqGSIb3DQEB
AQUAA4IBDwAwggEKAoIBAQDpQJo+Izt8v0zriSWwrChc1tnVj3E3h3XuyEHub7hj
y4bMu2PqKByFNr+iikEF3u0d6HrCRSDKt1BcLzL3TsTJ/hJBHAlTyqEgVce1knjL
2g9NnDbekRtJSJCxS9j+RWtP43Xdg5edb5hTCZqdNFHD8oNuSMGFBbHN4oi9eDXl
rvyVHJe+UkI1Ow6mW0+ln/IoKNFPovz+l+ds3fJ5+UHe2TaQPQc7tGZ33j7wfJQd
es8baFdK+lnmGdUOrW9BQE6ReMSezkz6dKdIZdy7jEs6xoflOzyWlgydmnkAvLnx
MBQDgDUbc5MuooVMAWa4yhtz0B9ZmdJDb8jzHDpTPqdRAgMBAAGjgY8wgYwwHQYD
VR0OBBYEFF8KPhl1xxV0110JiBs5iUEPoJ1IMFAGA1UdIwRJMEeAFF8KPhl1xxV0
110JiBs5iUEPoJ1IoRmkFzAVMRMwEQYDVQQDDAp2cG5zZXJ2ZXIxghQPZTF0TCC+
GGvTOrl8DEgMi54HYTAMBgNVHRMEBTADAQH/MAsGA1UdDwQEAwIBBjANBgkqhkiG
9w0BAQsFAAOCAQEAMnRLz3CBApSrjfUKsWYioNGQGvh77Smh/1hPGIu4eEldQSmZ
Aj7qclEaORdBxmqrVtA3Z9cX1L0xFrg14nLyddmuWHG3ZChc5ZMpYtD2YpOH265B
FFjDp96vK13dpixWKrVpvakLCCA4EvnC8CEjbm0oNFiCgSwKAoJFCcUzwC33swsU
B2w5/iT6CZKuKhSmET1IDpG8krGC/Ib2GNAS0szMI94P0ajZgVznMcXOJ7gUg4rM
sEB8OzM6GBEZTqbAa9uVMZnOZvZA5jGIbBuelUo0bqGdAyx2B68zzuL//qvsHsvw
kZCyKIaXH0NBV7vexMKWcwFLLBzWizFQbbFpFA==
-----END CERTIFICATE-----
</ca>

# vpnclient1.cert
<cert>
-----BEGIN CERTIFICATE-----
MIIDVjCCAj6gAwIBAgIQLhO4FTrqN5WZiQETULAwnzANBgkqhkiG9w0BAQsFADAV
MRMwEQYDVQQDDAp2cG5zZXJ2ZXIxMB4XDTIxMDIyMTE4NTYzM1oXDTI0MDIwNjE4
NTYzM1owFTETMBEGA1UEAwwKdnBuY2xpZW50MTCCASIwDQYJKoZIhvcNAQEBBQAD
ggEPADCCAQoCggEBALUFYXwk6JW/hRtoMs0Ug5jMcWXsjMUsCz8L8CeXNOs3wQrf
YBWF1TYCLPd2/vwXsvbqCE85IZwjsJ5mEx9YgQ5M1teDkLZqBn8y7VIyDAAU8RsN
NcrnpeMDV0LgZIBeUrHi4ZTooaw4FdJ5BBYRHR1APVaaHDWx59ohJuBDpriWhvWk
lWX0rpSJltXriIOCzky/yEwfw6ah5jWaTgfe41fXq8j3lx2IbgIL7I4//jhC6JYz
N7huTdT2uB2MUbYX0XWBffMG8wcBZtMI2XryZmPvFYWP7N5nZZsBXkLz/UngAu3k
jkYJOnJy/hdOFLN/yXj7VFydmivUSeekdjjxyAECAwEAAaOBoTCBnjAJBgNVHRME
AjAAMB0GA1UdDgQWBBSnLIQoTPLyECbJHfgYBHvQpcmfgzBQBgNVHSMESTBHgBRf
Cj4ZdccVdNddCYgbOYlBD6CdSKEZpBcwFTETMBEGA1UEAwwKdnBuc2VydmVyMYIU
D2UxdEwgvhhr0zq5fAxIDIueB2EwEwYDVR0lBAwwCgYIKwYBBQUHAwIwCwYDVR0P
BAQDAgeAMA0GCSqGSIb3DQEBCwUAA4IBAQBaBpYZXVYUzOcXOVSaijmOZAIVBTeJ
meQz9xBQjqDXaRvypWlQ1gQtO8WnK9ruafc1g/h7LtvqtiALnGiJ0NbshkH8C1KE
yen46UCau5B/Xi0gA7FoPildvYdKSn/jI6KySCsplubjnJK9H/6DjAcEuqFLcsaY
5vpKQGP9Vl7H7hEVs4f1aory1T4Ma/bdXEOqgzHmIARLmxYeJm90sUT/n7e7VXfy
fILZ+8D1fMxCbeQRBkg1e8wJfgEbMRY9aGGt1qAs9gkm9RPelGB18v4iCbyebv3X
4hVHmfjcixdbWiABC7yq/gisooQ0robW/92dgemcwO0awHZX+opNBgwr
-----END CERTIFICATE-----
</cert>

# vpnclient1.key
<key>
-----BEGIN ENCRYPTED PRIVATE KEY-----
MIIFHDBOBgkqhkiG9w0BBQ0wQTApBgkqhkiG9w0BBQwwHAQInjFvz5a4mY8CAggA
MAwGCCqGSIb3DQIJBQAwFAYIKoZIhvcNAwcECNsxQXxvMpN0BIIEyEZdgFwPnGup
vyhywXR6l6ihvHK2GRczIgH0mFIiwQDgDjZj2YsEnvSA/P3MHplkU/bgv9DJ5j2T
C5wPDmGN4yG1boHx9BQKbXqxGwdz/UcHwmNKur9qnSFrSVEvMDwvum+rmzWuKykf
gkKKBCT1JZ2DWKtjjDNYG9qhBn3S2zYVq311dDuLbBcruvo1UL031sDDYWTpVuuf
zZc0ozng0Nzb35bNkG6Ib+LYLzJi4stxzw0DTFl52lKv++R6xhmqb81IJE3vBs4H
DOutkYfifO1eGqEKksPQRl8n03UVkOtB5pH8VdQeLqEBBaq3qeIfU6FkH9XrPR/E
8VOg9BNpbyuUW7bQu0MzuJ8Ofkjy9K+HHdwFtGPyOatkeaXT/qcKVMvzWcbr8bPc
VncavzXdzo0Sb8FigsKYU1lNjgo00Phd3m0AOfptrweK6ucBds5SmqNrUFXiQ2JA
Ms3LUw4CXBBgvdu5TsA2xLGysip0RPKLyTnUPGnXxbBaaHMv8Jz3XRCrWgZbtAE3
XhE9fKw+ZMEP+2jpC/1mjN/N9VuJfYZEhgA84wzYMu6pt3zPkWZqR6yGTDFEDhvh
OAZYEpqrhe++nxDpuQlpCCl4IndSg9L9oX1ydrvPNHGbRVztd3+r9wr4Ub3fJ1g/
9ckCdanohEymKbjw34HEMmdx+fn5k2T9bLnl8fsYtcESkg04ChON3yOnZFKl6chT
BQ9X2Qmeg7FoawWiUY5o+7OHNKL7QpRt4jXPbXNuXFK9EYvuRzUqubLhL5DdmjuO
Se1vvZg7fT4C8qjYsoCa18idA00EN3ePFFf9AssHCoVW92GiUTTKG+qURCjtNtG6
dnPvxiSf98OBkkjeX3ni0cKdfMGoQTSdEy5GexvfRMF5HJrGO+CWXmqSBsuIlPUe
quqCsPmpaT2Ws/0UU9cKe4qaKjTL7CghtFmUEhH7t6Cd41Ki9gKi33j3541l9w7l
J1bgca4rRUCecp2BPF3IjJc/RnTvHkbUK4mDX9s8xJhYf9WE6JYsk3NBSNNIj/9G
FMJlo71x8H3OAdFzRN5bjV797HByZ+YidZIgGAx2dSko3PQPy7RSxdmzFbxfUvzj
9jcYEu+V9unbtDK2qZ9I+LqXGE+EXjPBui40IWp8XIYNlSLn2qgroH079lXhXKBY
+DzcBzyT7GTX2QeYE+yqqPRIFWHnbnsnD6dMnAa46h+Si+f5sq33rfRsF7UpK4gV
IhzFkncCM47/Taqi0OY04Q40LuSCDjmjFL+VzZOsAtWGRNYNzIgniThEehElJwfI
ErzClcVptjhtCer8BPuO7YaMIHk1hKecHFqw3RrimWzroL1iu9Q29m2oM+bVc6mD
we6r+t8JbaAFxoHBK4i6M0rcdJPICxDTIOjPC3Fg/MeqiCi7F0DFZvXwPGRD+0Of
MBnsDplEUjK06jbE5BjGQ7n7P+dwDxyp/aVO4CfX7ZOco6h9r3b6nqlzPVNE9erw
kS7WwT/TWraw/sfIO9sNSgle7PoRh2s/w/oGVhC6ymlMdXe+mhMzHFnGEbBRh2Rd
kd/EdYNubHg0k9+RLTwbgwZ+176cIJyOpqaoJGv0bsKM8X26Pk/fkyF6xgdQYQOx
8i9Whea8OjUOQAcgc7gUyA==
-----END ENCRYPTED PRIVATE KEY-----
</key>

# ta.key
<tls-auth>
-----BEGIN OpenVPN Static key V1-----
4eb35b44d1d8a82cfa51af394d4f58f3
69bf8fe8c0a0a032f38b0ee104889628
8a5dc89486736b39d64ad3c6831bf9ba
9f3f96c3307d322a5bf055b9bc3bfa74
929faf361c14de97445f5927794264bb
e3f71c925f2236cfb0109ecfd6406cef
857dfb39783a09ecd56d3cf09ebbc853
0f43b1c787f0db99dbecabcd2090cfbb
54c86d8102a5430fd6a7f37ab5ce8ed9
f6bec8984bde4267f78913ff702dd396
a205b6be9e7ab41cf1ebad3953c27c7c
f3b435345e02aede049ef7c9f1c2704f
2ed91110ccb19d0d3bd46a00f54c73e2
07b31160cdc54c3f5a7989bb999ac5f3
89c6de7e79fc93399924a8d298eab462
231234e690c319d5cbd832788f0dbcfb
-----END OpenVPN Static key V1-----
</tls-auth>

```

现在您只需要向客户分发一个文件。 （请参阅 Recipe 13.1 了解如何为 Linux、macOS、Windows、iOS 和 Android 客户端安装所需内容。）

在 Linux 中导入新的*.ovpn*文件的简便方法是使用 NetworkManager。打开“VPN Connections” → “Add a VPN connection”。这将打开“选择 VPN 连接类型”。选择“导入保存的 VPN 连接”，点击“创建”，然后在文件选择器中找到您的*.ovpn*文件。在“通用”和“VPN”选项卡上查看设置。

在“通用”选项卡上，请确保未选中“所有用户都可以连接到此网络”。这是一个简单但重要的安全措施，要求每个用户都有自己的独立 OpenVPN 配置。

在 VPN 选项卡上，请注意 NetworkManager 将内联证书转换为.pem 文件（Figure 13-2）。这是正常的，不是错误。您可以将这些与原始文件进行比较；单击右侧的小文件夹图标即可查看转换后的文件存储位置。

![将 .ovpn 客户端配置文件导入 NetworkManager](img/lcb2_1302.png)

###### 图 13-2\. 将 *.ovpn* 客户端配置文件导入 NetworkManager

对于所有其他客户端，该过程类似。按照他们的说明操作，如果一切顺利，您的客户端将在几分钟内运行起来。

NetworkManager 导入功能也适用于不是内联的客户端配置文件，例如第 13.7 节。在这种情况下，证书不会被转换，它们保留其原始文件名。

如果只有 Linux 客户端，则内联文件可能具有 *.conf* 扩展名。

## 另请参阅

+   [EasyRSA](https://oreil.ly/eKbsg)

+   [OpenVPN 文档](https://oreil.ly/Ah124)

+   *man 8 openvpn*

+   [OpenSSL Cookbook](https://oreil.ly/Ctm0X)

# 13.10 加固您的 OpenVPN 服务器

## 问题

您想了解一些使您的 OpenVPN 更安全的选项。

## 解决方案

OpenVPN 的默认设置相当不错，但它们设计用于更广泛的兼容性。您可以做一些改动，使您的服务器更强大。

下面的示例适用于服务器和客户端配置文件。这些选项最大化了 TLS 的效果。所有低于 TLS 1.2 的 SSL 和 TLS 协议均已废弃，不应允许。仅接受 TLS 1.2 或更高版本：

```
tls-version-min 1.2
tls-version-max 1.3 or-highest
```

使用更强大的数据通道密码，并通过禁用密码协商来强制使用它：

```
AES-128-GCM
ncp-disable
```

TLS 1.3 有许多变化，因此您需要为 TLS 1.2 和 1.3 配置两种不同的配置文件。这些都是更强大和更高效的加密密码：

```
# TLS 1.3
tls-ciphersuites TLS_CHACHA20_POLY1305_SHA256:TLS_AES_128_GCM_SHA256
# TLS 1.2
tls-cipher TLS-ECDHE-ECDSA-WITH-CHACHA20-POLY1305-SHA256:TLS-ECDHE-RSA-
WITH-CHACHA20-POLY1305-SHA256:TLS-ECDHE-ECDSA-WITH-AES-128-GCM-SHA256:
TLS-ECDHE-RSA-WITH-AES-128-GCM-SHA256
```

使用椭圆曲线 Diffie-Hellman 瞬态（ECDHE）代替旧的静态 Diffie-Hellman 密钥。您不必像在第 13.5 节中创建 *ta.key* 那样操作：

```
dh none
ecdh-curve secp384r1
# use tls-server on the server, tls-client on the client
tls-server
```

在服务器配置中添加 *float* 选项，允许客户端在不丢失连接的情况下漫游到不同的网络，只要它们通过所有其他身份验证测试。

*opt-verify* 选项仅在服务器配置中检查服务器和客户端设置的兼容性，并断开不匹配的客户端。*opt-verify* 检查 *dev-type, link-mtu, tun-mtu, proto, ifconfig, comp-lzo, fragment, keydir, cipher, auth, keysize, secret, no-replay, no-iv, tls-auth, key-method, tls-server* 和 *tls-client*。

查看讨论以获取完整的示例配置。

## 讨论

将这些增强功能全部整合到服务器配置中：

```
# *vpnserver1.conf*
port 1194
proto udp
dev tun
user nobody
group nobody

ca /etc/openvpn/keys/ca.crt
cert /etc/openvpn/keys/*vpnserver1.crt*
key /etc/openvpn/keys/*vpnserver1.key*

server 10.10.0.0 255.255.255.0
ifconfig-pool-persist ipp.txt
keepalive 10 120
persist-key
persist-tun
tls-server

remote-cert-tls client
verify-client-cert require
tls-cert-profile preferred
tls-version-min 1.2
tls-version-max 1.3 or-highest

float
opt-verify
AES-128-GCM
ncp-disable
dh none
ecdh-curve secp384r1

# TLS 1.3
tls-ciphersuites TLS_CHACHA20_POLY1305_SHA256:TLS_AES_128_GCM_SHA256
# TLS 1.2
tls-cipher TLS-ECDHE-ECDSA-WITH-CHACHA20-POLY1305-SHA256:TLS-ECDHE-RSA-
WITH-CHACHA20-POLY1305-SHA256:TLS-ECDHE-ECDSA-WITH-AES-128-GCM-SHA256:
TLS-ECDHE-RSA-WITH-AES-128-GCM-SHA256

status openvpn-status.log
verb 4
mute 20
explicit-exit-notify 1
```

一个示例客户端配置，使用内联文件格式（第 13.9 节）：

```
# *vpnclient1.conf*
client
dev tun
proto udp
remote *server1* 1194

persist-key
persist-tun
resolv-retry infinite
nobind

user nobody
group nobody
tls-client
remote-cert-tls server
verb 4

# Using inline keys
# ca.crt
<ca>
[...]
</ca>

# *client.crt*
<cert>
[...]
</cert>

# *client.key*
<key>
[...]
</key>

tls-version-min 1.2
tls-version-max 1.3 or-highest
AES-128-GCM
ncp-disable
dh none
ecdh-curve secp384r1

# TLS 1.3 encryption settings
tls-ciphersuites TLS_CHACHA20_POLY1305_SHA256:TLS_AES_128_GCM_SHA256
# TLS 1.2 encryption settings
tls-cipher TLS-ECDHE-ECDSA-WITH-CHACHA20-POLY1305-SHA256:TLS-ECDHE-RSA-WITH-
CHACHA20-POLY1305-SHA256:TLS-ECDHE-ECDSA-WITH-AES-128-GCM-SHA256:TLS-ECDHE-RSA-
WITH-AES-128-GCM-SHA256

status openvpn-status.log
verb 4
mute 20
explicit-exit-notify 1
```

这些选项加强了客户端和服务器之间的认证，并强制使用 TLS 1.2 及更高版本。如果您想知道使用哪些密码和密码套件，可以向专家询问。您可以进一步限制您的设置。例如，禁止用户保存密码，增加客户端-服务器认证的限制，使用 SELinux 或使用 chroot。这些都是此处未涵盖的高级主题。

## 另请参阅

+   [EasyRSA](https://oreil.ly/eKbsg)

+   [OpenVPN 文档](https://oreil.ly/Ah124)

+   *man 8 openvpn*

+   [OpenSSL Cookbook](https://oreil.ly/Ctm0X)

# 13.11 配置网络

## 问题

您的 OpenVPN 服务器正在运行，所有连接测试都正常工作，现在您需要知道如何设置您的网络，以便远程客户端可以找到您的服务器并正确路由流量。

## 解决方案

没有一种适合所有情况的解决方案。配置您的网络必须考虑到您的局域网设置，无论是连接个体客户端还是链接网络，IPv4、IPv6，您的互联网网关设置以及更多因素。请参阅 Jan Just Keijser 的优秀著作 [*OpenVPN Cookbook*](https://learning.oreilly.com/library/view/openvpn-cookbook-/9781786463128/)，第二版（O'Reilly），以找到您需要的答案。本书涵盖了 TUN 与 TAP、Windows 客户端、PAM 和 LDAP、IPv6、路由以及站点间配置。

## 讨论

网络是运行任何服务器的最具挑战性的部分，尤其是一个重要的安全服务器。学习和确保正确配置网络是非常值得的。OpenVPN 文档也有很多关于网络的有用信息。

## 参见

+   [OpenVPN 文档](https://oreil.ly/Ah124)

+   *man 8 openvpn*
