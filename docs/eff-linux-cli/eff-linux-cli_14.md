# 第十一章：最终节省时间

写这本书非常有趣，希望你们读起来也很愉快。在最后一个章节中，让我们涵盖一些前几章中没有完全适合的小主题。这些主题让我成为一个更好的 Linux 用户，也许它们也会帮助到你。

# 快速胜利

以下时间节省方法，几分钟内就能轻松掌握。

## 从`less`跳转到您的编辑器

当你使用`less`查看文本文件并想要编辑文件时，不要退出`less`。只需按`v`键启动你喜欢的文本编辑器。它会加载文件并将光标放在你在`less`中查看的位置。退出编辑器后，你将回到原来在`less`中的位置。

要使此技巧发挥最佳效果，请将环境变量`EDITOR`和/或`VISUAL`设置为编辑命令。这些环境变量代表你的默认 Linux 文本编辑器，可以通过各种命令启动，包括`less`、`lynx`、`git`、`crontab`和众多电子邮件程序。例如，要将`emacs`设置为默认编辑器，请在 shell 配置文件中添加以下任一行（或两行），然后执行：

```
VISUAL=emacs
EDITOR=emacs
```

如果你没有设置这些变量，你的默认编辑器将是你的 Linux 系统通常设置的`vim`。如果你进入`vim`而不知道如何使用它，请不要惊慌。按下 Escape 键，输入`:q!`（冒号、字母*q*和感叹号），然后按 Enter 键退出`vim`。要退出`emacs`，按 Ctrl-X，然后按 Ctrl-C。

## 编辑包含特定字符串的文件

想要编辑当前目录中包含特定字符串（或正则表达式）的每个文件？使用`grep -l`生成文件名列表，并通过命令替换将它们传递给你的编辑器。假设你的编辑器是`vim`，命令如下：

```
$ vim $(grep -l *string* *) 
```

通过将`-r`选项（递归）添加到`grep`并从当前目录（点）开始，编辑所有包含*string*的文件：

```
$ vim $(grep -lr *string* .) 
```

对于大型目录树的快速搜索，使用`find`和`xargs`而不是`grep -r`：

```
$ vim $(find . -type f -print0 | xargs -0 grep -l *string*) 
```

“技巧＃3：命令替换”提到了这种技术，但我想强调一下，因为它非常有用。记得要注意文件名中包含空格和其他特殊字符，因为它们可能会影响到结果，就像“特殊字符和命令替换”中解释的那样。

## 接受拼写错误

如果你经常拼错命令，请为你最常见的错误定义别名，以便正确的命令仍然运行：

```
alias firfox=firefox
alias les=less
alias meacs=emacs
```

注意不要意外地通过定义具有相同名称的别名来覆盖现有的 Linux 命令。首先使用`which`或`type`命令搜索你提议的别名，然后运行`man`命令，确保没有其他同名的命令：

```
$ type firfox
bash: type: firfox: not found
$ man firfox
No manual entry for firfox
```

## 快速创建空文件

在 Linux 中有几种创建空文件的方法。`touch`命令用于更新文件的时间戳，如果文件不存在，则也会创建文件：

```
$ touch newfile1
```

`touch`非常适合用于创建大量空文件进行测试：

```
$ mkdir tmp                           *Create a directory*
$ cd tmp
$ touch file{0000..9999}.txt          *Create 10,000 files*
$ cd ..
$ rm -rf tmp                          *Remove the directory and files*
```

`echo`命令如果将其输出重定向到文件，则会创建一个空文件，但仅当提供`-n`选项时：

```
$ echo -n > newfile2
```

如果忘记了`-n`选项，则生成的文件包含一个字符，即换行符，因此不是空文件。

## 逐行处理文件

当您需要逐行处理文件时，将文件`cat`到`while read`循环中：

```
$ cat myfile | while read line; do
 *...do something here...*
done
```

例如，要计算文件每行的长度，例如*/etc/hosts*，将每行管道传递给`wc -c`：

```
$ cat /etc/hosts | while read line; do
  echo "$line" | wc -c
done
65
31
1
⋮
```

此技术的一个更实际的示例见示例 9-3。

## 辨识支持递归的命令

在“find 命令”中，我介绍了`find -exec`，它可以递归地对整个目录树应用任何 Linux 命令：

```
$ find . -exec *your command here* \;
```

还有其他一些命令本身支持递归，如果您知道它们，可以节省时间，而不是构建一个`find`命令来实现递归。

`ls -R`

递归列出目录及其内容

`cp -r`或`cp -a`

递归复制目录及其内容

`rm -r`

递归删除目录及其内容

`grep -r`

通过正则表达式在整个目录树中搜索

`chmod -R`

递归更改文件保护

`chown -R`

递归更改文件所有权

`chgrp -R`

递归更改文件组所有权

## 阅读一个手册页

选择一个常用命令，如`cut`或`grep`，并彻底阅读其手册页。您可能会发现一两个从未使用过但很有价值的选项。定期重复此活动可完善并扩展您的 Linux 工具箱。

# 长期学习

下面的技术需要真正的努力学习，但您将在节省时间方面得到回报。我提供了每个主题的一点点味道，不是为了教授详细内容，而是为了激励您自己去发现更多。

## 阅读 bash 手册页

运行`man bash`以显示`bash`的完整官方文档，并阅读所有内容——是的，全部 46318 字：

```
$ man bash | wc -w
46318
```

花几天时间，慢慢来。您肯定会学到很多，使您的日常 Linux 使用更加轻松。

## 学习 cron、crontab 和 at

在“第一个示例：查找文件”中，有一段简短的关于如何安排命令在未来定期自动运行的说明。我建议学习`crontab`程序为自己设置定期命令。例如，您可以按计划备份文件到外部驱动器，或通过电子邮件为月度事件发送提醒。

在运行`crontab`之前，请按照“从 less 跳转到编辑器”中所示的方式定义您的默认编辑器。然后运行`crontab -e`来编辑您的个人定时命令文件。`crontab`会启动您的默认编辑器并打开一个空文件来指定命令。该文件称为您的*crontab*。

简而言之，在 crontab 文件中的计划命令，通常称为*cron 作业*，由六个字段组成，全部位于单个（可能很长的）行上。前五个字段确定作业的调度时间，依次为分钟、小时、每月的日期、月份和星期几。第六个字段是要运行的 Linux 命令。您可以按小时、每天、每周、每月、每年的某些特定日期或时间或其他更复杂的安排来启动命令。一些示例包括：

```
 * * * * * *command*             *Run command every minute*
30 7 * * * *command*             *Run command at 07:30 every day*
30 7 5 * * *command*             *Run command at 07:30 the 5th day of every month*
30 7 5 1 * *command*             *Run command at 07:30 every January 5*
30 7 * * 1 *command*             *Run command at 07:30 every Monday*

```

一旦创建了所有六个字段，保存了文件并退出了编辑器，该命令会根据您定义的时间表自动启动（由一个称为`cron`的程序执行）。计划的语法短而难懂，但在 man 页面（`man 5 crontab`）和许多在线教程（搜索*cron 教程*）中都有详细说明。

我还建议学习`at`命令，该命令可安排命令在指定的日期和时间运行一次，而不是重复运行。运行`man at`获取详细信息。以下是一个命令，它会在明天晚上 10 点向您发送一封电子邮件提醒刷牙：

```
$ at 22:00 tomorrow
warning: commands will be executed using /bin/sh
at> echo brush your teeth | mail $USER
at> ^D                                       *Type Ctrl-D to end input*
job 699 at Sun Nov 14 22:00:00 2021
```

要列出您待定的`at`作业，请运行`atq`：

```
$ atq
699     Sun Nov 14 22:00:00 20211 a smith
```

要查看`at`作业中的命令，请使用作业号运行`at -c`，并打印最后几行：

```
$ at -c 699 | tail
⋮
echo brush your teeth | mail $USER
```

在执行之前移除待定作业，请使用作业号运行`atrm`：

```
$ atrm 699
```

## 学习 rsync

要从一个磁盘位置复制完整目录及其子目录到另一个位置，许多 Linux 用户会使用命令`cp -r`或`cp -a`：

```
$ cp -a dir1 dir2
```

`cp`第一次可以完成工作，但如果稍后在目录*dir1*中修改了几个文件并再次执行复制，`cp`会浪费资源。它会忠实地再次复制*dir1*中的所有文件和目录，即使在*dir2*中已经存在完全相同的副本。

命令`rsync`是一个更智能的复制程序。它只复制第一个和第二个目录之间的*差异*。

```
$ rsync -a dir1/ dir2
```

###### 注意

前述命令中的斜杠表示复制*dir1*内的文件。如果没有斜杠，`rsync`会复制*dir1*本身，从而创建*dir2/dir1*。

如果稍后向目录*dir1*添加一个文件，`rsync`只会复制那一个文件。如果在*dir1*中的文件内修改一行，`rsync`只会复制那一行！在多次复制大型目录树时，这可以节省大量时间。`rsync`甚至可以通过 SSH 连接复制到远程服务器。

`rsync`有几十个选项。以下是一些特别有用的选项：

`-v`（表示“详细模式”）

在文件被复制时打印文件名

`-n`

假装复制；结合`-v`以查看*将要*被复制的文件

`-x`

告诉`rsync`不要跨越文件系统边界

我强烈推荐熟悉`rsync`以进行更高效的复制。阅读 man 页并查看 Korbin Brown 的文章[“Rsync Examples in Linux”](https://oreil.ly/7gHCi)中的示例。

## 学习另一种脚本语言

Shell 脚本方便且功能强大，但也有一些严重的缺陷。例如，它们无法处理文件名中包含空白字符的情况。考虑这个试图删除文件的简短`bash`脚本：

```
#!/bin/bash
BOOKTITLE="Slow Inefficient Linux"
rm $BOOKTITLE					# Wrong! Don't do this!
```

第二行看起来是在删除一个名为*Slow Inefficient Linux*的文件，但实际上并不是。它尝试删除三个名为*Slow*、*Inefficient*和*Linux*的文件。在调用`rm`之前，shell 会展开变量`$BOOKTITLE`，其展开结果是由空白分隔的三个单词，就像你输入了以下内容一样：

```
rm Slow Efficient Linux
```

然后 shell 使用三个参数调用`rm`，结果可能是错误的文件被删除了。正确的删除命令应该用双引号括起`$BOOKTITLE`：

```
rm "$BOOKTITLE"
```

shell 会将其展开为：

```
rm "Slow Efficient Linux"
```

这种微妙且潜在破坏性的怪癖只是表明了 shell 脚本在严肃项目中的不适用性之一。因此，我建议学习第二种脚本语言，如 Perl、PHP、Python 或 Ruby。它们都能正确处理空白字符。它们都支持真实的数据结构。它们都拥有强大的字符串处理函数。它们都能轻松进行数学计算。其优势不胜枚举。

使用 shell 启动复杂命令和创建简单脚本，但对于更重要的任务，请转向另一种语言。尝试在线的许多语言教程之一。

## 用于非编程任务的 make

程序`make`会根据规则自动更新文件。它设计用于加快软件开发，但稍加努力，`make`也可以简化 Linux 生活的其他方面。

假设您有三个文件分别命名为*chapter1.txt*、*chapter2.txt*和*chapter3.txt*，分开处理。还有第四个文件*book.txt*，它是这三个章节文件的组合。每当章节发生变化时，您需要重新组合它们并更新*book.txt*，可能会使用如下命令：

```
$ cat chapter1.txt chapter2.txt chapter3.txt > book.txt
```

这种情况非常适合使用`make`。

+   一堆文件

+   规则涉及文件，即*book.txt*在任何章节文件更改时都需要更新

+   执行更新的命令

`make`通过读取一个配置文件（通常命名为*Makefile*），该文件中充满了规则和命令来操作。例如，以下*Makefile*规则说明*book.txt*依赖于三个章节文件：

```
book.txt:	chapter1.txt chapter2.txt chapter3.txt
```

如果规则的目标（在本例中为*book.txt*）比其任何依赖项（章节文件）都要旧，则`make`认为目标已过期。如果在规则后的行上提供了命令，`make`将运行该命令以更新目标：

```
book.txt:	chapter1.txt chapter2.txt chapter3.txt
		cat chapter1.txt chapter2.txt chapter3.txt > book.txt
```

要应用规则，只需运行命令`make`：

```
$ ls
Makefile  chapter1.txt  chapter2.txt  chapter3.txt
$ make
cat chapter1.txt chapter2.txt chapter3.txt > book.txt       *Executed by make*
$ ls
Makefile  book.txt  chapter1.txt  chapter2.txt  chapter3.txt
$ make
make: 'book.txt' is up to date.
$ vim chapter2.txt                                           *Update a chapter*
$ make
cat chapter1.txt chapter2.txt chapter3.txt > book.txt
```

`make`是为程序员开发的，但是通过一点学习，您可以将其用于非编程任务。每当您需要更新依赖其他文件的文件时，编写一个*Makefile*通常可以简化您的工作。

`make`帮助我编写和调试这本书。我用一种称为 AsciiDoc 的排版语言写作，并定期将章节转换为 HTML 以在浏览器中查看。以下是一个`make`规则，将任何 AsciiDoc 文件转换为 HTML 文件：

```
%.html:	%.asciidoc
	asciidoctor -o $@ $<
```

它的意思是：要创建一个扩展名为*.html*的文件（`%.html`），请查找扩展名为*.asciidoc*的相应文件（`%.asciidoc`）。如果 HTML 文件比 AsciiDoc 文件旧，通过在依赖文件（`$<`）上运行`asciidoctor`命令并将输出发送到目标 HTML 文件（`-o $@`）来重新生成 HTML 文件。有了这个略微神秘但简短的规则，我只需输入一个简单的`make`命令，就可以创建您现在正在阅读的章节的 HTML 版本。`make`启动`asciidoctor`来执行更新：

```
$ ls ch11* 
ch11.asciidoc
$ make ch11.html
asciidoctor -o ch11.html ch11.asciidoc
$ ls ch11* 
ch11.asciidoc  ch11.html
$ firefox ch11.html                              *View the HTML file*
```

对于小任务，学习`make`通常不到一个小时就可以掌握基本技能。这是值得的。有一个有用的指南在[makefiletutorial.com](https://makefiletutorial.com/)上。

## 将版本控制应用于日常文件

您是否曾经想过编辑一个文件，但又担心您的更改可能会弄乱它？也许您制作了一个备份副本进行保管，并编辑了原始副本，知道如果出错可以恢复备份：

```
$ cp myfile myfile.bak
```

这种解决方案不具有可扩展性。如果您有几十个甚至几百个文件以及数十个甚至数百个人在处理它们，会怎么样？像 Git 和 Subversion 这样的版本控制系统通常可以解决这一问题，方便地跟踪多个版本的文件。

Git 在维护软件源代码中非常广泛，但我建议无论是个人文件还是操作系统文件，都学习并使用它，因为您的更改很重要。《“随环境旅行”》建议使用版本控制来维护您的`bash`配置文件。

在写这本书时，我使用了 Git，这样可以尝试不同的呈现材料的方式。几乎不费力气，我创建并维护了书的三个不同版本：一个是迄今为止的完整手稿，一个只包含我提交给编辑审查的章节，另一个用于实验性工作，尝试新的想法。如果我不喜欢自己写的东西，一个简单的命令就可以恢复以前的版本。

教授 Git 超出了本书的范围，但这里有一些示例命令，展示基本的工作流程并激发您的兴趣。将当前目录（及其所有子目录）转换为 Git 存储库：

```
$ git init
```

编辑一些文件。之后，将更改的文件添加到一个不可见的“暂存区”，这一操作声明了您创建新版本的意图：

```
$ git add .
```

创建新版本时，请提供注释以描述您对文件的更改：

```
$ git commit -m"Changed X to Y"
```

查看您的版本历史：

```
$ git log
```

在此过程中还有更多内容，比如检索旧版本的文件和将版本保存（*推送*）到另一个服务器上。获取一个[`git`教程](https://oreil.ly/0AlOu)，然后开始吧！

# 再见

非常感谢你通过这本书与我同行。我希望它能实现我在前言中对提升你的 Linux 命令行技能的承诺。请告诉我你在 dbarrett@oreilly.com 的体验。祝你计算愉快。
