## 附录。逐章的命令行回顾

### 1. 欢迎使用 Linux

+   `ls -lh /var/log`—列出/var/log/目录的内容和完整、人性化的详细信息。

+   `cd`—返回您的家目录。

+   `cp file1 newdir`—将名为 file1 的文件复制到名为 newdir 的目录中。

+   `mv file? /some/other/directory/`—将包含字母*file*和一个更多字符的所有文件移动到目标位置。

+   `rm -r *`—删除当前位置下的所有文件和目录—请谨慎使用。

+   `man sudo`—打开使用 sudo 与命令的 man 文档文件。

### 2. Linux 虚拟化：构建 Linux 工作环境

+   `apt install virtualbox`—使用`apt`从远程仓库安装软件包。

+   `dpkg -i skypeforlinux-64.deb`—直接在 Ubuntu 机器上安装下载的 Debian 包。

+   `wget https://example.com/document-to-download`—一个用于下载文件的命令行程序。

+   `dnf update`—或`yum update`或`apt update`—同步本地软件索引与在线仓库中可用的内容。

+   `shasum ubuntu-16.04.2-server-amd64.iso`—计算下载文件的校验和，以确认其与提供的值匹配，这意味着内容在传输过程中未被损坏。

+   `vboxmanage clonevm Kali-Linux-template --name newkali`—使用`vboxmanage`工具克隆现有的虚拟机。

+   `lxc-start -d -n mycont`—启动现有的 LXC 容器。

+   `ip addr`—显示系统每个网络接口的信息（包括它们的 IP 地址）。

+   `exit`—在不关闭机器的情况下离开 shell 会话。

### 3. 远程连接：安全访问网络机器

+   `dpkg -s ssh`—检查基于 Apt 的软件包的状态。

+   `systemctl status ssh`—检查系统进程（systemd）的状态。

+   `systemctl start ssh`—启动系统进程。

+   `ip addr`—列出计算机上的所有网络接口。

+   `ssh-keygen`—生成新的 SSH 密钥对。

+   `$ cat .ssh/id_rsa.pub | ssh ubuntu@10.0.3.142 "cat >> .ssh/authorized_keys"`—复制本地密钥，并将其粘贴到远程机器上。

+   `ssh -i .ssh/mykey.pem ubuntu@10.0.3.142`—指定特定的密钥对。

+   `scp myfile ubuntu@10.0.3.142:/home/ubuntu/myfile`—安全地将本地文件复制到远程计算机。

+   `ssh -X ubuntu@10.0.3.142`—登录到远程主机以进行图形化会话。

+   `ps -ef | grep init`—显示所有当前运行的系统进程，并通过字符串`init`过滤结果。

+   `pstree -p`—以可视树格式显示所有当前运行的系统进程。

### 4. 归档管理：备份或复制整个文件系统

+   `df -h`—以人类可读的格式显示所有当前活动的分区及其大小。

+   `tar czvf archivename.tar.gz /home/myuser/Videos/*.mp4`—从指定目录树中的视频文件创建压缩归档。

+   `split -b 1G archivename.tar.gz archivename.tar.gz.part`—将大文件分割成指定最大大小的较小文件。

+   `find /var/www/ -iname "*.mp4" -exec tar -rvf videos.tar {} \;`—查找满足特定标准的文件，并将它们的名称流式传输到 tar 以包含在存档中。

+   `chmod o-r /bin/zcat`—移除用户 *others* 的读取权限。

+   `dd if=/dev/sda2 of=/home/username/partition2.img`—创建 sda2 分区的镜像，并将其保存到您的家目录中。

+   `dd if=/dev/urandom of=/dev/sda1`—用随机字符覆盖分区以隐藏旧数据。

### 5. 自动化管理：配置自动离站备份

+   `#!/bin/bash`—所谓的“shebang 行”，告诉 Linux 您将使用哪个 shell 解释器来执行脚本。

+   `||`—在脚本中插入一个“或”，意味着左侧的命令成功或执行右侧的命令。

+   `&&`—在脚本中插入一个“和”，意味着如果左侧的命令成功，则执行右侧的命令。

+   `test -f /etc/filename`—检查指定的文件或目录名是否存在。

+   `chmod +x upgrade.sh`—使脚本文件可执行。

+   `pip3 install --upgrade --user awscli`—使用 Python 的 pip 软件包管理器安装 AWS 命令行界面。

+   `aws s3 sync /home/username/dir2backup s3://linux-bucket3040`—同步本地目录的内容与指定的 S3 存储桶。

+   `21 5 * * 1 root apt update && apt upgrade`—cron 指令，每天早上 5:21 执行两个 `apt` 命令。

+   `NOW=$(date +"%m_%d_%Y")`—将当前日期分配给脚本变量。

+   `systemctl start site-backup.timer`—激活 systemd 系统定时器。

### 6. 紧急工具：构建系统恢复设备

+   `sha256sum systemrescuecd-x86-5.0.2.iso`—计算 .ISO 文件的 SHA256 校验和。

+   `isohybrid systemrescuecd-x86-5.0.2.iso`—为实时启动镜像添加一个 USB 友好的 MBR。

+   `dd bs=4M if=systemrescuecd-x86-5.0.2.iso of=/dev/sdb && sync`—将实时启动镜像写入空驱动器。

+   `mount /dev/sdc1 /run/temp-directory`—将分区挂载到实时文件系统上的一个目录。

+   `ddrescue -d /dev/sdc1 /run/usb-mount/sdc1-backup.img /run/usb-mount/sdc1-backup.logfile`—将损坏分区上的文件保存到名为 sdc1-backup.img 的镜像中，并将事件写入日志文件。

+   `chroot /run/mountdir/`—在文件系统上打开 root shell。

### 7. 网络服务器：构建 MediaWiki 服务器

+   `apt install lamp-server^`—Ubuntu 命令，安装 LAMP 服务器的所有元素。

+   `systemctl enable httpd`—在 CentOS 机器的每次系统启动时启动 Apache。

+   `firewall-cmd --add-service=http --permanent`—允许 HTTP 浏览器流量进入 CentOS 系统。

+   `mysql_secure_installation`—重置您的 root 密码并加强数据库安全性。

+   `mysql -u root -p`—以 root 用户登录 MySQL（或 MariaDB）。

+   `CREATE DATABASE newdbname;`—在 MySQL（或 MariaDB）中创建一个新的数据库。

+   `yum search php- | grep mysql`—在 CentOS 机器上搜索与 PHP 相关的可用软件包。

+   `apt search mbstring`—搜索与多字节字符串编码相关的可用软件包。

### 8. 网络文件共享：构建 Nextcloud 文件共享服务器

+   `a2enmod rewrite`—启用重写模块，以便 Apache 可以在客户端和服务器之间移动 URL 时编辑它们。

+   `nano /etc/apache2/sites-available/nextcloud.conf`—为 Nextcloud 创建或编辑 Apache 主机配置文件。

+   `chown -R www-data:www-data /var/www/nextcloud/`—将所有网站文件的用户和组所有权更改为 www-data 用户。

+   `sudo -u www-data php occ list`—使用 Nextcloud CLI 列出可用命令。

+   `aws s3 ls s3://nextcloud32327`—列出 S3 存储桶的内容。

### 9. 保护你的 Web 服务器

+   `firewall-cmd --permanent --add-port=80/tcp`—打开端口 80 以允许传入 HTTP 流量，并在启动时配置它重新加载。

+   `firewall-cmd --list-services`—列出 firewalld 系统上当前活动的规则。

+   `ufw allow ssh`—使用 Ubuntu 上的 UncomplicatedFirewall 打开端口 22 以允许 SSH 流量。

+   `ufw delete 2`—删除`ufw status`命令列出的第二个`ufw`规则。

+   `ssh -p53987 username@remote_IP_or_domain`—使用非默认端口登录 SSH 会话。

+   `certbot --apache`—配置 Apache Web 服务器以使用 Let’s Encrypt 加密证书。

+   `selinux-activate`—在 Ubuntu 机器上激活 SELinux。

+   `setenforce 1`—在 SELinux 配置中切换强制模式。

+   `ls -Z /var/www/html/`—显示指定目录中文件的保密上下文。

+   `usermod -aG app-data-group otheruser`—将 otheruser 用户添加到 app-data-group 系统组。

+   `netstat -npl`—在服务器上扫描打开（监听）的网络端口。

### 10. 确保网络连接安全：创建 VPN 或 DMZ

+   `hostname OpenVPN-Server`—设置命令提示符描述，以便更容易跟踪你登录的是哪个服务器。

+   `cp -r /usr/share/easy-rsa/ /etc/openvpn`—将 Easy RSA 脚本和环境配置文件复制到工作 OpenVPN 目录。

+   `./build-key-server server`—生成名为*server*的 RSA 密钥对集。

+   `./pkitool client`—从现有的 RSA 密钥基础设施生成客户端密钥集。

+   `openvpn --tls-client --config /etc/openvpn/client.conf`—使用 client.conf 文件中的设置在 Linux 客户端上启动 OpenVPN。

+   `iptables -A FORWARD -i eth1 -o eth2 -m state --state NEW,ESTABLISHED,RELATED -j ACCEPT`—允许 eth1 和 eth2 网络接口之间的数据传输。

+   `man shorewall-rules`—显示 Shorewall 使用的规则文件的文档。

+   `systemctl start shorewall`—启动 Shorewall 防火墙工具。

+   `vboxmanage natnetwork add --netname dmz --network "10.0.1.0/24" --enable --dhcp on`—使用 VirtualBox CLI 创建和配置一个带有 DHCP 的虚拟 NAT 网络，用于 VirtualBox 虚拟机。

+   `vboxmanage natnetwork start --netname dmz`—启动虚拟 NAT 网络。

+   `dhclient enp0s3`—从 DHCP 服务器请求 enp0s3 接口的 IP 地址。

### 11. 系统监控：处理日志文件

+   Alt-F<*n*>—从非 GUI shell 打开虚拟控制台。

+   `journalctl -n 20`—显示最近的 20 条日志条目。

+   `journalctl --since 15:50:00 --until 15:52:00`—仅显示在 `since` 和 `until` 时间之间的事件。

+   `systemd-tmpfiles --create --prefix /var/log/journal`—指示 systemd 创建并维护一个持久的日志文件，而不是每次启动时都会被销毁的文件。

+   `cat /var/log/auth.log | grep -B 1 -A 1 failure`—显示匹配的行以及其前后的一行。

+   `cat /var/log/mysql/error.log | awk '$3 ~/[Warning]/' | wc`—在 MySQL 错误日志中搜索被分类为警告的事件。

+   `sed "s/^` **`[0-9]`** `//g" numbers.txt`—删除文件每行的开头数字。

+   `tripwire --init`—初始化 Tripwire 安装的数据库。

+   `twadmin --create-cfgfile --site-keyfile site.key twcfg.txt`—为 Tripwire 生成一个新的加密 tw.cfg 文件。

### 12. 在私有网络上共享数据

+   `/home 192.168.1.11(rw,sync)`—NFS 服务器 /etc/exports 文件中的一个条目，定义了远程客户端共享。

+   `firewall-cmd --add-service=nfs`—打开 CentOS 防火墙以允许客户端访问您的 NFS 共享。

+   `192.168.1.23:/home /nfs/home nfs`—NFS 客户端 /etc/fstab 文件中的一个典型条目，用于加载 NFS 共享。

+   `smbpasswd -a sambauser`—向现有的 Linux 用户账户添加 Samba 功能（以及一个唯一的密码）。

+   `nano /etc/samba/smb.conf`—Samba 由服务器上的 smb.conf 文件控制。

+   `smbclient //localhost/sharehome`—使用 Samba 用户账户登录本地 Samba 共享。

+   `ln -s /nfs/home/ /home/username/Desktop/`—创建一个符号链接，允许用户通过点击桌面图标轻松访问 NFS 共享。

### 13. 解决系统性能问题

+   `uptime`—返回过去 1、5 和 15 分钟的 CPU 负载平均值。

+   `cat /proc/cpuinfo | grep processor`—返回系统 CPU 处理器的数量。

+   `top`—显示正在运行的 Linux 进程的实时统计信息。

+   `killall yes`—关闭所有正在运行的 `yes` 命令实例。

+   `nice --15 /var/scripts/mybackup.sh`—提高 mybackup.sh 脚本的系统资源优先级。

+   `free -h`—显示系统和可用系统 RAM。

+   `df -i`—显示每个文件系统的可用和总 inodes。

+   `find . -xdev -type f | cut -d "/" -f 2 | sort | uniq -c | sort -n`—按父目录计数并显示文件数量。

+   `apt-get autoremove`—删除旧的未使用的内核头文件。

+   `nethogs eth0`—使用 eth0 接口显示与网络连接相关的进程和数据传输。

+   `tc qdisc add dev eth0 root netem delay 100ms`—通过 eth0 接口将所有网络传输速度减慢 100 毫秒。

+   `nmon -f -s 30 -c 120`—将一系列 `nmon` 扫描的数据记录到文件中。

### 14. 解决网络问题

+   `ip addr`—列出 Linux 系统上的活动接口。可以是 `ip a` 的缩写，也可以是 `ip address` 的扩展，由您选择。

+   `lspci`—列出当前连接到您的计算机的 PCI 设备。

+   `dmesg | grep -A 2 Ethernet`—在 `dmesg` 日志中搜索字符串 *Ethernet* 的引用，并显示引用及其后续两行输出。

+   `ip route add default via 192.168.1.1 dev eth0`—手动为计算机设置新的网络路由。

+   `dhclient enp0s3`—为 enp0s3 接口请求动态（DHCP）IP 地址。

+   `ip addr add 192.168.1.10/24 dev eth0`—将静态 IP 地址分配给 eth0 接口（在下次系统重启后不会持续）。

+   `ip link set dev enp0s3 up`—启动 enp0s3 接口（在编辑配置后很有用）。

+   `netstat -l | grep http`—扫描本地机器，查找在端口 80 上监听的 Web 服务。

+   `nc -z -v bootstrap-it.com 443 80`—扫描远程网站，查找在端口 443 或 80 上监听的服务。

### 15. 外设故障排除

+   `lshw -c memory`（或 `lshw -class memory`）—显示系统硬件配置的内存部分。

+   `` ls /lib/modules/`uname -r` ``—列出位于 /lib/modules/ 目录下的内容，该目录包含当前活动内核的模块。

+   `lsmod`—列出所有活动模块。

+   `modprobe -c`—列出所有可用的模块。

+   `find /lib/modules/$(uname -r) -type f -name ath9k*`—在可用的内核模块中搜索以 *ath9k* 开头的文件。

+   `modprobe ath9k`—将指定的模块加载到内核中。

+   `GRUB_CMDLINE_LINUX_DEFAULT="systemd.unit=runlevel3.target"`—/etc/default/grub 文件加载 Linux 为多用户、非图形会话。

+   `lp -H 11:30 -d Brother-DCP-7060D /home/user/myfile.pdf`—计划在 11:30 UTC 的时间将打印任务发送到 Brother 打印机。

### 16. DevOps 工具：使用 Ansible 部署脚本化服务器环境

+   `add-apt-repository ppa:ansible/ansible`—将 Debian Ansible 软件仓库添加到系统中，以便 `apt` 在 Ubuntu/Debian 机器上安装 Ansible。

+   `ansible webservers -m ping`—测试 webservers 主机组中的所有主机以检查网络连通性。

+   `ansible webservers -m copy -a "src=/home/ubuntu/stuff.html dest=/var/www/html/"`—将本地文件复制到 webservers 组中所有主机的指定文件位置。

+   `ansible-doc apt`—显示 `apt` 模块的语法和用法信息。

+   `ansible-playbook site.yml`—根据 site.yml playbook 启动操作。

+   `ansible-playbook site.yml --ask-vault-pass`—使用 Vault 密码进行身份验证并执行 playbook 操作。
