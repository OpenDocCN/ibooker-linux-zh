# 第七章：使用 rsync 和 cp 进行备份和恢复

您知道您需要对计算机文件进行良好的备份，并定期测试以查看是否可以还原文件。但是在 Linux 上如何实现这一点？不用担心，在 Linux 上备份和恢复非常容易理解，并且您的备份文件易于搜索和还原。

对于练习本章命令，拥有几个 USB 闪存驱动器和几个充满文件的目录是很有帮助的，这些文件如果出现问题也不会遗憾。

我们将使用*rsync*和*cp*。这两者都是基本的 Linux 工具，您可以放心它们已得到良好维护并且随处可用。

*cp*是 GNU *coreutils*包中包含的复制命令，默认安装在几乎所有 Linux 发行版上。*cp*用于简单的复制。这可能是您维护常规备份所需的全部功能。

*rsync*是一款高效的文件传输程序，其主要目的是保持不同文件系统之间的同步。当您用它来进行备份时，它会将本地文件与备份设备保持同步。由于它仅传输文件中的更改，因此速度快且效率高。与许多备份软件不同的是，它甚至可以镜像删除操作。正是由于这些特点，rsync 成为更新和镜像用户主目录、网站、git 仓库以及其他大型复杂文件树的首选工具。

有两种方法可以通过网络使用 rsync：通过 SSH 进行身份验证的登录和传输，或者将其作为守护进程运行。使用 SSH 需要用户在每台需要 rsync 访问的机器上拥有登录账户。当 rsync 以守护进程模式运行时，您可以使用其内置的身份验证方法来控制访问，以便用户无需在 rsync 服务器上拥有登录账户。守护进程模式非常适合局域网备份服务器。在不受信任的网络上访问是不安全的，除非您使用 VPN（参见第十三章）。

您会将备份存储在什么设备上？这取决于您的需求。对于单个用户，我喜欢使用 USB 存储介质。假设您有一台台式 Linux PC、一台笔记本、一台平板和一部智能手机。将手机和平板备份到您的 PC，然后将 PC 备份到 USB 硬盘驱动器上。重要的文件可以备份到在线备份服务。

对于多用户，一个好的解决方案是一个中央备份服务器。这可以是任何 Linux PC。

考虑长期性。您不能依赖数字存储介质的长期存储，因为即使介质（硬盘、USB 存储驱动器、CD/DVD）存活下来，也不能保证能够持久读取。硬件和文件格式会发生变化。您还能读取软盘吗？还记得 ZIP 盘吗？还有那些旧版 Microsoft Word 和 Powerpoint 文档的档案？使用开源文件格式，您总能找到恢复它们的方法。但是在供应商决定停止支持它们时，专有格式的恢复就很困难了。

纸张仍然是长期存储的冠军，值得考虑用于你最重要的文件和照片。

长期数字存储中，计划定期将你的存档转移到新的媒体，可能需要新的命令和新的文件系统格式。

那么备份你的备份服务器怎么样？没问题。建立一个远程 rsync 镜像来备份备份是一种常见策略，如果你的互联网连接足够强大来处理流量的话。但在建立大规模备份基础设施之前，请考虑你真正需要多少级冗余。离站点备份是对你站点灾难的保险。这可以是你控制的远程备份服务器，也可以是朋友的站点，或者是数据中心的租用空间。也要考虑恢复：你能快速获取你的备份吗？

永远记住备份的目的是*恢复*。定期测试你的备份，避免以最艰难的方式发现你的备份方法失败。

在本章中，你将学习如何使用*cp*简单地将文件复制到 USB 存储设备。对一些用户来说，这可能是他们所需要的全部。

大部分章节讲述使用*rsync*命令进行更快、更有效的复制。你可以使用*rsync*将文件备份到本地媒体或远程服务器。你将学习哪些文件应该备份，如何微调文件选择，保持文件的相同权限和时间戳，如何为多用户建立一个 rsync 备份服务器，以及如何进行安全的远程备份。

# 7.1 选择需要备份的文件

## 问题

你不确定应该备份哪些文件。是否需要备份系统文件？你真的需要备份所有的个人文件吗？有些文件是不需要备份的吗？

## 解决方案

任何你会后悔丢失的文件都是需要备份的文件。你的个人文件和系统数据文件是最重要的。恢复系统文件，如命令、应用程序和库，相对不那么重要，因为你总是可以重新下载和安装这些文件。

以下目录包含诸如配置文件、服务器数据文件（如 web、FTP 和邮件服务器）、日志文件、安装在非标准位置的应用程序以及共享目录等文件，都应该进行备份：

+   */boot/grub*，如果其中包含任何自定义内容，如主题、背景图片或字体。

+   */etc* 包含系统配置文件。

+   */home*，用户的个人文件。

+   */mnt*，临时文件系统挂载点。如果你有想要保留的挂载点，也需要备份它。

+   */opt*，用于专有或其他非标准安装的应用程序。

+   */root*，root 用户的个人文件。

+   */srv*，服务器数据，如 web、FTP 和 rsync 服务器的数据。

+   */tmp* 包含临时数据，根据需要会自动更新或删除。*/tmp* 中的一些数据是持久的，例如用户创建的文件和一些系统服务，它们应该备份。

+   */var* 存储许多类型的数据，如日志文件、邮件池、cron 作业和系统服务的数据，尽管大多数发行版已经迁移到使用 */srv* 用于系统服务。

如果你有任何共享目录、自定义命令和脚本，或者之前未列出的任何数据文件或目录，请备份它们。

*/proc*、*/sys* 和 */dev* 是存在于内存中的伪文件系统，不应备份。

*/media* 用于挂载可移动存储介质，应由系统管理，因此无需备份。如果你在 */media* 手动创建挂载点，则确实需要将它们移动到 */mnt*。

许多数据库不应该使用简单复制备份，因为它们有专门的实用程序和流程用于复制、备份和恢复。使用专为你的数据库制作的工具。一些示例是 PostrgreSQL、MariaDB 和 MySQL。

# 从备份中恢复

有些文件不应从备份中恢复；参见配方 7.2。

如果备份存储足够大，复制一切是简单的方法。你还可以通过创建文件复制或排除列表来精确调整文件选择；参见配方 7.8 和 7.9。

## 讨论

存储媒体现在价格很便宜，你可能不必关心存储空间的保留。如果你需要注意存储限制，请参阅本章关于文件选择的配方。

## 参见

+   [文件系统层次标准](https://oreil.ly/y1pJs)

# 7.2 从备份中选择要恢复的文件

## 问题

你正在从备份中恢复文件，并且想知道是否有文件不应该被恢复。

## 解决方案

根据情况，有些文件不应该被恢复。

在重新安装 Linux 后不要恢复 */etc/fstab*（配置静态文件系统挂载的文件）。每次安装 Linux 时，所有文件系统都会获得新的通用唯一标识符（UUID），因此它们将无法被识别，你的新安装将失败。

注意，恢复任何 */etc* 中的文件或家目录中的点文件（如 */home/.config* 或 */home/.local*），如果你要将其从备份恢复到不同版本或不同 Linux 发行版的新安装中，可能会存在配置选项或文件位置的不兼容性。逐个恢复它们，这样你可以快速发现任何问题。

## 参见

+   第一章

# 7.3 使用最简单的本地备份方法

## 问题

你想知道最简单、最简便的方法来定期备份到本地 USB 存储设备。

## 解决方案

答案是使用简单复制。弄一个好用的 USB 硬盘或 USB 存储棒。插上它，用你的文件管理器复制你的文件。简单易行，没麻烦，恢复文件也很简单。或者，使用 *cp* 命令（参见第 7.4 节）。

## 讨论

简单复制并不完全适合所有情况，但对于少量设备，如个人电脑、笔记本电脑和手机，效果还不错。重要的是定期进行备份，验证你可以从备份中恢复文件，并且不必担心是否足够“极客”。

从前，备份更复杂，因为存储空间昂贵，所以备份程序使用了很多技巧来节省空间。现在你可以花不到 200 美元购买支持多 TB 容量的外部 USB 3.0 硬盘。

## 参见

+   *man 1 cp*

# 7.4 自动化简单的本地备份

## 问题

你喜欢使用简单复制来将文件备份到外部 USB 存储驱动器，并且希望自动化这个过程。

## 解决方案

这需要使用 *cp* 命令和 *crontab* 来安排你的备份。

你可以列出要复制的单个文件和目录，用空格分隔：

```
duchess@pc:~$ cp -auv Pictures/cat-desk.jpg Pictures/cat-chair.png \
  ~/cat-pics /media/duchess/2tbdisk/backups/
```

下面的示例将 Duchess 的整个主目录复制到名为*2tbdisk*的外部 USB 驱动器上的*backups*目录中：

```
duchess@pc:~$ cp -auv ~ /media/duchess/2tbdisk/backups/
```

这将在备份设备上创建 */media/duchess/2tbdisk/backups/duchess/*。

复制目录内容而不复制目录本身：

```
duchess@pc:~$ cp -auv /home/duchess/* /media/duchess/2tbdisk/backups/

```

创建一个个人 cron 作业，在每晚 10:30 运行你的备份：

```
duchess@pc:~$ crontab -e
# m h  dom mon dow   command
30 22  *   *   *   /bin/cp -au /home/duchess /media/duchess/2tbdisk/backups/

```

## 讨论

如果你想保留文件属性（如所有权和权限），请使用支持文件属性的 Linux 文件系统格式化你的备份驱动器，例如 Ext4、XFS 或 Btrfs（见第十一章）。FAT 文件系统不保留所有权或权限。

留意备份运行的时间长短。如果超过了预定的备份间隔时间，cron 会按计划启动下一次备份，然后你就有麻烦了。

第一次运行时间最长，因为所有文件都是新的。随后的备份将更快，因为只会复制新文件和时间戳更新的文件。

波浪线，~，是当前用户主目录的快捷方式，在这个示例中，它是 */home/duchess* 的缩写。

*/home/duchess/* 中的星号表示复制 */home/duchess* 中的所有文件，但不包括目录 */home/duchess*。

*-a*、*-u* 和 *-v* 选项用于 *cp* 的含义是：

+   *-a, --archive* 递归复制并保留所有文件属性：模式、所有权、时间戳和扩展属性。

+   *-u, --update* 告诉 *cp* 仅复制比备份目录中的副本更新的文件，或尚未备份的新文件。

+   *-v, --verbose* 在复制操作期间打印活动消息。

还有一些其他有用的选项：

+   *-R, -r* 递归；当你不使用 *-a* 选项时，用它来复制目录。*-a* 保留文件属性，*-R, -r* 则不保留。FAT 和 exFAT 文件系统不支持文件属性，因此在这些文件系统中使用 *-R, -r*。

+   *--parents* 在目标上创建丢失的父目录。

+   *-x, --one-file-system* 这可以防止递归到其他分区和挂载的网络文件系统中。例如，如果你挂载了一个 NFS 共享，你可能不希望将其添加到备份中。

大多数 Linux 发行版在 */run/media* 或 */media* 中挂载 USB 设备。找到你的 USB 驱动器的文件路径的简单方法是查看文件管理器或使用 *lsblk* 命令：

```
$ lsblk
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
[...]
sdb      8:16   0  1.8T  0 disk
└─sdb1   8:17   0  1.5T  0 part /media/duchess/backups

```

## 参见

+   Recipe 3.7 了解更多关于使用 *cron* 的信息

+   *man 1 crontab*

+   *man 1 cp*

# 7.5 使用 rsync 进行本地备份

## 问题

你想要将备份存储到 USB 闪存或 USB 硬盘，并且你希望比简单复制更快更高效。你还希望使用标准 Linux 工具进行简单的文件恢复，而不需要特殊软件。

## 解决方案

*rsync* 是你想要的。它可以同步文件系统，无论是本地还是远程。*rsync* 快速高效，因为它只传输文件中的更改，你可以使用 *rsync* 命令、*cp* 命令、你的文件管理器或者你喜欢的任何复制工具来恢复文件。

下面的示例展示了如何备份一个主目录。首先，命名你的源目录，即你想要备份的目录，然后命名目标目录。这个示例将 Duchess 的 */home* 复制到名为 *2tbdisk* 的 USB 驱动器中：

```
duchess@pc:~$ rsync -av ~ /media/duchess/2tbdisk/
sending incremental file list
duchess/
duchess/Documents/
duchess/Downloads/
duchess/Music/
[...]

sent 27,708,209 bytes  received 20,948 bytes  11,091,662.80 bytes/sec
total size is 785,103,770,793  speedup is 28,313.29
```

你可以指定两个或更多的目录，用空格分隔的列表，传输到目标目录：

```
duchess@pc:~$ rsync -av ~/arias ~/overtures /media/duchess/2tbdisk/duchess/

```

将文件从备份设备复制到计算机上，反转源和目标：

```
duchess@pc:~$ rsync -av /media/duchess/2tbdisk/duchess/arias /home/duchess/

```

你可以安全地使用 *--dry-run* 选项测试你的 *rsync* 命令，而不复制任何文件：

```
duchess@pc:~$ rsync -av --dry-run \
~/Music/scores ~/Music/woodwinds /media/duchess/2tbdisk/duchess/

```

如果从源目录删除了任何文件，*rsync* 将不会从目标目录中删除它们，除非你使用 *delete* 选项显式告诉它要删除：

```
duchess@pc:~$ rsync -av --delete /home/duchess /media/duchess/2tbdisk/

```

## 讨论

波浪号 *~* 是你的家目录的快捷方式，所以在例子中它意味着 */home/duchess*。

带有换行的命令示例使用斜杠 \ 表示命令在下一行继续。你可以复制整个带有斜杠的命令，它应该可以工作。

如果在你的个人电脑上挂载了网络文件系统，比如 NFS 或 Samba，使用 *-x* 选项仅从本地文件系统复制，而不递归到远程文件系统。

添加尾部斜杠*~/*（*/home/duchess/*）只复制*duchess/*目录的内容，但不包括目录本身，结果为*/media/duchess/2tbdisk/[files]*。省略尾部斜杠会传输*/home/duchess*和*duchess*目录的内容，结果为*/media/duchess/2tbdisk/duchess/[files]*。尾部斜杠只在源目录上有影响，在目标目录上没有影响。

如果你不得不用手指计数或进行多次测试来提醒自己尾部斜杠的行为，不要感到难过，因为这困扰着每个人。可以将尾部斜杠想象成一个小篱笆，防止源目录逃逸。

*-a*和*-v*选项对*rsync*的含义是：

+   *-a, --archive*保留模式、时间戳、权限和所有权，并进行递归复制。这与**-rlptgoD**相同，它进行递归复制、复制符号链接、保留权限、保留文件修改时间、保留文件所有权和保留设备文件等特殊文件。

+   *-v, --verbose*显示活动消息。

你可能希望使用其中一些选项：

+   *-q, --quiet*抑制非错误消息。

+   *--progress*显示每个文件传输时的信息。

+   *-A, --als*保留访问控制列表（ACLs）。

+   *-X, --xattrs*保留扩展文件属性（xattrs）。

当然，你的文件名会与示例不同。*2tbdisk*是用户创建的文件系统标签（参见第 9.4 节）。它是“2TB 硬盘”的缩写。如果你没有创建标签，*udev*会创建一个，例如*/media/duchess/488B-7971/*。

你可以使用正常的 Linux 工具，比如*rsync*、文件管理器或*cp*命令来恢复文件。

## 参见

+   *man 1 rsync*

# 7.6 使用 rsync 通过 SSH 进行安全的远程文件传输

## 问题

你想使用*rsync*将文件复制到本地网络上的另一台计算机或通过互联网，并且希望进行加密传输和身份验证。

## 解决方案

*rsync*在向另一台机器传输文件时默认使用 SSH。远程机器必须运行 SSH 服务器，源机器必须已经设置好 SSH 客户端（参见第十二章）。

这个例子在本地网络上从 Duchess 的 PC 传输文件到她的笔记本电脑。Duchess 在她的笔记本电脑上的用户名是 Empress，她正在从她的 PC 的主目录复制文件到她笔记本电脑的主目录：

```
duchess@pc:~$ rsync -av ~/Music/arias empress@laptop:songs/
duchess@laptop's password:
building file list ... done
arias/
arias/o-mio-babbino-caro.ogg
arias/deh-vieni-non-tardar.ogg
arias/mi-chiamano-mimi.ogg
wrote 25984 bytes  read 68 bytes  7443.43 bytes/sec
total size is 25666  speedup is 0.99
```

如果目标目录不存在，*rsync*会创建它。

要通过互联网上传文件，请使用你要登录的服务器的完全限定域名：

```
duchess@pc:~$ rsync -av ~/Music/woodwinds \
 empress@remote.example.com:/backups/
```

从远程主机复制文件的语法是相反的。这个例子将远程主机的*/woodwinds*目录及其内容复制到 Duchess 的主目录：

```
duchess@pc:~$ rsync -av empress@remote.example.com:/backups/woodwinds \
 /home/duchess/Music/
```

## 讨论

你可能还记得以前必须显式指定 SSH 选项，例如*rsync -a -e ssh [options]*。现在不再需要这样做。

您可能会发现以下一些选项很有用：

+   *--partial* 在网络连接中断时保留部分下载的文件，并在恢复连接时从中断处恢复文件传输。

+   *-h, --human-readable* 显示文件大小为千字节、兆字节和千兆字节，而不是字节。

+   *--log-file=* 将每次传输的完整记录存储在文本文件中。像这样将它们放在一起：

```
duchess@pc:~$ rsync --partial --progress \
 --log-file=/home/duchess/rsynclog.txt \
 -hav ~/Music/arias empress@remote.example.com:/backups/
```

SSH 同时加密验证和传输。用户需要在所有要传输文件的机器上拥有 shell 帐户。请参见第十二章了解使用 SSH 进行安全远程管理的内容。

考虑设置一个中央备份服务器来简化管理。您的用户拥有自己的帐户和自己的*/home*目录，并可以管理自己的备份和还原，而不会打扰您。

作为备份服务器的另一个选择是将*rsync*作为服务运行。这样做的优点是*rsync*用户不需要服务器上的登录帐户。缺点之一是不支持加密传输。请参见 Recipe 7.13 了解详情。

## 参见

+   *man 1 rsync*

+   Recipe 12.5

+   Recipe 12.7

# 7.7 使用 cron 和 SSH 自动化 rsync 传输

## 问题

您希望创建 crontabs 以自动运行您的安全*rsync*传输。

## 解决方案

您需要在目标机器上设置 SSH 以进行无密码验证（请参阅 Recipe 12.10 和 Recipe 12.11），并且客户端需要访问目标机器的网络。

然后使用*/etc/crontab*来进行需要 root 权限的传输。以下示例每晚 10 点将*/etc*备份到名为*server1*的 LAN 服务器：

```
# m h dom mon dow user  command
00 22 * * * root /usr/bin/rsync -a /etc server1:/system-backups

```

使用个人 crontabs 来传输您自己的文件（请参阅 Recipe 3.7）。

## 讨论

OpenSSH 是一个很好的工具，提供安全的网络传输服务，适用于各种任务。任何可以在网络上运行的内容都可能可以通过 SSH 运行。

## 参见

+   第十二章

+   Recipe 12.10

+   Recipe 12.11

# 7.8 从备份中排除文件

## 问题

到目前为止，示例已经展示了如何传输整个目录。您想知道如何排除文件和目录以防止复制。

## 解决方案

为简单起见，以下示例演示了向 USB 驱动器进行本地传输的操作，但也适用于通过 SSH 进行远程传输；参见 Recipe 7.6。

当只有几个文件时，您可以使用命令行列出它们，并使用*--exclude=*来排除一个文件。例如，此示例从*/home/duchess/Music/arias*排除一个文件：

```
duchess@pc:~$ rsync -av --exclude=lho-perduta.wav \
 ~/Music/arias /media/duchess/2tbdisk/duchess/Music/
```

这很简单和可靠。但是，有一个需要注意的地方：如果源目录中有与排除文件相同名称的多个文件，则所有这些文件都将被排除。如果您不想排除重复文件，需要指定要排除的文件。在以下示例中，您只想排除*arias/*源目录中的复制文件：

```
duchess@pc:~$ rsync -av --exclude=arias/lho-perduta.wav \
 ~/Music/arias /media/duchess/2tbdisk/duchess/Music/
```

通过用单引号和逗号分隔它们来排除多个文件，将它们放在大括号中。等号和大括号之间不能有空格，逗号和单引号之间也不能有空格：

```
duchess@pc:~$ rsync -av \
--exclude={'arias/lho-perduta.wav','non-mi-dir.wav','un-bel-di-vedremo.flac'} \
~/Music/arias /media/duchess/2tbdisk/duchess/Music/
```

排除目录的工作方式与排除文件相同，您可以在排除列表中混合文件和目录：

```
duchess@pc:~$ rsync -av \
--exclude={'soprano/','tenor/','non-mi-dir.wav'} \
~/Music/arias /media/duchess/2tbdisk/duchess/Music/
```

参见 Recipe 7.11 了解如何将您的排除列表放入文件中。

## 讨论

*rsync*传输中的根目录是您从中传输文件的顶级目录。在本例中，这是*~/Music/arias*。*rsync*检查根目录中的所有文件和目录，并将它们与排除指令（*rsync*称为*模式*）进行比较。模式与根目录中的文件和目录进行匹配，从根目录开始并沿着目录层次结构向下进行。每次匹配模式时，它都会从传输中排除。如果模式*arias/lho-perduta.wav*在另一个位置重复，例如*2arias/lho-perduta.wav*，它也将被排除。当模式以斜杠（*/*）结尾时，*rsync*将仅匹配目录。

## 参见

+   *man 1 rsync*

+   Recipe 7.10

# 7.9 包括选定的备份文件

## 问题

您希望在备份中包含一组选定的文件，而不是定义要排除的文件列表。

## 解决方案

当您只想备份几个文件时，可以在命令行上执行此操作。*--include=* 操作与*--exclude=* 不同，因为它实际上并不意味着“包括”，而是“不排除”。它需要两个额外选项，*--include=*/* 和 *--exclude='*'，如此单个文件的传输示例所示：

```
duchess@pc:~$ rsync -av --include=*/ --include=lho-perduta.wav \
 --exclude='*' ~/Music/arias /media/duchess/2tbdisk/duchess/Music/
```

您可以传输文件列表：

```
duchess@pc:~$ rsync -av --include=*/ \
--include={'lho-perduta.wav','non-mi-dir.wav','un-bel-di-vedremo.flac'} \
--exclude='*' ~/Music/arias /media/duchess/2tbdisk/duchess/Music/
```

等号和大括号之间不能有空格，逗号和单引号之间也不能有空格。

如果在源目录中的不同位置有多个同名文件，则*rsync*将传输所有这些文件。在此示例中，仅传输了*/home/duchess/Music/arias/sopranos/lho-perduta.wav*，因为*arias/sopranos/lho-perduta.wav*模式在*/Music/arias*中是唯一的：

```
duchess@pc:~$ rsync -av --include=*/ --include=soprano/lho-perduta.wav
--exclude='*' ~/Music/arias /media/duchess/2tbdisk/duchess/Music/
Music/
Music/arias/
Music/arias/baritone/
Music/arias/soprano/
Music/arias/soprano/lho-perduta.wav
Music/arias/tenor/
[...]
```

仅传输一个文件，但所有子目录在*~/Music/arias*都会被复制。使用*-m, --prune-empty-dirs*选项来防止复制空目录，如下例：

```
duchess@pc:~$ rsync -avm --include=*/ --include=soprano/lho-perduta.wav
--exclude='*' ~/Music/arias /media/duchess/2tbdisk/duchess/Music/
Music/
Music/arias/soprano/
Music/arias/soprano/lho-perduta.wav
```

当您有多个要包含的文件时，请将列表存储在纯文本文件中（参见 7.10 和 7.11 的示例）。

## 讨论

*--include=*/* 告诉*rsync*遍历整个源目录。

*--include=[files]* 意味着不排除这些文件。

*--exclude='*'* 告诉 *rsync* 排除所有未被包含的内容。

记住所有文件路径都是相对于你的源目录而不是系统的根目录。

## 参见

+   *man 1 rsync*

+   配方 7.10

+   配方 7.11

# 7.10 使用简单包含文件管理包含内容

## 问题

你的包含内容对于命令行来说太多了，你希望将列表维护在一个 *rsync* 能读取的文件中。你也希望如果可能的话，这一切都简单些，多亏了以前与 *rsync* 的包含/排除文件的经验，那些东西从来就不会正常工作。

## 解决方案

维护清单的最简单方法，不要因为要理解 *rsync* 的包含/排除语法而感到烦躁，就是创建一个纯文本文件清单，然后将其提供给 *--files-from=* 选项。你不必担心它们的顺序，也不用使用 *rsync* 的过滤符号，只需一个包含你想要的文件和目录的简单清单即可。唯一需要注意的是你清单中的每个条目都必须是相对于你的源目录的路径。在下面的示例中，所有清单条目都相对于 */home/duchess*：

```
# include file list
#
/Documents/compositions/jazz/
/Documents/schedule.odt
/Videos/concerts/
.config
.local
/Music/courses/bassoon.avi</strong>
[...]
```

然后使用带有 *--files-from* 选项的列表：

```
duchess@pc:~$ rsync -av ~ --files-from ~/include-list.txt \
 duchess@remote.example.com:/backups/
```

## 讨论

这是维护备份文件和目录列表的最简单方法。没有排除项，没有通配符，没有奇怪的语法，只是一个清晰易懂的干净列表。

当你使用波浪线表示你的主目录时，与配方中的最后一个示例一样，不要加上 *--files-from=* 中的等号。

## 参见

+   *man 1 rsync*

+   配方 7.9

+   配方 7.11

# 7.11 使用排除文件管理包含和排除内容

## 问题

你喜欢配方 7.10 中的简单包含文件概念，但你真的想要同时包含和排除。

## 解决方案

你想要一个 *rsync* 排除文件。排除文件提供了更多的灵活性，并包含包括和排除的内容。下面的示例展示了一个基本的配置。每个条目必须以源根目录开始，本示例中为 */home/duchess*，并以排除源根目录结束：

```
# exclude file list
#
# include home directory
+ /duchess/
#
# include .config and .local, exclude all other dotfiles
+ /duchess/.config
+ /duchess/.local
- /duchess/.*
#
# include jazz/, exclude all other files in Documents
+ /duchess/Documents/
+ /duchess/Documents/compositions/
+ /duchess/Documents/compositions/jazz/
- /duchess/Documents/compositions/*
- /duchess/Documents/*
#
# include schedule.odt, include all .ogg files in
# arias/, exclude all other files in Music
+ /duchess/Music/
+ /duchess/Music/schedule.odt
+ /duchess/Music/arias/*.ogg
- /duchess/Music/arias/*
- /duchess/Music/*
#
# includes courses/, exclude all other files in Videos
+ /duchess/Videos/
+ /duchess/Videos/courses/
- /duchess/Videos/*
#
# exclude everything else
- /duchess/*
```

将其输入 *rsync* 的 *--exclude-from=* 选项：

```
duchess@pc:~$ rsync -av ~ \
 --exclude-from=/home/duchess/exclude-list.txt \
 /media/duchess/2tbdisk/
```

## 讨论

*exclude-list.txt* 示例演示了备份：

+   两个点文件，*.config* 和 *.local*

+   */Documents*、*/jazz* 中的单个子目录

+   */Music* 中的单个文件 *schedule.odt*，以及 */Music/arias/* 中的 *.ogg* 文件

+   */Videos*、*/courses* 中的单个目录

行间不应该有空格，评论符号（#）对于提醒每个部分的目的以及增加一些空白是有用的。包含的部分前面要加上加号，排除的部分前面要加上减号。

所有其他位于*/home/duchess*中的文件都被排除在备份之外。必须首先包括。

```
+ /duchess/Documents/compositions/
- /duchess/*
```

尝试包括*/Documents*：

```
+ /duchess/Documents/
+ /duchess/Documents/compositions/
- /duchess/*
```

现在*/Documents*中的所有子目录及其内容都被传输，而不仅仅是*/compositions*。要仅复制*/compositions*，您必须排除*/Documents*；仅排除*/duchess*是不够的。以下示例仅复制*/duchess/Documents/compositions/*而不复制其他任何内容：

```
+ /duchess/Documents/
+ /duchess/Documents/compositions/
- /duchess/Documents/*
- /duchess/*
```

您可以使用通配符按类型包括或排除文件。例如，包括所有*.ogg*和*.flac*文件，排除所有*.wav*文件，以及排除所有*cache*和*temp*目录：

```
# include home directory
+ /duchess/
#
# include all ogg and flac files
+ *.ogg
+ *.flac
#
# exclude wav files, all cache and temp dirs
- *.wav
- cache*
- temp*
```

您可以有多个源目录。

始终只有一个目标目录。

## 另请参见

+   *man 1 rsync*

+   Recipe 7.10

# 7.12 限制 rsync 的带宽使用

## 问题

大文件传输可能会使用大量网络带宽并减慢所有操作。您希望有一种简单的方法来限制*rsync*的带宽使用，而不实施像流量整形这样复杂的东西。

## 解决方案

使用*rsync*的*--bwlimit*选项。以下示例将其限制为 512 Kbps：

```
$ rsync --bwlimit=512 -ave ssh ~/Music/arias empress@laptop:songs/

```

## 讨论

*--bwlimit*只接受以千位为单位的值。

## 参见

+   *man 1 rsync*

# 7.13 构建 rsyncd 备份服务器

## 问题

您希望用户在中央备份服务器上备份自己的数据，但又不想在备份服务器上给他们提供 shell 帐户。

## 解决方案

设置一个中央备份服务器，并以守护进程模式运行*rsync*。您应该已经设置了名称服务，并且网络上的主机可以访问备份服务器。用户不需要在服务器上有登录帐户，因为您将使用*rsync*自己的访问控制和用户授权来控制对*rsync*存档的访问。

# 仅限局域网使用

这仅适用于局域网使用，不适用于不受信任的网络，因为*rsync*守护程序不会对认证或文件传输进行加密。要进行加密传输，您需要 OpenVPN（第十三章）。

在所有机器上必须安装*rsync*。*rsyncd*在备份服务器上运行，客户端将使用*rsync*命令连接服务器。

在备份服务器上编辑或创建*/etc/rsyncd.conf*以创建定义存档的*rsync*模块：

```
# modules
[*backup_dir1*]
   path = */backups*
   comment = *"server1 public archive"*
   list = yes
   read only = no
   use chroot = no
   uid = 0
   gid = 0

```

创建您的*/backups*目录，模式为 0700，由 root 所有，以防止任何有权访问服务器的人员未经授权访问：

```
$ sudo mkdir /backups/
$ sudo chmod 0700 /backups/

```

在服务器上以守护进程模式使用 systemd 启动*rsyncd*：

```
$ sudo systemctl start rsyncd.service
```

在 Debian/Ubuntu 上，它是*rsync.service*。

如果您的 Linux 系统没有 systemd，请使用*rsync*命令启动它：

```
admin@server1:~$ sudo rsync --daemon
```

在备份服务器上测试*rsyncd*是否正在监听并接受连接：

```
admin@server1:~$ rsync server1::
backup_dir1     "server1 public archive"
```

然后从网络上的另一台 PC 上使用服务器的主机名或 IP 地址进行测试：

```
duchess@pc:~$ rsync server1::
backup_dir1     "server1 public archive"

duchess@pc:~$ rsync 192.168.10.15::
backup_dir1     "server1 public archive"
```

现在您知道它已准备好传输文件。测试您是否可以将文件复制到您的新*rsyncd*服务器：

```
duchess@pc:~$ rsync -av ~/drawings server1::backup_dir1
building file list.....done
drawings/
drawings/aug_03
drawings/sept_03

wrote 1126399 bytes  read 104 bytes  1522.0 bytes/sec
total size is 1130228  speedup is 0.94
```

现在查看漂亮的新上传文件：

```
duchess@pc:~$ rsync server1::backup_dir1/drawings/
drwx------    4,096  2021/01/04  06:06:55    .
-rw-r--r--   21,560  2021/09/17  08:53:18    aug_03
-rw-r--r--   21,560  2021/10/14  16:42:16    sept_03
```

向服务器上传更多文件，然后从 *rsyncd* 服务器下载文件到另一台计算机：

```
madmax@buntu:~$ rsync -av server1::backup_dir1/drawings ~/downloads
receiving incremental file list
created directory /home/madmax/downloads
drawings/
drawings/aug_03
drawings/sept_03

sent 123 bytes  received 11562479 bytes  1755.00 bytes/sec
total size is 1141776 speedup is 1.00
```

一切顺利。休息一下，享受您的成功。

## 讨论

这不是一种安全的文件传输形式，因为没有加密，网络上的任何人都可以访问文件。适合在本地网络上进行简单的归档和文件共享。

连接到运行在守护程序模式下的 *rsync* 服务器时，*rsync [hostname]::* 需要双冒号。这告诉 *rsync* 查找模块名称。

这些是 */etc/rsyncd.conf* 示例中的命令选项：

*[backup_dir1]*

模块名称可以随意设置。

*path =*

定义模块要使用的目录。

*comment =*

这是一个简要描述，提醒您模块属于谁，或者用于什么用途。

*list=yes*

允许用户查看模块中的文件列表。*no* 将隐藏模块。

*read only = no*

允许用户上传文件到服务器。

*use chroot = no*

覆盖了 *use chroot = yes* 的默认设置。*chroot* 是 *change root*，有时被称为 *chroot jail*。*chroot jail* 是您文件系统内的一个独立环境，包含其自己的根文件系统、命令、库以及所有其他所需内容。虽然通常被视为安全工具，但这并不是一个安全的环境。对于 *rsync*，手册页面将其描述为对配置错误有用的保护措施。折衷之处在于，*rsync* 被阻止跟随符号链接到 *chroot* 环境外的文件，且通过名称保存 UID 和 GID 复杂化。正如人们所说，效果因人而异，对您可能是一个不错的选择。参见 *rsyncd.conf (5)* 的 *use chroot* 部分。

将 *uid* 和 *gid* 都设置为 *root* 或 *0*。这样可以保留 UID 和 GID，并正确管理权限。

如果您的任何传输失败，请查看 *rsync* 错误消息。它们会告诉您是否在文件路径中出错、拼写错误或无法连接到服务器，并提供有用的修正提示。

如果您使用的是没有 systemd 的 Linux，请查阅其文档，了解如何启动和停止 rsyncd。

参见 7.14 节 了解如何设置访问控制。

## 参见

+   *man 5 rsyncd.conf*

# 7.14 限制对 rsyncd 模块的访问

## 问题

您不希望开放 *rsyncd* 服务器，并希望用户拥有自己的受保护模块，其他用户无法访问。

## 解决方案

*rsyncd* 自带简单的身份验证和访问控制。创建一个新文件，包含用户名/密码对，并将 *auth users* 和 *secrets file* 指令添加到 */etc/rsyncd.conf*。

首先创建密码文件。在下面的示例中，*/etc/rsyncd-users* 设置了三个用户及其密码：

```
# rsync-users for server1
duchess:12345
madmax:23456
stash:34567
```

设置权限为仅根用户读写：

```
$ sudo chmod 0600 /etc/rsyncd-users
```

现在在*/etc/rsyncd.conf*为您的一个用户创建一个模块。此示例为 Duchess 创建了一个模块，使用*rsync*服务器上的*/backups/duchess*目录：

```
[*duchess_backup*]
   path = */backups/duchess*
   comment = *Duchess's private archive*
   list = yes
   read only = no
   auth users = *duchess*
   secrets file =*/etc/rsyncd-users*
   use chroot = no
   strict modes = yes
   uid = root
   gid = root

```

创建您的用户备份目录，例如 Duchess 的此示例，权限设置为 0700：

```
$ sudo mkdir */backups/duchess/*
$ sudo chmod -R 0700 */backups/duchess/*
```

现在尝试登录：

```
$ rsync *duchess@server1::duchess_backup*
Password: *12345*
drwxr-xr-x      4,096 2020/06/29   18:24:43 .
```

尝试传输一些文件：

```
$ rsync -av ~/logs *duchess@server1::duchess_backup*
Password:
sending incremental file list
logs/
logs/irc.log
logs/irc_#core-standup.log
logs/irc_#core.log
logs/irc_#desktop.log
logs/irc_#engineering.log
logs/irc_#mobile.log

sent 130,507 bytes  received 305 bytes  37,374.86 bytes/sec
total size is 129,383  speedup is 0.99
```

成功了！如果文件传输失败，请查看*rsync*日志了解原因。在 systemd Linux 上，阅读状态输出中的最新日志条目：

```
$ systemctl status rsyncd.service
```

在其他 Linux 发行版上，*rsyncd* 日志应该位于 */var/log*。

## 讨论

用户名/密码对是任意的，与系统用户帐户无关。*rsyncd*用户在其*rsync*共享之外没有访问主机系统的权限。

为了增加安全性，在*/etc/rsyncd.conf*中添加以下指令：

*hosts allow*

使用此选项列出允许访问*rsyncd*档案的主机。例如，您可以限制对单个子网上的主机的访问：

```
hosts allow = **.local.net*
hosts allow = *192.168.1.*
```

所有未允许的主机都会被拒绝，因此您不需要一个*hosts deny*指令。

*hosts deny*

通常情况下，如果使用*hosts allow*，则不需要这个。但对于拒绝访问引起麻烦的特定主机，它很有用。

密码文件以明文存储，因此必须限制为超级用户。

## 参见

+   *man 5 rsyncd.conf*

+   第 7.13 节中的讨论了解命令选项。

# 7.15 为 rsyncd 创建一条消息的尝试

## 问题

您正在运行一个*rsyncd*服务器，并且您认为向用户致以愉快的问候会很好。

## 解决方案

在一个纯文本文件中创建您的每日消息（MOTD），例如*/etc/rsync-motd*：

```
*Welcome to your local backup server! Please remember to actually back up
your files!*
```

然后在*/etc/rsyncd.conf*的顶部配置 MOTD 文件的位置：

```
[global]
motd file = */etc/rsync-motd*
```

当用户连接到您的服务器时，他们会看到您的消息：

```
$ rsync *server1::backup_dir1/*
Welcome to your local backup server! Please remember to actually backup your
files!

drwx------          4,096 2020/06/29 18:24:43 .
-rwxr-xr-x          6,400 2015/03/13 08:21:21 keytool
drwx------          4,096 2020/06/17 06:07:41 WIP
drwx------          4,096 2020/06/17 06:06:55 bin
drwxr-xr-x          4,096 2020/06/30 09:47:42 duchess
[...]
```

## 讨论

每日消息是一个古老的 Unix 传统。您可以用它来欢迎问候、维护停机公告、安全提示、备份技巧或任何您认为重要的事情。

## 参见

+   *man 5 rsyncd.conf*
