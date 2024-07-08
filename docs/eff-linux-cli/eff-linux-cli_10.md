# 第八章：构建大胆一行命令

还记得前言中那个复杂的命令吗？

```
$ paste <(echo {1..10}.jpg | sed 's/ /\n/g') \
        <(echo {0..9}.jpg | sed 's/ /\n/g') \
  | sed 's/^/mv /' \
  | bash
```

这样的魔法咒语被称为*大胆一行命令*。^(1) 我们来拆解一下这个命令，理解它的作用和原理。最内层的`echo`命令使用大括号展开来生成 JPEG 文件名列表：

```
$ echo {1..10}.jpg
1.jpg 2.jpg 3.jpg ... 10.jpg
$ echo {0..9}.jpg
0.jpg 1.jpg 2.jpg ... 9.jpg
```

将文件名传输到`sed`中，将空格字符替换为换行符：

```
$ echo {1..10}.jpg | sed 's/ /\n/g'
1.jpg
2.jpg
⋮
10.jpg
$ echo {0..9}.jpg | sed 's/ /\n/g'
0.jpg
1.jpg
⋮
9.jpg
```

`paste`命令会将两个列表并排打印出来。过程替换允许`paste`像它们是文件一样读取这两个列表：

```
$ paste <(echo {1..10}.jpg | sed 's/ /\n/g') \
        <(echo {0..9}.jpg | sed 's/ /\n/g')
1.jpg   0.jpg
2.jpg   1.jpg
⋮
10.jpg  9.jpg
```

在每一行前加上`mv`，可以打印一系列字符串，这些字符串是`mv`命令：

```
$ paste <(echo {1..10}.jpg | sed 's/ /\n/g') \
        <(echo {0..9}.jpg | sed 's/ /\n/g') \
  | sed 's/^/mv /'
mv 1.jpg   0.jpg
mv 2.jpg   1.jpg
⋮
mv 10.jpg  9.jpg
```

现在命令的目的已经显露出来：它生成 10 个命令来重命名图像文件*1.jpg*至*10.jpg*。新名称分别是*0.jpg*至*9.jpg*。将输出传输到`bash`执行这些`mv`命令：

```
$ paste <(echo {1..10}.jpg | sed 's/ /\n/g') \
        <(echo {0..9}.jpg | sed 's/ /\n/g') \
  | sed 's/^/mv /' \
  | bash
```

大胆的一行命令就像是解谜题。面对一个业务问题，比如重命名一组文件，你可以运用你的工具箱构建一个 Linux 命令来解决它。大胆的一行命令挑战你的创造力并增强你的技能。

在这一章中，你将逐步创建像前述那样的大胆一行命令，使用以下的神奇公式：

1.  发明一个能解决难题的命令。

1.  运行命令并检查输出。

1.  回想一下历史命令并调整它。

1.  重复步骤 2 和 3，直到命令产生期望的结果。

本章将让你的大脑得到锻炼。有时候，你可能会被示例搞得一头雾水。只需一步步来，边读边在计算机上运行这些命令。

###### 注意

本章中的一些大胆一行命令太长，无法容纳在一行内，所以我用反斜杠将它们分成多行。然而，我们不称它们为大胆两行（或大胆七行）。

# 准备好变得大胆

在你开始创建大胆一行命令之前，花点时间调整好心态：

+   要灵活。

+   想想从哪里开始。

+   熟悉你的测试工具。

我会依次讨论每一个想法。

## 要灵活。

撰写大胆的一行命令的关键是*灵活性*。到现在为止，你学会了一些强大的工具——一套核心的 Linux 程序（以及运行它们的无数方法），还有命令历史记录、命令行编辑等等。你可以以多种方式结合这些工具，而每个问题通常都有多个解决方案。

即使是最简单的 Linux 任务也有多种完成方式。想想你会如何列出当前目录中的*.jpg*文件。我打赌 99.9%的 Linux 用户会运行像这样的命令：

```
$ ls *.jpg
```

但这只是众多解决方案中的一个例子。例如，你可以列出目录中的*所有*文件，然后使用`grep`仅匹配以*.jpg*结尾的文件名：

```
$ ls | grep '\.jpg$'
```

为什么选择这种解决方案？嗯，您在“长参数列表”中看到了一个例子，当目录包含太多文件时，无法通过模式匹配列出它们。*按文件扩展名进行 grep 搜索* 的技术是解决各种问题的强大、通用方法。重要的是要灵活并理解您的工具，以便在需要时应用最佳工具。这是创建大胆一行命令时的巫术技能。

以下所有命令列出当前目录中的*.jpg*文件。试着弄清楚每个命令的工作原理：

```
$ echo $(ls *.jpg)
$ bash -c 'ls *.jpg'
$ cat <(ls *.jpg)
$ find . -maxdepth 1 -type f -name \*.jpg -print
$ ls > tmp && grep '\.jpg$' tmp && rm -f tmp
$ paste <(echo ls) <(echo \*.jpg) | bash
$ bash -c 'exec $(paste <(echo ls) <(echo \*.jpg))'
$ echo 'monkey *.jpg' | sed 's/monkey/ls/' | bash
$ python -c 'import os; os.system("ls *.jpg")'
```

结果是否相同，还是某些命令的行为有所不同？您能否想出其他合适的命令？

## 思考从哪里开始

每个大胆的一行命令都以一个简单命令的输出开始。该输出可能是文件的内容、文件的一部分、目录列表、一系列数字或字母、用户列表、日期和时间或其他数据。因此，您的第一个挑战是生成命令的初始数据。

例如，如果您想知道英语字母表的第 17 个字母，那么您的初始数据可以是通过大括号扩展产生的 26 个字母：

```
$ echo {A..Z}
A B C D E F G H I J K L M N O P Q R S T U V W X Y Z
```

一旦您能够生成此输出，下一步就是决定如何进行处理以使其符合您的目标。您是否需要按行或列切片输出？将输出与其他信息连接？以更复杂的方式转换输出？可以查看第一章和第五章中的程序，如`grep`、`sed`和`cut`，并使用第七章中的技术应用它们。

对于这个例子，您可以使用`awk`打印第 17 个字段，或者使用`sed`删除空格并使用`cut`定位第 17 个字符：

```
$ echo {A..Z} | awk '{print $(17)}'
Q
$ echo {A..Z} | sed 's/ //g' | cut -c17
Q
```

作为另一个例子，如果您想打印一年中的月份，您的初始数据可以再次通过大括号扩展生成数字 1 到 12：

```
$ echo {1..12}
1 2 3 4 5 6 7 8 9 10 11 12
```

从这里开始，扩展大括号以形成每个月第一天的日期（从`2021-01-01`到`2021-12-01`）；然后对每行运行`date -d`以生成月份名称：

```
$ echo 2021-{01..12}-01 | xargs -n1 date +%B -d
January
February
March
⋮
December
```

或者，假设您想知道当前目录中最长文件名的长度。您的初始数据可以是目录列表：

```
$ ls
animals.txt  cartoon-mascots.txt  ...  zebra-stripes.txt
```

从这里开始，使用`awk`生成命令以计算每个文件名中字符的数量，并使用`wc -c`：

```
$ ls | awk '{print "echo -n", $0, "| wc -c"}'
echo -n "animals.txt" | wc -c
echo -n "cartoon-mascots.txt | wc -c"
⋮
echo -n "zebra-stripes.txt | wc -c"
```

（`-n`选项可以防止`echo`打印换行字符，这会使每个计数多一个。）最后，将命令管道传输给`bash`运行，将数字结果从高到低排序，并使用`head -n1`获取最大值（第一行）：

```
$ ls | awk '{print "echo -n", $0, "| wc -c"}' | bash | sort -nr | head -n1
23
```

最后一个例子有些棘手，将管道生成为字符串并将其传递给进一步的管道。尽管如此，一般原则是相同的：找出您的起始数据并将其操纵以满足您的需求。

## 了解您的测试工具

构建一个大胆的一行代码可能需要反复试验。以下工具和技术将帮助您快速尝试不同的解决方案：

使用命令历史记录和命令行编辑。

在您尝试实验时，请不要重新输入命令。使用第三章中的技术来回忆先前的命令，调整它们并运行它们。

添加`echo`来测试您的表达式。

如果您不确定表达式将如何评估，请事先用`echo`打印它以查看 stdout 上的评估结果。

使用`ls`或添加`echo`来测试具有破坏性的命令。

如果您的命令调用`rm`、`mv`、`cp`或其他可能覆盖或移除文件的命令，请在它们前面加上`echo`以确认哪些文件将受到影响。（因此，不要执行`rm`，而是执行`echo rm`。）另一个安全策略是用`ls`替换`rm`以列出将被移除的文件。

插入一个`tee`来查看中间结果。

如果您想在长管道中间查看输出（stdout），插入`tee`命令将输出保存到文件以供检查。以下命令将`command3`的输出保存在文件*outfile*中，同时将相同的输出传递给`command4`：

```
$ *command1* | *command2* | *command3* | tee outfile | *command4* | *command5*
$ less outfile
```

好的，让我们来构建一些大胆的一行代码吧！

# 将文件名插入序列

这个大胆的一行代码与打开本章节的那个（重命名*.jpg*文件）相似，但更加详细。这也是我在写这本书时真实遇到的情况。像前一个一行代码一样，它结合了第七章中的两种技术：进程替换和管道到`bash`。结果是一个可重复使用的模式，用于解决类似问题。

我在一台使用名为[AsciiDoc](https://asciidoc.org)的排版语言的 Linux 计算机上写了这本书。这里并不重要语言的细节；重要的是每一章都是一个单独的文件，最初有 10 章：

```
$ ls
ch01.asciidoc  ch03.asciidoc  ch05.asciidoc  ch07.asciidoc  ch09.asciidoc
ch02.asciidoc  ch04.asciidoc  ch06.asciidoc  ch08.asciidoc  ch10.asciidoc
```

在某个时候，我决定在第二章和第三章之间插入第十一章。这意味着需要重命名一些文件。第 3 至第十章必须变为第 4 至第十一章，留下一个空隙，以便我可以创建一个新的第三章（*ch03.asciidoc*）。我本可以手动重命名文件，从*ch11.asciidoc*开始向后工作：^(2)

```
$ mv ch10.asciidoc ch11.asciidoc
$ mv ch09.asciidoc ch10.asciidoc
$ mv ch08.asciidoc ch09.asciidoc
⋮
$ mv ch03.asciidoc ch04.asciidoc
```

但这种方法很繁琐（想象一下如果有 1000 个文件而不是 11 个！），所以我生成了必要的`mv`命令并将它们传递给`bash`。好好看看前面的`mv`命令，并思考一下您可能如何创建它们。

首先专注于原始文件名*ch03.asciidoc*到*ch10.asciidoc*。你可以使用花括号扩展打印它们，比如`ch{10..03}.asciidoc`，就像本章节的第一个例子一样，但为了练习一些灵活性，使用`seq -w`命令来打印数字：

```
$ seq -w 10 -1 3
10
09
08
⋮
03
```

然后通过管道将这个数字序列转换为文件名到`sed`：

```
$ seq -w 10 -1 3 | sed 's/\(.*\)/ch\1.asciidoc/'
ch10.asciidoc
ch09.asciidoc
⋮
ch03.asciidoc
```

现在您有一个原始文件名列表。同样地，为第 4 至 11 章创建目标文件名：

```
$ seq -w 11 -1 4 | sed 's/\(.*\)/ch\1.asciidoc/'
ch11.asciidoc
ch10.asciidoc
⋮
ch04.asciidoc
```

要形成`mv`命令，您需要将原始文件名和新文件名并排打印出来。本章第一个示例使用`paste`解决了“并排打印”的问题，并使用进程替换将两个打印列表视为文件。在这里也要做同样的操作：

```
$ paste <(seq -w 10 -1 3 | sed 's/\(.*\)/ch\1.asciidoc/') \
        <(seq -w 11 -1 4 | sed 's/\(.*\)/ch\1.asciidoc/')
ch10.asciidoc   ch11.asciidoc
ch09.asciidoc   ch10.asciidoc
⋮
ch03.asciidoc   ch04.asciidoc
```

###### 提示

上述命令可能看起来很长，但是通过命令历史和 Emacs 风格的命令行编辑，其实并不复杂。要从单一的“`seq`和`sed`”行转到`paste`命令：

1.  使用向上箭头从历史记录中调用前一个命令。

1.  按下 Ctrl-A 然后 Ctrl-K 来剪切整行。

1.  输入单词`paste`，然后加上一个空格。

1.  按两次 Ctrl-Y 来创建`seq`和`sed`命令的两个副本。

1.  使用移动和编辑键来修改第二份副本。

1.  依此类推。

通过将输出导向`sed`来在每行前面添加`mv`，打印出您所需的`mv`命令：

```
$ paste <(seq -w 10 -1 3 | sed 's/\(.*\)/ch\1.asciidoc/') \
        <(seq -w 11 -1 4 | sed 's/\(.*\)/ch\1.asciidoc/') \
  | sed 's/^/mv /'
mv ch10.asciidoc    ch11.asciidoc
mv ch09.asciidoc    ch10.asciidoc
⋮
mv ch03.asciidoc    ch04.asciidoc
```

作为最后一步，将命令导向`bash`以执行：

```
$ paste <(seq -w 10 -1 3 | sed 's/\(.*\)/ch\1.asciidoc/') \
        <(seq -w 11 -1 4 | sed 's/\(.*\)/ch\1.asciidoc/') \
  | sed 's/^/mv /' \
  | bash
```

我在我的书中确实使用了这个解决方案。`mv`命令运行后，生成的文件是第 1、2 和 4-11 章，留下了一个新的第三章的空白：

```
$ ls ch*.asciidoc
ch01.asciidoc  ch04.asciidoc  ch06.asciidoc  ch08.asciidoc  ch10.asciidoc
ch02.asciidoc  ch05.asciidoc  ch07.asciidoc  ch09.asciidoc  ch11.asciidoc
```

我刚刚呈现的模式可以在各种情况下重复使用以运行一系列相关命令：

1.  在 stdout 上生成命令参数作为列表。

1.  使用`paste`和进程替换并排打印列表。

1.  使用`sed`将命令名称前置，通过替换行首字符（`^`）来添加程序名称和空格。

1.  将结果导向`bash`。

# 检查匹配的文件对

这个大胆的一行代码受到了 Mediawiki 的实际用例启发，这是驱动维基百科和成千上万其他维基站点的软件。Mediawiki 允许用户上传图片进行显示。大多数用户通过网页表单进行手动过程：点击“选择文件”以弹出文件对话框，浏览到图像文件并选择它，在表单中添加描述性注释，然后点击“上传”。维基管理员使用更自动化的方法：一个脚本读取整个目录并上传其图片。每个图像文件（例如*bald_eagle.jpg*）都与一个文本文件（*bald_eagle.txt*）配对，其中包含关于图像的描述性注释。

想象一下，你面对着一个充满数百个图像文件和文本文件的目录。你希望确认每个图像文件都有一个匹配的文本文件，反之亦然。这里是该目录的较小版本：

```
$ ls
bald_eagle.jpg  blue_jay.jpg  cardinal.txt  robin.jpg  wren.jpg
bald_eagle.txt  cardinal.jpg  oriole.txt    robin.txt  wren.txt
```

让我们开发两种不同的解决方案来识别任何不匹配的文件。对于第一个解决方案，创建两个列表，一个用于 JPEG 文件，一个用于文本文件，并使用`cut`去掉它们的文件扩展名*.txt*和*.jpg*：

```
$ ls *.jpg | cut -d. -f1
bald_eagle
blue_jay
cardinal
robin
wren
$ ls *.txt | cut -d. -f1
bald_eagle
cardinal
oriole
robin
wren
```

然后使用进程替换使用`diff`来比较列表：

```
$ diff <(ls *.jpg | cut -d. -f1) <(ls *.txt | cut -d. -f1)
2d1
< blue_jay
3a3
> oriole
```

您可以在这里停下来，因为输出表明第一个列表有一个额外的*blue_jay*（意味着*blue_jay.jpg*），第二个列表有一个额外的*oriole*（意味着*oriole.txt*）。尽管如此，让我们使结果更加精确。通过在每行开头 grep 字符`<`和`>`来消除不需要的行：

```
$ diff <(ls *.jpg | cut -d. -f1) <(ls *.txt | cut -d. -f1) \
  | grep '^[<>]'
< blue_jay
> oriole
```

然后使用`awk`根据文件名（`$2`）前面是`<`还是`>`来附加正确的文件扩展名：

```
$ diff <(ls *.jpg | cut -d. -f1) <(ls *.txt | cut -d. -f1) \
  | grep '^[<>]' \
  | awk '/^</{print $2 ".jpg"} /^>/{print $2 ".txt"}'
blue_jay.jpg
oriole.txt
```

现在你已经有了未匹配文件的列表。然而，这个解决方案存在一个微妙的 bug。假设当前目录包含文件名*yellow.canary.jpg*，其中有两个点。上述命令将产生错误的输出：

```
blue_jay.jpg
oriole.txt
yellow.jpg                       *This is wrong*
```

此问题发生是因为两个`cut`命令从第一个点开始而不是从最后一个点开始移除字符，所以*yellow.canary.jpg*被截断为*yellow*而不是*yellow.canary*。为了解决这个问题，用`sed`替换`cut`，从最后一个点到字符串末尾删除字符：

```
$ diff <(ls *.jpg | sed 's/\.[^.]*$//') \
       <(ls *.txt | sed 's/\.[^.]*$//') \
  | grep '^[<>]' \
  | awk '/</{print $2 ".jpg"} />/{print $2 ".txt"}'
blue_jay.txt
oriole.jpg
yellow.canary.txt

```

第一个解决方案现在完成了。第二个解决方案采用了不同的方法。不是将`diff`应用于两个列表，而是生成一个单独的列表并删除匹配的文件名对。首先使用`sed`（使用与之前相同的 sed 脚本）去掉文件扩展名，并用`uniq -c`计算每个字符串的出现次数：

```
$ ls *.{jpg,txt} \
  | sed 's/\.[^.]*$//' \
  | uniq -c
      2 bald_eagle
      1 blue_jay
      2 cardinal
      1 oriole
      2 robin
      2 wren
      1 yellow.canary
```

输出的每一行包含数字`2`，表示匹配的文件名对，或者`1`，表示未匹配的文件名。使用`awk`来隔离以空格开头和`1`开头的行，并只打印第二个字段：

```
$ ls *.{jpg,txt} \
  | sed 's/\.[^.]*$//' \
  | uniq -c \
  | awk '/^ *1 /{print $2}'
blue_jay
oriole
yellow.canary
```

对于最后一步，如何添加丢失的文件扩展名？不要费心进行任何复杂的字符串操作。只需使用`ls`列出当前目录中的实际文件。用`awk`在每行输出的末尾添加一个星号（通配符）：

```
$ ls *.{jpg,txt} \
  | sed 's/\.[^.]*$//' \
  | uniq -c \
  | awk '/^ *1 /{print $2 "*"}'
blue_jay*
oriole*
yellow.canary*
```

并通过命令替换将这些行传递给`ls`。Shell 执行模式匹配，而`ls`列出未匹配的文件名。完成！

```
$ ls -1 $(ls *.{jpg,txt} \
  | sed 's/\.[^.]*$//' \
  | uniq -c \
  | awk '/^ *1 /{print $2 "*"}')
blue_jay.jpg
oriole.txt
yellow.canary.jpg
```

# 从你的主目录生成一个 CDPATH

在章节“Organize Your Home Directory for Fast Navigation”中，你手动编写了一个复杂的`CDPATH`行。它以`$HOME`开头，然后是所有`$HOME`的子目录，并以相对路径`..`（父目录）结束：

```
CDPATH=$HOME:$HOME/Work:$HOME/Family:$HOME/Finances:$HOME/Linux:$HOME/Music:..
```

让我们创建一个大胆的一行命令来自动生成`CDPATH`行，适合插入到`bash`配置文件中。从`$HOME`中的子目录列表开始，使用子 Shell 防止`cd`命令改变你的 Shell 当前目录：

```
$ (cd && ls -d */)
Family/  Finances/  Linux/  Music/  Work/
```

使用`sed`在每个目录前面添加`$HOME/`：

```
$ (cd && ls -d */) | sed 's/^/$HOME\//g'
$HOME/Family/
$HOME/Finances/
$HOME/Linux/
$HOME/Music/
$HOME/Work/
```

前面的`sed`脚本稍微复杂，因为替换字符串`$HOME/`包含一个斜杠，并且`sed`替换也使用斜杠作为分隔符。这就是为什么我的斜杠被转义：`$HOME\/`。为了简化，回想一下在“Substitution and Slashes”中提到，`sed`接受任何方便的字符作为分隔符。让我们使用`@`符号代替斜杠，这样就不需要转义了：

```
$ (cd && ls -d */) | sed 's@^@$HOME/@g'
$HOME/Family/
$HOME/Finances/
$HOME/Linux/
$HOME/Music/
$HOME/Work/
```

接下来，使用另一个`sed`表达式去掉最后的斜杠：

```
$ (cd && ls -d */) | sed -e 's@^@$HOME/@' -e 's@/$@@'
$HOME/Family
$HOME/Finances
$HOME/Linux
$HOME/Music
$HOME/Work
```

使用`echo`和命令替换将输出打印在单行上。请注意，你不再需要显式地在`cd`和`ls`周围使用普通括号创建子 shell，因为命令替换会创建自己的子 shell：

```
$ echo $(cd && ls -d */ | sed -e 's@^@$HOME/@' -e 's@/$@@')
$HOME/Family $HOME/Finances $HOME/Linux $HOME/Music $HOME/Work
```

添加第一个目录`$HOME`和最终相对目录`..`：

```
$ echo '$HOME' \
       $(cd && ls -d */ | sed -e 's@^@$HOME/@' -e 's@/$@@') \
       ..
$HOME $HOME/Family $HOME/Finances $HOME/Linux $HOME/Music $HOME/Work ..
```

通过将到目前为止的所有输出管道传输到`tr`来将空格更改为冒号：

```
$ echo '$HOME' \
       $(cd && ls -d */ | sed -e 's@^@$HOME/@' -e 's@/$@@') \
       .. \
  | tr ' ' ':'
$HOME:$HOME/Family:$HOME/Finances:$HOME/Linux:$HOME/Music:$HOME/Work:..
```

最后，添加`CDPATH`环境变量，你就生成了一个变量定义，可以粘贴到`bash`配置文件中。将此命令存储在一个脚本中，随时生成该行，比如当你向`$HOME`添加新子目录时：

```
$ echo 'CDPATH=$HOME' \
       $(cd && ls -d */ | sed -e 's@^@$HOME/@' -e 's@/$@@') \
       .. \
  | tr ' ' ':'
CDPATH=$HOME:$HOME/Family:$HOME/Finances:$HOME/Linux:$HOME/Music:$HOME/Work:..
```

# 生成测试文件

在软件行业中的常见任务是测试——向程序提供各种数据以验证程序的预期行为。下一个勇敢的一行生成包含随机文本的一千个文件，这些文件可用于软件测试。数字一千是任意的；你可以生成任意数量的文件。

该解决方案将随机从大型文本文件中选择单词，并创建一千个包含随机内容和长度的较小文件。一个完美的源文件是系统字典*/usr/share/dict/words*，其中包含 102,305 个单词，每个单词占一行。

```
$ wc -l /usr/share/dict/words
102305 /usr/share/dict/words
```

要生成这个勇敢的一行，你需要解决四个谜题：

1.  随机洗牌字典文件

1.  从字典文件中随机选择几行

1.  创建一个输出文件来保存结果

1.  运行你的解决方案一千次

要将字典随机打乱顺序，使用命令`shuf`，命名得当。每次运行命令`shuf /usr/share/dict/words`都会产生超过十万行的输出，因此使用`head`查看前几行随机行：

```
$ shuf /usr/share/dict/words | head -n3
evermore
shirttail
tertiary
$ shuf /usr/share/dict/words | head -n3
interactively
opt
perjurer
```

你的第一个谜题解决了。接下来，你如何从洗牌后的字典中选择随机数量的行？`shuf`有一个选项`-n`，可以打印给定数量的行，但你希望每次创建输出文件时该值都会变化。幸运的是，`bash`有一个变量`RANDOM`，它保存一个介于 0 和 32,767 之间的随机正整数。每次访问该变量时，它的值都会改变：

```
$ echo $RANDOM $RANDOM $RANDOM
7855 11134 262
```

因此，运行带有选项`-n $RANDOM`的`shuf`以打印随机数量的随机行。同样，完整输出可能会非常长，因此将结果管道传输到`wc -l`以确认每次执行时行数会改变：

```
$ shuf -n $RANDOM /usr/share/dict/words | wc -l
9922
$ shuf -n $RANDOM /usr/share/dict/words | wc -l
32465
```

你已经解决了第二个谜题。接下来，你需要一千个输出文件，或更具体地说，一千个不同的文件名。要生成文件名，请运行程序`pwgen`，它生成字母和数字的随机字符串：

```
$ pwgen
eng9nooG ier6YeVu AhZ7naeG Ap3quail poo2Ooj9 OYiuri9m iQuash0E voo3Eph1
IeQu7mi6 eipaC2ti exah8iNg oeGhahm8 airooJ8N eiZ7neez Dah8Vooj dixiV1fu
Xiejoti6 ieshei2K iX4isohk Ohm5gaol Ri9ah4eX Aiv1ahg3 Shaew3ko zohB4geu
⋮
```

添加选项`-N1`以生成一个字符串，并将字符串长度（10）作为参数指定：

```
$ pwgen -N1 10
ieb2ESheiw
```

可选择使用命令替换使字符串看起来更像文本文件的名称：

```
$ echo $(pwgen -N1 10).txt
ohTie8aifo.txt
```

第三个谜题完成了！现在你拥有生成单个随机文本文件的所有工具。使用`shuf`的`-o`选项将其输出保存在一个文件中：

```
$ mkdir -p /tmp/randomfiles && cd /tmp/randomfiles
$ shuf -n $RANDOM -o $(pwgen -N1 10).txt /usr/share/dict/words
```

并检查结果：

```
$ ls                           *List the new file*
Ahxiedie2f.txt
$ wc -l Ahxiedie2f.txt         *How many lines does it contain?*
13544 Ahxiedie2f.txt
$ head -n3 Ahxiedie2f.txt      *Peek at the first few lines*
saviors
guerillas
forecaster
```

看起来不错！最后一个谜题是如何一千次运行前面的`shuf`命令。您当然可以使用循环：

```
for i in {1..1000}; do
  shuf -n $RANDOM -o $(pwgen -N1 10).txt /usr/share/dict/words
done
```

但这并不像创建大胆一行一句那么有趣。相反，让我们预先生成命令作为字符串，并通过`bash`管道传递它们。作为测试，使用`echo`打印您想要的命令一次。添加单引号以确保`$RANDOM`不被评估，`pwgen`不运行：

```
$ echo 'shuf -n $RANDOM -o $(pwgen -N1 10).txt /usr/share/dict/words'
shuf -n $RANDOM -o $(pwgen -N1 10).txt /usr/share/dict/words
```

这个命令可以轻松地通过`bash`管道执行：

```
$ echo 'shuf -n $RANDOM -o $(pwgen -N1 10).txt /usr/share/dict/words' | bash
$ ls
eiFohpies1.txt
```

现在，使用`yes`命令通过管道传递给`head`打印一千次您的命令，然后将结果传递给`bash`，您已经解决了第四个谜题：

```
$ yes 'shuf -n $RANDOM -o $(pwgen -N1 10).txt /usr/share/dict/words' \
  | head -n 1000 \
  | bash
$ ls
Aen1lee0ir.txt  IeKaveixa6.txt  ahDee9lah2.txt paeR1Poh3d.txt
Ahxiedie2f.txt  Kas8ooJahK.txt  aoc0Yoohoh.txt sohl7Nohho.txt
CudieNgee4.txt  Oe5ophae8e.txt  haiV9mahNg.txt uchiek3Eew.txt
⋮
```

如果您更喜欢一千个随机图像文件而不是文本文件，可以使用相同的技术（`yes`、`head`和`bash`），并用生成随机图像的命令替换`shuf`。以下是我从[Stack Overflow 上 Mark Setchell 的解决方案](https://oreil.ly/ruDwG)中改编的大胆一行一句。它运行来自图形包 ImageMagick 的`convert`命令，以产生大小为 100 x 100 像素的由多彩方块组成的随机图像：

```
$ yes 'convert -size 8x8 xc: +noise Random -scale 100x100 $(pwgen -N1 10).png' \
  | head -n 1000 \
  | bash
$ ls
Bahdo4Yaop.png  Um8ju8gie5.png  aing1QuaiX.png  ohi4ziNuwo.png
Eem5leijae.png  Va7ohchiep.png  eiMoog1kou.png  ohnohwu4Ei.png
Eozaing1ie.png  Zaev4Quien.png  hiecima2Ye.png  quaepaiY9t.png
⋮
$ display Bahdo4Yaop.png              *View the first image*
```

# 生成空文件

有时候，进行测试所需的仅仅是具有不同名称的大量文件，即使它们是空的。生成命名为*file0001.txt*至*file1000.txt*的一千个空文件就像这样简单：

```
$ mkdir /tmp/empties           *Create a directory for the files*
$ cd /tmp/empties
$ touch file{01..1000}.txt     *Generate the files*
```

如果你喜欢更有趣的文件名，可以随机从系统字典中选择。使用`grep`限制名称为小写字母以简化（避免空格、撇号和其他对 shell 特殊的字符）：

```
$ grep '^[a-z]*$' /usr/share/dict/words
a
aardvark
aardvarks
⋮
```

使用`shuf`混洗名称并用`head`打印前一千个：

```
$ grep '^[a-z]*$' /usr/share/dict/words | shuf | head -n1000
triplicating
quadruplicates
podiatrists
⋮
```

最后，将结果通过`xargs`管道传递给`touch`以创建文件：

```
$ grep '^[a-z]*$' /usr/share/dict/words | shuf | head -n1000 | xargs touch
$ ls
abases             distinctly      magnolia         sadden
abets              distrusts       maintaining      sales
aboard             divided         malformation     salmon
⋮
```

# 概要

希望本章的例子有助于培养你编写大胆一行一句的技能。其中几个提供了可重复使用的模式，你可能会在其他情况下发现它们有用。

注意：在城里，大胆的一行一句并非唯一的解决方案。它们只是在命令行高效工作的一种方法。有时候，编写一个 shell 脚本可能会更有价值。其他时候，使用像 Perl 或 Python 这样的编程语言可能会找到更好的解决方案。然而，编写大胆的一行一句是执行关键任务的一项重要技能，快速而富有风格。

^(1) 我所知道的此术语最早使用（来自 BSD Unix 4.x 中的[lorder(1)的 manpage](https://oreil.ly/ro621)）。感谢 Bob Byrnes 找到它。

^(2) 从*ch03.asciidoc*开始并向前工作可能是危险的——你能看出为什么吗？如果不能，使用命令`touch ch{01..10}.asciidoc`创建这些文件并自行尝试。
