# 第六章：安装和卸载软件

安装和卸载软件是基本的系统管理员任务。你可能不会每天执行这些任务，但这些是你和你的团队需要完成的常规任务。通常情况下，你会安装更新，这可以自动化进行。任何新安装的软件都应附有业务正当性、变更控制记录，并且从请求方获取的安全影响的书面理解（如果有的话）。安装已知存在漏洞的软件是恶意行为者侵入系统的一条简单路径。

卸载软件同样需要变更控制记录，因为移除某些其他关键系统或服务功能所需的包、目录或库可能会造成潜在危险。

有三种软件安装方法：使用包管理器从软件仓库安装、下载到本地文件系统的单独软件包安装，以及编译源代码。本章涵盖了这三种方法。卸载软件有两种标准方法：使用包管理工具和在编译软件情况下使用卸载过程。第三种非标准的卸载方法是通过移除目录、库和二进制文件手动卸载软件。

手动卸载软件是只有高级系统管理员才能执行的繁琐任务。本章的各节将教会你如何通过特定方法安装软件，然后使用相同方法卸载软件。

在讨论安装和卸载软件之前，我将向你展示如何更新你的系统。因为更新如此重要，值得优先讨论，并且你应在执行其他任务之前更新系统，因为更新系统的价值很高。在排查问题时，应始终更新系统，以检查简单的更新是否解决了你的问题。快速的系统更新可能会减轻需移除或安装新软件来解决问题的需求。

###### 注意

本章中的所有演示和示例均使用 CentOS 8.3（`server1`）和 Ubuntu Server 20.04 LTS（`server2`）。我首先在 `server1` 上执行所有任务，然后在 `server2` 上执行，并注意两个系统之间的任何差异。示例中使用的软件包是 Lynx，一个轻量级的基于文本的浏览器。

# 更新你的系统

我在本书中多次提到保持系统更新。这是一个重要的任务要记住。这应该是你的首要任务之一。更新是标准维护的一部分。许多系统管理员每周应用更新，这是一个良好的做法。但在需要时不要犹豫去应用补丁、更新和升级，以减轻漏洞的影响。安全是你的首要任务。接下来的两节说明了如何向你的系统应用更新。

## 应用基于 Red Hat Enterprise Linux 的系统更新

基于 Red Hat Enterprise Linux 的系统使用 YUM/DNF 实用程序来维护来自仓库的更新和软件安装。来自[官方 Red Hat 文档](https://oreil.ly/R2Mca)：

> YUM/DNF（yum/dnf）是获取、安装、删除、查询和管理来自官方 Red Hat 软件仓库以及其他第三方仓库的 Red Hat Enterprise Linux RPM 软件包的主要工具。YUM/DNF 用于 Red Hat Enterprise Linux 5 及更新版本。

DNF 是这个实用程序的最新版本，所以我已将两者结合起来。根据文档，DNF 是 YUM 的第四版，是从 Red Hat Enterprise Linux 版本 8 开始使用的工具。要开始更新，请发出`yum`或`dnf`命令：

```
$ sudo yum update
Last metadata expiration check: 1:52:37 ago on Sun 07 Nov 2021 07:14:51 PM CST.
Dependencies resolved.
=================================================================================
 Package                         Architecture    Version      Repository     Size
=================================================================================
Installing:
 kernel                          x86_64          4.18.0...    baseos        5.9 M
 kernel-core                     x86_64          4.18.0...    baseos         36 M
 kernel-modules                  x86_64          4.18.0...    baseos         28 M
Upgrading:
 NetworkManager                  x86_64          1:1.30...    baseos        2.6 M
 NetworkManager-config-server    noarch          1:1.30...    baseos        129 k
 NetworkManager-libnm            x86_64          1:1.30...    baseos        1.8 M
 NetworkManager-team             x86_64          1:1.30...    baseos        146

Removing:
 kernel                          x86_64          4.18.0...    @BaseOS         0 
 kernel-core                     x86_64          4.18.0...    @BaseOS        60 M
 kernel-modules                  x86_64          4.18.0...    @BaseOS        20 M

Transaction Summary
=================================================================================
Install   21 Packages
Upgrade  256 Packages
Remove     3 Packages

Total download size: 404 M
Is this ok [y/N]:
```

同意在此安装，以便将您的目标软件包升级到最新的稳定版本。为了自动化后续更新，使用`-y`选项来回答任何提示时选择“是”。以下演示了如何在`dnf`命令中使用`-y`选项：

```
$ sudo dnf -y update
```

这将自动接受安装而不会与您进行交互。这是在脚本中使用的一个很好的选择。接下来的部分将为您提供在基于 Debian 的系统上进行等效更新操作的方法。

## 应用基于 Debian 的系统更新

对于基于 Debian 的系统，您可以使用类似于 Red Hat Enterprise Linux 系统中使用的 DNF 命令的命令来应用更新，使用 Debian 的`apt`实用程序，如下所示。如果您的系统需要更新，您也将收到类似的响应。

```
$ sudo apt update
```

如果您的系统不需要更新，`apt`的响应将类似于以下内容：

```
Hit:1 http://us.archive.ubuntu.com/ubuntu focal InRelease
Get:2 http://us.archive.ubuntu.com/ubuntu focal-updates InRelease [114 kB]
Get:3 http://us.archive.ubuntu.com/ubuntu focal-backports InRelease [101 kB]
Get:4 http://us.archive.ubuntu.com/ubuntu focal-security InRelease [114 kB]
Fetched 328 kB in 52s (6,311 B/s)
Reading package lists... Done
Building dependency tree      
Reading state information... Done
All packages are up to date.
```

对于补丁、安全更新或应用程序版本升级没有特别的更新命令；此单一命令将处理所有类型或优先级的更新。您的系统将从所有配置的仓库检查更新，并在可用时应用它们。定期每周或更频繁地检查更新，并在计划的维护时间窗口内应用更新或根据关键的安全更新需要进行应用，是一种良好的做法。

本章的其余部分侧重于根据您的用户、管理或其他来源的按需服务请求安装软件。

# 从仓库安装软件

从仓库安装软件是在 Linux 系统上安装软件的最简单方法。之所以它是最简单的方法，是因为仓库会自动满足您的依赖关系，您无需做任何事情，只需请求安装。例如，如果您想在系统上安装 Apache HTTP 服务器，则需要满足几个依赖项才能安装。仓库包含所有依赖包，根据需要调用并安装它们以支持您的主要软件包。我将使用基于文本的 Lynx 浏览器进行以下安装演示。

## 安装一个应用程序

在 CentOS 系统中，输入以下命令：

```
$ sudo yum install lynx
Last metadata expiration check: 1:00:58 ago on Fri 05 Nov 2021 10:50:41 AM CDT.
No match for argument: lynx
Error: Unable to find a match: lynx
```

如果你收到此错误，意味着该软件包，比如`lynx`在这种情况下，不以该名称存在。你需要搜索软件包名称或包含你所需软件包的存储库。例如，我必须执行以下步骤来安装`lynx`：

```
$ sudo dnf install dnf-plugins-core
```

这安装了三个软件包：`dnf-plugins-core`、`python3-dnf-plugins-core`和`yum-utils`。然后，使用以下命令来启用 PowerTools 存储库，其中包含`lynx`：

```
$ sudo dnf config-manager --set-enabled powertools
```

现在，继续安装`lynx`及其依赖项：

```
$ sudo dnf install lynx
CentOS Linux 8 - PowerTools                           2.4 kB/s | 2.4 MB     16:50
Last metadata expiration check: 0:14:39 ago on Fri 05 Nov 2021 12:09:08 PM CDT.
Dependencies resolved.
=================================================================================
 Package                   Architecture    Version         Repository        Size
=================================================================================
Installing:
 lynx                      x86_64          2.8.9-2.el8     powertools       1.6 M
Installing dependencies:
 centos-indexhtml          noarch          8.0-0.el8       baseos           246 k

Transaction Summary
=================================================================================
Install  2 Packages

Total download size: 1.8 M
Installed size: 6.5 M
Is this ok [y/N]:
```

同意安装以继续：

```
Is this ok [y/N]: y
Downloading Packages:
(1/2): lynx-2.8.9-2.el8.x86_64.rpm                    208 kB/s | 1.6 MB     00:07
(2/2): centos-indexhtml-8.0-0.el8.noarch.rpm           30 kB/s | 246 kB     00:08
---------------------------------------------------------------------------------
Total                                                 207 kB/s | 1.8 MB     00:09
Running transaction check
Transaction check succeeded.
Running transaction test
Transaction test succeeded.
Running transaction
  Preparing        :                                                          1/1
  Installing       : centos-indexhtml-8.0-0.el8.noarch                        1/2
  Installing       : lynx-2.8.9-2.el8.x86_64                                  2/2
  Running scriptlet: lynx-2.8.9-2.el8.x86_64                                  2/2
  Verifying        : centos-indexhtml-8.0-0.el8.noarch                        1/2
  Verifying        : lynx-2.8.9-2.el8.x86_64                                  2/2

Installed:
  centos-indexhtml-8.0-0.el8.noarch                       lynx-2.8.9-2.el8.x86_64

Complete!
```

这个过程安装了 Lynx 应用程序及其依赖项`centos-indexhtml`。

在 Ubuntu 系统上，输入以下命令：

```
$ sudo apt install lynx
Reading package lists... Done
Building dependency tree      
Reading state information... Done
The following additional packages will be installed:
  libidn11 lynx-common
The following NEW packages will be installed:
  libidn11 lynx lynx-common
0 upgraded, 3 newly installed, 0 to remove and 0 not upgraded.
Need to get 1,586 kB of archives.
After this operation, 5,731 kB of additional disk space will be used.
Do you want to continue? [Y/n]
```

注意两个依赖项：*libidn11* 和 *lynx-common*。系统会先安装这些依赖项，然后再安装目标软件包。继续回答提示时输入是 (`y`) ：

```
Do you want to continue? [Y/n] y
Get:1 http://us.archive.ubuntu.com/ubuntu focal/main amd64 libidn11 amd64 1.33...
Get:2 http://us.archive.ubuntu.com/ubuntu focal/universe amd64 lynx-common all...
Get:3 http://us.archive.ubuntu.com/ubuntu focal/universe amd64 lynx amd64 2.9....
Fetched 1,586 kB in 4min 29s (5,890 B/s)
Selecting previously unselected package libidn11:amd64.
(Reading database ... 107982 files and directories currently installed.)
Preparing to unpack .../libidn11_1.33-2.2ubuntu2_amd64.deb ...
Unpacking libidn11:amd64 (1.33-2.2ubuntu2) ...
Selecting previously unselected package lynx-common.
Preparing to unpack .../lynx-common_2.9.0dev.5-1_all.deb ...
Unpacking lynx-common (2.9.0dev.5-1) ...
Selecting previously unselected package lynx.
Preparing to unpack .../lynx_2.9.0dev.5-1_amd64.deb ...
Unpacking lynx (2.9.0dev.5-1) ...
Setting up libidn11:amd64 (1.33-2.2ubuntu2) ...
Setting up lynx-common (2.9.0dev.5-1) ...
Setting up lynx (2.9.0dev.5-1) ...
update-alternatives: using /usr/bin/lynx to provide /usr/bin/www-browser ...
Processing triggers for libc-bin (2.31-0ubuntu9.2) ...
Processing triggers for man-db (2.9.1-1) ...
Processing triggers for mime-support (3.64ubuntu1) ...
```

`apt`软件包管理器安装了`lynx`软件包及其依赖项。这就是从存储库安装的全部内容。使用软件包安装程序命名你想要安装的应用程序软件包，软件包管理器会为你处理一切。在下一节中，你将学习如何卸载软件包。

## 卸载应用程序

以下是在基于 Red Hat Enterprise Linux 系统上使用软件包管理器卸载目标软件包的简单过程。Red Hat 包管理器`rpm`用于安装、卸载和查询各个软件包及其依赖关系。选项`-e`用于从系统中擦除（移除）目标软件包。接下来你将看到，当移除软件包时没有错误时，系统不会给出任何响应。`autoremove`步骤会自动删除未使用的依赖项。你的系统可能有多个未使用的依赖项。通常情况下可以安全地移除它们。

```
$ sudo rpm -e lynx
$ sudo dnf autoremove
Last metadata expiration check: 0:31:32 ago on Fri 05 Nov 2021 01:52:28 PM CDT.
Dependencies resolved.
=================================================================================
 Package                  Architecture      Version          Repository      Size
=================================================================================
Removing:
 centos-indexhtml         noarch            8.0-0.el8        @baseos        505 k

Transaction Summary
=================================================================================
Remove  1 Package

Freed space: 505 k
Is this ok [y/N]: y
Running transaction check
Transaction check succeeded.
Running transaction test
Transaction test succeeded.
Running transaction
  Preparing        :                                                          1/1
  Erasing          : centos-indexhtml-8.0-0.el8.noarch                        1/1
  Verifying        : centos-indexhtml-8.0-0.el8.noarch                        1/1

Removed:
  centos-indexhtml-8.0-0.el8.noarch                                                                                            

Complete!
```

这个过程移除了`lynx`软件包及其依赖项`centos-indexhtml`。

以下过程使用 Ubuntu 的`apt`和`purge`选项：

```
$ sudo apt purge lynx
Reading package lists... Done
Building dependency tree      
Reading state information... Done
The following packages were automatically installed and are no longer required:
  libidn11 lynx-common
Use 'sudo apt autoremove' to remove them.
The following packages will be REMOVED:
  lynx*
0 upgraded, 0 newly installed, 1 to remove and 0 not upgraded.
After this operation, 1,992 kB disk space will be freed.
Do you want to continue? [Y/n] y
(Reading database ... 108095 files and directories currently installed.)
Removing lynx (2.9.0dev.5-1) ...
(Reading database ... 108082 files and directories currently installed.)
Purging configuration files for lynx (2.9.0dev.5-1) ...
```

`purge`选项会移除`lynx`软件包但不会移除依赖项。如前所示的输出，你必须运行`sudo apt autoremove`来从系统中擦除这些文件。如果你使用`remove`选项，那么`apt`仅会从系统中移除二进制文件，而保留配置文件和其他文件不变。在下一节中，你将学习如何使用系统的软件包管理器安装和卸载单个软件包。

# 安装和卸载单个软件包

你需要从其他来源获取软件包，如供应商网站、GitHub 和 SourceForge，这些软件包不属于任何存储库。你必须在命令行手动安装这些单独的软件包。而不是使用存储库命令来安装这些软件包，你可以使用本地软件包管理器工具，如`rpm`和`dpkg`。

###### 提示

在尝试安装目标软件包之前，请确保阅读任何附带的文档。检查依赖项、配置以及任何安全警告。您需要先满足依赖关系，然后再安装目标软件包。

## 手动安装单个软件包

本节中的示例使用 `downloadonly` 选项从仓库下载而不安装软件包，以简化演示中软件包的定位。软件包的来源无关紧要，只要您已将它们下载到系统并在命令行手动安装即可。

在 CentOS 系统上，输入以下命令：

```
$ sudo dnf --downloadonly install lynx
Last metadata expiration check: 0:20:15 ago on Sat 06 Nov 2021 08:11:31 AM CDT.
Dependencies resolved.
=================================================================================
 Package                 Architecture      Version         Repository        Size
=================================================================================
Installing:
 lynx                    x86_64            2.8.9-2.el8     powertools       1.6 M
Installing dependencies:
 centos-indexhtml        noarch            8.0-0.el8       baseos           246 k

Transaction Summary
=================================================================================
Install  2 Packages

Total download size: 1.8 M
Installed size: 6.5 M
DNF will only download packages for the transaction.
Is this ok [y/N]:y
Downloading Packages:
(1/2): lynx-2.8.9-2.el8.x86_64.rpm                    287 kB/s | 1.6 MB     00:05
(2/2): centos-indexhtml-8.0-0.el8.noarch.rpm           37 kB/s | 246 kB     00:06
---------------------------------------------------------------------------------
Total                                                 252 kB/s | 1.8 MB     00:07
Complete!
The downloaded packages were saved in cache until the next successful transact...
You can remove cached packages by executing 'dnf clean packages'.
```

使用此方法下载软件包时，它们存储在 */var/cache/dnf* 目录的子目录中。软件包下载到的子目录取决于软件包的来源仓库。例如，在我的 CentOS 系统中，软件包可以下载到以下任何一个位置：

```
/var/cache/dnf/appstream-a520ed22b0a8a736
/var/cache/dnf/AppStream-a520ed22b0a8a736
/var/cache/dnf/baseos-929b586ef1f72f69
/var/cache/dnf/BaseOS-929b586ef1f72f69
/var/cache/dnf/epel-6519ee669354a484
/var/cache/dnf/epel-modular-95d9a0c53e492cbd
/var/cache/dnf/extras-2770d521ba03e231
/var/cache/dnf/powertools-25a6a2b331e53e98
```

在此示例中，`lynx` 包下载到 */var/cache/dnf/powertools-⁠25a6a2b3​31e53e98/packages*，而 `centos-indexhtml` 下载到 */var/cache/dnf/⁠baseos-​929b586ef1f72f69/packages*。

我将首先尝试安装 `lynx` 包，但会因为依赖于 `centos-indexhtml` 包而失败：

```
$ sudo rpm -i lynx-2.8.9-2.el8.x86_64.rpm
error: Failed dependencies:
      redhat-indexhtml is needed by lynx-2.8.9-2.el8.x86_64
```

###### 注意

由于 CentOS 是与 Red Hat Enterprise Linux 兼容的二进制分发，我们的 `centos-indexhtml` 包相当于 `redhat-indexhtml` 包。它将正常工作，因为包名称是类似的。

根据前面的错误提示，首先安装 `centos-indexhtml` 包，然后再安装 `lynx` 包：

```
$ sudo rpm -i centos-indexhtml-8.0-0.el8.noarch.rpm
```

该软件包安装时没有错误：

```
$ sudo rpm -i lynx-2.8.9-2.el8.x86_64.rpm
```

`lynx` 包已成功安装。`rpm` 开关 (`-i`) 的意思是 *install*。

对于 Ubuntu 系统，手动下载和安装过程如下：

```
$ sudo apt install --download-only lynx
Reading package lists... Done
Building dependency tree      
Reading state information... Done
The following additional packages will be installed:
  libidn11 lynx-common
The following NEW packages will be installed:
  libidn11 lynx lynx-common
0 upgraded, 3 newly installed, 0 to remove and 0 not upgraded.
Need to get 960 kB/1,586 kB of archives.
After this operation, 5,731 kB of additional disk space will be used.
Do you want to continue? [Y/n]y
Get:1 http://us.archive.ubuntu.com/ubuntu focal/main amd64 libidn11 amd64 1.33...
Get:2 http://us.archive.ubuntu.com/ubuntu focal/universe amd64 lynx-common all...
Get:2 http://us.archive.ubuntu.com/ubuntu focal/universe amd64 lynx-common all...
Fetched 214 kB in 2min 24s (1,491 B/s)
Download complete and in download only mode
```

在 Ubuntu 等基于 Debian 的系统上，通过这种方式下载的文件存储在 */var/cache/apt/archives* 中，格式为 *.deb* 包，您可以从该位置安装它们。请注意，`apt` 实用程序对话框显示软件包将被安装，但最终消息“Download complete and in download only mode”表示软件包已下载但未安装。在以下示例中，我尝试安装 `lynx` 并忽略与其一同下载的依赖项：

```
$ sudo dpkg -i lynx_2.9.0dev.5-1_amd64.deb
Selecting previously unselected package lynx.
(Reading database ... 108026 files and directories currently installed.)
Preparing to unpack lynx_2.9.0dev.5-1_amd64.deb ...
Unpacking lynx (2.9.0dev.5-1) ...
dpkg: dependency problems prevent configuration of lynx:
 lynx depends on libidn11 (>= 1.13); however:
  Package libidn11 is not installed.
 lynx depends on lynx-common; however:
  Package lynx-common is not installed.

dpkg: error processing package lynx (--install):
 dependency problems - leaving unconfigured
Errors were encountered while processing:
 lynx
```

系统不允许您在没有与 `lynx` 同一目录中下载的依赖项的情况下安装：

```
$ ls -1 /var/cache/apt/archives
libidn11_1.33-2.2ubuntu2_amd64.deb
lynx_2.9.0dev.5-1_amd64.deb
lynx-common_2.9.0dev.5-1_all.deb
```

首先安装依赖项，然后再安装 `lynx`：

```
$ sudo dpkg -i libidn11_1.33-2.2ubuntu2_amd64.deb
Selecting previously unselected package libidn11:amd64.
(Reading database ... 108039 files and directories currently installed.)
Preparing to unpack libidn11_1.33-2.2ubuntu2_amd64.deb ...
Unpacking libidn11:amd64 (1.33-2.2ubuntu2) ...
Setting up libidn11:amd64 (1.33-2.2ubuntu2) ...
Processing triggers for libc-bin (2.31-0ubuntu9.2) …

$ sudo dpkg -i lynx-common_2.9.0dev.5-1_all.deb
Selecting previously unselected package lynx-common.
(Reading database ... 108044 files and directories currently installed.)
Preparing to unpack lynx-common_2.9.0dev.5-1_all.deb ...
Unpacking lynx-common (2.9.0dev.5-1) ...
Setting up lynx-common (2.9.0dev.5-1) ...
Processing triggers for mime-support (3.64ubuntu1) ...
Processing triggers for man-db (2.9.1-1) ...

$ sudo dpkg -i lynx_2.9.0dev.5-1_amd64.deb
(Reading database ... 108136 files and directories currently installed.)
Preparing to unpack lynx_2.9.0dev.5-1_amd64.deb ...
Unpacking lynx (2.9.0dev.5-1) over (2.9.0dev.5-1) ...
Setting up lynx (2.9.0dev.5-1) ...
update-alternatives: using /usr/bin/lynx to provide /usr/bin/www-browser ...
```

`dpkg` 命令的 `-i` 开关表示安装，就像 `rpm` 实用程序一样。接下来，我们将手动卸载相同的软件包。

## 卸载单个软件包

要卸载手动安装的软件包，您必须反向进行。这意味着您需要按照安装顺序的相反顺序进行卸载，即首先卸载所有依赖项，然后再卸载软件包本身。如果存在依赖项，系统会指示您它们是哪些。

在 CentOS 系统上，输入以下命令：

```
$ sudo rpm -e centos-indexhtml
error: Failed dependencies:
    redhat-indexhtml is needed by (installed) lynx-2.8.9-2.el8.x86_64

$ sudo rpm -e lynx

$ sudo rpm -e centos-indexhtml
```

您已成功卸载 `lynx` 及其依赖项 `centos-indexhtml`。

在 Ubuntu 系统上，当您尝试卸载手动安装的软件包时，不会有关于依赖关系的警告。要在 Ubuntu 上卸载 Lynx，有三个软件包，`lynx` 和其两个依赖项：`libidn11` 和 `lynx-common`。

查看在尝试从 Ubuntu 系统中分别卸载 `lynx` 及其依赖项时的差异。这里展示了三个分开的命令，演示了使用不同命令时如何移除或不移除软件包和依赖项。我在这个演示中对每个命令都选择了否 (`n`)：

```
$ sudo apt purge lynx
Reading package lists... Done
Building dependency tree      
Reading state information... Done
The following packages will be REMOVED:
  lynx*
0 upgraded, 0 newly installed, 1 to remove and 0 not upgraded.
After this operation, 1,992 kB disk space will be freed.
Do you want to continue? [Y/n]

$ sudo apt purge lynx-common
Reading package lists... Done
Building dependency tree      
Reading state information... Done
The following packages will be REMOVED:
  lynx* lynx-common*
0 upgraded, 0 newly installed, 2 to remove and 0 not upgraded.
After this operation, 5,481 kB disk space will be freed.
Do you want to continue? [Y/n]

$ sudo apt purge libidn11
Reading package lists... Done
Building dependency tree      
Reading state information... Done
The following packages will be REMOVED:
  libidn11* lynx*
0 upgraded, 0 newly installed, 2 to remove and 0 not upgraded.
After this operation, 2,242 kB disk space will be freed.
Do you want to continue? [Y/n]
```

如果只移除系统上的 `lynx`，`lynx_common` 和 `libidn11` 将被保留。执行命令 `sudo apt autoremove` 不会像在从仓库安装 Lynx 时那样移除未使用的依赖项。接下来的部分描述了如何查找特定软件包的依赖关系。

## 查找软件包依赖关系

在安装之前了解软件包的依赖关系很有帮助。以下是在基于 Red Hat Enterprise Linux 的系统上查找它们的方法：

```
$ dnf deplist lynx
CentOS Linux 8 - AppStream                                37 kB/s | 9.6 MB  04:28
CentOS Linux 8 - BaseOS                                   39 kB/s | 8.5 MB  03:44
CentOS Linux 8 - Extras                                  7.8 kB/s |  10 kB  00:01
CentOS Linux 8 - PowerTools                              121 kB/s | 2.4 MB  00:20
Extra Packages for Enterprise Linux Modular 8 - x86_64    20 kB/s | 955 kB  00:48
Extra Packages for Enterprise Linux 8 - x86_64            29 kB/s |  11 MB  06:08
package: lynx-2.8.9-2.el8.x86_64
  dependency: libc.so.6(GLIBC_2.15)(64bit)
   provider: glibc-2.28-151.el8.x86_64
  dependency: libcrypto.so.1.1()(64bit)
   provider: openssl-libs-1:1.1.1g-15.el8_3.x86_64
  dependency: libcrypto.so.1.1(OPENSSL_1_1_0)(64bit)
   provider: openssl-libs-1:1.1.1g-15.el8_3.x86_64
  dependency: libdl.so.2()(64bit)
   provider: glibc-2.28-151.el8.x86_64
  dependency: libncursesw.so.6()(64bit)
   provider: ncurses-libs-6.1-7.20180224.el8.x86_64
  dependency: libssl.so.1.1()(64bit)
   provider: openssl-libs-1:1.1.1g-15.el8_3.x86_64
  dependency: libssl.so.1.1(OPENSSL_1_1_0)(64bit)
   provider: openssl-libs-1:1.1.1g-15.el8_3.x86_64
  dependency: libtinfo.so.6()(64bit)
   provider: ncurses-libs-6.1-7.20180224.el8.x86_64
  dependency: libz.so.1()(64bit)
   provider: zlib-1.2.11-17.el8.x86_64
  dependency: redhat-indexhtml
   provider: centos-indexhtml-8.0-0.el8.noarch
  dependency: rtld(GNU_HASH)
   provider: glibc-2.28-151.el8.i686
   provider: glibc-2.28-151.el8.x86_64
```

正如本章前面提到的，系统已安装了大多数所需的依赖项。所需的 `centos-indexhtml` 并没有特别突出。我知道的唯一方法是尝试安装目标软件包以隔离系统所需的任何依赖项。

以下清单展示了在 Ubuntu 系统上执行相同依赖列表查询的过程：

```
$ sudo apt show lynx
Package: lynx
Version: 2.9.0dev.5-1
Priority: extra
Section: universe/web
Origin: Ubuntu
Maintainer: Ubuntu Developers <ubuntu-devel-discuss@lists.ubuntu.com>
Original-Maintainer: Debian Lynx Packaging Team <pkg-lynx-maint@lists.aliot...
Bugs: https://bugs.launchpad.net/ubuntu/+filebug
Installed-Size: 1,992 kB
Provides: news-reader, www-browser
Depends: libbsd0 (>= 0.0), libbz2-1.0, libc6 (>= 2.15), libgnutls30 (>= 3.6.12...
Recommends: mime-support
Conflicts: lynx-ssl
Breaks: lynx-cur (<< 2.8.9dev8-2~), lynx-cur-wrapper (<< 2.8.8dev.8-2)
Replaces: lynx-cur (<< 2.8.9dev8-2~), lynx-cur-wrapper (<< 2.8.8dev.8-2)
Homepage: https://lynx.invisible-island.net/
Download-Size: 626 kB
APT-Manual-Installed: yes
APT-Sources: http://us.archive.ubuntu.com/ubuntu focal/universe amd64 Packages
Description: classic non-graphical (text-mode) web browser
 In continuous development since 1992, Lynx sets the standard for
 text-mode web clients. It is fast and simple to use, with support for
 browsing via FTP, Gopher, HTTP, HTTPS, NNTP, and the local file system.
```

如果您是 Linux 纯粹主义者，并且喜欢编译软件以便拥有最大的控制权，下一节演示了如何从源代码安装软件包。

# 从源代码安装软件

一些系统管理员喜欢从源代码安装所有软件（也称为“源码”），因为这是最灵活的软件安装方法。从源代码安装允许您为特定需求自定义安装。您可以更改安装路径、启用功能、禁用功能，并对应用程序的每个可能的配置选项进行微调。

从源代码安装存在一些缺点。主要的缺点是您必须满足所安装软件的依赖关系。这可能是令人沮丧、耗时和乏味的。我个人曾追踪递归依赖性，直到忘记原本需要安装的应用程序名称为止。另一个缺点是必须在系统上安装一整套开发工具，这将消耗大量磁盘空间。还有一个缺点是，从源代码安装的软件升级到新版本与安装原始版本一样困难和耗时。如果之前的版本没有完全被覆盖或删除，则可能会遇到相当难以解决的版本冲突。

## 满足前提条件：构建开发环境

在从源代码安装任何应用程序之前，您需要通过安装代码编译器和支持软件来设置开发环境。最简单的方法是从您的 Linux 供应商的存储库安装一组软件包。

对于基于红帽企业 Linux 的系统，使用`groupinstall`选项并将`"Development Tools"`标识为安装目标组是最佳选择。不幸的是，此组选择会安装许多不必要且潜在非安全的软件包，用于在命令行编译源代码，例如一系列图形工具。因此，通常希望设置一个专门用于软件开发的特定系统。要创建软件开发系统，请使用以下命令：

```
$ sudo dnf groupinstall "Development Tools"
Last metadata expiration check: 2:41:45 ago on Sun 07 Nov 2021 09:57:05 AM CST.
Dependencies resolved.
================================================================================
 Package                          Architecture  Version        Repository   Size
================================================================================
Upgrading:
 automake                         noarch        1.16.1-7.el8   appstream   713 k
 binutils                         x86_64        2.30-93.el8    baseos      5.8 M
 cpp                              x86_64        8.4.1-1.el8    appstream    10 M
 elfutils-libelf                  x86_64        0.182-3.el8    baseos      216 k
 elfutils-libs                    x86_64        0.182-3.el8    baseos      293 k
 gcc                              x86_64        8.4.1-1.el8    appstream    23 M

...

xorg-x11-font-utils               x86_64        1:7.5-40.el8   appstream   103 k
 xorg-x11-fonts-ISO8859-1-100dpi  noarch        7.5-19.el8     appstream   1.1 M
 xorg-x11-server-utils            x86_64        7.7-27.el8     appstream   198 k
 zlib-devel                       x86_64        1.2.11-17.el8  baseos       58 k
 zstd                             x86_64        1.4.4-1.el8    appstream   393 k
Installing weak dependencies:
 gcc-gdb-plugin                   x86_64        8.4.1-1.el8    appstream   117 k
 kernel-devel                     x86_64        4.18.0-305...  baseos       18 M
Enabling module streams:
 javapackages-runtime                           201801                    
Installing Groups:
 Development Tools                                                          

Transaction Summary
================================================================================
Install  159 Packages
Upgrade   23 Packages

Total download size: 219 M
Is this ok [y/N]:
```

在 Ubuntu 系统上，等效安装使用`build-essential`选项在您的系统上安装所有必要的开发工具：

```
$ sudo apt install build-essential
Reading package lists... Done
Building dependency tree      
Reading state information... Done
The following additional packages will be installed:
  binutils binutils-common binutils-x86-64-linux-gnu cpp cpp-9 dpkg-dev fakero...
g++ g++-9 gcc gcc-9 gcc-9-base libalgorithm-diff-perl libalgorithm-diff-xs-per...
libasan5 libatomic1 libbinutils libc-dev-bin libc6-dev libcc1-0 libcrypt-dev l...
libctf0 libdpkg-perl libfakeroot libfile-fcntllock-perl libgcc-9-dev libgomp1 ...
libitm1 liblsan0 libmpc3 libquadmath0 libstdc++-9-dev libtsan0 libubsan1 linux...
make manpages-dev
Suggested packages:
  binutils-doc cpp-doc gcc-9-locales debian-keyring g++-multilib g++-9-multili...
gcc-multilib autoconf automake libtool flex bison gdb gcc-doc gcc-9-multilib g...
bzr libstdc++-9-doc make-doc
The following NEW packages will be installed:
  binutils binutils-common binutils-x86-64-linux-gnu build-essential cpp cpp-9...
fakeroot g++ g++-9 gcc gcc-9 gcc-9-base libalgorithm-diff-perl libalgorithm-di...
libalgorithm-merge-perl libasan5 libatomic1 libbinutils libc-dev-bin libc6-dev...
libcrypt-dev libctf-nobfd0 libctf0 libdpkg-perl libfakeroot libfile-fcntllock-...
libgcc-9-dev libgomp1 libisl22 libitm1 liblsan0 libmpc3 libquadmath0 libstdc++...
libtsan0 libubsan1 linux-libc-dev make manpages-dev
0 upgraded, 41 newly installed, 0 to remove and 0 not upgraded.
Need to get 43.0 MB of archives.
After this operation, 189 MB of additional disk space will be used.
Do you want to continue? [Y/n]
```

确认安装并继续。安装软件包组可能需要几分钟时间。一旦设置好开发环境，您将需要为 Lynx 下载源代码。从源代码安装的指令对任何 Linux 发行版都是相同的；但是，我将在 CentOS 和 Ubuntu 系统上执行此安装，并注意文本中的任何差异和错误。用户可以下载、编译和制作二进制文件，但只有 root 用户可以将二进制文件安装到系统目录。

## 下载、解压、编译和安装您的软件

使用诸如`wget`之类的实用程序下载您的压缩源代码：

```
$ wget https://invisible-mirror.net/archives/lynx/tarballs/lynx2.8.9rel.1.tar.gz
```

从压缩存档中提取源代码：

```
$ tar zxvf lynx2.8.9rel.1.tar.gz
```

切换到由提取过程创建的`lynx`源代码树目录中：

```
$ cd lynx2.8.9rel.1
```

在运行 `configure` 之前，请花几分钟查找并阅读通常存在于源代码树中的 *README* 文件。这个文件包含有关源代码和安装说明的有价值的指导和信息。*README* 文件通常引用描述安装选项的 *INSTALLATION* 文件。出于演示目的，我接受所有默认设置，并简单地运行了配置脚本。如果所有依赖项都满足，配置检查将顺利进行，创建 makefile，然后将您退回到您的 shell 提示符。这个过程可能需要几分钟来完成：

```
$ ./configure
```

两个配置脚本都因以下错误而失败：

```
checking for screen type... curses
checking for specific curses-directory... no
checking for extra include directories... no
checking if we have identified curses headers... none
configure: error: No curses header-files found
```

当你遇到错误时，配置脚本（configure）会停止，但会保留其位置，这样你可以通过再次运行脚本来满足依赖项并继续之前的工作。为了满足当前的依赖关系，我在 CentOS 系统上安装了 `ncurses-devel` 包（`sudo dnf -y install ncurses-devel`）。配置脚本成功完成。对于 Ubuntu 系统，我安装了 `lib32ncurses-dev` 包（`sudo apt install lib32ncurses-dev`），配置脚本也成功完成。

根据 *INSTALLATION* 文件，现在您必须运行 `make` 命令来编译源代码。这个过程将需要几分钟的时间来完成：

```
$ make
```

在满足配置脚本中的失败依赖项后，两个编译过程都成功完成。要将 Lynx 安装到正确的位置并设置正确的权限，请运行 `sudo make install`：

```
$ sudo make install
/bin/sh -c "P=`echo lynx|sed 's,x,x,'`; \
if test -f /usr/local/bin/$P ; then \
      mv -f /usr/local/bin/$P /usr/local/bin/$P.old; fi"
/usr/bin/install -c lynx /usr/local/bin/`echo lynx|sed 's,x,x,'`
/usr/bin/install -c -m 644 ./lynx.man /usr/.../man1/`echo lynx|sed 's,x,x,'`.1
** installing ./lynx.cfg as /usr/local/etc/lynx.cfg
** installing ./samples/lynx.lss as /usr/local/etc/lynx.lss

Use make install-help to install the help-files
Use make install-doc to install extra documentation files
```

在这一点上，大多数指令建议您运行 `make clean` 来从系统中删除所有的对象代码和其他临时文件：

```
$ make clean
rm -f WWW/Library/*/*.[aoib]
rm -f WWW/Library/*/.created
cd ./WWW/Library/Implementation && make  DESTDIR="" CC="gcc" clean
make[1]: Entering directory '/home/khess/lynx2.8.9rel.1/.../Implementation'
rm -f core *.core *.leaks *.[oi] *.bak tags TAGS
rm -f dtd_util
rm -f ./*.o
make[1]: Leaving directory '/home/khess/lynx2.8.9rel.1/.../Implementation'
cd ./src && make  DESTDIR="" CC="gcc" clean
make[1]: Entering directory '/home/khess/lynx2.8.9rel.1/src'
rm -f lynx core *.core *.leaks *.i *.o *.bak tags TAGS test_*
cd chrtrans && make  DESTDIR="" CC="gcc" clean
make[2]: Entering directory '/home/khess/lynx2.8.9rel.1/src/chrtrans'
rm -f makeuctb *.o *uni.h *uni2.h *.i
make[2]: Leaving directory '/home/khess/lynx2.8.9rel.1/src/chrtrans'
make[1]: Leaving directory '/home/khess/lynx2.8.9rel.1/src'
rm -f *.b ./src/lynx *.leaks cfg_defs.h LYHelp.h lint.*
rm -f help_files.sed
rm -f core *.core
```

运行这个命令是个人喜好问题。如果您需要从系统中删除编译的软件，下一节将引导您完成此过程。

## 卸载源码安装的软件包

如果你的源代码树中还存在原始的 makefile，你可以相当轻松地卸载一个软件包，但你必须有 makefile 才能这样做。如果你不想为每个编译的程序保留所有的源代码树在你的系统上，那么请备份 makefile，例如将其复制到一个名为 *makefile.lynx289r1* 或类似的备份目录中。在卸载时，makefile 必须位于你当前的目录中：

```
$ sudo make uninstall
rm -f /usr/local/bin/`echo lynx|sed 's,x,x,'`
rm -f /usr/local/share/man/man1/`echo lynx|sed 's,x,x,'`.1
rm -f /usr/local/etc/lynx.cfg
rm -f /usr/local/etc/lynx.lss
/bin/sh -c 'if test -d "/usr/local/share/lynx_help" ; then \
    WD=`cd "/usr/local/share/lynx_help" && pwd` ; \
    TAIL=`basename "/usr/local/share/lynx_help"` ; \
    HEAD=`echo "$WD"|sed -e "s,/${TAIL}$,,"` ; \
    test "x$WD" != "x$HEAD" && rm -rf "/usr/local/share/lynx_help"; \
    fi'
/bin/sh -c 'if test -d "/usr/local/share/lynx_doc" ; then \
    WD=`cd "/usr/local/share/lynx_doc" && pwd` ; \
    TAIL=`basename "/usr/local/share/lynx_doc"` ; \
    HEAD=`echo "$WD"|sed -e "s,/${TAIL}$,,"` ; \
    test "x$WD" != "x$HEAD" && rm -rf "/usr/local/share/lynx_doc"; \
    fi'
/bin/sh -c 'if test -d "/usr/local/share/lynx_help" ; then \
    WD=`cd "/usr/local/share/lynx_help" && pwd` ; \
    TAIL=`basename "/usr/local/share/lynx_help"` ; \
    HEAD=`echo "$WD"|sed -e "s,/'${TAIL}'$,,"` ; \
    test "x$WD" != "x$HEAD" ; \
    cd "/usr/local/share/lynx_help" && rm -f COPYING COPYHEADER ; \
    fi'
```

如果你没有你的 makefile 或它不在你当前的目录中，你会收到以下错误：

```
$ sudo make uninstall
make: *** No rule to make target 'uninstall'.  Stop.
```

如果要重新创建 makefile，请提取与之前使用的相同版本的源代码树，并记住你的配置选项。否则，你可以使用前面卸载的结果来指导手动卸载。

# 摘要

本章指导你如何从不同来源（仓库、本地软件包文件和源代码）安装软件。你会发现，通过一些练习，安装软件是相当容易的。请记住，保持系统平稳运行和安全的责任在于你。仅仅因为安装软件快捷简单，并不意味着你应该忽视它对系统性能、安全性和磁盘使用的影响。

下一章将教你如何向系统添加磁盘空间。
