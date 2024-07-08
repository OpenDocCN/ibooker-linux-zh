# 第七章：11 种更多运行命令的方式

现在你的工具箱中有很多命令，并且对 shell 有了深入的理解，是时候学习……如何运行命令了。等一下，你自本书开始就一直在运行命令吧？是的，但只有两种方式。第一种是普通执行简单命令：

```
$ grep Nutshell animals.txt
```

第二种是简单命令的管道，如第一章中所述：

```
$ cut -f1 grades | sort | uniq -c | sort -nr
```

在本章中，我将展示 11 种更多运行命令的方式以及为什么你应该学习它们。每种技术都有其利弊，你掌握的技术越多，与 Linux 的互动就越灵活、高效。我现在将专注于每种技术的基础知识；在接下来的两章中，你将看到更复杂的例子。

# 技术清单

一个列表是单个命令行上的命令序列。你已经见过一种列表类型——管道，但 shell 支持其他具有不同行为的列表：

条件列表

每个命令依赖于前一个命令的成功或失败。

无条件列表

命令只是一个接着一个运行。

## 技术 #1：条件列表

假设你想在目录 *dir* 中创建一个文件 *new.txt*。一个典型的命令序列可能是：

```
$ cd dir            *Enter the directory*
$ touch new.txt     *Make the file*
```

注意第二个命令依赖于第一个命令的成功。如果目录 *dir* 不存在，则运行 `touch` 命令没有意义。Shell 允许你明确地表达这种依赖关系。如果在单行上的两个命令之间放置 `&&` 运算符（读作“and”）：

```
$ cd dir && touch new.txt
```

接着第二个命令（`touch`）只有在第一个命令（`cd`）成功后才会运行。上面的例子是两个命令的*条件列表*。（要了解命令“成功”的含义，请参见“退出代码表示成功或失败”。）

很可能，你每天都在运行依赖于之前命令的命令。例如，你曾经为了备份文件而修改原文件并在完成后删除备份吗？

```
$ cp myfile.txt myfile.safe      *Make a backup copy*
$ nano myfile.txt                *Change the original*
$ rm myfile.safe                 *Delete the backup*
```

每个命令仅在前一个命令成功时才有意义。因此，这个序列适合作为条件列表：

```
$ cp myfile.txt myfile.safe && nano myfile.txt && rm myfile.safe
```

再举个例子，如果你使用版本控制系统 Git 来维护文件，你可能熟悉在修改文件后的以下命令序列：运行 `git add` 准备提交的文件，然后 `git commit`，最后 `git push` 来分享你的提交变更。如果其中任何一个命令失败，你将不会运行余下的命令（直到修复失败的原因）。因此，这三个命令很好地作为一个条件列表：

```
$ git add . && git commit -m"fixed a bug" && git push
```

就像 `&&` 运算符仅在第一个命令成功时运行第二个命令一样，相关的运算符 `||`（读作“or”）仅在第一个命令失败时运行第二个命令。例如，以下命令尝试进入 *dir*，如果失败，则创建 *dir*：^(1)

```
$ cd dir || mkdir dir
```

你会经常在脚本中看到`||`运算符，如果发生错误，脚本会退出：

```
# If a directory can't be entered, exit with an error code of 1
cd dir || exit 1
```

结合`&&`和`||`运算符来设置更复杂的成功和失败操作。以下命令尝试进入目录*dir*，如果失败，则创建该目录并进入。如果全部失败，则打印失败消息：

```
$ cd dir || mkdir dir && cd dir || echo "I failed"
```

条件列表中的命令不必是简单命令；它们也可以是管道和其他组合命令。

## 技巧 #2：无条件列表

列表中的命令不必彼此依赖。如果用分号分隔命令，它们只是顺序运行。命令的成功或失败不会影响后续的命令。

我喜欢无条件列表，在我下班后启动临时命令。这是一个睡眠（什么都不做）两个小时（7,200 秒），然后备份我的重要文件的例子：

```
$ sleep 7200; cp -a ~/important-files /mnt/backup_drive
```

这是一个类似的命令，作为一个简单的提醒系统，睡眠五分钟，然后发送给我一封电子邮件：^(3)

```
$ sleep 300; echo "remember to walk the dog" | mail -s reminder $USER
```

无条件列表是一个便利特性：它们产生与逐个输入命令并按 Enter 键相同的结果（大多数情况下）。唯一显著的区别与退出代码有关。在无条件列表中，单个命令的退出代码被丢弃，除了最后一个。只有列表中最后一个运行的命令的退出代码被分配给 shell 变量`?`：

```
$ mv file1 file2; mv file2 file3; mv file3 file4
$ echo $?
0                           *The exit code for "mv file3 file4"*
```

# 替换技术

*替换*意味着自动用其他文本替换命令的文本。我将向你展示两种具有强大可能性的类型：

命令替换

一个命令被其输出替换。

进程替换

一个命令被一个文件（某种程度上）替换。

## 技巧 #3：命令替换

假设你有几千个文本文件，代表歌曲。每个文件包括歌曲标题，艺术家名，专辑标题和歌词：

```
Title: Carry On Wayward Son
Artist: Kansas
Album: Leftoverture

Carry on my wayward son
There'll be peace when you are done
⋮
```

你想按艺术家将文件组织到子目录中。为了手动执行此任务，你可以使用`grep`搜索所有 Kansas 的歌曲文件：

```
$ grep -l "Artist: Kansas" *.txt
carry_on_wayward_son.txt
dust_in_the_wind.txt
belexes.txt
```

然后将每个文件移动到一个目录*kansas*中：

```
$ mkdir kansas
$ mv carry_on_wayward_son.txt kansas
$ mv dust_in_the_wind.txt kansas
$ mv belexes.txt kansas
```

很乏味，对吧？如果你能告诉 Shell：“移动所有包含字符串*Artist: Kansas*的文件到目录*kansas*”，那不是很棒吗？用 Linux 术语来说，你想从前面的`grep -l`命令中获取名称列表，并将其交给`mv`。好吧，你可以通过一个称为*命令替换*的 Shell 特性轻松地实现这一点：

```
$ mv $(grep -l "Artist: Kansas" *.txt) kansas
```

语法：

```
$(*any command here*)
```

执行括号内的命令，并用其输出替换命令。因此，在前述命令行上，`grep -l`命令被打印的文件名列表替换，就好像你已经像这样输入文件名一样：

```
$ mv carry_on_wayward_son.txt dust_in_the_wind.txt belexes.txt kansas
```

每当你发现自己在将一个命令的输出复制到后续命令行中时，通常可以通过命令替换节省时间。甚至可以在命令替换中包含别名，因为它的内容在一个子 shell 中运行，其中包括其父 shell 的别名的副本。

# 特殊字符和命令替换

带有 `grep -l` 的前面的示例对大多数 Linux 文件名效果很好，但对包含空格或其他特殊字符的文件名则不适用。在输出交给 `mv` 之前，shell 会评估这些字符，可能会产生意外的结果。例如，如果 `grep -l` 打印了 *dust in the wind.txt*，shell 会将空格视为分隔符，`mv` 将尝试移动四个名为 *dust*、*in*、*the* 和 *wind.txt* 的不存在文件。

这里是另一个例子。假设你有几年的银行对账单以 PDF 格式下载。下载的文件名包含对账单的年份、月份和日期，例如 *eStmt_2021-08-26.pdf* 表示 2021 年 8 月 26 日的对账单。^(4) 你想在当前目录中查看最近的对账单。你可以手动完成：列出目录，找到最新日期的文件（这将是列表中的最后一个文件），并使用 Linux PDF 查看器如 `okular` 显示它。但为什么要做所有这些手动工作呢？让命令替换为你省时。创建一个命令来打印目录中最新的 PDF 文件名：

```
$ ls eStmt*pdf | tail -n1
```

并使用命令替换将其提供给 `okular`：

```
$ okular $(ls eStmt*pdf | tail -n1)
```

`ls` 命令列出所有的声明文件，`tail` 仅打印最后一个，例如 *eStmt_2021-08-26.pdf*。命令替换将这个单个文件名直接放在命令行上，就像你输入了 `okular eStmt_2021-08-26.pdf`。

###### 注意

命令替换的原始语法是反引号（backticks）。以下两个命令是等效的：

```
$ echo Today is $(date +%A).
Today is Saturday.
$ echo Today is `date +%A`.
Today is Saturday.
```

大多数 shell 支持反引号（backticks）。`$()` 语法更容易嵌套，然而：

```
$ echo $(date +%A) | tr a-z A-Z                    *Single*
SATURDAY
echo Today is $(echo $(date +%A) | tr a-z A-Z)!   *Nested*
Today is SATURDAY!
```

在脚本中，命令替换的常见用途是将命令的输出存储在变量中：

```
*VariableName*=$(*some command here*)
```

例如，要获取包含 Kansas 歌曲的文件名并将它们存储在一个变量中，可以像这样使用命令替换：

```
$ kansasFiles=$(grep -l "Artist: Kansas" *.txt) 
```

输出可能有多行，因此为了保留任何换行符，请确保在使用该值时引用它：

```
$ echo "$kansasFiles"
```

## 技巧 #4：进程替换

正如你刚刚看到的，命令替换将一个命令的输出直接替换成一个字符串。*进程替换* 也替换一个命令的输出，但它将输出视为存储在文件中。这个强大的区别一开始可能看起来令人困惑，所以我会一步步解释。

假设你在一个包含 JPEG 图像文件的目录中，文件名从 *1.jpg* 到 *1000.jpg*，但某些文件神秘地丢失了，你想要识别它们。使用以下命令生成这样一个目录：

```
$ mkdir /tmp/jpegs && cd /tmp/jpegs
$ touch {1..1000}.jpg
$ rm 4.jpg 981.jpg
```

一种找到丢失文件的较差方法是列出目录并按数字排序，然后用肉眼查找间隙：

```
$ ls -1 | sort -n | less
1.jpg
2.jpg
3.jpg
5.jpg            *4.jpg is missing*
⋮
```

更健壮、自动化的解决方案是将现有文件名与从*1.jpg*到*1000.jpg*的完整列表进行比较，使用`diff`命令。实现此解决方案的一种方法是使用临时文件。将现有的按排序后的文件名存储在一个临时文件*original-list*中：

```
$ ls *.jpg | sort -n > /tmp/original-list
```

然后使用`seq`生成整数 1 到 1000，并将“.jpg”附加到每一行，将完整的文件名列表从*1.jpg*到*1000.jpg*打印到另一个临时文件*full-list*：

```
$ seq 1 1000 | sed 's/$/.jpg/' > /tmp/full-list
```

使用`diff`命令比较两个临时文件，发现*4.jpg*和*981.jpg*丢失，然后删除这些临时文件：

```
$ diff /tmp/original-list /tmp/full-list
3a4
> 4.jpg
979a981
> 981.jpg
$ rm /tmp/original-list /tmp/full-list       *Clean up afterwards*
```

这是很多步骤。直接比较两个文件名列表并且不再使用临时文件，这岂不是一件了不起的事情吗？挑战在于`diff`无法比较来自标准输入的两个列表；它需要文件作为参数。^5 进程替换解决了这个问题。它使得`diff`将这两个列表都看作是文件。（侧边栏 “进程替换的工作原理” 提供了技术细节。）语法：

```
<(*any command here*)
```

在子 shell 中运行命令并将其输出呈现为文件的形式。例如，以下表达式表示`ls -1 | sort -n`的输出，就像它被包含在一个文件中一样：

```
<(ls -1 | sort -n)
```

您可以使用`cat`命令来查看文件：

```
$ cat <(ls -1 | sort -n)
1.jpg
2.jpg
⋮
```

您可以使用`cp`命令复制文件：

```
$ cp <(ls -1 | sort -n) /tmp/listing
$ cat /tmp/listing
1.jpg
2.jpg
⋮
```

如您现在所看到的，您可以将文件与另一个文件进行`diff`比较。从生成您的两个临时文件的两个命令开始：

```
ls *.jpg | sort -n
seq 1 1000 | sed 's/$/.jpg/'
```

应用进程替换，使得`diff`能够将它们视为文件，并获得与之前相同的输出，但不使用临时文件：

```
$ diff <(ls *.jpg | sort -n) <(seq 1 1000 | sed 's/$/.jpg/')
3a4
> 4.jpg
979a981
> 981.jpg
```

通过使用`grep`查找以`>`开头的行并使用`cut`去掉前两个字符来清理输出，您就得到了丢失文件的报告：

```
$ diff <(ls *.jpg | sort -n) <(seq 1 1000 | sed 's/$/.jpg/') \
    | grep '>' \
    | cut -c3-
4.jpg
981.jpg
```

进程替换改变了我使用命令行的方式。只从磁盘文件读取的命令突然可以从标准输入读取。通过实践，以前看似不可能的命令变得很容易。

# 命令作为字符串的技术

每个命令都是一个字符串，但有些命令比其他命令更“字符串化”。我将向您展示几种逐步构建字符串并作为命令运行的技术：

+   将命令作为参数传递给`bash`

+   将命令通过 stdin 管道传递给`bash`

+   使用`ssh`将命令发送到另一台主机

+   使用`xargs`运行一系列命令

###### 警告

以下技术可能存在风险，因为它们将看不见的文本发送到 shell 以执行。绝对不要盲目执行这些命令。在执行之前，一定要理解文本（并信任其来源）。您不希望因错误执行字符串`"rm -rf $HOME"`而删除所有文件。

## 技术 #5：将命令作为`bash`的参数传递

`bash` 是一个像其他任何普通命令一样的命令，正如在“Shell 可执行文件” 中解释的那样，因此您可以在命令行上按名称运行它。默认情况下，运行 `bash` 会启动一个交互式 shell，用于输入和执行命令，就像您看到的那样。或者，您可以通过 `-c` 选项将命令作为字符串传递给 `bash`，`bash` 将运行该字符串作为命令并退出：

```
$ bash -c "ls -l"
-rw-r--r-- 1 smith smith 325 Jul  3 17:44 animals.txt
```

这为什么有用？因为新的 `bash` 进程是具有自己环境的子进程，包括当前目录、具有值的变量等等。对子 shell 的任何更改都不会影响您当前运行的 shell。以下是一个 `bash -c` 命令，它将目录更改为 */tmp*，只足够删除一个文件，然后退出：

```
$ pwd
/home/smith
$ touch /tmp/badfile                           *Create a temporary file*
$ bash -c "cd /tmp && rm badfile"
$ pwd
/home/smith                                    *Current directory is unchanged*
```

`bash -c` 最具教育性和美丽的用法之一是在您以超级用户身份运行某些命令时产生的。具体来说，`sudo` 和输入/输出重定向的组合会产生一个有趣（有时是疯狂的）的情况，其中 `bash -c` 是成功的关键。

假设您想在系统目录 */var/log* 中创建一个日志文件，这是普通用户无法写入的。您运行以下 `sudo` 命令以获得超级用户权限并创建日志文件，但它神秘地失败了：

```
$ sudo echo "New log file" > /var/log/custom.log
bash: /var/log/custom.log: Permission denied
```

等一下 — `sudo` 应该允许您在任何地方创建任何文件。这个命令为什么会失败呢？为什么 `sudo` 甚至没有提示您输入密码？答案是：因为 `sudo` 没有运行。您将 `sudo` 应用于 `echo` 命令，但没有应用于首先运行并失败的输出重定向。详细来看：

1.  您按下了 Enter 键。

1.  shell 开始评估整个命令，包括重定向 (`>`).

1.  shell 尝试在受保护目录 */var/log* 中创建文件 *custom.log*。

1.  您没有权限写入 */var/log*，因此 shell 放弃并打印“权限被拒绝”消息。

这就是为什么 `sudo` 从未运行。要解决这个问题，您需要告诉 shell：“以超级用户身份运行整个命令，包括输出重定向。”这正是 `bash -c` 解决得很好的情况。构建您想要作为字符串运行的命令：

```
'echo "New log file" > /var/log/custom.log'
```

并将其作为参数传递给`sudo bash -c`：

```
$ sudo bash -c 'echo "New log file" > /var/log/custom.log'
[sudo] password for smith: xxxxxxxx
$ cat /var/log/custom.log
New log file
```

这一次，您已经以超级用户身份运行了 `bash`，而不仅仅是 `echo`，`bash` 执行整个字符串作为命令。重定向成功了。每当将 `sudo` 与重定向配对使用时，请记住这个技巧。

## 技巧 #6：将命令管道传输到 bash

shell 会读取您在 stdin 上键入的每个命令。这意味着 `bash` 程序可以参与管道。例如，打印字符串 `"ls -l"` 并将其管道到 `bash`，`bash` 将该字符串视为命令并运行它：

```
$ echo "ls -l"
ls -l
$ echo "ls -l" | bash
-rw-r--r-- 1 smith smith 325 Jul  3 17:44 animals.txt
```

###### 警告

请记住，永远不要盲目地将文本管道到 `bash`。要意识到您正在执行什么。

当你需要连续运行许多相似命令时，这种技术非常棒。如果可以将命令打印为字符串，那么可以将字符串通过管道传输到`bash`以执行。假设你在一个包含许多文件的目录中，并且想要按它们的第一个字符将它们组织到子目录中。一个名为*apple*的文件将被移动到子目录*a*，一个名为*cantaloupe*的文件将移动到子目录*c*，依此类推。^(6)（为简单起见，我们假设所有文件名以小写字母开头且不包含空格或特殊字符。）

首先，列出按排序的文件。我们假设所有名称至少为两个字符长（与模式`??*`匹配），因此我们的命令不会与子目录*a*到*z*发生冲突：

```
$ ls -1 ??* 
apple
banana
cantaloupe
carrot
⋮
```

通过大括号扩展创建你需要的 26 个子目录：

```
$ mkdir {a..z}
```

现在生成你需要的`mv`命令，作为字符串。从一个为`sed`设计的正则表达式开始，它捕获文件名的第一个字符作为表达式＃1（`\1`）：

```
^\(.\)
```

捕获剩余的文件名作为表达式＃2（`\2`）：

```
\(.*\)$
```

连接这两个正则表达式：

```
^\(.\)\(.*\)$
```

现在用单词*mv*后跟一个空格，完整的文件名（`\1\2`），再加一个空格和第一个字符（`\1`）来形成一个`mv`命令：

```
mv \1\2 \1
```

完整的命令生成器是：

```
$ ls -1 ??* | sed 's/^\(.\)\(.*\)$/mv \1\2 \1/'
mv apple a
mv banana b
mv cantaloupe c
mv carrot c
⋮
```

其输出包含你所需的确切`mv`命令。通过将其管道传输到`less`以进行逐页查看来确认它的正确性：

```
$ ls -1 ??* | sed 's/^\(.\)\(.*\)$/mv \1\2\t\1/' | less
```

当您确信生成的命令正确时，请将输出通过管道传输到`bash`以执行：

```
$ ls -1 ??* | sed 's/^\(.\)\(.*\)$/mv \1\2\t\1/' | bash
```

你刚刚完成的步骤是一个可重复的模式：

1.  通过操作字符串来打印一系列命令。

1.  使用`less`查看结果以检查正确性。

1.  将结果管道传输到`bash`。

## 技术＃7：使用 ssh 远程执行字符串

*免责声明*：仅当您熟悉用于登录远程主机的安全外壳 SSH 时，此技术才会有意义。建立主机之间的 SSH 关系超出了本书的范围；要了解更多信息，请寻找 SSH 教程。

除了通常的远程主机登录方式：

```
$ ssh myhost.example.com
```

您还可以通过在命令行上将字符串附加到`ssh`的其余部分来在远程主机上执行单个命令。：

```
$ ssh myhost.example.com ls
remotefile1
remotefile2
remotefile3
```

这种技术通常比登录，运行命令和退出更快。如果命令包含特殊字符（例如需要在远程主机上评估的重定向符号），请引用或转义它们。否则，它们将由您的本地 shell 评估。以下两个命令都在远程运行`ls`，但输出重定向发生在不同的主机上：

```
$ ssh myhost.example.com ls > outfile       *Creates outfile on local host*
$ ssh myhost.example.com "ls > outfile"     *Creates outfile on remote host*
```

您还可以通过管道将命令传输到`ssh`以在远程主机上运行它们，就像在本地运行它们时将它们传输到`bash`一样：

```
$ echo "ls > outfile" | ssh myhost.example.com
```

在将命令传输到`ssh`时，远程主机可能会打印诊断或其他消息。这些通常不会影响远程命令，并且您可以将它们抑制。

+   如果您看到关于伪终端或伪 tty 的消息，例如“因为 stdin 不是终端，所以不会分配伪终端”，请使用`ssh`的`-T`选项运行，以防止远程 SSH 服务器分配终端：

    ```
    $ echo "ls > outfile" | ssh -T myhost.example.com
    ```

+   如果您看到通常在登录时出现的欢迎消息（“欢迎使用 Linux！”）或其他不需要的消息，请尝试显式告诉`ssh`在远程主机上运行`bash`，这些消息应该会消失：

    ```
    $ echo "ls > outfile" | ssh myhost.example.com bash
    ```

## 技术＃8：使用 xargs 运行命令列表

许多 Linux 用户从未听说过`xargs`命令，但它是一个强大的工具，用于构建和运行多个相似的命令。学习`xargs`是我 Linux 教育中的又一个转折点，我也希望对您有帮助。

`xargs`接受两个输入：

+   在标准输入上：由空格分隔的字符串列表。例如由`ls`或`find`产生的文件路径，但任何字符串都可以。我将它们称为*输入字符串*。

+   在命令行上：一个不完整的命令，缺少一些参数，我将其称为*命令模板*。

`xargs`将输入字符串和命令模板合并，生成并运行新的完整命令，我将其称为*生成的命令*。我将通过一个玩具示例来演示这个过程。假设您在一个有三个文件的目录中：

```
$ ls -1
apple
banana
cantaloupe
```

将目录列表通过管道传递给`xargs`作为其输入字符串，并提供`wc -l`作为命令模板，如下所示：

```
$ ls -1 | xargs wc -l
3 apple
4 banana
1 cantaloupe
8 total
```

正如承诺的那样，`xargs`将`wc -l`命令模板应用于输入字符串，并计算每个文件的行数。要使用`cat`打印相同的三个文件，只需将命令模板更改为`cat`：

```
$ ls -1 | xargs cat
```

我对`xargs`的玩具示例有两个缺点，一个是致命的，一个是实际的。致命的缺点是，如果输入字符串包含特殊字符（例如空格），`xargs`可能会执行错误的操作。在侧边栏“使用 find 和 xargs 时的安全性”中有一个健壮的解决方案。

实际的缺点是，在这里您不需要`xargs`—您可以通过文件模式匹配更简单地完成相同的任务：

```
$ wc -l * 
3 apple
4 banana
1 cantaloupe
8 total
```

那么为什么要使用`xargs`呢？当输入字符串比简单的目录列表更有趣时，它的强大之处就显现出来了。假设您想递归地计算一个目录及其所有子目录中所有 Python 源文件（以*.py 结尾）的行数。使用`find`轻松生成这样的文件路径列表：

```
$ find . -type f -name \*.py -print
fruits/raspberry.py
vegetables/leafy/lettuce.py
⋮
```

当前，`xargs`可以将命令模板`wc -l`应用于每个文件路径，生成一个递归的结果，否则将很难获得。为了安全起见，我将选项`-print`替换为`-print0`，并将`xargs`替换为`xargs -0`，原因在侧边栏“使用 find 和 xargs 时的安全性”中有解释：

```
$ find . -type f -name \*.py -print0 | xargs -0 wc -l
6 ./fruits/raspberry.py
3 ./vegetables/leafy/lettuce.py
⋮
```

通过结合`find`和`xargs`，你可以使任何命令递归运行到文件系统中，仅影响符合你指定条件的文件（和/或目录）。在某些情况下，你可以仅使用`find`的`-exec`选项达到相同效果，但`xargs`通常是一个更干净的解决方案。

`xargs`有许多选项（参见`man xargs`），用于控制它如何创建和运行生成的命令。在我看来，除了`-0`之外，最重要的选项是`-n`和`-I`。`-n`选项控制`xargs`在每个生成的命令中添加多少个参数。默认行为是尽可能添加适合 shell 限制的参数数目：^(7)

```
$ ls | xargs echo                     *Fit as many input strings as possible:*
apple banana cantaloupe carrot          *echo apple banana cantaloupe carrot*
$ ls | xargs -n1 echo                 *One argument per echo command:*
apple                                   *echo apple*
banana                                  *echo banana*
cantaloupe                              *echo cantaloupe*
carrot                                  *echo carrot*
$ ls | xargs -n2 echo                 *Two arguments per echo command:*
apple banana                            *echo apple banana*
cantaloupe carrot                       *echo cantaloupe carrot*
$ ls | xargs -n3 echo                 *Three arguments per echo command:*
apple banana cantaloupe                 *echo apple banana cantaloupe*
carrot                                  *echo carrot*
```

`-I`选项控制输入字符串在生成命令中的位置。默认情况下，它们附加到命令模板，但你可以将它们放置在其他位置。在`-I`后跟任意字符串（你选择的字符串），那个字符串将成为命令模板中的占位符，指示输入字符串应该插入的确切位置：

```
$ ls | xargs -I XYZ echo XYZ is my favorite food      *Use XYZ as a placeholder*
apple is my favorite food
banana is my favorite food
cantaloupe is my favorite food
carrot is my favorite food
```

我随意选择“XYZ”作为输入字符串的占位符，并将其放置在`echo`后面，将输入字符串移动到每个输出行的开头。请注意，`-I`选项限制`xargs`每个生成的命令只接受一个输入字符串。我建议仔细阅读`xargs`的手册，以了解你还能控制哪些内容。

# 长参数列表

当命令行变得非常长时，`xargs`是一个解决方案。假设当前目录包含 100 万个名为*file1.txt*至*file1000000.txt*的文件，如果你尝试通过模式匹配来删除它们：

```
$ rm *.txt
bash: /bin/rm: Argument list too long
```

模式`*.txt`的评估结果是一个超过 1400 万字符的字符串，这比 Linux 支持的长度还长。要解决此限制，请将文件列表传输到`xargs`以便删除。`xargs`将文件列表分割成多个`rm`命令。通过将完整目录列表管道传输到`grep`，仅匹配以*.txt*结尾的文件名，然后管道传输给`xargs`：

```
$ ls | grep '\.txt$' | xargs rm
```

这种解决方案比文件模式匹配(`ls *.txt`)更好，后者会产生相同的“Argument list too long”错误。更好的方法是像“使用 find 和 xargs 安全删除文件”中描述的那样运行`find -print0`：

```
$ find . -maxdepth 1 -name \*.txt -type f -print0 \
  | xargs -0 rm
```

# 进程控制技术

到目前为止，我讨论的所有命令都占据父 shell 直到完成。让我们考虑几种与父 shell 建立不同关系的技术：

后台命令

立即返回提示符并在视线外执行

明确的子 shell

可以在组合命令的中间启动

进程替换

取代父 shell

## 技术#9：后台运行命令

到目前为止，我们所有的技术都是等待命令完成，然后显示下一个 shell 提示符。但你不必等待，特别是对于执行时间长的命令。你可以以特殊的方式启动命令，让它们从视线中消失（或者说几乎消失），但继续运行，立即释放当前 shell 运行更多的命令。这个技术称为 *后台运行* 命令或 *在后台运行命令*。相比之下，占据 shell 的命令称为 *前台* 命令。一个 shell 实例最多同时运行一个前台命令，加上任意数量的后台命令。

### 在后台启动一个命令

要在后台运行一个命令，只需附加一个&符号。shell 会用一个看起来神秘的消息回应，指示命令已被后台运行，并显示下一个提示符：

```
$ wc -c my_extremely_huge_file.txt &     *Count characters in a huge file*
[1] 74931                                *Cryptic-looking response*
$
```

你可以继续在这个 shell 中运行前台命令（或更多的后台命令）。后台命令的输出可能随时出现，甚至在你输入时也可能出现。如果后台命令成功完成，shell 将会用 *Done* 消息通知你：

```
59837483748 my_extremely_huge_file.txt
[1]+  Done               wc -c my_extremely_huge_file.txt
```

或者如果失败，你会看到一个带有退出码的 *Exit* 消息：

```
[1]+  Exit 1             wc -c my_extremely_huge_file.txt
```

###### 提示

这个&符号也是列表操作符，类似于 `&&` 和 `||`：

```
$ *command1* & *command2* & *command3* &   *All 3 commands*
[1] 57351                            *in background*
[2] 57352
[3] 57353
$ *command4* & *command5* & echo hi      *All in background*
[1] 57431                            *but "echo"*
[2] 57432
hi
```

### 暂停命令并发送到后台

一个相关的技巧是运行前台命令，在执行过程中改变主意，并将其发送到后台。按下 Ctrl-Z 暂停命令（称为 *挂起* 命令）并返回到 shell 提示符；然后输入 `bg` 来恢复在后台运行该命令。

### 工作和作业控制

后台命令是 shell 的一个特性，称为 *作业控制*，可以以各种方式操作正在运行的命令，如后台运行、挂起和恢复。一个 *作业* 是 shell 的工作单位：在 shell 中运行的命令的单个实例。简单命令、管道和条件列表都是作业的例子——基本上任何你可以在命令行运行的东西。

一个作业不仅仅是一个 Linux 进程。一个作业可以由一个进程、两个进程或更多进程组成。例如，一个包含六个程序的管道是一个单一的作业，其中至少包括六个进程。作业是 shell 的构造。Linux 操作系统并不跟踪作业，只跟踪底层的进程。

在任何时刻，一个 shell 可能会有多个作业在运行。给定 shell 中的每个作业都有一个正整数 ID，称为作业 ID 或作业号。当你在后台运行命令时，shell 会打印作业号和作业内第一个进程的 ID。在以下命令中，作业号为 1，进程 ID 为 74931：

```
$ wc -c my_extremely_huge_file.txt &
[1] 74931
```

### 常见的作业控制操作

Shell 对控制作业有内置命令，列在表 7-1 中。我将通过运行一系列作业并对其进行操作来演示最常见的作业控制操作。为了保持作业简单和可预测性，我将运行命令`sleep`，它什么也不做（“睡眠”）一段时间，然后退出。例如，`sleep 10`表示休眠 10 秒。

表 7-1\. 作业控制命令

| Command | 含义 |
| --- | --- |
| bg | 将当前挂起的作业移到后台 |
| bg %*n* | 将挂起的作业号*n*移至后台（例如：`bg %1`） |
| fg | 将当前后台作业移到前台 |
| fg %*n* | 将后台作业号为*n*的作业移到前台（例如：`fg %2`） |
| kill %*n* | 终止后台作业号为*n*的作业（例如：`kill %3`） |
| jobs | 查看 shell 的作业 |

将一个作业在后台运行至完成：

```
$ sleep 20 &                        *Run in the background*
[1] 126288
$ jobs                              *List this shell's jobs*
[1]+  Running          sleep 20 &
$
*...eventually...*
[1]+  Done             sleep 20
```

###### 注意

当作业完成时，“完成”消息可能不会立即显示，直到您再次按 Enter 键。

运行一个后台作业并将其切换至前台：

```
$ sleep 20 &                        *Run in the background*
[1] 126362
$ fg                                *Bring into the foreground*
sleep 20
*...eventually...*
$
```

运行一个前台作业，将其暂停，然后将其切换回前台：

```
$ sleep 20                          *Run in the foreground*
^Z                                  *Suspend the job*
[1]+  Stopped          sleep 20
$ jobs                              *List this shell's jobs*
[1]+  Stopped          sleep 20
$ fg                                *Bring into the foreground*
sleep 20
*...eventually...*
[1]+  Done             sleep 20
```

运行前台作业并将其发送到后台：

```
$ sleep 20                          *Run in the foreground*
^Z                                  *Suspend the job*
[1]+  Stopped          sleep 20
$ bg                                *Move to the background*
[1]+ sleep 20 &
$ jobs                              *List this shell's jobs*
[1]+  Running          sleep 20 &
$
*...eventually...*
[1]+  Done             sleep 20
```

处理多个后台作业。通过作业号加上百分号（`%1`，`%2`等）进行引用：

```
$ sleep 100 &                        *Run 3 commands in the background*
[1] 126452
$ sleep 200 &
[2] 126456
$ sleep 300 &
[3] 126460
$ jobs                               *List this shell's jobs*
[1]   Running          sleep 100 &
[2]-  Running          sleep 200 &
[3]+  Running          sleep 300 &
$ fg %2                              *Bring job 2 into the foreground*
sleep 200
^Z                                   *Suspend job 2*
[2]+  Stopped          sleep 200
$ jobs                               *See job 2 is suspended ("stopped")*
[1]   Running          sleep 100 &
[2]+  Stopped          sleep 200
[3]-  Running          sleep 300 &
$ kill %3                            *Terminate job 3*
[3]+  Terminated       sleep 300
$ jobs                               *See job 3 is gone*
[1]-  Running          sleep 100 &
[2]+  Stopped          sleep 200
$ bg %2                              *Resume suspended job 2 in the background*
[2]+ sleep 200 &
$ jobs                               *See job 2 is running again*
[1]-  Running          sleep 100 &
[2]+  Running          sleep 200 &
$
```

### 在后台进行输出和输入

后台命令可能会在不方便或混乱的时间写入标准输出。如果按预期对 Linux 字典文件（有 100,000 行）进行排序并在后台打印前两行时，Shell 会立即打印作业号（1）、进程 ID（81089）和下一个提示符：

```
$ sort /usr/share/dict/words | head -n2 &
[1] 81089
$
```

如果等到作业完成再打印两行到标准输出，输出可能会显得杂乱无章。在此情况下，光标位于第二个提示符处，所以您会得到这种看起来不整齐的输出：

```
$ sort /usr/share/dict/words | head -n2 &
[1] 81089
$ A
A's
```

按 Enter 键，Shell 将打印“作业完成”消息：

```
[1]+  Done               sort /usr/share/dict/words | head -n2
$
```

后台作业的屏幕输出可能会在作业运行时的任何时候出现。为避免这种混乱，将标准输出重定向到文件，然后在方便时检查文件内容：

```
$ sort /usr/share/dict/words | head -n2 > /tmp/results &
[1] 81089
$
[1]+  Done               sort /usr/share/dict/words | head -n2 > /tmp/results
$ cat /tmp/results
A
A's
$
```

当后台作业尝试从标准输入读取时，会发生一些奇怪的事情。Shell 暂停该作业，打印*已停止*消息，并在后台等待输入。通过不带参数后台化`cat`来演示这一点：

```
$ cat &
[1] 82455
[1]+  Stopped            cat
```

后台作业无法读取输入，因此使用`fg`将作业切换至前台，然后提供输入：

```
$ fg
cat
Here is some input
Here is some input
⋮
```

提供所有输入后，执行以下任何一项操作：

+   在前台继续运行命令，直到完成。

+   通过按 Ctrl-Z 然后 `bg`，将命令暂停并移到后台。

+   用 Ctrl-D 结束输入，或用 Ctrl-C 终止命令。

### 后台化提示

后台运行非常适合那些需要长时间运行的命令，比如在长时间编辑会话期间的文本编辑器，或者任何打开自己窗口的程序。例如，程序员可以通过挂起他们的文本编辑器而不是退出来节省大量时间。我见过有经验的工程师修改他们文本编辑器中的一些代码，保存并退出编辑器，测试代码，然后重新启动编辑器并搜索他们离开的代码位置。他们每次退出编辑器都会损失 10 到 15 秒的工作切换时间。如果他们代替这样做挂起编辑器（Ctrl-Z），测试他们的代码，然后恢复编辑器（`fg`），他们避免了不必要的时间浪费。

后台运行也非常适合使用条件列表在后台运行一系列命令。如果列表中的任何命令失败，其余命令将不会运行，作业完成。（只需注意读取输入的命令，因为它们会导致作业挂起并等待输入。）

```
$ *command1* && *command2* && *command3* &
```

## 技巧 #10：显式子 shell

每次启动简单命令时，它都在子进程中运行，就像你在“父进程和子进程”中看到的那样。命令替换和进程替换创建子 shell。然而，有时明确启动额外的子 shell 很有帮助。要做到这一点，只需将命令括在括号中，它就会在子 shell 中运行：

```
$ (cd /usr/local && ls)
bin   etc   games   lib   man   sbin   share
$ pwd
/home/smith                   *"cd /usr/local" occurred in a subshell*
```

当应用于整个命令时，这种技巧并不是特别有用，除非也许可以帮助你避免运行第二个`cd`命令返回到先前的目录。但是，如果你在组合命令的一部分周围放置括号，你可以执行一些有用的技巧。一个典型的例子是在执行过程中更改目录的管道。假设你下载了一个压缩的`tar`文件，*package.tar.gz*，并且你想要提取文件。一个用于提取文件的`tar`命令是：

```
$ tar xvf package.tar.gz
Makefile
src/
src/defs.h
src/main.c
⋮
```

提取发生在当前目录的相对位置。^(8) 如果你想将它们提取到另一个目录怎么办？你可以首先`cd`到其他目录，然后运行`tar`（然后再`cd`回来），但你也可以用一个命令完成这个任务。窍门在于将被打包的数据传输到一个执行目录操作并在从 stdin 读取时运行`tar`的子 shell 中：^(9)

```
$ cat package.tar.gz | (mkdir -p /tmp/other && cd /tmp/other && tar xzvf -)
```

这个技巧还适用于使用两个`tar`进程将文件从一个目录*dir1*复制到另一个现有目录*dir2*，一个写入 stdout，另一个从 stdin 读取：

```
$ tar czf - dir1 | (cd /tmp/dir2 && tar xvf -)
```

相同的技巧也可以通过 SSH 将文件复制到另一个主机上的现有目录：

```
$ tar czf - dir1 | ssh myhost '(cd /tmp/dir2 && tar xvf -)'
```

###### 警告

看起来像是`bash`中的括号仅仅是将命令组合在一起，就像数学中的括号一样诱人。但实际上并不是这样。每一对括号都会启动一个子 shell。

## 技巧 #11：进程替换

通常情况下，当你运行一个命令时，shell 会将其运行在一个单独的进程中，当命令退出时该进程被销毁，如“父进程和子进程”中所述。你可以通过 shell 内置命令`exec`改变这种行为。它会将正在运行的 shell（一个进程）替换为你选择的另一个命令（另一个进程）。当新命令退出时，不会出现 shell 提示符，因为原始 shell 已经不存在。

为了演示这一点，手动运行一个新的 shell 并更改其提示符：

```
$ bash                   *Run a child shell*
$ PS1="Doomed> "         *Change the new shell's prompt*
Doomed> echo hello       *Run any command you like*
hello
```

现在`exec`一个命令并观察新 shell 的关闭：

```
Doomed> exec ls          *ls replaces the child shell, runs, and exits*
animals.txt
$                        *A prompt from the original (parent) shell*
```

# 运行 exec 可能是致命的

如果你在 shell 中运行`exec`，那么 shell 在此后会退出。如果 shell 是在终端窗口中运行的，窗口会关闭。如果 shell 是登录 shell，你会被注销。

为什么会运行`exec`？一个原因是通过不启动第二个进程来节省资源。Shell 脚本有时会利用这种优化，在脚本的最后一个命令上运行`exec`。如果脚本运行多次（比如，数百万或数十亿次执行），这种节省可能是值得的。

`exec` 还有第二个能力——它可以重新分配当前 shell 的 stdin、stdout 和/或 stderr。这在 shell 脚本中最实用，比如这个玩具示例，将信息打印到文件 */tmp/outfile*：

```
#!/bin/bash
echo "My name is $USER"                                 > /tmp/outfile
echo "My current directory is $PWD"                     >> /tmp/outfile
echo "Guess how many lines are in the file /etc/hosts?" >> /tmp/outfile
wc -l /etc/hosts                                        >> /tmp/outfile
echo "Goodbye for now"                                  >> /tmp/outfile
```

不要单独将每个命令的输出重定向到 */tmp/outfile*，而是使用`exec`将整个脚本的 stdout 重定向到 */tmp/outfile*。随后的命令可以简单地输出到 stdout：

```
#!/bin/bash
# Redirect stdout for this script
exec > /tmp/outfile2
# All subsequent commands print to /tmp/outfile2
echo "My name is $USER"
echo "My current directory is $PWD"
echo "Guess how many lines are in the file /etc/hosts?"
wc -l /etc/hosts
echo "Goodbye for now"
```

运行这个脚本并检查文件 */tmp/outfile2* 以查看结果：

```
$ cat /tmp/outfile2
My name is smith
My current directory is /home/smith
Guess how many lines are in the file /etc/hosts?
122 /etc/hosts
Goodbye for now
```

你可能不经常使用`exec`，但当你需要时它就在那里。

# 总结

现在你掌握了运行命令的 13 种技术——本章的 11 种技术加上简单命令和管道。表 7-2 概述了不同技术的常见用例。

表 7-2\. 运行命令的常见习惯用法

| 问题 | 解决方案 |
| --- | --- |
| 将一个程序的 stdout 发送到另一个程序的 stdin | 管道传输 |
| 将输出（stdout）插入到一个命令中 | 命令替换 |
| 提供输出（stdout）给一个不读取 stdin，但读取磁盘文件的命令 | 进程替换 |
| 将一个字符串作为命令执行 | `bash -c`，或将其传输到`bash` |
| 在标准输出上打印多个命令并执行它们 | 管道传输至`bash` |
| 连续执行多个类似的命令 | 使用`xargs`，或构造命令字符串并将其传输到`bash` |
| 管理依赖于彼此成功的命令 | 条件列表 |
| 同时运行多个命令 | 后台运行 |
| 同时运行多个依赖于彼此成功的命令 | 后台运行的条件列表 |
| 在远程主机上运行一个命令 | 运行 `ssh` *`host command`* |
| 在管道中间更改目录 | 显式子 shell |
| 以后执行一个命令 | 使用`sleep`延时后紧跟命令的无条件列表 |
| 重定向到/从受保护文件 | 运行 `sudo bash -c "`*`command`* `>` *`file`*`"` |

下两章将教你如何结合技术以高效实现业务目标。

^(1) 命令 `mkdir -p dir` 可以创建一个目录路径，仅在该路径不存在时才创建，这在这里是一个更优雅的解决方案。

^(2) 这种行为与许多编程语言相反，其中零表示失败。

^(3) 或者，你可以使用`cron`进行备份作业和使用`at`设置提醒，但 Linux 注重灵活性——寻找实现同一目标的多种方法。

^(4) 目前，美国银行的可下载对账单文件是这样命名的。

^(5) 从技术上讲，如果你提供一个短划线作为文件名，`diff`可以从标准输入读取一个列表，但不能读取两个列表。

^(6) 这个目录结构类似于带有链表的哈希表。

^(7) 精确数字取决于你的 Linux 系统对长度限制；参见`man xargs`。

^(8) 假设`tar`归档是使用相对路径构建的——这在下载软件时很典型，而不是绝对路径。

^(9) 可以更简单地使用`tar`选项`-C`或`--directory`解决这个具体问题，该选项指定目标目录。我只是演示了使用子 shell 的一般技巧。
