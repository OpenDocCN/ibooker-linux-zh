# 第三章：定制用户体验

本章讨论了为自己和您的用户定制用户体验。系统管理员经常需要对用户环境进行微小更改或对系统上所有用户的默认环境进行全局更改（后者称为全局更改）。只要任何请求的更改和增强不会损害系统安全或违反公司政策，那么进行符合用户需求和工作流的更改就没有坏处。作为系统管理员，我们的职责首先是对公司负责（然后才是对用户负责）。用户就是您的客户。

全局定制默认用户环境会更改系统上所有用户的环境。但是，您或用户可以覆盖某些全局参数。在第二章中，当您为用户账户添加新的 `umask` 时，您就做了这样的覆盖。通过在设置全局 umask 之后设置个人 umask 偏好，您超越了系统设置的 umask。用户定制他们可以控制的环境是一个常见的做法。

本章将通过编辑每个用户家目录中的关键文件，讨论如何定制您和您的用户的环境。作为系统管理员，您还将探索这些环境文件的“全局”版本，这些文件可以进行更改或添加，从而为您的用户创建特定的体验。

# 更改家目录选项

在每个用户的家目录中，几个隐藏文件控制着大部分用户的环境。由于许多 Linux 用户使用 bash，因此本章节将重点讨论默认和自定义用户环境的相关内容。（其他如 ash、zsh、csh 和 ksh 等 shell 也可供选择，它们的隐藏、可编辑文件在名称、功能和结构上相似。）

由于一些用户没有足够的技能来进行必要的更改，您可能需要代表用户进行更改。相关文件如下：

+   *.bashrc*

+   *.bash_logout*

根据您的 Linux 发行版和之前的配置更改，您的家目录中还可能会看到名为 *.profile*、*.bash_profile*、*.bash_login* 和 *.bash_history* 的文件。

您不必对它们都进行更改。例如，*.bash_history* 文件不需要任何更改。它记录了发出的命令，没有用户可配置的项目。当您登录 Linux 系统时，*.bashrc* 文件首先执行，然后执行 *.bash_profile*。*.bash_logout* 文件在注销时执行。

###### 警告

在您的启动文件中放置的程序、脚本和消息要小心，因为如果它们损坏、损坏或部分打开，您可能会发现您的登录要么延迟，要么您完全无法登录，需要其他系统管理员进行救援。也把这个警告传递给您的用户。

当您交互登录到 Linux 系统时，一组文件会自动执行，以构建您的用户环境，如以下部分所述。

## 登录与非登录 Shell

您会听到和阅读关于两种类型的交互式 shell：登录和非登录。交互式登录 shell 是您通过 SSH 登录或直接输入用户名和密码或 SSH 密钥登录的 shell。交互式非登录 shell 是您从命令行调用子 shell 的 shell：

```
$ bash
$ echo $SHLVL
2
```

`$SHLVL` 是一个跟踪您的 shell 级别的变量。当您首次使用用户名和密码或密钥交互式登录 shell 时，您的`$SHLVL` 为`1`。调用第一个之后的子 shell 会增加`SHLVL` 变量。以这种方式调用的子 shell 是交互式的非登录 shell，因为它是交互式的，但不涉及新的登录。

## /etc/bashrc

当您交互登录到 Linux 系统时，*/etc/bashrc* 是第一个执行的个性化文件。*/etc/bashrc* 文件还会在交互非登录 shell 上执行。*/etc/bashrc* 文件是一个全局的个性化文件，为您的登录 shell（bash）提供函数和别名。这个文件应该保持不变——即使作为系统管理员，您也不应该编辑它。

此处显示的信息和警告重新列印自*/etc/bashrc* 文件。如果您尝试编辑文件，您会看到它们：

```
# System-wide functions and aliases
# Environment stuff goes in /etc/profile

# It's NOT a good idea to change this file unless you know what you
# are doing. It's much better to create a custom.sh shell script in
# /etc/profile.d/ to make custom changes to your environment, as this
# will prevent the need for merging in future updates.
```

*/etc/bashrc* 文件类似于您家目录中的*.bashrc* 文件。如果需要更改函数和别名，请在那里更改。

## /etc/profile

*/etc/profile* 文件是一个系统范围的启动文件。它为所有用户提供通用的变量、路径和其他设置。该文件中的警告如下代码清单所示，指出除非您知道自己在做什么，否则不应编辑此文件：

```
# System-wide environment and startup programs for login setup
# Functions and aliases go in /etc/bashrc

# It's NOT a good idea to change this file unless you know what you
# are doing. It's much better to create a custom.sh shell script in
# /etc/profile.d/ to make custom changes to your environment, as this
# will prevent the need for merging in future updates.
```

正如警告所述，最好编辑用户家目录中的个性化文件，或在*/etc/profile.d* 下创建其他全局个性化文件。

然而，这些全局设置可以被位于家目录中的个人化启动文件覆盖。这是您登录 Linux 系统时执行的第二个环境个性化文件。奇怪的是，它也调用*/etc/bashrc* 文件，因此*/etc/bashrc* 被执行两次。

*/etc/profile* 文件类似于您家目录中的*.bash_profile* 文件。在那里更改任何环境设置。

## .bashrc

`.bashrc`文件是您家目录中的一个隐藏文件。它是隐藏的，因为在使用通配符重命名或删除文件时，您不希望直接访问它。您可以完全拥有该文件，并可以随意编辑它。该文件用于设置和包含您可能需要的任何函数，并为命令设置别名。全局个性化文件执行后，`.bashrc`文件将执行。`.bashrc`文件与其全局类似，在交互式非登录 shell 中执行。以下是未更改的`.bashrc`文件的列表。您应该在此文件中增加或更改您的`PATH`，以便您的 shell 在登录和非登录实例中表现类似：

```
# .bashrc

# Source global definitions
if [ -f /etc/bashrc ]; then
    . /etc/bashrc
fi

# User specific environment
if ! [[ "$PATH" =~ "$HOME/.local/bin:$HOME/bin:" ]]
then
    PATH="$HOME/.local/bin:$HOME/bin:$PATH"
fi
export PATH

# Uncomment the following line if you don't like systemctl's auto-paging feature:
# export SYSTEMD_PAGER=

# User specific aliases and functions
```

别名是对命令及其选项的有用快捷方式。例如，如果您希望常用命令的“啰嗦”版本，在您的`.bashrc`文件中创建以下别名：

```
alias rm='rm -i'
alias cp='cp -i'
alias mv='mv -i'
```

`-i` 选项意味着*交互式*，并要求您在执行命令之前进行确认。当您发出新的别名`rm`命令时，您将收到一个验证消息。这个选项特别适用于破坏性命令，比如`rm`，因为 Linux 并不“啰嗦”。例如，当您删除文件时，它不提供任何反馈。`-i` 选项通过提示您确认操作，提供了更类似于 Windows 命令的体验：

```
$ rm file4.txt
rm: remove regular empty file 'file4.txt'?
```

如果您经常运行相同的命令和选项，别名非常方便。例如，我在使用的每个系统上都创建了几个默认别名。我最有用的一个是用于长列表：

```
alias ll='ls -l'
```

您还可以按会话创建别名。这意味着您可以简单地在命令行上发出别名命令，而无需将其保存到文件中，并且它保持有效，直到您注销或终止当前 shell。

###### 提示

许多来源建议将所有定制内容放在`.bashrc`文件中，因为它适用于交互式登录和非登录 shell。在`.bashrc`文件中进行 shell 定制可以保证 shell 的一致性。

## `.bash_profile`

在交互式登录上执行的最后一个文件是您的家目录中的`.bash_profile`文件：

```
# .bash_profile

# Get the aliases and functions
if [ -f ~/.bashrc ]; then
    . ~/.bashrc
fi

# User specific environment and startup programs
```

你可以在此文件中放置定制内容，这些内容将在交互式登录 shell 中运行，但在交互式非登录 shell 中不可用。

## `.bash_logout`

在注销时执行的最后一个文件是`.bash_logout`文件。此文件是可选的。它仅存在以允许用户在退出时清理临时文件。您还可以使用它来使用 shell 记录时间或在注销时发送消息。

在下一节中，您将了解这些 shell 个性化文件的起源，以及如何为每个用户更改默认设置。

# `/etc/skel`目录

*/etc/skel* 目录是一个特殊目录，用于保存您希望在创建用户账户时每个用户都收到的文件。您在 */etc/skel* 中创建的文件不必是隐藏文件，尽管默认是这样。这些文件在创建账户时会被复制到用户的主目录中。这些是环境个性化文件的全局副本。Red Hat Enterprise Linux 系统上的 */etc/skel* 默认文件列表如下所示：

```
# ls -la /etc/skel
total 28
drwxr-xr-x.   2 root root   76 Jul  4 12:45 .
drwxr-xr-x. 144 root root 8192 Jul  4 11:07 ..
-rw-r--r--.   1 root root   18 Apr 21 10:04 .bash_logout
-rw-r--r--.   1 root root  141 Apr 21 10:04 .bash_profile
-rw-r--r--.   1 root root  376 Apr 21 10:04 .bashrc
-rw-r--r--.   1 root root  658 Mar  3  2020 .zshrc
```

如果您在 */etc/skel* 目录中创建文件，则在创建账户时，这些文件将被复制到新用户的主目录中。已存在的用户在您创建他们的账户后，将不会接收到放置在 */etc/skel* 中的文件。您需要手动将这些文件复制到每个用户的主目录，并更改权限，以便用户完全控制它们。

###### 注意

Linux 有一个称为手册页或简称 *man 页* 的本地帮助系统。如果您需要有关命令、配置或系统设置的帮助，请输入 `man` 和关键字以查看是否存在相关文档。例如，要获取 `ls` 命令的帮助，您可以使用 `$ man ls`。您可以使用 vi（Vim）导航命令浏览 man 页。

# 自定义 Shell 提示符

通常，用户接受系统呈现的默认提示符。通常，它看起来像 `[username@hostname pwd]$`，或者在我的具体情况下，像 `[khess​@server1 ~]⁠$`。波浪号 (`~`) 表示用户的主目录。例如，在前面讨论个性化脚本的部分中，它们通常表示为 `~/.bashrc` 和 `~/.bash_profile`，以说明这些文件位于用户的主目录中。

要设置自定义提示符，Shell 提供了一组转义字符，表示位置、用户名、时间、回车等。例如，提示符环境变量 (`PS1`) 的默认提示符由以下代码给出：`PS1="[\u@\h \W]\\$ "`。系统在 */etc/bashrc* 文件中设置此默认提示符。您可以在您的 *~/.bashrc* 文件中覆盖它。

Bash 允许通过插入多个反斜杠转义的特殊字符来定制这些提示字符串，如 表 3-1 所示解码。

表 3-1\. 反斜杠转义的特殊字符

| 特殊字符 | 描述 |
| --- | --- |
| `\a` | ASCII 响铃字符（07） |
| `\d` | “星期 月份 日期” 格式的日期（例如，“周二 5 月 26 日”） |
| `\D{format}` | 格式将传递给 `strftime(3)`，并将结果插入到提示字符串中；空格式将得到本地化的时间表示。大括号是必需的。 |
| `\e` | ASCII 转义字符（033） |
| `\h` | 直到第一个 `'.'` 的主机名 |
| `\H` | 主机名 |
| `\j` | 当前由 Shell 管理的作业数 |
| `\l` | Shell 终端设备名称的基本名称 |
| `\n` | 换行符 |
| `\r` | 回车符 |
| `\s` | Shell 的名称，即`$0`的基本名称（最后斜杠后的部分） |
| `\t` | 当前时间，24 小时制的 HH:MM:SS 格式 |
| `\T` | 当前时间，12 小时制的 HH:MM:SS 格式 |
| `\@` | 当前时间，12 小时制的上午/下午格式 |
| `\A` | 当前时间，24 小时制的 HH:MM 格式 |
| `\u` | 当前用户的用户名 |
| `\v` | Bash 的版本（例如，2.00） |
| `\V` | Bash 的发布版本，版本+补丁级别（例如，2.00.0） |
| `\w` | 当前工作目录，以`$HOME`用波浪号缩写（使用`PROMPT_DIRTRIM`变量的值） |
| `\W` | 当前工作目录的基本名称，以`$HOME`用波浪号缩写 |
| `\!` | 此命令的历史编号 |
| `\#` | 此命令的命令编号 |
| `\$` | 如果有效 UID 为`0`，则为`#`，否则为`$` |
| `\nnn` | 对应于八进制数字`nnn`的字符 |
| `\\` | 一个反斜杠 |
| `\[` | 开始一系列非打印字符，可用于将终端控制序列嵌入提示符 |
| `\]` | 结束一系列非打印字符 |

默认提示符对大多数用户来说足够了，但一些用户和系统管理员可能希望有所不同，因此他们可以自由更改它。一些聪明的用户已经设计出了彩色和艺术风格的提示符代码，例如这个圣诞主题的提示符：

```
PS1="\[\e[33;41m\][\[\e[m\]\[\e[32m\]\u\[\e[m\]\[\e[36m\]@\[\e[m\] \
\[\e[34m\]\h\[\e[m\]\[\e[33;41m\]]\[\e[m\] "

```

在网上搜索“有趣的 Linux 提示符”，尽情享受。玩耍后，您可以注销并重新登录以重置您的提示符，或输入`PS1="[\u@\h \W]\\$ "`以恢复默认设置。

# 摘要

在本章中，您学习了如何通过个性化文件编辑用户环境，并了解了添加更多文件到新帐户的默认位置和配置。您还学习了这些文件的位置、加载顺序以及应该为特定效果编辑哪些文件。最后，您对 shell 提示符有了简要的概述以及如何修改它。

在第四章中，您将获得有关用户管理的概述，从用户帐户创建到通过组管理用户，再到如何授予资源访问权限。
