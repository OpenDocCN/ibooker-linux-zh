## 第五章. 自动化管理：配置自动远程备份

*本章涵盖*

+   使用脚本自动化管理任务

+   提高安全性和系统效率

+   备份本地数据

+   安排自动化任务

如果我在上一章中已经足够清晰地说明了某一点，那就是定期且可靠的系统备份绝对是**关键**。但是，从许多方面来看，困难的部分在于**定期**。跟上具有直接后果的重要任务已经足够困难；记得运行一些无聊的每日或每周备份几乎是不可能的。

没有什么秘密：解决这个问题的最佳方案是配置一个自动调度器来为你执行任务，然后忘记它。直到最近，Linux 上使用的调度器几乎肯定是一些软件工具 cron 的变体；实际上，这仍然是一个很好的选择。但是，你在第三章（kindle_split_011.xhtml#ch03）中学到的 systemd 进程管理器已经将 systemd 定时器添加到其中。

我将在本章中介绍这两种方法，但也会向你展示如何将备份和其他管理任务打包到脚本中，这些脚本本身也可以被安排在自动化的时间表中。为了展示这一切在实际世界中的工作原理，我将创建一个命令，将一些数据备份到 AWS 简单存储解决方案（S3）桶中，然后使用该命令通过 cron 和 systemd 定时器创建调度器。

### 5.1. 使用 Bash 脚本

Linux 的 *脚本* 是一个包含一个或多个符合 Bash（或某些其他 shell 解释器）命令的纯文本文件。能够在单个文件中串联多个命令，使得创建可以与编程语言在复杂性和多功能性上相媲美的可执行例程成为可能。

#### 5.1.1. 系统文件备份的示例脚本

为了说明一个工作脚本可能的样子，让我给你展示一个简短、写得很好的例子，这个例子可能已经在你的机器上运行了。一旦我们逐行分析完脚本，我会告诉你这一切与本章有什么关系。而且别忘了这个练习的另一个重要收获：如果你能读懂脚本，你也能编写它们。这是完全相同的技能集。

此脚本使用一系列强大的工具来完成实际上相当简单的事情：创建四个重要系统文件的加密备份，以确保在原始文件被意外损坏时有一个可用的替代品。图 5.1 说明了脚本的操作流程图。请注意，`$FILE` 是一个用于表示脚本处理的文件集的变量。

##### 图 5.1. passwd 脚本跟踪的决策流程

![图片](img/05fig01.jpg)

转到 /etc/cron.daily/ 目录并列出内容。你可能会看到一个名为 passwd 的文件。使用 `less`（或 `cat`、`nano` 或 `vim`）显示该文件。由你选择。如果它恰好不在那里，它看起来是这样的：

```
#!/bin/sh

cd /var/backups || exit 0

for FILE in passwd group shadow gshadow; do
        test -f /etc/$FILE              || continue
        cmp -s $FILE.bak /etc/$FILE     && continue
        cp -p /etc/$FILE $FILE.bak && chmod 600 $FILE.bak
done
```

通常 `#` 字符引入的注释不会被解释器读取。在这个特定的例子中，由于同时使用了 `#` 和 `!`，Linux 将读取注释并使用其值（/bin/sh）作为活动 shell。这个字符串通常被称为 *shebang 行*，尽管我不清楚为什么。虽然 `sh` 是 `bash` 的一个替代品，但就我们现在的目的而言，两者之间没有实际的区别。

脚本的下一行将更改目录到 /var/backups/。如果不存在这样的目录，它将退出脚本并发出退出状态码 `0`，这表示命令成功：

```
cd /var/backups || exit 0
```

`||` 序列（有时被称为双竖线）可以读作单词 *or*。所以这一行意味着：要么更改目录到 /var/backups/，要么退出脚本。如果一切按计划进行，后续的脚本操作将在 /var/backups/ 目录中进行。

| |
| --- |

##### 注意

当 Linux 命令完成时，会传递退出码。`0` 将被传递以表示成功，而不同的数字可以配置来指定某种错误。

| |
| --- |

在代码的下一部分，以 `for` 开头的行引入了一个循环。对于有编程经验的你们来说，理解这里发生的事情不会有任何困难：脚本将依次将以下四个字符串（`passwd`、`group` 等等）作为变量 `FILE` 的值。然后它执行 `do` 和 `done` 保留词之间的代码块：

```
for FILE in passwd group shadow gshadow; do
```

这里是一些概念的简要定义：

+   *循环*—由保留词分隔的一系列操作，直到满足指定条件为止

+   *字符串*—字符的连续序列

+   *变量*—一个可以更改并且可以动态地纳入脚本操作中的值

+   *保留词*—由 shell 根据预定义的含义解释的术语

回到脚本，下面第一行将检查 /etc/ 目录中是否存在一个文件，其名称与变量 `$FILE` 的当前值匹配。如果 /etc/ 中没有以该名称命名的文件，则脚本将继续通过将下一个字符串（下一个文件名）分配给 `$FILE` 变量并检查其存在性来继续：

```
test -f /etc/$FILE              || continue
```

如果 /etc/ 中存在这样的文件，则脚本将比较该文件的内容与当前目录（/var/backups/）中具有相同名称加上 .bak 文件扩展名的文件的内容。如果比较操作（`&&`）成功，shell 将继续 `for` 循环并尝试下一个字符串。另一方面，如果两个文件的内容不匹配，则它将移动到下一行：

```
cmp -s $FILE.bak /etc/$FILE     && continue
```

然后，最后，脚本将交付其有效载荷：它将 /etc/ 目录中的当前版本复制到 /var/backups/ 目录，将其名称添加 .bak，并调整文件权限以防止未经授权的用户读取它。此操作将覆盖具有相同名称的任何现有文件。示例中的 `-p` 标志保留源文件的原始所有权属性和时间戳：

```
cp -p /etc/$FILE $FILE.bak && chmod 600 $FILE.bak
```

那个脚本做什么？它旨在创建指定配置文件的副本，这些文件自上次备份以来已更新。以下是工作原理：如果指定的名称在活动 /etc/ 目录中存在，并且其内容与 /var/backups/ 目录中类似名称的文件不同，那么 /etc/ 中的文件将被复制到 /var/backups/，并适当地重命名和加密。

但那四个文件（passwd、group、shadow 和 gshadow）是怎么回事？它们的内容决定了单个用户和组如何能够访问特定资源。例如，如果你查看 /etc/passwd 的内容，你会看到每个账户都有一行。在下面的摘录中，你可以看到常规用户账户被分配了用户和组 ID（例如 ubuntu 的 `1000`），家目录（/home/ubuntu/），以及默认的 shell（bash）。一些系统用户（如 syslog）也有一个默认的 shell，奇怪的是，设置为 /bin/false。这是一种防止人类用户使用该账户登录系统的方法，这会不安全：

```
$ cat /etc/passwd
[...]
syslog:x:104:108::/home/syslog:/bin/false
_apt:x:105:65534::/nonexistent:/bin/false
sshd:x:106:65534::/var/run/sshd:/usr/sbin/nologin
ubuntu:x:1000:1000::/home/ubuntu:/bin/bash           *1*
mysql:x:107:111:MySQL Server,,,:/nonexistent:/
   bin/false                                         *2*
bind:x:108:112::/var/cache/bind:/bin/false
newuser:x:1002:1002:,,,:/home/newuser:/bin/bash
messagebus:x:109:114::/var/run/dbus:/bin/false
```

+   ***1* ubuntu 的用户 ID（1000）、家目录（/home/ubuntu）和默认 shell（bash）**

+   ***2* 不应使用非用户账户进行登录 (/bin/false)。**

当你使用以下方式将新用户添加到你的系统时

```
# useradd -m alan
```

将在新添加的 passwd、shadow 和 group 文件中添加新行。实际上，所有相关的用户管理操作都可以从命令行（或通过脚本）执行，而无需直接编辑这些文件。


##### 注意

Ubuntu 倾向于使用 `adduser username` 而不是 `useradd username`，尽管两者都可以工作。`adduser` 的一个优点是会自动创建家目录，而 `useradd` 需要使用 `-m` 参数。`adduser` 命令还会为新用户提示输入密码。如果你使用 `useradd`，你需要单独运行 `sudo passwd new-user-name` 来设置密码。


以前，每个用户的密码的加密版本也会包含在这里。出于实际原因，因为 passwd 文件必须由系统上的任何人可读，所以包括加密密码被认为是不明智的。那些密码被移动到了 /etc/shadow。使用 `sudo` 权限，你应该在自己的系统上查看包含加密密码的该文件。以下是方法：

```
$ sudo cat /etc/shadow
```

/etc/group 文件包含有关所有当前存在的系统和用户组的基本信息。您可以手动编辑组文件来管理组成员。例如，您可以通过将新加入团队的用户姓名添加到 sudo 组中来授予他们管理权限。该行将看起来像这样：

```
sudo:x:27:steve,newuser,neweruser
```

在名称和逗号之间不要添加任何空格。这样做会导致立即的不愉快。

最后一个文件：/etc/gshadow 文件包含用于在您有时想允许非组成员访问组资源时使用的加密组密码版本。

如您可能已经猜到的，这个脚本是这个章节的一个很好的例子，因为它位于：/etc/cron.daily/ 目录。保存到 /cron.daily/ 目录的脚本将每天执行。我们很快就会回到所有这些。现在，作为另一个简单的例子，这里有一个名为 upgrade.sh 的脚本文件，用于让 `apt` 自动更新所有已安装的软件：

```
#!/bin/bash
# Script to automate regular software upgrades

apt update
apt upgrade -y
```

如您所无疑问地回忆起，`apt update` 命令将与在线仓库的索引同步，确保 APT 了解所有最新的软件包和版本。`apt upgrade` 将下载并安装任何相关的更新。`-y` 将在询问确认操作时自动回答是。

您仍然还没有准备好运行您的脚本。因为您将以程序的方式运行脚本，您需要更改文件属性使其可执行。`chmod +x` 后跟文件名即可完成此操作：

```
$ chmod +x upgrade.sh
```

就这样。现在您可以自由地将文件复制到 /etc/cron.daily/ 目录中，这样它就可以与 passwd 和其他文件一起每天运行：

```
# cp upgrade.sh /etc/cron.daily/
```

由于它运行 `apt`，新的脚本需要管理员权限，但不需要在命令本身中包含 `sudo`。默认情况下，Cron 总是以 root 身份运行。如果您想直接从命令行运行脚本，则需要添加 `sudo` 并在文件名前加上点号和正斜杠，以告诉 Linux 您引用的命令位于当前目录中：

```
$ sudo ./upgrade.sh
```

#### 5.1.2\. 一个用于更改文件名的示例脚本

让我再向您介绍一些脚本工具。您可能已经遇到了这样一个事实，即 Linux shell 有时可能会错误地解释包含空格的文件名。如果您尝试 `cat` 一个名为 big name 的文件的 内容，它看起来会是这样：

```
$ cat big name
cat: big: No such file or directory
cat: name: No such file or directory
```

简单的解决方案是将完整的文件名用单引号或双引号括起来，如下所示：

```
$ cat 'big name'
Hello world
```

但这个选项并不总是可用。在这种情况下，您可以通过将文件名中的空格自动转换为，例如，下划线字符来自动化一个过程。然后，在您的旅途中，当您遇到包含大量违规文件名的目录时，您将能够执行一个脚本来快速修复问题。好吧，这就是它：

```
#!/bin/bash
echo "which directory would you like to check?"
read directory
   find $directory -type f | while read file; do
    if [[ "$file" = *[[:space:]]* ]]; then
    mv "$file" `echo $file | tr ' ' '_'`
    fi;
   done
```

`echo` 行将文本打印到屏幕上，然后等待用户输入。用户将输入一个有效的目录，如 /home/ubuntu/files/，它将被分配为变量 `directory` 的值。`find` 命令将被调用以返回指定目录中的所有文件对象 (`-type f`)。`find` 的文件名集将逐个在 `while` 循环中读取，每个文件都会检查是否存在空格。如果找到空格，`then` 将文件名中的任何空格（‘ ’）更改为下划线（‘_’）。`fi;` 当目录中没有更多文件名时停止循环。

要尝试这个，创建一个包含带有空格的文件名的几个文件的目录，然后运行脚本。一个看起来像这样的目录

```
$ ls
file name  file - name
```

应该看起来像这样：

```
$ ls
file_name  file_-_name
```

| |
| --- |

##### 注意

仔细思考组成脚本的每个步骤，并确保你确切地了解正在发生什么。

| |
| --- |

让我们花一点时间提醒自己我们现在在哪里。确保你不是只盯着树木而错过了整个森林总是一个好主意。到目前为止发生的事情如下：

+   本章是关于使用脚本创建自动备份。

+   你探索了从 /etc/ 到 /var/backup/ 的用户管理员文件的备份脚本。

+   你学习了那些用户管理员文件的照顾和喂养。

+   你编写了自己的简单脚本。

接下来还将发生什么：

+   你将把自己的数据备份到 AWS S3 存储桶。

+   你将使用 cron 和 anacron 来安排定期备份。

+   你将学习如何使用 systemd 定时器来完成这项工作。

脚本可以用于比备份和文件名多得多的用途。随着对服务器和网络环境需求的不断增长，有时需要数百甚至数千个动态生成的虚拟微服务，手动管理几乎是不可能的。在你作为系统管理员的职业生涯中，你可能会需要创建脚本来配置和启动单个和成群的虚拟机，以及监控庞大且不断变化的环境。

### 5.2\. 将数据备份到 AWS S3

我选择亚马逊的 AWS S3 作为我将要使用的备份示例的目标，有以下两个原因：

+   总是保持重要数据离线副本至关重要。

+   将存档到 S3 是目前流行的事情，而且非常简单。

就这样。这个事实是，这是一个极好的机会来毫不掩饰地推广我的书《一个月午餐时间学习亚马逊网络服务》（Manning，2017），这本书充满了你可能需要知道的所有关于 AWS 的知识，这绝对没有影响我的选择。没有。好吧，也许只是有一点。

无论如何，如果您还没有自己的 AWS 账户，您仍然可以跟随下一节，并用您自己的备份脚本替换我将向您展示的 AWS 脚本。或者，您也可以访问 [`aws.amazon.com`](https://aws.amazon.com) 并注册一个新账户。开设账户不会花费您任何费用，并且根据免费层，许多服务（包括 5 GB 的 S3 存储）在第一年都是免费的。

顺便说一句，即使您的 AWS 免费层权限已经结束，存储费用仍然大约为每月每 GB 0.025 美元。这可能足够便宜，足以让您改变对异地存档的一般看法。

#### 5.2.1\. 安装 AWS 命令行界面 (CLI)

您可以在 AWS 控制台中通过浏览器执行许多 AWS 管理任务，但这并不是真正的 Linux 管理员完成任务的方式。如果您打算将备份到 S3 的操作集成到脚本中，那么它必须在命令行上工作。为此，您无需寻找其他选择，只需使用亚马逊自己的 AWS CLI。因为它运行在 Python 上，所以您至少需要运行 Python 2（版本 2.6.5）或 Python 3（版本 3.3）。此外，您还需要 pip Python 包管理器来处理安装。截至本文撰写时，Ubuntu 正在努力将 Python 3 作为其默认版本，尽管其他发行版可能仍然更倾向于 Python 2。

如果以下安装命令不起作用，那么您可能需要安装 pip。（使用 `apt install python-pip` 或 `apt install python3-pip`。）以下是超秘密、内部人士专属、我可以向您透露但之后我必须杀掉您的隐藏代码，它告诉您需要哪种 pip 安装。如果它有效，那就是正确的。如果不起作用，请尝试另一个：

```
$ pip3 install --upgrade --user awscli
Collecting awscli
  Downloading awscli-1.11.112-py2.py3-none-any.whl (1.2MB)             *1*
    100% |################################| 1.2MB 946kB/s
Collecting PyYAML<=3.12,>=3.10 (from awscli)
  Downloading PyYAML-3.12.tar.gz (253kB)
    100% |################################| 256kB 2.2MB/s
[...]
Collecting jmespath<1.0.0,>=0.7.1 (from botocore==1.5.75->awscli)
  Downloading jmespath-0.9.3-py2.py3-none-any.whl
Collecting six>=1.5 (from python-dateutil<3.0.0,>=2.1->
    botocore==1.5.75->awscli)
  Downloading six-1.10.0-py2.py3-none-any.whl
Building wheels for collected packages: PyYAML
  Running setup.py bdist_wheel for PyYAML ... done
  Stored in directory: /home/ubuntu/.cache/pip/wheels
    /2c/f7/79/13f3a12cd723892437c0cfbde1230ab4d82947ff7b3839a4fc
Successfully built PyYAML
Installing collected packages: PyYAML, pyasn1, rsa, colorama, six,
    python-dateutil, docutils, jmespath, botocore, s3transfer, awscli  *2*
Successfully installed PyYAML awscli botocore colorama docutils
    jmespath pyasn1 python-dateutil rsa s3transfer six
```

+   ***1* pip 可以很好地显示实时进度详情。**

+   ***2* pip 安装的包的摘要**

#### 5.2.2\. 配置您的 AWS 账户

现在，您已经准备好将本地 AWS CLI 与您的 AWS 账户链接。为此，您需要检索一些访问密钥。在控制台中的任何页面，点击带有您账户名称的下拉菜单标题（页面右上角），然后点击“我的安全凭证”链接（图 5.2）。

##### 图 5.2\. AWS 控制台，直接链接到数十个 AWS 服务。这里可以看到“我的安全凭证”链接。

![](img/05fig02_alt.jpg)

一旦您进入“我的安全凭证”页面，点击展开“访问密钥”（访问密钥 ID 和秘密访问密钥）部分，并注意可能出现的警告：现有根密钥无法检索。然后点击“创建新访问密钥”按钮。您将看到您的新访问密钥 ID 和其相应的秘密访问密钥。前者与登录名具有相同的功能，后者则像密码一样。您可以将访问密钥下载并保存到计算机上的安全位置，或者选择、复制并粘贴到其他地方。

为了设置这些，在你的本地机器上打开一个终端，并从命令行运行 `aws configure`。你将被要求输入你的访问密钥 ID、你的秘密访问密钥、你想要设置为默认的 AWS 区域，以及你想要使用的输出格式。如果你愿意，可以留出最后两个值。以下 AWS 文档中的示例使用了假凭证（你永远不应该公开显示真实的密钥集）：

```
$ aws configure
AWS Access Key ID [None]: AKIAIOSFODNN7EXAMPLE
AWS Secret Access Key [None]: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
Default region name [None]: us-east-1                                   *1*
Default output format [None]:                                           *2*
```

+   ***1* 确定资源将从中启动的 AWS 地理中心**

+   ***2* 选择以文本（默认）、JSON 或表格格式显示输出。**


##### 注意

在设置默认区域名称时，请记住，地方监管制度可能会限制将某些数据备份到海外服务器。


现在，你应该已经准备好开始工作了。以下命令行

```
$ aws s3 ls
```

将列出你账户中的所有 S3 存储桶。*存储桶* 是 AWS 用于我们称之为目录的术语。

#### 5.2.3\. 创建你的第一个存储桶

假设这是一个全新的账户，将不会显示任何内容。你应该使用创建存储桶命令 `mb` 创建一个新的存储桶。在选择存储桶名称时要注意的是，它必须在整个 S3 系统中是唯一的。如图所示，像 mybucket 这样的名称可能不会被接受：

```
$ aws s3 mb s3://mybucket
make_bucket failed: s3://mybucket/ An error occurred (BucketAlreadyExists)
 when calling the CreateBucket operation: The requested bucket name is not
 available. The bucket namespace is shared by all users of the system.
 Please select a different name and try again.
```

相反，使用不太常见的词汇并添加几个数字可能会效果更好：

```
$ aws s3 mb s3://linux-bucket3040
```

再多一步，我们就有了离线备份工作。假设你需要备份的文件位于名为 /dir2backup 的目录中，该目录位于你的主目录中，以下是它的工作方式：

```
aws s3 sync /home/username/dir2backup s3://linux-bucket3040
```

`s3 sync` 工具的工作方式与你在第四章中遇到的 `rsync` 工具非常相似。第一次运行它时，源目录中的所有内容都将上传到你的 S3 存储桶；随后，只有新文件或更改的文件将被传输。创建一个运行该 `sync` 命令的脚本将非常直接。下面是如何看起来：

```
#!/bin/bash
/usr/local/bin/aws s3 sync \
 /home/username/dir2backup s3://linux-bucket3040
```

注意我添加了 `aws` 命令的完整路径 (/usr/local/bin/aws)。这是为了确保 Bash 知道在系统中的哪个位置可以找到该命令。你可以使用 `whereis` 命令来确认 `aws` 的位置：

```
$ whereis aws
aws: /usr/local/bin/aws
```

虽然已经部署了优秀的备份工具，但这并不意味着你会使用它。这需要某种类型的任务调度器。你将在下一节中了解几个这样的调度器。

### 5.3\. 使用 cron 定期安排备份

Cron 有多种风味。因为那个数字大于一，你可以预期通常有不止一种方法来完成特定的任务。为了了解有哪些内容，列出 /etc/ 目录中包含字母 *cron* 的对象：

```
$ ls /etc | grep cron
anacrontab
cron.d
cron.daily
cron.hourly
cron.monthly
crontab
cron.weekly
```

在这些中，只有 anacrontab 和 crontab 是文件；其余的是目录。让我们先看看这些目录是如何工作的。

如果你有一个文件系统备份脚本，你希望它在设定的时间间隔内运行，你可以将其复制到相应的目录：`cron.hourly/` 以每小时执行一次，`cron.daily/` 以每天执行一次，依此类推。`cron.d/` 目录略有不同。它是为那些内容精确控制命令执行时间的文件设计的。

假设你想要运行我之前编写的那个软件升级任务，但不需要脚本，每周一运行一次。你可以在 `/etc/cron.d` 目录中创建一个包含以下内容的文件：

```
21 5  * * 1  root apt update && apt upgrade
```

这个例子将在每周一早上 5:21 运行升级。这是怎么工作的？看看前几个字符：`21 5`。第一个字段（`21`）代表你希望命令运行的每小时中的分钟数。在这种情况下，它将在 21 分钟后运行。下一个字段是你可以指定一天中的哪个小时（例如，8 或 23）。这个是 `5`，意味着早上 5:00。接下来的两个星号后面跟着一个 `1` 表示你想要在每月的每一天、每年的每个月、每周的每个星期一（1 代表星期一）遵循该计划。为了避免混淆，可以使用 0 或 7 来表示星期日。`root` 参数意味着命令将以 root 用户身份运行。

为什么这么早？因为，假设在每个人都进入办公室之前，对网络带宽的竞争需求会较低。为什么是 5:21 而不是 5:00？因为你不想养成将所有脚本都安排在每小时顶点（或任何特定时间）的习惯，因为这可能会最终导致冲突。最好是错开时间。

你可以直接将那一行添加到 `/etc/crontab` 文件中，它将以相同的方式运行，但不需要创建一个单独的文件。但我不建议这样做。这样做可能不会在世界范围内引发僵尸末日，但仍然是一个糟糕的想法。你看，在系统升级期间，crontab 文件可能会被覆盖，你的自定义命令将会丢失。

那么，crontab 文件有什么用呢？正如你应该能够从文件内容中看到的那样（图 5.3），这是执行 `/cron.?` 目录中脚本的调度器。花一两分钟自己查看每个命令。

##### 图 5.3。详细说明的 `/etc/crontab` 文件，展示了四个任务，每个任务都旨在启动 `/etc/cron.?` 目录的内容

![图片](img/05fig03_alt.jpg)

图 5.3 中最后三个命令开头的 `test -x` 确认了一个名为 anacron 的二进制程序存在且可执行。如果其中任何一个不成立，那么 `/cron.?` 目录中的脚本将会运行。`test -x` 命令在脚本编写中通常很有用，当你需要在启动相关操作之前确认对象的状态时。

你肯定不想把/etc/crontab 文件留给专业人士。但鉴于 Linux 关心你的感受，他们给了你自己的 crontab 来玩。这个 crontab 将以你的用户身份而不是 root 身份运行命令，只有你拥有的权限——并且在升级过程中不会被覆盖。

想看看你已经安排了什么？运行`crontab -l`。如果你还没有添加任何命令，你可能会看到以下消息：没有为*yourname*设置 Crontab。

```
$ crontab -l
no crontab for ubuntu
```

你可以使用`crontab -e`编辑你的 crontab。第一次编辑时，你会被要求选择一个文本编辑器。如果你已经熟悉 Nano，就选择它，因为它可能是列表中三个中最容易使用的：

```
$ crontab -e
no crontab for ubuntu - using an empty one            *1*

Select an editor. To change, run 'select-editor'.     *2*
  1\. /bin/nano                                        *3*
  2\. /usr/bin/vim.basic
  3\. /usr/bin/vim.tiny

Choose 1-3 [1]:
```

+   ***1* 检查现有任务**

+   ***2* 选择一个文本编辑器。**

+   ***3* Nano 编辑器选择**


##### 注意

默认情况下，在某些系统上，如果没有创建包含用户名的文件/etc/cron.allow，用户将无法创建 crontab 任务。然而，Debian/Ubuntu 允许直接使用单个 crontab 任务。


所有基于 cron 的工具对于可能一直运行的计算机（如生产服务器）都工作得很好。但如果你想在经常关闭的笔记本电脑上执行重要任务怎么办？告诉 cron（或 cron.daily 等）在周一早上 5:21 备份你的文件听起来很好，但你有多可能记得及时开机？正好——几乎不可能。是时候谈谈 anacron 了。

### 5.4\. 使用 anacron 安排不规则的备份

我们还没有讨论的一个 cron 文件是*anacrontab*，这是你安排在每次系统启动后固定时间运行操作的地方。如果你想在笔记本电脑上备份这些文件，但又不能保证在一天中的任何给定时间都能开机，你可以在 anacrontab 中添加一行。

在 anacrontab 文件中，你应该注意到的就是条目只有两列用于控制时间，而不是`cron`命令中使用的五列。这是因为 anacron 不是基于绝对时间工作，而是相对于最近的系统启动。以下是文件的全部内容：

```
# /etc/anacrontab: configuration file for anacron
# See anacron(8) and anacrontab(5) for details.
SHELL=/bin/sh
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/
       usr/sbin:/usr/bin                                *1*
HOME=/root
LOGNAME=root

# These replace cron's entries
1 5 cron.daily run-parts --report /etc/cron.daily         *2*
7 10 cron.weekly     run-parts --report /etc/cron.weekly
@monthly 15 cron.monthly run-parts --report
   /etc/cron.monthly
```

+   ***1* anacrontab 的 PATH 中的目录。这些位置中的文件可以在不引用完整目录树的情况下进行引用。**

+   ***2* 此任务每天执行一次/etc/cron.daily/目录中的任何脚本。**

例如，anacrontab 文件中的`cron.daily`行在启动后每隔一天运行，正好在启动后 5 分钟，无论何时发生。另一方面，`cron.weekly`命令在每隔七天（即自上次运行以来至少过去了七天）后运行，启动后 10 分钟。

你可能也对那些`cron.?`命令感到好奇；它们不是已经通过`/etc/crontab`文件运行了吗？是的，但前提是系统上 anacron 没有激活。正如你从 crontab 文件中看到的，anacron 被赋予了比 cron 更高的优先级。考虑到所有这些，下面是如何为你的笔记本电脑运行备份脚本的方法：

```
1    10    myBackupJob    /home/myname/backup.sh
```

这个命令每天在系统启动后 10 分钟内最多运行一次`backup.sh`脚本。作业标识符是`myBackupJob`，一个包含作业状态信息的同名日志文件将被保存到`/var/spool/anacron/`目录中。

#### 5.4.1. 运行 S3 同步任务

现在，最后，你已经学习了关于脚本的所有知识——AWS S3、cron 和 anacron——你终于准备好做出明智的选择：安排备份的最佳方式是什么？而且，像往常一样，正确答案取决于上下文。

如果你想要确保备份即使在机器不总是运行的情况下也能完成，你可以在 anacrontab 文件中添加类似这样的行（假设你已经在`/etc/`中有一个同名的可执行脚本文件）：

```
1 10 myDailyBackup /etc/s3script.sh
```

这个例子每天都会运行脚本（`1`），在系统启动后 10 分钟（`10`）执行。对于 24/7 的服务器，你可以将特定于语法的指令添加到用户的 crontab 中，或者在一个文件中添加到`/etc/cron.d/`目录中（尽管你可能需要将用户名如 root 添加到 cron.d 版本中）。

下一个例子每天早上 5:47 运行脚本，每个月每一天，每周每一天（`* * *`）：

```
47 5 * * * /home/myusername/s3script.sh           *1*
```

+   ***1. crontab 语法每天早上 5:47 运行脚本**

在这个例子中，因为我们谈论的是一个简单的脚本，你也可以通过将`aws`命令本身插入到 cron 文件（如 anacrontab）中来同样有效地做到这一点：

```
47 5 * * * username /usr/local/bin/aws s3 sync
   /home/username/dir2backup s3://linux-bucket3040
```

可以直接将像这种备份操作的具体命令插入到 crontab 文件中。注意用户名和`aws`的绝对位置。这清楚地确立了你要运行的过程的所有权和运行它的二进制文件的文件系统位置。

### 5.5. 使用 systemd 定时器安排定期备份

如果对新的 systemd 定时器替代 cron 有批评，那可能是因为它明显更复杂，需要更多步骤来设置。你不仅需要创建你的脚本，然后将文件复制到正确的 cron 目录中，你还需要创建并保存两个单独的文件到不为人知的目录位置（这还不是所有 Linux 发行版的行业标准），然后运行两个`systemctl`命令。

可能复杂，但绝对不是不可逾越的。systemd 定时器带来了一些显著的优势，包括与其他系统服务的更深层次集成（包括日志）以及根据系统状态的变化执行命令的能力（例如，有人连接 USB 设备），而不仅仅是设定时间。

我会留给你时间去探索那些你感兴趣的特定、深入的功能。但我会带你一步步完成一个简单的备份。你的任务？生成一个动态命名的 tar 归档文件，用于备份你服务器上运行的 Apache 网站。为了演示目的，你可以选择任何目录进行备份，而且你不需要配置和启动一个网站来完成这个任务。

首先，看看你可能已经运行的所有定时器。我的定时器通过`systemctl list-timers --all`命令显示在图 5.4 中。

##### 图 5.4\. `systemctl list-timers --all`命令为所有现有的 systemd 定时器作业提供了丰富的历史数据。

![](img/05fig04_alt.jpg)

这些定时器都是系统自动创建的。要创建你自己的，从一个备份脚本开始。深入挖掘我丰富多彩、富有创意的内心世界，我会把我的脚本文件命名为 site-backup.sh。下面是这个脚本的样子：

```
#!/bin/bash
NOW=$(date +"%m_%d_%Y")                                     *1*
tar czvf /var/backups/site-backup-$NOW.tar.gz /var/www
```

+   ***1* 将系统日期分配给环境变量$NOW**

如果我的归档文件名总是包含创建日期，那么识别归档文件会容易得多。为了做到这一点，我将当前系统日期分配给`$NOW`变量的值，并将其包含在新归档的文件名中。下面是结果文件名可能的样子：

```
site-backup-11_28_2017.tar.gz
```

不要忘记使你的脚本文件可执行（`chmod +x site-backup-11_28_2017.tar.gz`）。实际上，永远不要忘记使你的脚本文件可执行！

```
$ chmod +x site-backup.sh
```

现在，你需要创建你的.service 和.timer 文件。正如我之前所写的，没有单一的位置来保存所有的服务文件，但/lib/systemd/system/和/etc/systemd/system/都可以工作。如果可以选择，我更喜欢/etc/systemd/system/，因为对我来说，这是一个容易记住且逻辑上合理的地方。你的体验可能会有所不同。

我将从.service 文件开始，我将称之为 site-backup.service。*服务文件*在 systemd 操作中是通用的。它们旨在以统一、可预测的方式描述和定义系统服务。`Description`值应包含你认为可以准确描述服务的任何文本，而`ExecStart`行指向可执行资源的位置：在这个例子中是脚本。这是 systemd 需要了解你想要做什么的所有信息：

```
[Unit]
Description=Backup Apache website

[Service]
Type=simple
ExecStart=/home/username/site-backup.sh         *1*

[Install]
WantedBy=multi-user.target
```

+   ***1* 服务要运行的可执行资源**

.timer 文件，这是 systemd 定时器特有的，告诉 systemd 何时运行关联的服务。关联是通过`[Timer]`部分的`Unit`行设置的，在这个例子中，它指向我的 site-backup.service 文件。请注意，在这个例子中，`OnCalendar`的值被设置为每天早上 5:51 执行（`*-*-*`），而`Unit`的值是 site-backup.service 文件：

```
[Unit]
Description=Backup Apache website - daily

[Timer]
OnCalendar=*-*-* 5:51:00          *1*
Unit=site-backup.service          *2*

[Install]
WantedBy=multi-user.target
```

+   ***1* 控制服务执行的计划设置**

+   ***2* 与此定时器关联的服务单元**

有这些文件在位，您可以使用 `systemctl start` 启动服务。此外，您可以使用 `systemctl enable` 将其设置为每次系统启动时自动加载：

```
# systemctl start site-backup.timer
# systemctl enable site-backup.timer
```

对您服务的状态感到好奇？`is-enabled` 和 `is-active` 应该是一个好的开始：

```
# systemctl is-enabled backup.timer
enabled
# systemctl is-active backup.timer
active
```

最后，当您编辑您的 .timer 文件时，您需要更新系统。在练习本章所学内容的过程中，您可能需要进行大量的编辑。当然，您会想知道如何做到这一点。好吧，这里就是：

```
# systemctl daemon-reload
```

### 摘要

+   编写良好的 Bash 脚本可以高效且可靠地自动化复杂和简单的管理任务。

+   Linux 在 /etc 目录中用纯文本文件（名为 passwd、group、shadow 和 gshadow）存储用户账户和认证信息。

+   您可以将本地数据备份到 S3 存储桶，并通过命令行直接管理其生命周期。

+   将可执行脚本复制到 /etc/cron.? 中的一个目录会导致它以适当的间隔运行。

+   在 anacrontab 文件中添加指令将执行与系统启动相关的命令，而不是在绝对时间。

+   systemd 计时器可以根据绝对时间和系统事件（如硬件状态的变化）来设置运行。

### 关键术语

+   所有 Linux 命令在完成时都会输出 *退出代码*：0 表示成功执行，但所有正整数都可以由程序设置来表示各种失败状态。

+   通过使用 *访问密钥*（访问密钥 ID 和秘密访问密钥）来保护对 AWS 资源的命令行访问。

+   *存储桶* 是 AWS 对象，它在操作系统目录中工作方式几乎相同。

### 安全最佳实践

+   将系统账户（如 syslog 和理想情况下甚至 root）锁定，以防止它们被用于远程登录。

+   在您的安全规划中包含异地备份，这为数据可靠性增加了另一层。

+   总是保护您的（AWS）访问密钥，不用说密码和加密密钥对，免得任何形式的公开暴露。

### 命令行审查

+   `#!/bin/bash`（所谓的“shebang 行”）告诉 Linux 您将使用哪个 shell 解释器来执行脚本。

+   `||` 在脚本中插入一个 *或* 条件。想想看，这是“左边的命令成功”或“执行右边的命令”。

+   `&&` - 在脚本中插入一个 *和* 条件。想想看，这是“如果左边的命令成功”然后“执行右边的命令”。

+   `test -f /etc/filename` 检查指定的文件或目录名是否存在。

+   `chmod +x upgrade.sh` 使脚本文件可执行。

+   `pip3 install --upgrade --user awscli` 使用 Python 的 pip 软件包管理器安装 AWS 命令行界面。

+   `aws s3 sync /home/username/dir2backup s3://linux-bucket3040` 将本地目录的内容与指定的 S3 存储桶同步。

+   `21 5 * * 1 root apt update && apt upgrade`（一个 cron 指令）在每天早晨 5:21 执行两个 `apt` 命令。

+   `NOW=$(date +"%m_%d_%Y")` 将当前日期分配给脚本变量。

+   `systemctl start site-backup.timer` 激活一个 systemd 系统定时器。

### 测试自己

> **1**
> 
> Linux 脚本中用于引入注释的字符是什么？
> 
> 1.  !
> 1.  
> 1.  //
> 1.  
> 1.  #
> 1.  
> 1.  ^
> 1.  
> **2**
> 
> Linux 脚本中`||`的作用是什么？
> 
> 1.  或者
> 1.  
> 1.  和
> 1.  
> 1.  If
> 1.  
> 1.  Comment
> 1.  
> **3**
> 
> /etc/shadow 文件中保存的数据类型是什么？
> 
> 1.  账户组及 shell 数据
> 1.  
> 1.  组成员数据
> 1.  
> 1.  加密账户密码
> 1.  
> 1.  加密组密码
> 1.  
> **4**
> 
> 以下哪个操作将在你的 AWS S3 账户中创建一个新的存储桶？
> 
> 1.  `s3 mb s3://mybucket`
> 1.  
> 1.  `aws s3 mb s3://mybucket`
> 1.  
> 1.  `aws s3 cb s3://mybucket`
> 1.  
> 1.  `aws s3 sync mb s3://mybucket`
> 1.  
> **5**
> 
> 哪个命令可以让你输入一个以你自己的用户身份运行的 cron 指令？
> 
> 1.  `nano anacrontab`
> 1.  
> 1.  `crontab -l`
> 1.  
> 1.  `nano /etc/crontab`
> 1.  
> 1.  `crontab -e`
> 1.  
> **6**
> 
> 以下哪个将在每周一早上运行命令？
> 
> 1.  `21 * 1 * * root apt update && apt upgrade`
> 1.  
> 1.  `21 5 * * 1 root apt update && apt upgrade`
> 1.  
> 1.  `21 1 * * 0 root apt update && apt upgrade`
> 1.  
> 1.  `21 5 * 4 * root apt update && apt upgrade`
> 1.  
> **7**
> 
> 以下哪个选项不适用于始终运行的计算器？
> 
> 1.  crontab
> 1.  
> 1.  anacron
> 1.  
> 1.  systemd timer
> 1.  
> 1.  anacrontab
> 1.  
> **8**
> 
> `systemctl enable site-backup.timer`命令的作用是什么？
> 
> 1.  手动加载网站备份定时器
> 1.  
> 1.  配置网站备份定时器在系统启动时加载
> 1.  
> 1.  显示网站备份定时器的当前状态
> 1.  
> 1.  强制网站备份定时器在计算机关闭前运行

#### 答案键

> **1.**
> 
> c
> 
> **2.**
> 
> a
> 
> **3.**
> 
> c
> 
> **4.**
> 
> b
> 
> **5.**
> 
> d
> 
> **6.**
> 
> b
> 
> **7.**
> 
> a
> 
> **8.**
> 
> b
