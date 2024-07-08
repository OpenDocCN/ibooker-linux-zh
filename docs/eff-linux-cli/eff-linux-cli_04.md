# 第三章：重新运行命令

假设您刚刚执行了一个包含详细管道的长命令，例如来自“检测重复文件”的命令：

```
$ md5sum *.jpg | cut -c1-32 | sort | uniq -c | sort -nr
```

而您希望再次运行它。不要重新输入！相反，请让 Shell 回溯历史记录并重新运行命令。在幕后，Shell 会记录您调用的命令，因此您可以轻松地通过几个按键回忆并重新运行它们。这个 Shell 功能称为*命令历史*。熟练的 Linux 用户大量使用命令历史来加快工作速度，避免浪费时间。

类似地，假设在运行之前您在输入前述命令时出现拼写错误，比如将“jpg”拼写为“jg”：

```
$ md5sum *.jg | cut -c1-32 | sort | uniq -c | sort -nr
```

要修正错误，请不要按下退格键几十次并重新键入所有内容。相反，请在原地更改命令。Shell 支持*命令行编辑*，用于修正拼写错误和执行各种修改，就像文本编辑器一样。

本章将向您展示如何通过利用命令历史和命令行编辑来节省大量时间和输入。像往常一样，我不会试图面面俱到——我将专注于这些 Shell 功能中最实用和最有用的部分。（如果您使用的 Shell 不是`bash`，请参阅附录 B 获取额外的注意事项。）

# 学会盲打

如果您能快速打字，本书中所有建议都会对您有所帮助。无论您有多么博学，如果您每分钟打字 40 个单词，而您的同样博学的朋友每分钟打字 120 个，他们的工作速度将比您快三倍。搜索“打字速度测试”以测量您的速度，然后搜索“打字教程”，培养一项终生技能。努力达到每分钟 100 个单词。这是值得努力的。

# 查看命令历史

*命令历史*简单来说就是您在交互式 Shell 中执行的先前命令列表。要查看 Shell 的历史记录，请运行`history`命令，这是一个 Shell 内置命令。命令按时间顺序显示，并附有 ID 编号以便于参考。输出看起来类似于这样：

```
$ history
 1000  cd $HOME/Music
 1001  ls
 1002  mv jazz.mp3 jazzy-song.mp3
 1003  play jazzy-song.mp3
 ⋮                                      *Omitting 479 lines*
 1481  cd
 1482  firefox https://google.com
 1483  history                         *Includes the command you just ran*
```

`history` 命令的输出可能有几百行长（或更多）。通过添加整数参数来限制它仅打印最近的命令行数，该参数指定要打印的行数：

```
$ history 3                            *Print the 3 most recent commands*
 1482  firefox https://google.com
 1483  history
 1484  history 3
```

由于`history`输出到 stdout，您也可以使用管道处理输出。例如，逐屏查看您的历史记录：

```
$ history | less                      *Earliest to latest entry*
$ history | sort -nr | less           *Latest to earliest entry*
```

或者仅打印包含单词`cd`的历史命令：

```
$ history | grep -w cd
 1000  cd $HOME/Music
 1092  cd ..
 1123  cd Finances
 1375  cd Checking
 1481  cd
 1485  history | grep -w cd
```

要清除（删除）当前 Shell 的历史记录，请使用 `-c` 选项：

```
$ history -c
```

# 从历史记录中召回命令

我将向您展示三种从 Shell 历史中召回命令的省时方法：

光标移动

学起来极其简单，但在实践中通常较慢

历史扩展

更难学（坦白地说，它很神秘），但可以非常快速

渐进搜索

简单而快速

每种方法在特定情况下都是最好的，所以我建议学习所有三种。你掌握的技术越多，就能在任何情况下更好地选择合适的方法。

## 历史记录浏览

要在给定 shell 中回忆你的上一个命令，按上箭头键。就是这么简单。继续按上箭头以逆时间顺序回忆较早的命令。按下箭头向另一个方向移动（向更近的命令）。当你到达想要的命令时，按 Enter 运行它。

浏览命令历史记录是 Linux 用户学习的两种最常见的加速方法之一。（另一种是使用`*`进行文件名模式匹配，如你在第二章中看到的。）如果你想要的命令在历史中附近——最多是两三个命令之前——浏览是有效的，但是要达到更远的命令则很烦琐。连续按上箭头键 137 次很快就会令人厌倦。

光标浏览的最佳用例是回忆和运行即时前一个命令。在许多键盘上，上箭头键靠近 Enter 键，因此您可以快速连续按这两个键。在标准美式 QWERTY 键盘上，我将右手的无名指放在上箭头上，食指放在 Enter 上，可以高效地轻敲这两个键。（试试看。）

## 历史扩展

历史扩展是一种利用特殊表达式访问命令历史的 shell 特性。这些表达式以感叹号开头，传统上发音为“bang”。例如，两个感叹号连续使用（“bang bang”）将评估立即前一个命令：

```
$ echo Efficient Linux
Efficient Linux
$ !!                            *"Bang bang" = previous command*
echo Efficient Linux            *The shell helpfully prints the command being run*
Efficient Linux
```

要引用以特定字符串开头的最近命令，请在该字符串前面加上感叹号。因此，要重新运行最近的`grep`命令，请运行“bang grep”：

```
$ !grep
grep Perl animals.txt
alpaca	Intermediate Perl	2012	Schwartz, Randal
```

要引用包含给定字符串*任意位置*的最近命令，请将字符串用问号包围：^(1)

```
$ !?grep?
history | grep -w cd
 1000  cd $HOME/Music
 1092  cd ..
⋮
```

你还可以通过 shell 历史记录中命令的绝对位置来检索特定命令——在`history`输出中其左侧的 ID 号码。例如，表达式`!1203`（“bang 1023”）表示“历史记录中位置 1023 处的命令”：

```
$ history | grep hosts
 1203  cat /etc/hosts
$ !1203                         *The command at position 1023*
cat /etc/hosts
127.0.0.1       localhost
127.0.1.1       example.oreilly.com
::1             example.oreilly.com
```

负值通过历史记录中的相对位置检索命令，而不是绝对位置。例如，`!-3`（“bang minus three”）表示“你执行的三个命令前的命令”：

```
$ history
 4197  cd /tmp/junk
 4198  rm *
 4199  head -n2 /etc/hosts
 4199  cd
 4200  history
$ !-3                         *The command you executed three commands ago*
head -n2 /etc/hosts
127.0.0.1       localhost
127.0.1.1       example.oreilly.com
```

历史扩展快速方便，虽然有点晦涩。但如果提供错误的值并盲目执行，可能会有风险。仔细看看前面的例子。如果计数错误，输入`!-4`而不是`!-3`，你会意外地运行`rm *`而不是预期的`head`命令，从而删除家目录中的文件！为了减少这种风险，在命令后面附加修饰符`:p`以打印历史命令但不执行它：

```
$ !-3:p
head -n2 /etc/hosts              *Printed, not executed*
```

Shell 将未执行的命令（`head`）附加到历史记录中，因此如果看起来没问题，您可以方便地使用快速的“叹号叹号”来运行它：

```
$ !-3:p
head -n2 /etc/hosts              *Printed, not executed, and appended to history*
$ !!                             *Run the command for real*
head -n2 /etc/hosts              *Printed and then executed*
127.0.0.1       localhost
127.0.1.1       example.oreilly.com
```

有些人将历史扩展称为“叹号命令”，但像`!!`和`!grep`这样的表达式并不是命令。它们是您可以在*任何地方*放置的字符串表达式。作为演示，使用`echo`在标准输出上打印`!!`的值，而不执行它，并用`wc`计算单词数：

```
$ ls -l /etc | head -n3       *Run any command*
total 1584
drwxr-xr-x  2 root     root       4096 Jun 16 06:14 ImageMagick-6/
drwxr-xr-x  7 root     root       4096 Mar 19  2020 NetworkManager/

$ echo "!!" | wc -w           *Count the words in the previous command*
echo "ls -l /etc | head -n3" | wc -w
6
```

这个玩具例子演示了历史扩展比执行命令更有用途。在下一节中，您将看到一个更实用、更强大的技术。

我在这里只介绍了命令历史的一些功能。要获取完整信息，请运行`man history`。

# 命令历史中没有历史表达式。

如我在“关于命令历史的常见问题”中所述，shell 将命令原封不动地追加到历史中，不经过评估。唯一的例外是历史扩展。它的表达式总是在添加到命令历史之前进行评估：

```
$ ls               *Run any command*
hello.txt
$ cd Music         *Run some other command*
$ !-2              *Use history expansion*
ls
song.mp3
$ history          *View the history*
 1000  ls
 1001  cd Music
 1002  ls          *"ls" appears in the history, not "!-2"*
 1003  history
```

这个例外情况是有道理的。想象一下，试图理解一个充满像`!-15`和`!-92`这样引用其他历史条目的表达式的命令历史。您可能不得不通过眼睛追踪整个历史记录路径，才能理解一个单独的命令。

## 再也不会因为误删错误文件（感谢历史扩展）。

您是否曾经打算使用模式（例如`*.txt`）删除文件，但却意外地错误输入了模式，导致删除了错误的文件？这里有一个带有星号后意外空格字符的示例：

```
$ ls
123  a.txt   b.txt   c.txt  dont-delete-me  important-file  passwords
$ rm * .txt       *DANGER!! Don't run this! Deletes the wrong files!*
```

避免这种危险的最常见解决方案是将`rm`的别名设置为运行`rm -i`，这样在每次删除之前都会提示确认：

```
$ alias rm='rm -i'                  *Often found in a shell configuration file*
$ rm *.txt
/bin/rm: remove regular file 'a.txt'? y
/bin/rm: remove regular file 'b.txt'? y
/bin/rm: remove regular file 'c.txt'? y
```

因此，多余的空格字符不会致命，因为来自`rm -i`的提示将警告您正在删除错误的文件：

```
$ rm * .txt
/bin/rm: remove regular file '123'?      *Something is wrong: kill the command*
```

然而，别名解决方案很麻烦，因为大多数时候您可能不想要或不需要`rm`提示您。如果您在另一台没有别名的 Linux 机器上登录，它也不起作用。我将向您展示一种更好的方法来避免使用模式匹配错误的文件名。该技术有两个步骤，并依赖于历史扩展：

1.  *验证*。在运行`rm`之前，使用所需模式运行`ls`以查看匹配的文件。

    ```
    $ ls *.txt
    a.txt   b.txt   c.txt
    ```

1.  *删除*。如果`ls`的输出看起来正确，请运行`rm !$`以删除匹配的相同文件[²]。

    ```
    $ rm !$
    rm *.txt
    ```

历史扩展`!$`（“叹号美元”）表示“前一个命令中您键入的最后一个词”。因此，这里的`rm !$`是“删除我刚刚用`ls`列出的任何东西”的简写，即`*.txt`。如果您在星号后意外添加了一个空格，`ls`的输出将明确显示——安全地——出现了问题。

```
$ ls * .txt
/bin/ls: cannot access '.txt': No such file or directory
123  a.txt   b.txt   c.txt  dont-delete-me  important-file  passwords
```

很幸运你先运行了 `ls` 而不是 `rm`！现在你可以修改命令以去除多余的空格并安全进行。这个由`ls`后跟`rm !$`组成的两步命令序列是你 Linux 工具箱中很好的安全功能。

一个相关的技巧是在删除文件之前使用 `head` 查看文件内容，确保你操作的是正确的文件，然后运行 `rm !$`：

```
$ head myfile.txt
*(first 10 lines of the file appear)*
$ rm !$
rm myfile.txt
```

shell 还提供了一个历史扩展 `!*`（“bang star”），它匹配你在前一个命令中键入的所有参数，而不仅仅是最后一个参数：

```
$ ls *.txt *.o *.log
a.txt   b.txt   c.txt   main.o   output.log   parser.o
$ rm !* 
rm *.txt *.o *.log
```

在实践中，我使用 `!*` 的频率要比 `!$` 少得多。它的星号与手动输入类似 `*.txt` 一样存在风险，因为如果你误输入了，它可能会被解释为文件名的模式匹配字符，所以并不比手动输入更安全。

## 命令历史的增量搜索

如果你可以输入命令的几个字符，剩下的会立即显示并准备运行，那不是太棒了吗？事实上你可以。这个 shell 的快速功能，称为 *增量搜索*，类似于网页搜索引擎提供的交互式建议。在大多数情况下，增量搜索是从历史记录中召回命令的最简单和最快速的技术，即使是你很久以前运行的命令也是如此。我强烈建议将其添加到你的工具箱中：

1.  在 shell 提示符下，按 Ctrl-R（*R* 表示反向增量搜索）。

1.  开始输入前一个命令的 *任意部分* ——开头、中间或结尾。

1.  每输入一个字符，shell 显示最近的匹配你输入的历史命令。

1.  当你看到想要的命令时，按 Enter 运行它。

假设你一段时间前输入了命令 `cd $HOME/Finances/Bank` 并希望重新运行它。在 shell 提示符下按 Ctrl-R。提示符会改变以指示进行增量搜索：

```
(reverse-i-search)`':
```

开始输入想要的命令。例如，输入 `c`：

```
(reverse-i-search)`': c
```

shell 显示最近包含字符串 `c` 的命令，突出显示你已输入的内容：

```
(reverse-i-search)`': less /etc/hosts
```

输入下一个字母 `d`：

```
(reverse-i-search)`': cd
```

shell 显示最近包含字符串 `cd` 的命令，再次突出显示你已输入的内容：

```
(reverse-i-search)`': cd /usr/local
```

继续输入命令，加上空格和美元符号：

```
(reverse-i-search)`': cd $
```

命令行变成了：

```
(reverse-i-search)`': cd $HOME/Finances/Bank
```

这就是你想要的命令。按 Enter 运行它，你用了五个快捷键完成了。

我在这里假设 `cd $HOME/Finances/Bank` 是历史记录中最近的匹配命令。如果不是呢？如果你输入了一大堆包含相同字符串的命令？如果是这样，前面的增量搜索会显示另一个匹配项，比如：

```
(reverse-i-search)`': cd $HOME/Music
```

现在怎么办？你可以继续输入更多字符以精确到你想要的命令，但是，你可以再按一次 Ctrl-R。这个按键会导致 shell 跳转到历史记录中的 *下一个* 匹配命令：

```
(reverse-i-search)`': cd $HOME/Linux/Books
```

继续按 Ctrl-R 直到找到所需的命令：

```
(reverse-i-search)`': cd $HOME/Finances/Bank
```

并按 Enter 运行它。

这里有几个与增量搜索相关的小技巧：

+   要回忆最近搜索并执行的字符串，请从按两次 Ctrl-R 开始。

+   要停止增量搜索并继续在当前命令上工作，请按 Escape 键，或 Ctrl-J，或任何用于命令行编辑的键（本章的下一个主题），例如左右箭头键。

+   要退出增量搜索并清除命令行，请按 Ctrl-G 或 Ctrl-C。

花时间精通增量搜索。您将很快以惊人的速度定位命令。^(3)

# 命令行编辑

编辑命令有各种原因，无论是在输入时还是运行后：

+   修正错误

+   逐步创建命令，例如先输入命令的末尾，然后移动到行首输入开头部分

+   要基于先前的命令历史创建新命令（这是构建复杂管道的关键技能，您将在第八章中看到）

在本节中，我将向您展示三种编辑命令的方法，以提升您的技能和速度：

光标定位

再次强调，这是最慢且功能最弱的方法，但学习起来很简单。

插入符号表示法

一种历史扩展形式

Emacs 或 Vim 风格的按键

以强大的方式编辑命令行

与之前一样，我建议您学习所有三种技术以增强灵活性。

## 光标在命令中的定位

只需按下左箭头和右箭头键，在命令行上前后移动，逐个字符进行操作。使用退格键或删除键删除文本，然后输入所需的更正内容。表 3-1 总结了这些以及其他标准编辑命令行的按键操作。

光标来回移动很简单但效率低下。在更改小而简单时效果最佳。

表 3-1\. 简单命令行编辑的光标键

| 按键 | 动作 |
| --- | --- |
| 左箭头 | 向左移动一个字符 |
| 右箭头 | 向右移动一个字符 |
| Ctrl + 左箭头 | 向左移动一个单词 |
| Ctrl + 右箭头 | 向右移动一个单词 |
| 起始 | 移动到命令行的开头 |
| 结尾 | 移动到命令行的末尾 |
| 退格 | 删除光标前的一个字符 |
| 删除 | 删除光标下的一个字符 |

## 使用插入符号进行历史扩展

假设您误运行了以下命令，输入`jg`而不是`jpg`：

```
$ md5sum *.jg | cut -c1-32 | sort | uniq -c | sort -nr
md5sum: '*.jg': No such file or directory
```

要正确运行命令，您可以从命令历史中调用它，将光标移到错误处并修复，但有一种更快的方法来实现您的目标。只需键入旧（错误的）文本、新（修正的）文本和一对插入符号（`^`），如下所示：

```
$ ^jg^jpg
```

按 Enter，正确的命令将显示并运行：

```
$ ^jg^jpg
md5sum *.jpg | cut -c1-32 | sort | uniq -c | sort -nr
⋮
```

*插入符号语法*是历史扩展的一种类型，表示“在上一个命令中，将`jg`替换为`jpg`”。请注意，shell 会在执行之前打印新命令，这是历史扩展的标准行为。

此技术仅会更改命令中源字符串（`jg`）的第一次出现。如果原始命令中 `jg` 出现多次，只会将第一次更改为 `jpg`。

## Emacs 或 Vim 风格的命令行编辑

使用受启发于文本编辑器 Emacs 和 Vim 的熟悉按键来编辑命令行是最强大的方法。如果你已经熟练掌握其中一种编辑器，你可以立即跳入这种风格的命令行编辑中。如果不熟悉，Table 3-2 将帮助你开始使用最常见的移动和编辑按键。请注意，Emacs 的“Meta”键通常是 Escape（按下并释放）或 Alt（按住）。

Shell 的默认编辑风格是 Emacs 风格，我推荐它因为更易学易用。如果你喜欢 Vim 风格的编辑，运行以下命令（或将其添加到你的 *$HOME/.bashrc* 文件并使其生效）：

```
$ set -o vi
```

要使用 Vim 按键编辑命令，请按 Escape 键进入命令编辑模式，然后使用 Table 3-2 中的 Vim 按键。要切换回 Emacs 风格编辑，请执行以下操作：

```
$ set -o emacs
```

现在，不断练习，直到这些按键（无论是 Emacs 的还是 Vim 的）成为你的第二天性。相信我，你很快就能因节省的时间而得到回报。

Table 3-2\. Emacs 或 Vim 风格编辑的按键^(a)

| 动作 | Emacs | Vim |
| --- | --- | --- |
| 向前移动一个字符 | Ctrl-f | h |
| 向后移动一个字符 | Ctrl-b | l |
| 向前移动一个单词 | Meta-f | w |
| 向后移动一个单词 | Meta-b | b |
| 移动到行首 | Ctrl-a | 0 |
| 移动到行尾 | Ctrl-e | $ |
| 交换两个字符 | Ctrl-t | xp |
| 交换两个单词 | Meta-t | *n/a* |
| 将下一个单词的首字母大写 | Meta-c | w~ |
| 将下一个单词全部转换为大写 | Meta-u | *n/a* |
| 将下一个单词全部转换为小写 | Meta-l | *n/a* |
| 改变当前字符的大小写 | *n/a* | ~ |
| 直接插入下一个字符，包括控制字符 | Ctrl-v | Ctrl-v |
| 向前删除一个字符 | Ctrl-d | x |
| 向后删除一个字符 | Backspace *或* Ctrl-h | X |
| 向前删除一个单词 | Meta-d | dw |
| 向后删除一个单词 | Meta-Backspace *或* Ctrl-w | db |
| 从光标处剪切到行首 | Ctrl-u | d^ |
| 从光标处剪切到行尾 | Ctrl-k | D |
| 删除整行 | Ctrl-e Ctrl-u | dd |
| 粘贴（yank）最近删除的文本 | Ctrl-y | p |
| 粘贴（yank）下一个被删除的文本（在之前的粘贴后） | Meta-y | *n/a* |
| 撤销上一个编辑操作 | Ctrl-_ | u |
| 撤销所有编辑操作 | Meta-r | U |
| 从插入模式切换到命令模式 | *n/a* | Escape |
| 从命令模式切换到插入模式 | *n/a* | i |
| 中止正在进行的编辑操作 | Ctrl-g | *n/a* |
| 清除显示 | Ctrl-l | Ctrl-l |
| ^(a) 标记为*n/a*的操作没有简单的按键组合，但可能通过更长的按键序列实现。 |

要了解更多关于 Emacs 风格编辑的细节，请参阅 GNU 的`bash`手册中的[“可绑定的 Readline 命令”](https://oreil.ly/rAQ9g)章节。要了解 Vim 风格编辑，请参阅文档[“Readline VI 编辑模式速查表”](https://oreil.ly/Zv0ba)。

# 总结

练习本章中的技巧，你将大大加快命令行的使用速度。其中三项特别的技术彻底改变了我使用 Linux 的方式，希望它们也能对你有所帮助：

+   删除文件时使用`!$`以确保安全

+   使用 Ctrl-R 进行增量搜索

+   Emacs 风格命令行编辑

^(1) 在这里你可以省略尾随的问号—`!?grep`—但在某些情况下它是必需的，比如 sed 风格的历史扩展（见“使用历史扩展进行更强大的替换”）。

^(2) 我假设在`ls`步骤后你的背后没有添加或删除匹配的文件。不要依赖这种技术在快速变化的目录中。

^(3) 在撰写本书期间，我经常重新运行版本控制命令，如`git add`、`git commit`和`git push`。增量搜索使重新运行这些命令变得轻而易举。
