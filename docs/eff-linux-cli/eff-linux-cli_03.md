# 第二章：介绍 Shell

所以，你可以在提示符下运行命令。但是那个提示符*到底是什么*？它从哪里来，你的命令是如何运行的，这又为什么重要呢？

那个小提示符是一个叫做 *shell* 的程序产生的。它是一个用户界面，位于你和 Linux 操作系统之间。Linux 提供了几种 shell，最常见的（也是本书的标准）叫做 `bash`。（有关其他 shell 的说明，请参见附录 B。）

`bash` 和其他 shell 做的远不止运行命令这么简单。例如，当一个命令包含通配符 (`*`) 来一次引用多个文件时：

```
$ ls *.py
data.py    main.py    user_interface.py
```

通配符完全由 shell 处理，而不是由程序 `ls` 处理。Shell 评估表达式 `*.py` 并在 `ls` 运行之前*隐式地替换它为匹配的文件列表*。换句话说，`ls` *从不看到通配符*。从 `ls` 的角度来看，你输入了以下命令：

```
$ ls data.py main.py user_interface.py
```

Shell 还处理你在第一章中看到的管道。它透明地重定向 stdin 和 stdout，这样涉及的程序并不知道它们正在相互通信。

每次运行命令时，一些步骤是被调用程序的责任，比如 `ls`，而另一些则是 shell 的责任。专家用户明白哪个是哪个。这就是他们能够头脑风暴出长而复杂命令并成功运行的一个原因。他们在按下 Enter 键之前就*已经知道命令会做什么*，部分原因是他们理解了 shell 与其调用的程序之间的分离。

在本章中，我们将启动你对 Linux shell 的理解。我将采用与第一章命令和管道相同的最简方法。与其涵盖数十种 shell 功能，不如仅提供足够的信息来带你迈向学习之路的下一步：

+   用于文件名的模式匹配

+   存储值的变量

+   输入和输出的重定向

+   使用引号和转义字符禁用某些 shell 功能

+   用于查找要运行程序的搜索路径

+   保存对你的 shell 环境的更改

# Shell 词汇

单词*shell*有两个含义。有时它指的是 Linux shell 的*概念*，如“shell 是一个强大的工具”或“`bash` 是一个 shell”。其他时候它指的是在给定的 Linux 计算机上运行的特定*实例*，等待你下一个命令。

在本书中，“shell”的含义大部分时间应该根据上下文清楚。必要时，我会提到第二个含义，即“shell 实例”、“运行中的 shell”或者你的“当前 shell”。

一些 shell 实例具有提示符，这样你可以与它们交互。我将使用术语*交互式 shell*来指代这些实例。其他 shell 实例是非交互式的—它们运行一系列命令然后退出。

# 用于文件名的模式匹配

在第一章中，你使用了几个接受文件名作为参数的命令，如 `cut`、`sort` 和 `grep`。这些命令（及许多其他命令）接受多个文件名作为参数。例如，你可以一次在一百个文件中搜索*Linux*一词，文件名从*chapter1*到*chapter100*：

```
$ grep Linux chapter1 chapter2 chapter3 chapter4 chapter5 *...and so on...*
```

按名称列出多个文件是一种繁琐且浪费时间的做法，因此 shell 提供了特殊字符作为文件或目录的简写。许多人称这些字符为通配符，但更普遍的概念称为*模式匹配*或*通配符展开*。模式匹配是 Linux 用户学习的两种最常见的加速技术之一（另一种是按上箭头键来回忆 shell 的先前命令，我在第三章中描述了这个技巧）。

大多数 Linux 用户熟悉星号或星号字符（`*`），它匹配文件或目录路径中的任意长度的零个或多个字符（不包括前导点）^(1)：

```
$ grep Linux chapter* 
```

在幕后，shell（而不是 `grep`！）展开模式 `chapter*` 为一系列匹配的文件名。然后 shell 运行 `grep`。

许多用户还看到了问号（`?`）特殊字符，它匹配任意单个字符（除了前导点）。例如，通过提供一个问号来使 shell 匹配单个数字，你可以仅在第 1 至第九章中搜索*Linux*一词：

```
$ grep Linux chapter?
```

或者在第 10 至第九十九章中使用两个问号来匹配两位数：

```
$ grep Linux chapter??
```

较少的用户熟悉方括号（`[]`），它请求 shell 从集合中匹配单个字符。例如，你可以仅搜索前五章：

```
$ grep Linux chapter[12345]
```

同样，你可以使用连字符提供字符范围：

```
$ grep Linux chapter[1-5]
```

你还可以结合星号和方括号来匹配以偶数数字结尾的文件名，以搜索偶数章节：

```
$ grep Linux chapter*[02468]
```

方括号中可以出现任何字符，而不仅仅是数字。例如，以大写字母开头，包含下划线，并以`@`符号结尾的文件名将被 shell 匹配到：

```
$ ls [A-Z]*_*@
```

# 术语：评估表达式和展开模式

在命令行上输入的字符串，如 `chapter*` 或 `Efficient Linux`，称为*表达式*。像 `ls -l chapter*` 这样的整个命令也是一个表达式。

当 shell 解释和处理表达式中的特殊字符（如星号和管道符号）时，我们称 shell *评估*该表达式。

模式匹配是一种评估方式。当 shell 评估包含模式匹配符号（例如 `chapter*`）的表达式，并用匹配该模式的文件名替换时，我们称 shell *展开*了该模式。

模式几乎可以应用于您在命令行上提供文件或目录路径的任何地方。例如，您可以使用模式列出目录*/etc*中以*.conf*结尾的所有文件：

```
$ ls -1 /etc/*.conf
/etc/adduser.conf
/etc/appstream.conf
⋮
/etc/wodim.conf
```

谨慎使用仅接受一个文件或目录参数的命令与模式一起使用，例如`cd`。您可能得不到您期望的行为：

```
$ ls
Pictures   Poems    Politics
$ cd P*                                   *Three directories will match*
bash: cd: too many arguments
```

如果一个模式不匹配任何文件，shell 将其保留为未更改的命令参数文字传递。在以下命令中，模式`*.doc`在当前目录中找不到任何匹配项，因此`ls`寻找一个名为`*.doc`的文件名并失败：

```
$ ls *.doc
/bin/ls: cannot access '*.doc': No such file or directory
```

在使用文件模式时，有两个非常重要的要点需要记住。首先，正如我已经强调的，模式匹配由 shell 执行，而不是调用的程序。我知道我一直在重复这一点，但我经常对多少 Linux 用户不知道它并且会对某些命令成功或失败发展出迷信感到惊讶。

第二个重要点是 shell 模式匹配仅适用于文件和目录路径。它不适用于用户名、主机名和某些命令接受的其他类型的参数。您也不能在命令行开头键入（例如）`s?rt`并期望 shell 运行`sort`程序。（某些 Linux 命令如`grep`、`sed`和`awk`执行它们自己的模式匹配，我们将在第五章中探讨。）

# 文件名模式匹配和您自己的程序

所有接受文件名作为参数的程序都自动“使用”模式匹配，因为 shell 在程序运行之前评估模式。即使是您自己编写的程序和脚本也是如此。例如，如果您编写了一个程序`english2swedish`，它将文件从英语翻译成瑞典语并接受命令行上的多个文件名，您可以立即使用模式匹配运行它：

```
$ english2swedish *.txt
```

# 变量评估

运行中的 shell 可以定义变量并将值存储在其中。shell 变量与代数中的变量很像——它有一个名称和一个值。一个例子是 shell 变量`HOME`。它的值是您的 Linux 主目录路径，例如*/home/smith*。另一个例子是`USER`，其值是您的 Linux 用户名，我将在本书中假设为`smith`。

要在 stdout 上打印`HOME`和`USER`的值，请运行`printenv`命令：

```
$ printenv HOME
/home/smith
$ printenv USER
smith
```

当 shell 评估一个变量时，它将变量名替换为其值。只需在名称前面放置一个美元符号来评估变量。例如，`$HOME`评估为字符串`/home/smith`。

观察 shell 评估命令行最简单的方法是运行`echo`命令，该命令简单地打印其参数（在 shell 完成评估后）：

```
$ echo My name is $USER and my files are in $HOME    *Evaluating variables*
My name is smith and my files are in /home/smith
$ echo ch*ter9                                       *Evaluating a pattern*
chapter9
```

## 变量的来源

`USER`和`HOME`等变量由 shell 预定义。它们的值在你登录时自动设置。（稍后详细介绍这个过程。）传统上，这些预定义变量使用大写名称。

你也可以随时通过使用以下语法为变量分配一个值来定义或修改变量：

```
*name*=*value*
```

例如，如果你经常在目录*/home/smith/Projects*中工作，你可以将其名称分配给一个变量：

```
$ work=$HOME/Projects
```

并将其用作`cd`的便捷快捷方式：

```
$ cd $work
$ pwd
/home/smith/Projects
```

你可以将`$work`提供给任何期望一个目录的命令：

```
$ cp myfile $work
$ ls $work
myfile
```

定义变量时，等号周围不允许有空格。如果你忘记了，shell 会错误地假设命令行上的第一个单词是要运行的程序，等号和值则是其参数，你会看到一个错误消息：

```
$ work = $HOME/Projects               *The shell assumes "work" is a command*
work: command not found
```

类似`work`这样的用户定义变量与`HOME`这样的系统定义变量一样合法且可用。唯一的实际区别是，一些 Linux 程序会根据`HOME`、`USER`和其他系统定义变量的值内部改变其行为。例如，具有图形界面的 Linux 程序可能会从 shell 中检索你的用户名并显示它。这些程序不会关注像`work`这样的虚构变量，因为它们没有被编程来这么做。

## 变量与迷信

当你使用`echo`打印变量值时：

```
$ echo $HOME
/home/smith
```

你可能会认为`echo`命令会检查`HOME`变量并打印其值。实际情况并*不*是这样的。`echo`对变量一无所知。它只会打印你传递给它的参数。真正发生的是，在运行`echo`之前，shell 会评估`$HOME`。从`echo`的角度来看，你输入的是：

```
$ echo /home/smith
```

这种行为非常重要，特别是在我们深入了解更复杂的命令时。shell 在执行命令前会评估命令中的变量，以及模式和其他 shell 结构。

## 模式与变量

让我们测试一下你对模式和变量评估的理解。假设你在一个包含两个子目录*mammals*和*reptiles*的目录中，奇怪的是*mammals*子目录包含名为*lizard.txt*和*snake.txt*的文件：

```
$ ls
mammals   reptiles
$ ls mammals
lizard.txt  snake.txt
```

在现实世界中，蜥蜴和蛇不是哺乳动物，所以这两个文件应该移动到*reptiles*子目录中。以下是两种提议的方法。一种有效，一种无效：

```
mv mammals/*.txt reptiles                     *`Method` `1`*

FILES="lizard.txt snake.txt"
mv mammals/$FILES reptiles                    *`Method` `2`*
```

方法 1 有效，因为模式匹配整个文件路径。看看目录名*mammals*如何成为`mammals/*.txt`两个匹配的一部分：

```
$ echo mammals/*.txt
mammals/lizard.txt mammals/snake.txt
```

因此，方法 1 操作就像你输入以下正确的命令一样：

```
$ mv mammals/lizard.txt mammals/snake.txt reptiles
```

方法 2 使用的是变量，它们只评估为它们的字面值。它们对文件路径没有特殊处理：

```
$ echo mammals/$FILES
mammals/lizard.txt snake.txt
```

因此，方法 2 操作就像你输入以下有问题的命令一样：

```
$ mv mammals/lizard.txt snake.txt reptiles
```

此命令在当前目录中查找*snake.txt*文件，而不是*mammals*子目录中，所以失败了：

```
$ mv mammals/$FILES reptiles
/bin/mv: cannot stat 'snake.txt': No such file or directory
```

要使变量在这种情况下起作用，请使用 `for` 循环，在每个文件名之前添加目录名 *mammals*：

```
FILES="lizard.txt snake.txt"
for f in $FILES; do
  mv mammals/$f reptiles
done
```

# 简化命令使用别名

变量是代表值的名称。Shell 还有一种代表命令的名称，它们称为*别名*。通过发明一个名称，并在名称后跟等号和一个命令来定义别名：

```
$ alias g=grep                 *A command with no arguments*
$ alias ll="ls -l"             *A command with arguments: quotes are required*
```

通过键入其名称作为命令来运行别名。当别名比调用的命令更短时，你可以节省打字时间：

```
$ ll                                            *Runs "ls -l"*
-rw-r--r-- 1 smith smith 325 Jul  3 17:44 animals.txt
$ g Nutshell animals.txt                        *Runs "grep Nutshell animals.txt"*
horse   Linux in a Nutshell     2009    Siever, Ellen
donkey  Cisco IOS in a Nutshell 2005    Boney, James
```

###### 提示

始终将别名定义在单独的行上，而不是作为组合命令的一部分。（有关技术细节，请参阅 `man bash`。）

你可以定义一个与现有命令同名的别名，从而在你的 shell 中有效地替换该命令。这种做法称为*屏蔽*命令。假设你喜欢 `less` 命令用于阅读文件，但你希望它在显示每一页之前清除屏幕。这可以通过 `-c` 选项启用，因此定义一个名为 `less` 的别名，运行 `less -c`：^(2)

```
$ alias less="less -c"
```

别名优先于具有相同名称的命令，因此你现在在当前 shell 中已经屏蔽了 `less` 命令。我将在“搜索路径和别名”中解释*优先级*的含义。

要列出 shell 的别名及其值，请无参数运行 `alias`：

```
$ alias
alias g='grep'
alias ll='ls -l'
```

要查看单个别名的值，请运行 `alias`，然后跟随其名称：

```
$ alias g
alias g='grep'
```

要从 shell 中删除别名，请运行 `unalias`：

```
$ unalias g
```

# 重定向输入和输出

Shell 控制其运行的命令的输入和输出。你已经见过一个例子：管道，它将一个命令的 stdout 重定向到另一个命令的 stdin。管道语法 `|` 是 shell 的一个特性。

另一个 shell 特性是将 stdout 重定向到文件。例如，如果你使用 `grep` 从 Example 1-1 中的 *animals.txt* 文件中打印匹配行，则该命令默认将输出写入 stdout：

```
$ grep Perl animals.txt
alpaca	Intermediate Perl	2012	Schwartz, Randal
```

你可以使用称为*输出重定向*的 shell 功能将该输出发送到文件中。只需添加符号 `>`，然后是接收输出的文件名即可：

```
$ grep Perl animals.txt > outfile                      *(displays no output)*
$ cat outfile
alpaca	Intermediate Perl	2012	Schwartz, Randal
```

刚刚将 stdout 重定向到 *outfile* 文件而不是显示器。如果文件 *outfile* 不存在，则会创建它。如果存在，则重定向会覆盖其内容。如果你想要追加而不是覆盖输出文件，请使用 `>>` 符号：

```
$ grep Perl animals.txt > outfile              *Create or overwrite outfile*
$ echo There was just one match >> outfile     *Append to outfile*
$ cat outfile
alpaca	Intermediate Perl	2012	Schwartz, Randal
There was just one match
```

输出重定向还有一个伙伴，*输入重定向*，它将 stdin 重定向到来自文件而不是键盘。使用符号 `<`，然后是文件名来重定向 stdin。

许多 Linux 命令在没有参数运行时，接受文件名作为参数，并从这些文件中读取，同时也可以从 stdin 中读取。例如，用于统计文件中行数、单词数和字符数的 `wc` 命令就是一个例子：

```
$ wc animals.txt                            *Reading from a named file*
  7  51 325 animals.txt
$ wc < animals.txt                          *Reading from redirected stdin*
  7  51 325
```

理解这两个 `wc` 命令在行为上的差异*非常重要*：

+   在第一个命令中，`wc`接收到文件名*animals.txt*作为参数，所以`wc`知道文件的存在。`wc`会在磁盘上打开文件并读取其内容。

+   在第二个命令中，`wc`被调用时没有参数，所以它从标准输入读取数据，通常是键盘输入。然而，Shell 却巧妙地将标准输入重定向到*animals.txt*文件。`wc`并不知道文件*animals.txt*的存在。

Shell 可以在同一条命令中重定向输入和输出：

```
$ wc < animals.txt > count
$ cat count
  7  51 325
```

并且可以同时使用管道。在这里，`grep`从重定向的标准输入读取数据，并将结果通过管道传递给`wc`，后者将结果写入重定向的标准输出，生成文件*count*：

```
$ grep Perl < animals.txt | wc > count
$ cat count
      1       6      47
```

你将在第八章深入探讨这类组合命令，并在整本书中看到许多其他重定向的示例。

# 用引号和转义字符禁用评估

通常，Shell 使用空格作为单词之间的分隔符。以下命令包含四个单词——一个程序名称后面跟着三个参数：

```
$ ls file1 file2 file3
```

然而有时候，你需要 Shell 将空格视为显著的字符，而不是分隔符。一个常见的例子是文件名中的空格，比如*Efficient Linux Tips.txt*：

```
$ ls -l
-rw-r--r-- 1 smith smith 36 Aug  9 22:12 Efficient Linux Tips.txt
```

如果在命令行中引用此类文件名，你的命令可能会失败，因为 Shell 会将空格字符视为分隔符：

```
$ cat Efficient Linux Tips.txt
cat: Efficient: No such file or directory
cat: Linux: No such file or directory
cat: Tips.txt: No such file or directory
```

强制 Shell 将空格视为文件名的一部分，你有三种选择——单引号、双引号和反斜杠：

```
$ cat 'Efficient Linux Tips.txt'
$ cat "Efficient Linux Tips.txt"
$ cat Efficient\ Linux\ Tips.txt
```

单引号告诉 Shell 将字符串中的每个字符都视为文字，即使该字符通常对 Shell 具有特殊含义，例如空格和美元符号：

```
$ echo '$HOME'
$HOME
```

双引号告诉 Shell 将所有字符都视为文字，除了某些美元符号和其他几个你稍后会了解的字符：

```
$ echo "Notice that $HOME is evaluated"                  *Double quotes*
Notice that /home/smith is evaluated
$ echo 'Notice that $HOME is not'                        *Single quotes*
Notice that $HOME is not
```

反斜杠，也称为*转义字符*，告诉 Shell 按照字面意义处理下一个字符。以下命令包含了一个转义的美元符号：

```
$ echo \$HOME
$HOME
```

即使在双引号内，反斜杠也作为转义字符：

```
$ echo "The value of \$HOME is $HOME"
The value of $HOME is /home/smith
```

但不适用于单引号内：

```
$ echo 'The value of \$HOME is $HOME'
The value of \$HOME is $HOME
```

使用反斜杠转义双引号字符在双引号内部：

```
$ echo "This message is \"sort of\" interesting"
This message is "sort of" interesting
```

行尾的反斜杠可以禁用不可见的换行符的特殊性质，使 Shell 命令可以跨越多行：

```
$ echo "This is a very long message that needs to extend \
onto multiple lines"
This is a very long message that needs to extend onto multiple lines
```

最后的反斜杠非常适合使管道更易读，就像从“Command #6: uniq”中的这个一样：

```
$ cut -f1 grades \
  | sort \
  | uniq -c \
  | sort -nr \
  | head -n1 \
  | cut -c9
```

当以这种方式使用时，反斜杠有时被称为*行继续字符*。

别名前面的反斜杠会使别名无效，Shell 会查找同名命令，忽略任何遮蔽：

```
$ alias less="less -c"        *Define an alias*
$ less myfile                 *Run the alias, which invokes less -c*
$ \less myfile                *Run the standard less command, not the alias*
```

# 查找要运行的程序

当 Shell 首次遇到一个简单命令，比如`ls *.py`时，它只是一串毫无意义的字符。Shell 立即将字符串分成两个单词，“ls”和“*.py”。在这种情况下，第一个单词是磁盘上的程序名称，Shell 必须找到该程序才能运行它。

程序 `ls` 实际上是目录 */bin* 中的可执行文件。您可以使用此命令验证其位置：

```
$ ls -l /bin/ls
-rwxr-xr-x 1 root root 133792 Jan 18  2018 /bin/ls
```

或者您可以使用 `cd /bin` 更改目录并运行这个可爱的，看起来神秘的命令：

```
$ ls ls
ls
```

使用 `ls` 命令列出可执行文件 *ls*。

shell 如何在 */bin* 目录中定位 `ls`？在幕后，shell 会咨询一个预先安排好的目录列表，称为*搜索路径*，这个列表存储为 shell 变量 `PATH` 的值：

```
$ echo $PATH
/home/smith/bin:/usr/local/bin:/usr/bin:/bin:/usr/games:/usr/lib/java/bin
```

搜索路径中的目录由冒号 (`:`) 分隔。为了更清晰的视角，通过管道将输出传输到 `tr` 命令，将冒号转换为换行符，`tr` 命令能够将一个字符翻译成另一个字符（更多细节参见第五章）：

```
$ echo $PATH | tr : "\n"
/home/smith/bin
/usr/local/bin
/usr/bin
/bin
/usr/games
/usr/lib/java/bin
```

当 shell 定位像 `ls` 这样的程序时，它按顺序从搜索路径中的目录进行查询。“*/home/smith/bin/ls* 存在吗？不。*/usr/local/bin/ls* 存在吗？也不。那 */usr/bin/ls* 呢？还是不。或许 */bin/ls* 呢？是的，找到了！我将运行 */bin/ls*。” 这个搜索过程非常迅速，几乎察觉不到。^(3)

要在搜索路径中定位程序，请使用 `which` 命令：

```
$ which cp
/bin/cp
$ which which
/usr/bin/which
```

或者更强大（和冗长的）`type` 命令，这是一个 shell 内建命令，也可以定位别名，函数和 shell 内建命令：^(4)

```
$ type cp
cp is hashed (/bin/cp)
$ type ll
ll is aliased to ‘/bin/ls -l’
$ type type
type is a shell builtin
```

您的搜索路径可能在不同目录中包含相同命名的命令，例如 */usr/bin/less* 和 */bin/less*。Shell 将运行出现在路径中较早目录的命令。通过利用这种行为，您可以在搜索路径中较早的目录（例如个人 *$HOME/bin* 目录）中放置相同命名的命令来覆盖 Linux 命令。

# 搜索路径和别名

当 shell 按名称搜索命令时，在检查搜索路径之前会先检查该名称是否为别名。这就是为什么别名可以覆盖同名命令的原因。

搜索路径是一个很好的例子，展示了 Linux 中某些神秘事物其实有普通的解释。Shell 不会凭空提取命令或通过魔法定位它们。它会有条不紊地检查列表中的目录，直到找到所请求的可执行文件。

# 环境和初始化文件，简要版

运行中的 shell 中保存了许多重要信息在变量中：搜索路径，当前目录，首选文本编辑器，定制的 shell 提示符等等。运行中的 shell 中的变量总称为 shell 的*环境*。当 shell 退出时，它的环境被销毁。

逐个手动定义每个 shell 的环境将非常乏味。解决方案是在称为*启动文件*和*初始化文件*的 shell 脚本中定义环境，并让每个 shell 在启动时执行这些脚本。效果是某些信息似乎“全局”或“已知”于您的所有运行中的 shell。

我将深入讲解“配置你的环境”的细节。现在，我将教你一个初始化文件，让你能够顺利通过接下来的几章。它位于你的主目录中，名为 *.bashrc*（读作“点 bash R C”）。因为它的名字以点开头，`ls` 默认不会列出它：

```
$ ls $HOME
apple   banana   carrot
$ ls -a $HOME
.bashrc   apple   banana    carrot
```

如果 *$HOME/.bashrc* 不存在，请使用文本编辑器创建它。你放置在这个文件中的命令将在 shell 启动时自动执行，^(5) 因此这是定义 shell 环境变量和其他对 shell 重要的事物（如别名）的好地方。这是一个示例 *.bashrc* 文件。以 `#` 开头的行是注释：

```
# Set the search path
PATH=$HOME/bin:/usr/local/bin:/usr/bin:/bin
# Set the shell prompt
PS1='$ '
# Set your preferred text editor
EDITOR=emacs
# Start in my work directory
cd $HOME/Work/Projects
# Define an alias
alias g=grep
# Offer a hearty greeting
echo "Welcome to Linux, friend!"
```

你对 *$HOME/.bashrc* 的任何更改都不会影响任何正在运行的 shell，只会影响未来的 shell。你可以使用以下任何一个命令强制运行中的 shell 重新读取和执行 *$HOME/.bashrc*：

```
$ source $HOME/.bashrc                 *Uses the builtin "source" command*
$ . $HOME/.bashrc                      *Uses a dot*
```

这个过程称为“源化”初始化文件。如果有人告诉你“源化你的点-bash-R-C 文件”，他们的意思是运行上述命令之一。

###### 警告

在现实生活中，不要将所有 shell 配置放在 *$HOME/.bashrc* 中。一旦你阅读了“配置你的环境”的详细信息，请检查你的 *$HOME/.bashrc* 并根据需要将命令移到适当的文件中。

# 总结

我只涵盖了一小部分 `bash` 的功能及其最基本的用法。在接下来的章节中，特别是第六章，你会看到更多。现在，你最重要的任务是理解以下概念：

+   shell 存在并且承担着重要责任。

+   shell 在运行任何命令之前评估命令行。

+   命令可以重定向标准输入、标准输出和标准错误。

+   引用和转义可以防止特殊的 shell 字符被评估。

+   shell 使用目录搜索路径来定位程序。

+   通过在文件 *$HOME/.bashrc* 中添加命令，你可以更改 shell 的默认行为。

你越了解 shell 与它调用的程序之间的分隔，命令行就会更加合理，你按下 Enter 运行命令前能预测发生的情况也越好。

^(1) 这就是为什么命令 `ls *` 不会列出以点开头的文件名（即点文件）。

^(2) `bash` 通过不将第二个 `less` 扩展为别名来防止无限递归。

^(3) 一些 shell 会记住（缓存）程序的路径，这样可以减少未来的搜索。

^(4) 请注意，命令 `type which` 会产生输出，但命令 `which type` 不会。

^(5) 这个声明过于简化；更多细节见表 6-1。
