# 附录 A. 有用的配方

在本附录中，我汇编了一些常见任务的配方列表。这只是我随时间收集的一些配方的选择，这些任务我经常执行，并且喜欢作为参考保留。这些配方并不是对 Linux 使用和管理员任务的全面或深入覆盖。对于全面的配方集合，我强烈推荐您查看 Carla Schroder 的[*Linux Cookbook*](https://oreil.ly/53Pk9)（O'Reilly），详细介绍了各种配方。

# 收集系统信息

要了解 Linux 版本、内核和其他相关信息，请使用以下任一命令：

```
cat /etc/*-release
cat /proc/version
uname -a
```

要了解基本硬件设备（CPU、RAM、磁盘），请执行以下操作：

```
cat /proc/cpuinfo
cat /proc/meminfo
cat /proc/diskstats
```

要了解系统的硬件详细信息，如 BIOS 等，请使用：

```
sudo dmidecode -t bios
```

关于前一个命令的说明：`-t`的其他有趣选项包括`system`和`memory`。

查询整体主存储器和交换使用情况，执行以下命令：

```
free -ht
```

要查询进程可以拥有多少文件描述符，请使用：

```
ulimit -n
```

# 处理用户和进程

您可以使用`who`或`w`列出已登录用户（更详细的输出）。

要显示特定用户`SOMEUSER`的每个进程的系统指标（CPU、内存等），请使用以下命令：

```
top -U SOMEUSER
```

使用以下命令以树形式详细列出所有用户的所有进程：

```
ps faux
```

查找特定进程（此处为`python`）：

```
ps -e | grep python
```

要终止进程，如果知道其 PID，请使用它（如果进程忽略此信号，请添加`-9`作为参数）：

```
kill *`PID`*
```

或者，您可以使用`killall`按名称终止进程。

# 收集文件信息

要查询文件详细信息（包括文件系统信息如 inode），请使用：

```
stat *`somefile`*
```

要了解命令的工作方式、shell 如何解释它以及可执行文件的位置，请使用：

```
type *`somecommand`*
which *`somebinary`*
```

# 处理文件和目录

要显示名为`afile`的文本文件的内容：

```
cat afile
```

要列出目录的内容，请使用`ls`，并可能进一步使用输出。例如，要计算目录中文件的数量，请使用：

```
ls -l /etc |  wc -l
```

查找文件和文件内容：

```
find /etc -name "*.conf" ![1](img/1.png)
find . -type f -exec grep -H FINDME {} \; ![2](img/2.png)
```

![1](img/#co_helpful_recipes_CO1-1)

在目录*/etc*中查找以*.conf*结尾的文件。

![2](img/#co_helpful_recipes_CO1-2)

通过执行`grep`在当前目录中查找“FINDME”。

要显示文件差异，请使用：

```
diff -u *`somefile`* *`anotherfile`*
```

要替换字符，请使用`tr`如下所示：

```
echo 'Com_Acme_Library' | tr '_A-Z' '.a-z'
```

替换字符串的另一种方法是使用`sed`（请注意分隔符不一定是`/`，这对于在路径或 URL 中替换内容的情况非常方便）：

```
cat 'foo bar baz' | sed -e 's/foo/quux/'
```

要创建指定大小的文件（用于测试），可以使用`dd`命令，如下所示：

```
dd if=/dev/zero of=output.dat bs=1024 count=1000 ![1](img/1.png)
```

![1](img/#co_helpful_recipes_CO2-1)

创建名为*output.dat*的 1 MB 文件（由 1000 个 1 KB 块组成），其中填充了零。

# 使用重定向和管道

在“流”中，我们讨论了文件描述符和流。以下是围绕这个主题的几个配方。

文件 I/O 重定向：

```
*command* 1> *file* ![1](img/1.png)
*command* 2> *file* ![2](img/2.png)
*command* &> *file* ![3](img/3.png)
*command* >*file* 2>&1 ![4](img/4.png)
*command* > /dev/null ![5](img/5.png)
*command* < *file* ![6](img/6.png)
```

![1](img/#custom_co_helpful_recipes_CO3-1)

将`*command*`的`stdout`重定向到`*file*`。

![2](img/#custom_co_helpful_recipes_CO3-2)

将 `*command*` 的 `stderr` 重定向到 `*file*`。

![3](img/#custom_co_helpful_recipes_CO3-3)

将 `*command*` 的 `stdout` 和 `stderr` 都重定向到 `*file*`。

![4](img/#custom_co_helpful_recipes_CO3-4)

将 `*command*` 的 `stdout` 和 `stderr` 重定向到 `*file*` 的替代方法。

![5](img/#custom_co_helpful_recipes_CO3-5)

丢弃 `*command*` 的输出（通过将其重定向到 */dev/null*）。

![6](img/#custom_co_helpful_recipes_CO3-6)

将 `*file*` 的输入重定向到 `*command*` 的 `stdin`。

要将一个进程的 `stdout` 连接到另一个进程的 `stdin`，请使用管道 (`|`)：

```
*`cmd1`* | *`cmd2`* | *`cmd3`*
```

要显示管道中每个命令的退出代码：

```
echo ${PIPESTATUS[@]}
```

# 处理时间和日期

要查询与时间相关的信息，例如本地时间、UTC 时间以及同步状态，请使用：

```
timedatectl status
```

处理日期时，通常需要获取当前时间的日期或时间戳，或将现有时间戳从一种格式转换为另一种格式。

要以 `YYYY-MM-DD` 格式获取日期（例如 `2021-10-09`），请使用以下命令：

```
date +"%Y-%m-%d"
```

要生成 Unix 时间戳（例如 `1633787676`），请执行：

```
date +%s
```

要为 UTC 创建 ISO 8601 时间戳（例如 `2021-10-09T13:55:47Z`），可以使用以下方法：

```
date -u +"%Y-%m-%dT%H:%M:%SZ"
```

同样的 ISO 8601 时间戳格式，但用于本地时间：

```
date +%FT%TZ
```

# 使用 Git

要克隆一个 Git 仓库（即在您的 Linux 系统上创建一个本地副本），使用以下命令：

```
git clone https://github.com/*`exampleorg`*/*`examplerepo`*.git
```

在上述 `git clone` 命令完成后，Git 仓库将位于 *examplerepo* 目录中，您应该在此目录中执行接下来的所有命令。

要以颜色显示本地更改并显示已添加和已删除行，请使用：

```
git diff --color-moved
```

要查看本地发生的更改（编辑的文件、新文件、已删除的文件），请执行：

```
git status
```

添加所有本地更改并提交它们：

```
git add --all && git commit -m "adds a super cool feature"
```

要找出当前提交的提交 ID，请使用：

```
git rev-parse HEAD
```

要用标签 `ATAG` 标记 ID 为 `HASH` 的提交，请执行：

```
git tag ATAG HASH
```

将本地更改推送到远程（上游）仓库，并附上标签 `ATAG`：

```
git push origin ATAG
```

要查看提交历史记录，请使用 `git log`；具体来说，要获取摘要，请执行：

```
git log (git describe --tags --abbrev=0)..HEAD --oneline
```

# 系统性能

有时您需要查看设备的速度或者 Linux 系统在负载下的性能。以下是生成系统负载的一些方法。

模拟内存负载（同时消耗一些 CPU 循环）使用以下命令：

```
yes | tr \\n x | head -c 450m | grep z
```

在上述管道中，`yes` 生成无限数量的 `y` 字符，每个字符位于自己的一行上，然后 `tr` 命令将其转换为一个连续的 `yx` 流，`head` 命令将其截断为大约 450 百万字节（约 450 MB）。最后，我们让 `grep` 消耗结果的 `yx` 块以寻找不存在的内容（`z`），因此我们看不到输出，但它仍在生成负载。

更详细的目录磁盘使用情况：

```
du -h /home
```

列出空闲磁盘空间（全局，本例中）：

```
df -h
```

使用以下命令对磁盘进行负载测试并测量 I/O 吞吐量：

```
dd if=/dev/zero of=/home/some/file bs=1G count=1 oflag=direct
```
