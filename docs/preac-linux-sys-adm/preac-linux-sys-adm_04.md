# 第四章：管理用户

作为系统管理员，您将花费大部分时间管理用户。您还将花时间解决与账户或权限无关的用户问题，如连接问题、应用程序故障、数据损坏、培训问题、安全问题和用户创建的问题。

用户管理涵盖以下任务：

+   创建用户账户

+   修改用户账户

+   移除用户账户

+   授予对文件和目录的访问权限

+   限制对文件和目录的访问

+   强制执行安全策略

+   对文件和目录设置权限

本书可以学到一些用户管理任务，而其他任务则纯粹是您的经验和在职培训。没有两个用户环境完全相同，没有两个用户体验完全相同。在本章中，您将学习一些预防性的用户管理方法，但问题仍然会发生。本章中学到的技术将帮助您开始处理各种与用户相关的问题。

# 用户和组 ID 编号约定

在 Linux 系统上创建和维护用户账户有一些指导原则，如表 4-1 所示。这些不是硬性规定，但通常在大多数企业系统中遵循。

表 4-1\. 用户和组账户编号约定

| UID | GID | 描述 |
| --- | --- | --- |
| 0 | 0 | Root |
| 1–999 | 1–999 | 系统/服务账户 |
| 1000+ | 1000+ | 用户账户 |

用户账户的 UID 和 GID 通常从 1000 开始，每新增一个账户递增一个数。root 用户的 UID 和 GID 始终为 0；系统上没有其他用户拥有这些用户和组 ID。

系统和服务账户不是人类用户账户，通常没有关联的交互式 shell。这些账户的 UID 和 GID 范围从 1 到 999。这些分离使得系统维护比随机分配 UID 和 GID 给用户账户要容易得多。

# 创建用户账户

像 Linux 中的大多数任务一样，创建用户账户有多种方法。对于本书，我坚持使用两种主流方法来创建账户：`useradd` 和 `adduser`。

## 使用 `useradd` 添加用户

`useradd` 命令是在 Linux 系统中添加新用户的标准命令行方法。`useradd` 命令很简单；您真正需要提供的只是一个用户名作为参数：

```
# useradd jsmith
```

这将创建主目录 */home/jsmith*，填充默认的隐藏环境文件，并将条目添加到 */etc/passwd* 中。当我使用 `useradd` 创建用户账户时，我只需提供一个参数和一些信息（用户的全名），否则需要编辑 */etc/passwd* 文件：

```
# useradd -c "Jane Smith" jsmith
```

`-c` 选项将您提供的信息写入到 */etc/passwd* 文件的第五个字段中。如果您希望提供更多信息，例如电话号码、电子邮件地址或其他任何您想包含的信息，请使用逗号（`,`）来分隔这些信息：

```
# useradd -c "Jane Smith,Room 26,212-555-1000,jsmith@example.com" jsmith
```

新用户的 */etc/passwd* 条目：

```
jsmith:x:1007:1007:Jane Smith,Room 26,212-555-1000,jsmith@example.com:/home/
jsmith:/bin/bash
```

用户的 */etc/passwd* 条目中各个字段依次是（从左到右）：

+   用户名

+   */etc/shadow* 密码字段

+   用户 ID

+   主组 ID

+   注释字段

+   家目录

+   默认 shell

密码不存储在 */etc/passwd* 文件中。*/etc/shadow* 字段指的是包含每个用户加密密码的 */etc/shadow* 文件，只有 root 用户可以读取。请注意，在基于 Red Hat Enterprise Linux 的系统中，*/etc/shadow* 文件的权限为 `000`。在不同的发行版中，文件权限会有所不同，但从不允许普通用户读取该文件：

```
----------. 1 root root 1547 Jul 17 10:55 /etc/shadow
```

尽管 Jane Smith 的用户账户已经创建，她的家目录也存在，并且在 */etc/passwd* 文件中有一个条目来表示该账户，但是 Jane 无法登录系统。知道为什么吗？因为这个账户没有密码。作为系统管理员，您需要为 Jane 提供一个初始密码，以便用户可以登录。由于您尚未为该账户提供密码，因此 */etc/shadow* 条目显示没有密码：

```
jsmith:!!:18825:0:99999:7:::
```

使用 `passwd` 命令为账户设置密码：

```
# passwd jsmith
Changing password for user jsmith.
New password:
Retype new password:
passwd: all authentication tokens updated successfully.
```

现在，您必须在 Jane 成功登录系统之前为她设置密码。

## 使用 adduser 添加用户

在一些 Linux 发行版中，`adduser` 是指向 `useradd` 的符号链接：

```
lrwxrwxrwx. 1 root root 17 Oct 26 2020 /usr/sbin/adduser -> /usr/sbin/useradd
```

在其他发行版中，`adduser` 是一个交互式 Perl 脚本，引导您逐步添加新用户，而 `useradd` 是一个独立的实用程序，带有其标准开关和参数。

# 修改用户帐户

几乎不存在静态用户账户。因此，`usermod` 命令存在以帮助您进行这些必要的更改，而无需编辑 */etc/passwd*、家目录或配置文件。`usermod` 命令是用于更改与用户账户相关的所有内容的综合命令。以下是您可以使用 `usermod` 命令进行的修改的简要列表：

+   将用户添加到附加组

+   更改 */etc/passwd* 中的用户注释字段

+   更改用户的家目录

+   设置账户过期日期

+   删除过期日期

+   更改用户的登录名（用户名）

+   锁定/解锁用户账户

+   移动用户家目录的内容

+   更改用户的登录 shell

+   更改用户的 ID

这些选项中有些比其他选项更常用。例如，更改用户的登录 shell、锁定和解锁账户、设置或删除过期日期或将用户添加到附加组是完全合理的。一旦在账户创建过程中设置了用户的 ID，很少会再次更改用户的 ID，就像很少会重新定位用户的家目录一样。

在接下来的章节中，我将给出用户账户最常被请求修改的示例。如果你需要改变用户账户的其他方面，请查阅手册获取详细信息。

## 添加一个附加组

当你创建一个新用户账户时，系统会分配给用户一个用户 ID（UID）和一个主组 ID（GID）。它们可以是相同的连续号码，但并非总是如此。例如，对于之前为简·史密斯创建的账户，UID 是`1007`，GID 是`1007`：

```
jsmith:x:1007:1007:
```

简的主要 GID 是`1007`，但她可能还在公司的 IT、工程或应用开发等领域工作，需要访问一个由组拥有的目录。在这个案例中，简在工程部门担任副工程师。工程部门的共享目录 GID 是`8020`。使用`usermod`，以下是如何授予简访问该组共享目录的方法：

```
# usermod -a -G 8020 jsmith
```

这将简的用户账户添加到*/etc/group*文件中的工程组：

```
engineering:x:8020:bjones,kdoe,vkundra,adundee,jsmith
```

现在简可以访问工程组的共享目录了。为了正确地将简的账户添加到一个新组中，使用`-a`（追加）和`-G`（附加组）选项一起使用。例如，如果你希望简访问财务部门的共享目录，你必须将她追加到该组中。在添加用户时，你可以使用 GID 号码或组名：

```
# usermod -a -G finance jsmith
```

###### 警告

你必须同时使用`-a`（追加）和`-G`（附加组）选项。如果你不使用`-a`选项，你的用户将从所有其他附加组中删除，并且只添加到你指定的一个组中。

用户可以是多个其他组的成员。例如，用户可能在财务部门（GID `8342`）工作，但也需要访问人力资源（GID `8901`）信息。你也可以一次性将用户添加到多个组中：

```
# usermod -a -G 8342,8901 jsmith
```

使用此命令，可以一次性将简·史密斯添加到财务和人力资源组中。

## 更改用户注释字段

更改用户注释（GECOS）字段是一个常见的任务。你可以直接编辑*/etc/passwd*文件，但这样做存在很大的风险。经验法则是，如果有工具可以执行某个操作或任务，你应该使用它而不是直接编辑配置文件。你可以使用`usermod`命令和`-c`选项轻松更改 GECOS 字段。

假设公司最近雇佣了另一位名叫简·史密斯的员工，因此你需要通过在第一位简·史密斯的 GECOS 字段中添加中间名来区分它们：

```
# usermod -c "Jane R Smith" jsmith
```

此命令将`简·史密斯`替换为`简·R·史密斯`。

`-c`选项告诉`usermod`命令，你正在编辑“注释”字段。你也可以使用`chfn`命令更改这些信息：

```
# chfn -f "Janie Smith" jsmith
```

`chfn` 命令更改你的 finger 信息。Finger 是早期 Unix 系统和一些 Linux 系统上运行的一个旧守护程序，提供有关用户的信息。由于安全问题，现在几乎没有人再使用它，但信息仍被称为 finger 信息。 `-f` 选项为指定的账户更改用户的全名字段。还有其他选项用于办公室（`-o`）、办公室电话（`-p`）和家庭电话（`-h`）。通常，只使用用户的全名或服务的名称和目的用于 GECOS 字段。

## 设置账户过期日期

如果用户在公司给出通知、转移到不同的业务单位或休产假，系统管理员可能决定出于安全原因禁用该用户的账户，直到该人返回或在从系统中移除账户之前：

```
# usermod -e 2021-07-23 rsmith
```

罗布·史密斯的账户将在指定的 YYYY-MM-DD 格式日期上失效（过期）。 `-e` 选项设置账户过期。

## 更改用户登录 shell

默认的 Linux shell 是 bash，但一些用户更喜欢使用其他可用的 shell，因此他们请求将其默认 shell 更改为其中一种。有三种方法可以更改用户的默认 shell：`usermod`、`chsh` 和直接编辑 */etc/passwd*。不建议直接编辑 */etc/passwd* 文件。

`usermod` 命令方法使用 `-s` 选项、新 shell 和用户名来进行更改：

```
# usermod -s /bin/sh jsmith
```

更新后的 */etc/passwd* 文件显示如下：

```
jsmith:x:1007:1007:Janie Smith:/home/jsmith:/bin/sh
```

只有具有 root 权限的用户才能编辑 */etc/passwd* 文件或使用 `usermod` 命令。但是，任何用户都可以使用 `chsh` 命令更改其 shell：

```
$ chsh -s /bin/zsh
Changing shell for jsmith.
Password:
Shell changed.
```

结果为 */etc/passwd* 的条目如下：

```
jsmith:x:1007:1007:Janie Smith:/home/jsmith:/bin/zsh
```

对于其他更改，请参阅 `usermod` 的 man 手册。

###### 注意

在任何 man 手册的末尾，你会找到相关的替代命令列表、外部文档链接以及“参见”部分引用的配置文件。这些都很方便探索，并且可能在进行更改时更有效。以下是从 [`usermod` man 手册](https://oreil.ly/nqaH_) 中摘录的“参见”部分：

> 参见
> 
> chfn(1)、chsh(1)、passwd(1)、crypt(3)、gpasswd(8)、groupadd(8)、groupdel(8)、groupmod(8)、login.defs(5)、useradd(8)、userdel(8)。

现在你已经学会了如何创建和修改用户账户，让我们讨论如何移除用户账户。

# 移除用户账户

幸运的是，系统管理员和开发人员命名命令时考虑了使其易于记忆的方式。命令名称通常描述其功能。`useradd` 命令就是一个例子。要从系统中移除用户账户，可以使用 `userdel` 命令，它像 `useradd` 一样易于使用。

要从系统中移除用户账户，使用 `userdel` 命令并提供账户的用户名：

```
# userdel jsmith
```

此命令从*/etc/passwd*和*/etc/shadow*中删除用户条目，但保留用户的主目录（*/home/jsmith*）不变。您认为这是一个好选择的原因是什么？系统管理员通常在用户离开公司或在公司内部更换工作但不再需要访问系统时保留用户的主目录。保留用户的主目录确保只有根用户可以访问用户可能留下的对公司重要的任何文档。

如果您每晚备份用户的主目录，您不一定需要保留用户的主目录。以下`userdel`命令将删除用户的主目录及其中的所有文件：

```
# userdel -r jsmith
```

###### 警告

`userdel`和`rm`等破坏性 Linux 命令是不可逆的，一旦执行就无法撤销。在按下回车键之前，请务必确认您选择了正确的用户账户，并且进行了良好的备份。

当需要更改密码时，您需要知道如何强制用户执行此操作。我们的下一节将向您展示如何操作。

# 强制密码更改

当您为用户提供初始密码时，用户更改其密码是基于信任的问题。定期审核用户密码的系统管理员意识到这种“诚信制度”的信任并非始终有效。您可以使用`chage`命令轻松审核用户账户设置。`-l`选项列出指定用户账户的当前设置：

```
# chage -l rsmith
Last password change                    : Jul 17, 2021
Password expires                        : never
Password inactive                        : never
Account expires                        : never
Minimum number of days between password change    : 0
Maximum number of days between password change    : 99999
Number of days of warning before password expires    : 7
```

如您所见，此账户的密码永不过期，这是一个安全违规，需要修复。除了定期强制更改外，您还应设置最小更改周期。例如，如下代码所示，我将`rsmith`账户设置为每 90 天强制更改密码（`-M 90`），并设置最小更改天数为 1 天（`-m 1`）。设置最小更改天数可确保用户不会多次更改密码（或者无论系统设置的记忆密码数量是多少），以将其重置为原始密码，这实际上没有净密码更改。

```
# chage -m 1 -M 90 rsmith

# chage -l rsmith
Last password change                    : Jul 17, 2021
Password expires                        : Oct 15, 2021
Password inactive                        : never
Account expires                        : never
Minimum number of days between password change    : 1
Maximum number of days between password change    : 90
Number of days of warning before password expires    : 7
```

系统设置的到期日期是最后一次密码更改后的 90 天。如果最后一次密码更改是在过去，用户必须在下次登录时更改其密码。

###### 注意

来自[`chage` man page](https://oreil.ly/-NeGX)：“`chage`命令用于更改密码更改之间的天数和最后一次密码更改日期。系统使用此信息确定用户何时必须更改密码。”

密码是一种弱身份验证形式，因为它们可以被猜测、破解或者如果用户将其写下，可以以纯文本形式读取。因此，您必须确保密码经常更改且不重复使用。

# 处理服务账户

没有什么比服务账户的提及更能激起系统管理员和安全管理员之间的争议了。我不确定所有争议的原因，因为每个 Linux 系统都有超过 30 个服务账户。

服务帐户的一个例子是*nobody*帐户，这是内核溢出用户帐户：

```
nobody:x:65534:65534:Kernel Overflow User:/:/sbin/nologin
```

通常，你可以通过*/etc/passwd*文件来识别服务帐户，看到用户帐户没有分配的 shell。服务帐户有*/sbin/nologin*，用户的 shell 应该在那里。这意味着服务帐户没有交互式 shell 或密码。这并不意味着他们的密码是空白或 null，它们根本不存在。换句话说，如果一个用户帐户的 shell 是*/sbin/nologin*，他们无法使用任何密码登录系统。甚至 root 用户也不能通过`su`或`sudo`切换到这些帐户：

```
# su - nobody
This account is currently not available.
```

因为服务帐户没有交互式 shell，也没有通过`su`或`sudo`切换到交互式 shell 的方法，所以在系统上拥有服务帐户不会导致安全违规。争议来自于一些系统管理员不知道服务帐户通常没有交互式 shell 或密码。某些服务可能需要一个交互式 shell 帐户才能使其服务正常运行。对于这些服务，应该采取极端审查和其他安全措施来阻止潜在的秘密登录。

# 管理组而不是用户

在管理一组用户的权限时，定义和管理一个组比单独管理每个用户更方便。组管理允许你做以下事情：

+   管理资产（如文件夹和文件）的权限

+   根据工作职能管理权限

+   更改大量用户的权限而不是为每个用户单独更改

+   轻松地将用户添加到组的共享文件夹和文件中，并从中删除用户

+   限制对敏感文件夹和文件的权限

对于单个用户管理权限非常困难，因为如果这些权限需要更改，你必须在用户在每个拥有帐户的系统上追踪该用户的权限。管理组的权限允许系统管理员在更全局的层面上管理细粒度的用户访问。

例如，如果你有一个在人力资源（HR）部门工作的用户，然后转移到财务部门，很容易将该用户从 HR 组中移除并添加到财务组中。用户立即可以访问所有其他财务组成员可以访问的共享文件和文件夹。而且用户不再可以访问 HR 文件和文件夹。

在本章的前面，你学习了如何将用户添加到附加组中。你应该练习将目录添加到系统中，添加一个组帐户，将该特定组的所有权设置为该目录，然后将用户添加到该组中。然后你可以`su - *用户名*`，成为该用户以测试你的权限设置。

下面是一个你可以通过学习来了解组管理的可能场景。这个例子假设你已经有一个财务组，并且分配了用户给它。请注意，在这个例子中，root 用户退出并返回到一个属于财务组的普通账户。用户可以改变他们拥有的文件的组所有权和权限：

```
$ su - root
Password:

# mkdir /opt/finance

# chgrp finance /opt/finance

# ls -la /opt
total 0
drwxr-xr-x.  3 root root     21 Aug 11 21:56 .
dr-xr-xr-x. 18 root root    239 Aug 11 21:08 ..
drwxr-xr-x.  2 root finance   6 Aug 11 21:56 finance

# chmod 770 /opt/finance

# ls -la /opt
total 0
drwxr-xr-x.  3 root root     21 Aug 11 21:56 .
dr-xr-xr-x. 18 root root    239 Aug 11 21:08 ..
drwxrwx---.  2 root finance   6 Aug 11 21:56 finance

# exit
logout

$ cd /opt/finance

$ touch budget.txt

$ ls -la
total 0
drwxrwx---. 2 root  finance 24 Aug 11 21:58 .
drwxr-xr-x. 3 root  root    21 Aug 11 21:56 ..
-rw-rw-r--. 1 khess khess    0 Aug 11 21:58 budget.txt

$ chgrp finance budget.txt

$ ls -la
total 0
drwxrwx---. 2 root  finance 24 Aug 11 21:58 .
drwxr-xr-x. 3 root  root    21 Aug 11 21:56 ..
-rw-rw-r--. 1 khess finance  0 Aug 11 21:58 budget.txt

$ chmod 660 budget.txt

$ ls -la
total 0
drwxrwx---. 2 root  finance 24 Aug 11 21:58 .
drwxr-xr-x. 3 root  root    21 Aug 11 21:56 ..
-rw-rw----. 1 khess finance  0 Aug 11 21:58 budget.txt
```

如果你理解这个例子中发生的一切，那么你已经准备好进入下一章了。如果你还没有掌握这些概念，请再次通过章节示例进行学习，然后返回到这个练习。请记住，创建用户、组和目录以及更改权限是系统管理员的日常任务，练习这些技能是掌握它们并且熟练使用它们的唯一途径。

# 总结

本章教会了你如何创建、删除和修改用户账户。你还学习了如何设置服务账户，并简要介绍了管理组的概述。在下一章中，你将学习关于 Linux 网络的知识，从网络的重要性基础到网络故障排除等更复杂的概念。
