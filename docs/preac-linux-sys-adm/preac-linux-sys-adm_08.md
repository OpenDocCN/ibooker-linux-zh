# 第八章：维护系统健康

维护系统健康是一个广泛的主题，包括预防性维护、日常管理、补丁、安全任务、用户维护以及监控和减少各种类型的扩展。保持系统健康是一项主动任务，而不是被动的。

监控是有帮助的。定期清理特定区域可以帮助。自动化更新也有帮助，但您必须积极地查看日志、检查空间和扩展，并处理用户维护任务。如果您能自动化系统健康维护的每个方面，系统管理员的需求将大大减少。

本章涵盖了自动化和手动系统健康维护任务。

# 保持系统无杂物

日常管理是没人愿意做的事情。它繁琐、耗时，可能会激怒用户，甚至向管理层举报你的行为，这从来不是好事。只要您遵守公司政策，不应用任何自己的政策，您将有材料可以提醒生气的用户。清理系统的规则与您进行的任何维护相同：如果意外删除了关键文件或目录，请确保有可靠的备份。以下各节详细介绍了如何保持系统无杂物。

## 清理 /tmp 目录

*/tmp* 目录是一个共享目录。它与所有用户、应用程序和系统进程共享。任何人都可以写入此目录，这对管理员来说是不利的，因为拥有对系统目录的无限访问权限可能会产生致命后果。如果原始管理员没有将 */tmp* 目录创建为一个具有有限空间访问权限的单独文件系统，用户或应用程序可能会填满所有空间。因此，出于这个原因，*/tmp* 目录绝不能作为与 */* 相同分区的一部分。

对于系统管理员来说，*/tmp* 目录在日常管理方面有些棘手，因为用户、应用程序或系统没有任何限制。因为任何用户都可以写入它，他们可能会认为这是额外的可用空间，可以存储下载和其他文件。为了使该目录成为一个不太理想的存储位置，您应该创建一个 `cron` 作业，每晚在备份后运行，以删除任何非系统文件。

###### 注意

在许多企业中，自动脚本每晚都会从 */tmp* 目录中删除用户创建的文件，并且大多数情况下将 */tmp* 目录排除在备份之外。

如果您不想为 */tmp* 创建单独的文件系统并将其挂载到 */tmp* 上，解决方案是启用 *tmp.mount*。此服务会创建一个临时文件系统（tmpfs），并将其挂载为 */tmp*。其中一个特点是它是易失性存储（RAM），填充它不会导致系统稳定性问题。

要在您的系统上启用 *tmp.mount*，请完成以下步骤。

首先，检查 */tmp* 目录的挂载点，看是否需要配置 *tmp.mount*：

```
$ df -h /tmp
Filesystem           Size  Used Avail Use% Mounted on
/dev/mapper/cl-root  6.2G  3.3G  3.0G  53% /
```

正如您所见，我的 */tmp* 有一个缺陷，即它是 */* 文件系统的子目录。启用 *tmp.mount*，然后启动它：

```
$ sudo systemctl enable tmp.mount
Created symlink /etc/systemd/system/local-fs.target.wants/tmp.mount 
→ \ /usr/lib/systemd/system/tmp.mount.

$ sudo systemctl start tmp.mount

$ df -h /tmp
Filesystem      Size  Used Avail Use% Mounted on
tmpfs           405M     0  405M   0% /tmp
```

现在，*/tmp* 目录不再是 */* 文件系统的子目录。启用 *tmp.mount* 使此配置持久化，这意味着这是一个永久性的更改，并且每次系统启动时都会自动创建。

下一部分介绍如何保持 */home* 目录对所有人可用。

## 使 /home 对每个人都是一个宜居的空间

*/home* 目录是一个共享目录，因为它包含除 root 用户外所有用户的主目录。例如，假设您的系统上有用户 `tux`、`penguin` 和 `gentoo`。它们的主目录如下：

```
/home/tux
/home/penguin
/home/gentoo
```

在系统中，如果 */home* 目录与 */* 目录位于同一文件系统中，则可能导致用户填满 */* 目录并引发系统问题，就像填满 */tmp* 一样，在前一节中讨论过。由于 */home* 中包含的文件的性质，您无法为其创建和挂载易失性文件系统。*/home* 目录必须是一个永久挂载的文件系统，最好不是 */* 的一部分。如果它是 */* 的一部分，则有两种方法可以更正它。第一种方法如下所述：

+   缩小现有的 LVM 文件系统

+   创建分区和文件系统

+   将该文件系统挂载为 */home*

第二种方法涉及以下步骤：

+   安装新硬盘

+   创建新分区

+   将新分区挂载为 */home*

在本章演示中，我将使用第二种方法为 */home* 安装新硬盘。

完成这些步骤的详细过程如下：

1.  创建或安装一个新硬盘。

1.  在硬盘上创建新的分区。

1.  在新分区上创建文件系统。

1.  将新分区挂载到 */mnt*。

1.  将所有文件从 */home* 复制或移动到 */mnt*。

1.  从 */home* 中删除所有文件。

1.  卸载位于 */mnt* 的新分区。

1.  将新分区挂载到 */home*。

1.  在 */etc/fstab* 中为 */home* 添加条目。

我向我的虚拟机添加了一个 1 GB 硬盘来演示这些步骤。以下各节详细说明如何完成此过程中的每一步。

### 在硬盘上创建新的分区

通过列出系统上的所有磁盘块设备来识别新硬盘：

```
$ lsblk | grep disk
sda                   8:0    0    8G  0 disk
sdb                   8:16   0  1.5G  0 disk
sdc                   8:32   0    1G  0 disk
```

新磁盘设备（*/dev/sdc*）是新增的 1 GB 硬盘。为了在该硬盘上创建新的分区，请使用 `fdisk` 实用程序：

```
$ sudo fdisk /dev/sdc

Welcome to fdisk (util-linux 2.32.1).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.

Device does not contain a recognized partition table.
Created a new DOS disklabel with disk identifier 0x3c239df6.

Command (m for help): n
Partition type
   p   primary (0 primary, 0 extended, 4 free)
   e   extended (container for logical partitions)
Select (default p): p
Partition number (1-4, default 1):
First sector (2048-2097151, default 2048):
Last sector, +sectors or +size{K,M,G,T,P} (2048-2097151, default 2097151):

Created a new partition 1 of type 'Linux' and of size 1023 MiB.

Command (m for help): w
The partition table has been altered.
Calling ioctl() to re-read partition table.
Syncing disks.
```

列出 `sdc` 块设备以验证分区 */dev/sdc1* 是否存在：

```
$ lsblk | grep sdc
sdc                   8:32   0    1G  0 disk
└─sdc1                8:33   0 1023M  0 part
```

*/dev/sdc1* 分区已存在并准备好在其上创建文件系统。

### 创建文件系统

对于 */home* 目录，将其格式化为 `ext4` 文件系统类型：

```
$ sudo mkfs.ext4 /dev/sdc1

mke2fs 1.45.6 (20-Mar-2020)
Creating filesystem with 261888 4k blocks and 65536 inodes
Filesystem UUID: bcd18fcc-3774-4b94-be0a-e2abf8aa4d31
Superblock backups stored on blocks:
    32768, 98304, 163840, 229376

Allocating group tables: done                           
Writing inode tables: done                           
Creating journal (4096 blocks): done
Writing superblocks and filesystem accounting information: done
```

确认你在 */dev/sdc1* 上正确创建了 `ext4` 文件系统：

```
$ lsblk -o NAME,FSTYPE,SIZE | grep sdc
sdc                                1G
└─sdc1              ext4        1023M
```

### 挂载新分区

要使用新分区，必须将其挂载。我使用 */mnt* 作为临时挂载点，将文件从 */home* 复制到最终将挂载为 */home* 的新分区。如果现在将 */dev/sdc1* 挂载到 */home* 上，就无法复制文件，因为在 */home* 上挂载会隐藏其文件。将设备挂载到非空目录会隐藏其内容并使其文件无法访问：

```
$ sudo mount /dev/sdc1 /mnt

$ mount | grep sdc
/dev/sdc1 on /mnt type ext4 (rw,relatime,seclabel)
```

### 从 */home* 复制文件到 */mnt*

将所有文件从 */home* 目录复制以保留所有 (`-a`) 权限、链接和时间戳：

```
$ sudo cp -a /home/* /mnt

$ ls -la /home
total 0
drwxr-xr-x.  4 root  root   32 Nov  3  2020 .
dr-xr-xr-x. 19 root  root  258 Dec 10 08:37 ..
drwx------.  2  1001  1001  62 Nov 13  2020 diane
drwx------.  7 khess khess 273 Dec 14 21:03 khess

$ ls -la /mnt
total 28
drwxr-xr-x.  5 root  root   4096 Dec 15 08:33 .
dr-xr-xr-x. 19 root  root    258 Dec 10 08:37 ..
drwx------.  2  1001  1001  4096 Nov 13  2020 diane
drwx------.  7 khess khess  4096 Dec 14 21:03 khess
drwx------.  2 root  root  16384 Dec 15 08:17 lost+found
```

### 删除 */home* 下的所有文件

我删除原始 */home* 目录中的所有文件的原因是将磁盘空间释放回 */* 文件系统，这也是我们进行此操作的原因之一。以下命令删除 */home* 下的所有目录，而不删除 */home* 目录本身。无需删除 */home*：

```
$ sudo rm -rf /home/*
```

### 从 */mnt* 卸载 */dev/sdc1*

从 */mnt* 卸载 */dev/sdc1*：

```
$ sudo umount /mnt
```

### 将新分区挂载到 */home*

将新分区挂载到 */home* 目录：

```
$ sudo mount /dev/sdc1 /home
```

通过作为用户登录并检查一切是否按预期工作来对此操作进行真实测试。

### 创建 */etc/fstab* 条目

*/home* 目录在系统重启后不会自动挂载，因为 */home* 在不在 */etc/fstab* 中列出的另一个分区上。您必须为其创建一个 */etc/fstab* 条目。

我的 */etc/fstab* 条目为 */dev/sdc1* 如下：

```
/dev/sdc1    /home                ext4    defaults    0 0
```

此 */etc/fstab* 条目使挂载持久化。

下一部分涉及将共享目录中“过时”和重复的文件归档和删除。

# 清理共享目录

保持共享目录无杂物是困难的。可能是不可能的，但您可以使用系统工具和一些计划来解决这个问题。本节探讨了一些可以简化清理和维护的实用程序。

## 使用 fdupes 进行文件去重

`fdupes` 是一个流行的重复文件删除实用程序。`fdupes` 可以在给定的一组目录中找到重复的文件。[`fdupes` 手册页面](https://oreil.ly/XERaf)描述了该命令的功能如下：

> 在给定路径搜索重复文件。这些文件通过比较文件大小和 MD5 签名来找到，然后进行逐字节比较。

您可以从您的发行版仓库安装 `fdupes`，但最好使用以下步骤从 GitHub 仓库获取修补版本：

```
$ git clone https://github.com/tobiasschulz/fdupes
$ cd fdupes
$ make fdupes
$ sudo make install
```

这个打补丁的版本有更多选项，特别是在生产环境中作为系统管理员最有可能使用的选项之一。当你在一个目录上使用 `fdupes` 时，你可以只查看重复项，删除重复项（不推荐），或删除重复项并为其中一个文件创建软链接（推荐）。只有当原始文件不存在时，这种解决方案才会出现问题。但是提供一个文件的软链接比删除重复项要少得多，后者可能导致工单、电话和训斥。示例 8-1 到 8-3 展示了如何使用 `fdupes` 查找重复文件，生成报告，然后仅为其中一个文件提供链接。

##### 示例 8-1\. 列出重复文件及其大小

```
$ fdupes -rS /opt/shared
29 bytes each:                         
/opt/shared/docs/building/test.rtf
/opt/shared/docs/building/test.txt
/opt/shared/a/b/list.txt
/opt/shared/a/b/c/stuff.doc
/opt/shared/a/todo.list
/opt/shared/one/foo.lst
/opt/shared/x/y/listed.here

20 bytes each:
/opt/shared/docs/hr/list.txt
/opt/shared/x/got.txt
/opt/shared/a/b/none.doc
```

`-r` 选项是递归的，这意味着检查指定目录及其所有子目录。`-S` 选项表示报告文件大小。

在 示例 8-2 中，`-m` 选项告诉 `fdupes` 打印任何找到的重复文件的报告。

##### 示例 8-2\. 使用 `-m` 选项生成重复文件报告

```
$ fdupes -mr /opt/shared
8 duplicate files (in 2 sets), occupying 214 bytes.
```

在 示例 8-3 中，`fdupes` 使用 `-L` 选项用硬链接替换重复文件。每组重复文件中仅保留第一个找到的文件。

##### 示例 8-3\. 为其中一个文件提供链接

```
$ fdupes -rL /opt/shared
[+] /opt/shared/docs/building/test.rtf
[h] /opt/shared/docs/building/test.txt
[h] /opt/shared/a/b/list.txt
[h] /opt/shared/a/b/c/stuff.doc
[h] /opt/shared/a/todo.list
[h] /opt/shared/one/foo.lst
[h] /opt/shared/x/y/listed.here

[+] /opt/shared/docs/hr/list.txt
[h] /opt/shared/x/got.txt
[h] /opt/shared/a/b/none.doc
```

使用选择性删除选项 `-d`，你会被提示选择要保留的文件。再次强调，不建议删除文件。请自行决定是否继续：

```
$ fdupes -rd /opt/shared
[1] /opt/shared/a/b/goo.txt            
[2] /opt/shared/a/false.doc
[3] /opt/shared/one/two/three/three
[4] /opt/shared/docs/building/test.rtf
[5] /opt/shared/x/y/new.txt
[6] /opt/shared/docs/building/test.txt
[7] /opt/shared/a/b/list.txt
[8] /opt/shared/a/b/c/stuff.doc
[9] /opt/shared/a/todo.list
[10] /opt/shared/one/foo.lst
[11] /opt/shared/x/y/listed.here

Set 1 of 2, preserve files [1 - 11, all]:
```

此选项除了保留的文件外都会删除文件，并且不会创建链接。查看 `fdupes` 的在线帮助和手册页面获取其他选项。

## 处理 /home 文件过度增长问题与配额

一些系统管理员和用户认为使用配额对付文件过度增长是一个严厉的解决方案。我认为它是一个在各种情况下都很好用的解决方案，可以帮助处理那些经常占用超过其应有份额的磁盘空间的用户。你可以在任何目录上实施配额，但肯定要考虑为共享目录和 */home* 目录设置配额。

第一个任务是安装 `quota`，如果你的系统还没有的话：

```
$ sudo yum install quota
$ sudo apt install quota
```

要使用 `quota` 系统，你需要准备并挂载一个 `xfs` 文件系统。例如，我的挂载在 */home* 下。为 `xfs` 文件系统在 */etc/fstab* 中创建一个条目。以下是 */etc/fstab* 中关于 */home* 的条目，我希望强制执行配额：

```
/dev/sdc1    /home                xfs    defaults,usrquota,grpquota    0 0
```

接下来，在 */home* 下创建两个新的空文件：*quota.group* 和 *quota.user*。在任何配置了配额的目录中，这两个文件必须存在，配额系统使用它们：

```
$ sudo touch /home/quota.group /home/quota.user
```

然后，使用 `quotaon` 命令启用配额：

```
$ sudo quotaon /home
quotaon /home
quotaon: Enforcing group quota already on /dev/sdc1
quotaon: Enforcing user quota already on /dev/sdc1
quotaon: Enable XFS project quota accounting during mount
```

系统现在已准备好为用户账户应用配额。 以下命令为演示目的设置了一个非常低的`quota`值。 软限制为 50 MB，硬限制为 80 MB。 当您超过软限制时，您会收到警告，但`quota`系统会阻止您超过硬限制。 您还可以限制用户可能消耗的索引节点（文件元数据结构）的数量。 索引节点限制可能更严格，因为用户可能生成成千上万个只占用几兆字节的小文本文件，但它限制了用户可以创建的文件数量。 索引节点也有硬限制和软限制。

```
[root@server1 home1]# sudo xfs_quota -x -c \
'limit -u bsoft=50m bhard=80m isoft=60 \ ihard=80 djones' /home
```

以下是用户在超过配额时看到的示例。 `head`命令创建一个 51 MB 的文件，以说明配额如何截断超过设定配额限制的文件：

```
$ su - djones
Password:
Last login: Mon Jan 10 20:16:49 CST 2022 on pts/0
[djones@server1 ~]$ head -c 51MB /dev/urandom > fillit.txt
[djones@server1 ~]$ head -c 51MB /dev/urandom > fillit1.txt
head: error writing 'standard output': Disk quota exceeded
$ ls -l
total 81892
-rw-rw-r--. 1 djones djones 32854016 Jan 10 20:21 fillit1.txt
-rw-rw-r--. 1 djones djones 51000000 Jan 10 20:21 fillit.txt
```

您可以看到，当用户达到他们的硬限制时，系统截断了他们试图创建的文件（*fillit1.txt*）。 如果用户没有设置配额，他们将拥有两个 51 MB 的文件。

配额防止用户在文件系统或目录上消耗超过一定量的空间。 但它们只应对违反公司政策或在公司系统上极端存储个人文件的用户执行。 您可以通过将所有限制设置为`0`轻松地从用户账户中删除配额限制：

```
$ sudo xfs_quota -x -c 'limit -u bsoft=0 bhard=0 isoft=0 ihard=0 djones' /home
[sudo] password for khess:
[khess@server1 home]$ su - djones
Password:
Last login: Mon Jan 10 20:20:41 CST 2022 on pts/0
[djones@server1 ~]$ head -c 51MB /dev/urandom > fillit2.txt
[djones@server1 ~]$ ls -l
total 131700
-rw-rw-r--. 1 djones djones        0 Jan 10 20:21 blah.txt
-rw-rw-r--. 1 djones djones 32854016 Jan 10 20:21 fillit1.txt
-rw-rw-r--. 1 djones djones 51000000 Jan 10 22:37 fillit2.txt
-rw-rw-r--. 1 djones djones 51000000 Jan 10 20:21 fillit.txt
```

现在关于超过限制的警告（或截断）已经消失。 `djones`账户不再对其施加配额限制。

这是对配额的一个很好的介绍，作为管理员，它们能为您做些什么，以及它们如何通过空间使用情况来控制您的用户。 接下来要探讨的话题是补丁，这是任何想要保持系统顺畅运行的管理员必不可少的任务。

# 补丁：保持系统健康的途径

对于新的系统管理员来说，补丁听起来可能像是在系统上放置胶带，暂时将其固定在一起，同时您寻找问题的真正解决方案。 补丁是一项基本任务，也许对于它的称呼并不准确：更新由开发者提供的系统实用程序、守护进程和应用程序修复。 软件补丁修复了特定的问题。 一个很好的例子是对 SSH 守护进程应用补丁以解决新的漏洞，这可能使系统容易受到攻击和妥协。

补丁并不总是专注于安全。通常，补丁修复稳定性问题、堵塞内存泄漏或修复损坏的功能。请记住，对 Linux 系统进行补丁几乎不意味着您必须启动重新启动，但如果更新了一个服务，您将需要重新启动该服务，除非补丁过程已经为您处理了这一点。如果补丁过程重新启动了一个服务，屏幕上将出现消息通知您重新启动。为了安全起见，在主要补丁事件后，我经常重新启动以确保系统及其服务得到重新启动。*主要*补丁事件包括多个修复以及通常的内核更新。如果您在维护窗口内并更新了多个服务、内核或其他关键系统软件，我建议您重新启动系统。

一些系统管理员通过设置`cron`作业来自动化他们的补丁管理，定期检查并安装补丁。这种场景有效，但你应该有一个或两个测试系统，手动更新以查看需要更新的内容，系统对这些更新的反应以及重启后系统的响应。你不希望也不需要因为糟糕的补丁而遇到任何意外。

现在，让我们看看如何对两种类型的 Linux 系统进行补丁管理——基于 Red Hat 的系统和基于 Debian 的系统。

## **Red Hat Enterprise Linux-Based 系统的补丁**

要在基于 Red Hat Enterprise Linux 的系统上启动补丁过程，请使用`yum`或`dnf`。在此演示中，我使用`yum`，但在 Red Hat Enterprise Linux 8 及更高版本中，`dnf`是首选命令：

```
$ sudo yum update
Last metadata expiration check: 0:23:24 ago on Sun 30 Jan 2022 08:27:15 AM CST.
Dependencies resolved.
========================================================================
 Package             Arch        Version            Repo          Size
========================================================================
Installing:
 kernel              x86_64      4.18.0-348...      baseos        7.0 M
 kernel-core         x86_64      4.18.0-348...      baseos        38 M
 kernel-modules      x86_64      4.18.0-348...      baseos        30 M
Upgrading:
 NetworkManager      x86_64      1:1.32.10-...      baseos        2.6 M

...

 openssh             x86_64      8.0p1-10.el8       baseos        522 k
 openssh-clients     x86_64      8.0p1-10.el8       baseos        668 k
 openssh-server      x86_64      8.0p1-10.el8       baseos        485 k
 openssl             x86_64      1:1.1.1k-5...      baseos        709 k
 openssl-libs        x86_64      1:1.1.1k-5...      baseos        1.5 M

...

 yum                 noarch      4.7.0-4.el8        baseos        205 k
 yum-utils           noarch      4.0.21-3.el8       baseos        73 k
Installing dependencies:
 libbpf              x86_64      0.4.0-1.el8        baseos        110 k
Removing:
 kernel              x86_64      4.18.0-240...      @BaseOS       0 
 kernel-core         x86_64      4.18.0-240...      @BaseOS       62 M
 kernel-modules      x86_64      4.18.0-240...      @BaseOS       21 M

Transaction Summary
========================================================================
Install    4 Packages
Upgrade  287 Packages
Remove     3 Packages

Total download size: 558 M
Is this ok [y/N]:
```

正如你从上面截断的列表中可以看到，我已经很久没有更新这个系统了（287 个软件包需要升级）。我包括了我想让你看到的部分，例如一个新的内核更新和新的 OpenSSH 服务器（SSHD），以演示我在更新后将重新启动此系统。我喜欢通过不提供`-y`选项手动安装，这样我可以在继续之前查看所有需要升级的内容。

###### **注意**

YUM/DNF 更新的默认行为是不安装它们。你将收到以下提示：

```
Is this ok [y/N]:
```

如果按下 Enter 键，更新将不会安装，因为`N`是默认响应。你必须在此处明确批准安装，通过输入`y`来确认。

如果你在测试系统上安装并验证了更新，通常可以放心在生产系统上继续更新。安装未在测试系统上验证的补丁可能会导致稳定性问题或应用程序/库冲突，特别是如果你在运行某些旧程序版本。

## **Debian-Based Linux 系统的补丁**

对于基于 Debian 的系统，补丁过程类似于基于 Red Hat Enterprise Linux 的系统，但有一些细微的差异。第一个差异是你使用`apt`命令来安装系统更新：

```
$ sudo apt update
Hit:1 http://us.archive.ubuntu.com/ubuntu focal InRelease
Get:2 http://us.archive.ubuntu.com/ubuntu focal-updates InRelease [114 kB]
Get:3 http://us.archive.ubuntu.com/ubuntu focal-backports InRelease [108 kB]
Get:4 http://us.archive.ubuntu.com/ubuntu focal-security InRelease [114 kB]
Fetched 336 kB in 1s (407 kB/s)  
Reading package lists... Done
Building dependency tree      
Reading state information... Done
20 packages can be upgraded. Run 'apt list --upgradable' to see them.
khess@server2:~$ sudo apt upgrade
Reading package lists... Done
Building dependency tree      
Reading state information... Done
Calculating upgrade... Done
The following packages will be upgraded:
  cloud-init command-not-found libasound2 libasound2-data libnetplan0 libssl1.1
linux-firmware netplan.io openssh-client openssh-server openssh-sftp-server ...
python3-commandnotfound python3-software-properties rsync software-propertie... 
ubuntu-advantage-tools ufw update-notifier-common wget
20 upgraded, 0 newly installed, 0 to remove and 0 not upgraded.
Need to get 120 MB of archives.
After this operation, 20.3 MB of additional disk space will be used.
Do you want to continue? [Y/n]
```

另一个不同之处在于批准程序。请注意`是否要继续？[Y/n]`提示符。`Y`是大写的，这意味着如果您按 Enter 键，更新将安装在您的系统上。请记住，在更新基于 Red Hat Enterprise Linux 的系统时，如果按 Enter 键，更新将不会安装。但请注意，`apt`没有像`yum`或`dnf`那样的自动批准选项（`-y`）。

###### 注意

`unattended-upgrades`是一个可以安装的包，它将允许您自动更新基于 Debian 的系统。

如前所述，您应始终首先在测试系统上安装更新和升级。如果您设置了自动更新，除非您为更新系统创建一个类似以下的巧妙时间表，否则这将更加困难：

测试系统

每周二手动更新

开发系统

每隔两周的星期四手动更新

生产系统

每月更新每月最后一个星期日

这个计划或类似计划允许您在将自动更新安装到生产系统之前检查它们。如果您有一个允许您在出现问题时暂停`cron`作业的自动化或企业管理工具，您可以更好地控制自动修补。如果您没有这样的工具并且需要管理多个系统，我建议您探索像 Ansible 这样的自动化工具，以便向您的系统交付新的配置文件、脚本等。

当向您的系统应用零日或其他紧急补丁时，自动化系统也是有帮助的。像备份一样，打补丁需要您的密切关注。通过定期应用最新的补丁，保持系统平稳安全对您作为系统管理员的职责至关重要。

# 保护您的系统

尽管定期打补丁应成为整体安全计划的一部分，但与打补丁主题无关的一些基本安全设置我将在本节中进行介绍。同样，像 Ansible 这样的自动化系统可以大大节省维护几十、几百或几千个系统的时间和精力。

并非所有系统在应用安全措施时可以被平等对待。例如，您的 DMZ（面向互联网）系统、数据库系统、Web 服务器、应用服务器和文件存储系统每个（作为一个组）都需要不同的安全设置。您几乎不可能对每种类型的系统都应用单一的安全策略。一些通用的安全措施适用于每种系统，无论其功能如何，但声明您将设置一个统一的通用安全计划是不现实的。您必须为每种系统类型应用安全配置，以便专注于特定的漏洞。

例如，你能否想到适用于 Web 服务器但不适用于文件服务器的特定安全设置？在 Web 服务器的情况下，您需要实施证书来保护您的 Web 通信。您应该集中精力保护文件服务器上的共享目录和用户帐户。您是否打算在文件服务器上允许 Samba 共享？您是否打算允许向 Web 服务器上传文件？您和您的用户如何与系统交互决定了其安全问题，而不是它是一个通用的 Linux 系统的事实。

表 8-1 展示了针对具有特定目的和工作负载的 Linux 系统的一些安全指南示例。

表 8-1\. Linux 服务安全指南

| 目的/工作负载 | 示例 | 安全措施 |
| --- | --- | --- |
| 所有服务器 | 杂项服务 | SSH, */etc/hosts.allow* 和 */etc/hosts.deny* 限制, 防火墙, SELinux |
| Web 服务器 | Apache, NGINX, IHS, 其他 | TLS, 证书, HTTPS |
| 数据库服务器 | MySQL, PostgreSQL, 其他 | SSH 隧道, 限制连接到本地主机 |
| 文件服务器 | Samba 共享, NFS 共享, SSH | 为 Samba 集成的 Active Directory, 双因素身份验证 |
| 应用服务器 | Tomcat, WebSphere, 其他 | HTTPS, TLS, 证书 |
| 邮件服务器 | SMTP, IMAP, POP-3 协议 | 使用安全协议和邮件服务 |

如您在表 8-1 中所见，每个系统都应实施某些安全功能，无论其用途如何，除非有某些供应商相关的原因禁止此操作。您应该尽可能地锁定每个系统，并在保持用户功能和可访问性的同时保持严格限制。

# 维护用户和群组帐户

用户帐户维护不仅仅是添加和删除帐户。它还包括制定帐户命名约定、创建设定禁用帐户多久后删除的政策、在用户离开组或公司后保留用户主目录的时间标准、群组帐户的撤销以及用户和群组 ID 的撤销或重复使用。所有这些活动都可以防止用户帐户泛滥。以下是帐户泛滥的一些示例：

+   系统中超过策略限制而保留的已禁用帐户

+   系统中超过策略限制而保留的个人目录

+   系统中保留的活跃和已终止的用户帐户

+   没有成员的群组帐户

+   在 */etc/sudoers* 中仍存在的已禁用或已删除帐户

+   系统中保留的活跃临时帐户

+   具有 shell 或密码的活跃服务帐户

## 设置命名约定

防止用户账户蔓延的最简单方法之一是创建用户账户命名约定。命名约定通过设定一套标准，使你和你的团队可以创建用户账户。例如，如果你将命名约定设置为姓氏的第一个字母和姓氏的前七个字符，现在你有了一个可用的系统。例外情况是当两个或更多人拥有相同的名字和姓氏时。在这些情况下，你会选择使用名字的第一个字母、中间字母和姓氏的前六个字符。

表 8-2 展示了标准命名约定的示例。

表 8-2\. 标准账户命名约定

| 名字 | 姓氏 | 用户名 |
| --- | --- | --- |
| 何塞 | 阿尔瓦雷斯 | `jalvarez` |
| 保拉 | 安德森 | `panderso` |
| 维克 | 库恩德拉 | `vkundra` |
| 西尔维亚 | 戈德斯坦 | `sgoldste` |

表 8-3 展示了重复姓名的例外情况的示例。

表 8-3\. 重复姓名的例外情况

| 名字 | 姓氏 | 用户名 |
| --- | --- | --- |
| 何塞 | 阿尔瓦雷斯 | `jalvarez` |
| 何塞 | 阿尔瓦雷斯 | `jqalvare` |
| 保拉 | 安德森 | `panderso` |
| 保拉 | 安德森 | `pmanders` |

如果你的用户没有中间名字，那么你可以按照字母表的倒序来进行，如表 8-4 所示。

表 8-4\. 没有中间名的用户的账户名解决方案

| 名字 | 姓氏 | 用户名 |
| --- | --- | --- |
| 保拉 | 安德森 | `panderso` |
| 保拉 | 安德森 | `pzanders` |
| 保拉 | 安德森 | `pyanders` |
| 保拉 | 安德森 | `pxanders` |

当然，你可以制定自己的命名约定，但我已经看到其他企业采用过这种方法，所以在这里值得一提。我还看到其他公司使用数字系统来避免重复的用户名。例如，如果有两个名为 Vivek Kundra 的用户，那么系统中创建的第一个将被创建为`vkundra`，第二个 Vivek Kundra 则为`vkundra2`。

命名约定的整个目的是开发一个系统，然后坚持使用它。有了命名约定，管理员就不会随意创建可能重复其他账户的用户账户。

例如，假设你为 Sylvia Goldstein 创建了一个用户账户`sgoldste`，另一位系统管理员创建了一个账户`syliag`。现在你面临一个安全问题，因为在同一个系统上有一个用户有两个账户，还有用户账户蔓延的问题。

用户账户蔓延不是一个小问题。假设 Sylvia 登录并使用两个账户来创建文件、安装软件，可能为她所属的群组创建依赖关系。现在的问题比简单地将文件从一个账户移动到另一个账户、更改用户和群组所有权、从系统中删除使用较少的账户要复杂得多。

如果系统管理员有一个命名规则，第二个管理员可能不会创建第二个账户。接下来，我将讨论创建账户保留政策，这是另一种防止账户扩展的方法。

## 创建账户保留政策

账户保留政策是一种良好的安全实践和日常管理实践。没有人在超过 90 天没有访问的活跃账户上悬挂安全扫描。一些高度安全的环境将强制每 45 天更改一次密码，在账户不活跃 90 天后禁用账户，并删除保持不活跃超过六个月的账户。这种激进的账户保留政策并不适合每个环境，但确实保护和挑战敏感系统用户维护或放弃他们的账户。

我并不建议您采用如此严格的政策。但是，您应该制定一种保留政策，并将其作为整体安全和雇佣教育计划的一部分提供给您的用户。

您的账户保留政策应该是一个书面政策，同时也是一个系统级政策。系统级政策意味着您必须配置全系统范围的不活跃账户锁定和删除设置。

### 更改不活跃账户状态

为了朝向系统解决方案迈进，请审计密码过期后系统禁用账户的默认天数，也称为`INACTIVE`变量：

```
$ sudo useradd -D | grep INACTIVE
INACTIVE=-1
```

`-1`表示未设置默认的不活跃值。

根据公司的安全政策设置无活动值。如果没有涉及此值的安全政策，将其设置为`15`天。有些系统管理员将此值设置为`30`。此设置的目的是禁用不活跃账户并保护系统安全：

```
$ sudo useradd -D -f 15
```

这将系统范围内的默认不活跃值设置为`15`天。如果用户收到更新密码的消息，则在系统禁用账户之前有 15 天的时间：

```
$ sudo useradd -D | grep INACTIVE
INACTIVE=15
```

如果用户尝试登录系统并失败，您可以检查他们的账户状态：

```
$ sudo passwd -S *username*
$ sudo passwd -S ndavis
ndavis PS 2022-02-13 0 99999 7 15 (Password set, SHA512 crypt.)
```

此用户账户状态正常。也许他们尝试登录到一个不是他们打算的主机，或者输入了一个错误的密码。但是，另一位用户`asmith`向您投诉说他无法登录相同的系统：

```
$ sudo passwd -S asmith
asmith LK 2022-02-12 0 99999 7 15 (Password locked.)
```

用户`asmith`被系统锁定。您需要解锁他们的账户，然后他们才能再次登录：

```
$ sudo passwd -u asmith
passwd: Success
```

用户可以再次登录。接下来，我将演示如何使用`chage`命令进一步保护账户。

### 保护用户账户

`chage`命令有几个选项，可进一步保护用户账户，例如强制定期更改密码、设置最小密码更改持续时间等。本节展示如何修改这些设置以保护您的用户和系统。

我在管理的系统上为所有用户更改的三个参数是 `不活跃天数`、`密码更改间隔的最小天数` 和 `密码更改间隔的最大天数`。我根据公司政策进行更改，如果有的话。否则，我会按照我的“默认”进行更改：

```
Inactive days:                                      15
Minimum number of days between password changes:    1
Maximum number of days between password changes:    90

$ sudo chage -m 1 -M 90 --inactive 15 asmith

$ sudo chage --list asmith
Last password change                                : Feb 13, 2022
Password expires                                    : May 14, 2022
Password inactive                                   : May 29, 2022
Account expires                                     : never
Minimum number of days between password change      : 1
Maximum number of days between password change      : 90
Number of days of warning before password expires   : 7
```

这会使用户的密码每 90 天过期一次。如果他们不更改密码，则在更改密码日期后的 15 天内其账户将被禁用。如果他们在提示时更改密码，则在一天后才能再次更改密码。

您只能使用 `chage` 命令逐个更改用户。如果您有多个用户和几台服务器，这可能会变得很麻烦。一个覆盖所有用户的脚本版本要高效得多。我提供了我的脚本如下：

```
#!/bin/bash
egrep ^[^:]+:[^\!*] /etc/shadow | cut -d: -f1 | grep -v root > user-list.txt
for user in `more user-list.txt`
do
chage -m 1 -M 90 -I 15 $user
done
```

## 退休组账户

一旦组为空且不活跃，仍保留在系统中的组账户是账户扩展的常见来源，但很容易修复。`groupmems` 命令可以通过回答特定组中的成员是谁来为您节省大量时间和挫折感：

```
$ sudo groupmems -g operations -l
ajones bhaas

$ sudo groupmems -g hr -l
asmith
```

本示例中使用的 `groupmems` 命令选项为 `-g` 表示组名，`-l` 表示列表。

如果没有组成员，则命令不显示任何输出：

```
$ sudo groupmems -g engineering -l
$
```

在此系统上，应该退休工程师组（从系统中移除因为空）。但在移除组之前，您必须弄清楚工程师组是否留下了任何他们拥有的文件：

```
$ sudo find / -group engineering
find: '/proc/3368/task/3368/fd/7': No such file or directory
find: '/proc/3368/task/3368/fdinfo/7': No such file or directory
find: '/proc/3368/fd/6': No such file or directory
find: '/proc/3368/fdinfo/6': No such file or directory
/shared/engineering
/shared/engineering/one
/shared/engineering/two
/shared/engineering/three
/shared/engineering/four
/shared/engineering/five
/shared/engineering/six
/shared/engineering/seven
/shared/engineering/eight
/shared/engineering/nine
/shared/engineering/ten
```

您应将文件转移到另一个用户或更改所有权为 root 以确保其安全，然后移除空组账户：

```
$ sudo groupdel engineering
```

您已成功移除了工程师账户。如果需要，您可以将其来日重建。

现在我已经讨论了扩展问题，接下来将讨论监控系统健康。

# 监控系统健康

系统监控并不明确属于扩展问题，但是它是维护系统健康的一部分。有数十种商业监控工具可供购买，但 `sysstat` 是 Linux 上的一个免费工具。它易于安装，在基于 Red Hat 的系统上，它会自动配置并立即开始收集数据。在基于 Debian 的系统上，您必须手动启用 `sysstat` 来收集数据。

要启用 `sysstat` 的系统活动报告（`sar`）工具以开始收集数据，请编辑 */etc/default/sysstat* 文件，并将 `ENABLED="false"` 改为 `ENABLED="true"`。

然后重新启动 `sysstat` 服务：

```
$ sudo service sysstat restart
```

给系统一些时间开始生成活动报告。稍后在本章中您将学习如何生成活动报告。

`sysstat` 软件包包含用于检查性能和格式化系统统计信息的新二进制文件。表 8-5 列出了这些二进制文件及其功能。我从各自的 man 页面复制了每个二进制文件的描述。

表 8-5\. `sysstat` 软件包中的二进制文件

| 二进制文件 | 名称 | 描述 |
| --- | --- | --- |
| `cifsiostat` | CIFS 统计 | `cifsiostat`命令显示关于 CIFS 文件系统读取和写入操作的统计信息。 |
| `iostat` | CPU 和设备 I/O 统计 | `iostat`命令用于通过观察设备的活动时间与其平均传输速率的关系来监视系统输入/输出设备负载。 |
| `mpstat` | 处理器相关统计 | `mpstat`命令会将每个可用处理器的活动写入标准输出，其中处理器 0 是第一个。 |
| `pidstat` | 任务统计 | `pidstat`命令监视 Linux 内核当前管理的各个任务。 |
| `sadf` | `sar`数据格式化 | `sadf`命令用于显示由`sar`命令创建的数据文件的内容。但与`sar`不同，`sadf`可以将其数据写入许多不同的格式（CSV、XML 等）。 |
| `sar` | 系统活动报告 | `sar`命令会将操作系统中选定的累积活动计数器的内容写入标准输出。 |
| `tapestat` | 磁带统计 | `tapestat`命令用于监视连接到系统的磁带驱动器的活动。 |

如果你熟悉`vmstat`命令，它不是`sysstat`的一部分，你就知道其它“stat”命令的工作原理。例如，在每个 Linux 系统上，`vmstat`要求你提供以秒为间隔的时间延迟快照和一个特定数量（计数），典型值为`5`：

```
vmstat [*options*] [delay [*count*]]
```

`vmstat`命令的第一次运行给出自上次重启以来的平均值：

```
$ vmstat 5 5
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id  wa st
 3  0  44616 343888    244 334676    0    0     1     1    1   24  0  0 100  0  0
 0  0  44616 343768    244 334676    0    0     0     0   48   91  0  0 100  0  0
 0  0  44616 343768    244 334676    0    0     0     2   53  104  0  0 100  0  0
 1  0  44616 343768    244 334676    0    0     0     4   78  141  0  0 100  0  0
 0  0  44616 343768    244 334676    0    0     0     0   75  141  0  0 100  0  0
```

每隔五秒钟运行一次`vmstat`命令，共采集五个快照。其它“stat”命令也是类似的工作方式。你需要提供一个延迟和一个计数。以下是一个包含两个快照和五秒延迟的`iostat`命令示例：

```
$ iostat 5 2
Linux 4.18.0-348.7.1.el8_5.x86_64 (server1)   02/13/2022   _x86_64_  (1 CPU)

avg-cpu:  %user   %nice %system %iowait  %steal   %idle
           0.01    0.00    0.24    0.01    0.00   99.74

Device             tps    kB_read/s    kB_wrtn/s    kB_read    kB_wrtn
sda               0.16         0.98         1.34    1592993    2187576
scd0              0.00         0.00         0.00          1          0
sdc               0.00         0.01         0.00       8975       2786
sdb               0.00         0.00         0.00       2921       2048
dm-0              0.17         0.94         1.33    1529818    2170543
dm-1              0.02         0.02         0.06      38232      94228
loop0             0.00         0.00         0.00       1192          0
loop1             0.00         0.00         0.00        253          0
loop2             0.00         0.00         0.00       1202          0
dm-2              0.00         0.00         0.00       1223       2048

avg-cpu:  %user   %nice %system %iowait  %steal   %idle
           0.00    0.00    0.50    0.00    0.00   99.50

Device             tps    kB_read/s    kB_wrtn/s    kB_read    kB_wrtn
sda               0.00         0.00         0.00          0          0
scd0              0.00         0.00         0.00          0          0
sdc               0.00         0.00         0.00          0          0
sdb               0.00         0.00         0.00          0          0
dm-0              0.00         0.00         0.00          0          0
dm-1              0.00         0.00         0.00          0          0
loop0             0.00         0.00         0.00          0          0
loop1             0.00         0.00         0.00          0          0
loop2             0.00         0.00         0.00          0          0
dm-2              0.00         0.00         0.00          0          0
```

## 收集系统活动报告

`sar`命令在系统活动信息的收集、报告和保存方面既全面又灵活。你可以像“stat”命令那样使用它，提供一个间隔和一个计数，使用选项如`-b`来显示 I/O 统计信息，或者结合两者在特定间隔内显示所选性能计数器。

我将以一个带有开关的`sar`使用示例开始。继续使用`-b`示例，以下是部分 I/O 统计报告：

```
$ sar -b
Linux 4.18.0-348.7.1.el8_5.x86_64 (server1)   02/13/2022   _x86_64_  (1 CPU)

12:00:15 AM       tps      rtps      wtps   bread/s   bwrtn/s
12:10:15 AM      0.22      0.03      0.19      2.32      4.30
12:20:15 AM      0.10      0.00      0.10      0.00      0.79
12:30:15 AM      0.16      0.00      0.16      0.00      1.76
12:40:15 AM      0.08      0.00      0.08      0.00      0.5
...
12:20:15 PM      0.08      0.00      0.08      0.00      0.73
12:30:15 PM      0.10      0.00      0.10      0.00      0.87
12:40:15 PM      0.08      0.00      0.08      0.00      0.78
12:50:15 PM      0.11      0.00      0.11      0.00      0.90
Average:         0.25      0.11      0.14      6.29      2.35
```

如报告所示，数据显示从午夜开始，这就是我不得不截断它的原因。以下是一个使用`-b`，计数为五，间隔为五秒的 I/O 统计数据示例：

```
$ sar -b 5 5
Linux 4.18.0-348.7.1.el8_5.x86_64 (server1)   02/13/2022   _x86_64_  (1 CPU)

12:57:13 PM       tps      rtps      wtps   bread/s   bwrtn/s
12:57:18 PM      0.00      0.00      0.00      0.00      0.00
12:57:23 PM      0.00      0.00      0.00      0.00      0.00
12:57:28 PM      0.00      0.00      0.00      0.00      0.00
12:57:33 PM      0.00      0.00      0.00      0.00      0.00
12:57:38 PM      0.00      0.00      0.00      0.00      0.00
Average:         0.00      0.00      0.00      0.00      0.00
```

最后，你可以使用`sar`命令和一个计数以及一个间隔：

```
$ sar 5 5
Linux 4.18.0-348.7.1.el8_5.x86_64 (server1)   02/13/2022   _x86_64_  (1 CPU)

01:02:11 PM     CPU     %user     %nice   %system   %iowait    %steal     %idle
01:02:16 PM     all      0.00      0.00      0.20      0.00      0.00     99.80
01:02:21 PM     all      0.00      0.00      0.40      0.00      0.00     99.60
01:02:26 PM     all      0.00      0.00      0.20      0.00      0.00     99.80
01:02:31 PM     all      0.00      0.00      0.20      0.00      0.00     99.80
01:02:36 PM     all      0.00      0.00      0.20      0.00      0.00     99.80
Average:        all      0.00      0.00      0.24      0.00      0.00     99.76
```

###### 注意

`sar`命令的默认输出是 CPU 统计信息。要显示其他统计信息，你必须使用一个开关，例如用于 I/O 数据的`-b`。

接下来的部分展示如何运行系统活动报告并将`sar`数据保存到文件中。

## 格式化系统活动报告

显示`sar`数据能让您看到过去 24 小时的快照和性能情况，但如果您想以特定格式（如 CSV 或 XML）将这些数据捕获到文件中（系统活动报告），则可以使用`sadf`命令。

为了帮助区分`sadf`和`sar`选项，请记住在命令中首先出现`sadf`选项，然后是`sar`选项跟随双破折号。请参考 man 手册获取这两个命令的所有可用选项。

我最喜欢的`sadf`命令使用`-d`开关来创建逗号分隔的文件，非常适合导入数据库：

```
$ sadf -d
# hostname;interval;timestamp;CPU;%user;%nice;%system;%iowait;%steal;%idle
server1;600;2022-02-13 06:10:15 UTC;-1;0.01;0.00;0.24;0.01;0.00;99.74
server1;600;2022-02-13 06:20:15 UTC;-1;0.00;0.00;0.21;0.01;0.00;99.78
server1;600;2022-02-13 06:30:15 UTC;-1;0.01;0.02;0.23;0.01;0.00;99.74
server1;600;2022-02-13 06:40:15 UTC;-1;0.00;0.00;0.22;0.01;0.00;99.77
```

系统活动日志位于*/var/log/sa*目录下，并以“sa”后跟当天日期的数字作为文件名。例如，用于当月第 10 天的系统活动日志数据文件是*sa10*。您可以指定从其他日期的活动日志数据文件：

```
$ sadf -d /var/log/sa/sa10 -- -d
# hostname;interval;timestamp;DEV;tps;rkB/s;wkB/s;areq-sz;aqu-sz;await;...
server1;600;2022-02-10 06:10:15 UTC;dev8-0;0.35;8.46;2.17;30.81;0.00;0.62;...
server1;600;2022-02-10 06:10:15 UTC;dev11-0;0.00;0.00;0.00;0.00;0.00;0.00;...
server1;600;2022-02-10 06:10:15 UTC;dev8-32;0.00;0.00;0.00;0.00;0.00;0.00;...
server1;600;2022-02-10 06:10:15 UTC;dev8-16;0.00;0.00;0.00;0.00;0.00;0.00;...
server1;600;2022-02-10 06:10:15 UTC;dev253-0;0.30;8.35;2.27;35.42;0.00;0.90;...
server1;600;2022-02-10 06:10:15 UTC;dev253-1;0.03;0.11;0.00;4.00;0.00;1.00;...
server1;600;2022-02-10 06:10:15 UTC;dev7-0;0.00;0.00;0.00;0.00;0.00;0.00;...
server1;600;2022-02-10 06:10:15 UTC;dev7-1;0.00;0.00;0.00;0.00;0.00;0.00;...
server1;600;2022-02-10 06:10:15 UTC;dev7-2;0.00;0.00;0.00;0.00;0.00;0.00;...
server1;600;2022-02-10 06:10:15 UTC;dev253-2;0.00;0.00;0.00;0.00;0.00;0.00;...
```

###### 注释

在某些 Linux 发行版中，`sysstat`日志存储在*/var/log/sa*中，但在其他发行版中，这些日志则存储在*/var/log/sysstat*中。

`--`选项告诉`sadf`命令其后跟随的选项是`sar`命令选项，而非`sadf`选项。在命令`sadf -d /var/log/sa/sa10 -- -d`中，跟在`--`后的`-d`选项将显示`sar`设备信息。也许一个不那么令人困惑的例子是显示来自`sar`命令的网络信息：

```
$ sadf -d /var/log/sa03 -- -n DEV
# hostname;interval;timestamp;IFACE;rxpck/s;txpck/s;rxkB/s;txkB/s;rxcmp/s;...
server1;600;2022-02-10 06:10:15 UTC;lo;0.00;0.00;0.00;0.00;0.00;...
server1;600;2022-02-10 06:10:15 UTC;enp0s3;0.50;0.01;0.12;0.00;0.00;...
server1;600;2022-02-10 06:20:15 UTC;lo;0.00;0.00;0.00;0.00;0.00;...
server1;600;2022-02-10 06:20:15 UTC;enp0s3;0.42;0.00;0.10;0.00;0.00;...
server1;600;2022-02-10 06:30:15 UTC;lo;0.00;0.00;0.00;0.00;0.00;...
server1;600;2022-02-10 06:30:15 UTC;enp0s3;0.47;0.00;0.10;0.00;0.00;...
```

`-n`是网络活动的`sar`选项。`DEV`表示网络设备，如`lo`（环回）、`eth0`、`enp0s3`等。

# 摘要

我只是初步了解了系统健康的表面，而这一章可能需要写成一本整书。健康监控在维护客户 SLA 方面非常重要。跟踪系统健康是影响容量和性能、安全性以及标准系统维护的重要系统管理员任务。它如此重要，以至于一些大型企业专门成立了整个团队来处理这一任务。

在第九章中，我讨论了非健康系统监控和网络库存系统。
