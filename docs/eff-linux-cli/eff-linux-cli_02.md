# 第一章：组合命令

当你在 Windows、macOS 和大多数其他操作系统中工作时，你可能会花费大量时间运行诸如网页浏览器、文字处理器、电子表格和游戏等应用程序。典型的应用程序包含大量功能：设计师认为用户需要的一切。因此，大多数应用程序是自给自足的。它们不依赖于其他应用程序。你可能偶尔在应用程序之间复制粘贴，但基本上它们是独立的。

Linux 命令行与众不同。与具有大量功能的大型应用程序不同，Linux 提供了数千个功能很少的小命令。例如，命令`cat`只是在屏幕上打印文件而已。`ls`列出目录中的文件，`mv`重命名文件，依此类推。每个命令都有一个简单而相当明确的目的。

如果你需要做一些更复杂的事情怎么办？别担心。Linux 可以很容易地*组合命令*，使它们各自的功能一起工作，以实现你的目标。这种工作方式会让你对计算机有非常不同的思维方式。不再是问“我应该启动哪个应用程序？”来实现某个结果，而是变成了“我应该组合哪些命令？”

在这一章中，你将学习如何以不同的组合方式安排和运行命令来完成你的需求。为了保持简单，我将介绍只有六个 Linux 命令及其最基本的用法，这样你可以专注于更复杂和有趣的部分——命令的组合，而无需经历陡峭的学习曲线。这有点像只用六种成分学习烹饪，或者只用锤子和锯子学习木工。（在第五章中，我会向你的 Linux 工具箱添加更多命令。）

你将使用*管道*来组合命令，这是 Linux 的一个功能，将一个命令的输出连接到另一个命令的输入。在我介绍每个命令（`wc`、`head`、`cut`、`grep`、`sort`和`uniq`）时，我将立即演示它们与管道的使用。有些示例对日常 Linux 使用很实用，而其他示例只是演示一个重要功能的玩具示例。

# 输入、输出和管道

大多数 Linux 命令从键盘读取输入，将输出写入屏幕，或者两者兼而有之。Linux 为这种读取和写入赋予了很多花哨的名字：

stdin（读作“标准输入”或“标准输入”）

Linux 从你的键盘读取的输入流。当你在提示符下输入任何命令时，你就是在 stdin 上提供数据。

stdout（读作“标准输出”或“标准输出”）

Linux 写入到你的显示器的输出流。当你运行`ls`命令来打印文件名时，结果就会显示在 stdout 上。

现在来看看有趣的部分。你可以将一个命令的 stdout 连接到另一个命令的 stdin，这样第一个命令就会向第二个命令输入数据。我们从熟悉的`ls -l`命令开始，以长格式列出一个大目录，比如 */bin*：

```
$ ls -l /bin
total 12104
-rwxr-xr-x 1 root root 1113504 Jun  6  2019 bash
-rwxr-xr-x 1 root root  170456 Sep 21  2019 bsd-csh
-rwxr-xr-x 1 root root   34888 Jul  4  2019 bunzip2
-rwxr-xr-x 1 root root 2062296 Sep 18  2020 busybox
-rwxr-xr-x 1 root root   34888 Jul  4  2019 bzcat
⋮
-rwxr-xr-x 1 root root    5047 Apr 27  2017 znew
```

这个目录包含的文件比你的显示器能显示的行数多得多，因此输出很快就会滚动到屏幕外。`ls` 无法一次打印信息，直到你按下键盘继续。但等等：另一个 Linux 命令有这个功能。`less` 命令以一页一页地显示文件：

```
$ less myfile                        *View the file; press q to quit*
```

你可以连接这两个命令，因为 `ls` 输出到 stdout，而 `less` 可以从 stdin 读取。使用管道将 `ls` 的输出发送到 `less` 的输入：

```
$ ls -l /bin | less
```

这个组合命令一次显示目录的内容一页一页。命令之间的竖线 (`|`) 是 Linux 的管道符号。^(1) 它连接第一个命令的 stdout 到下一个命令的 stdin。任何包含管道的命令行被称为 *管道*。

命令通常不知道它们是管道的一部分。`ls` 认为它在写入显示器，而实际上它的输出已被重定向到 `less`。而 `less` 则认为它从键盘读取输入，而实际上它正在读取 `ls` 的输出。

# 开始学习的六个命令

管道是 Linux 专家不可或缺的一部分。让我们通过一小组 Linux 命令来提升你的管道技能，这样无论你以后遇到哪些命令，你都能准备好将它们组合起来使用。

这六个命令——`wc`、`head`、`cut`、`grep`、`sort` 和 `uniq`——有许多选项和操作模式，我将大部分跳过，专注于管道。要了解任何命令的更多信息，请运行 `man` 命令以显示完整文档。例如：

```
$ man wc
```

为了演示我们的六个命令的作用，我将使用一个名为 *animals.txt* 的文件，其中列出了一些 O’Reilly 书籍信息，显示在 示例 1-1 中。

##### 示例 1-1\. *animals.txt* 文件内部

```
python	Programming Python	2010	Lutz, Mark
snail	SSH, The Secure Shell	2005	Barrett, Daniel
alpaca	Intermediate Perl	2012	Schwartz, Randal
robin	MySQL High Availability	2014	Bell, Charles
horse	Linux in a Nutshell	2009	Siever, Ellen
donkey	Cisco IOS in a Nutshell	2005	Boney, James
oryx	Writing Word Macros	1999	Roman, Steven
```

每行包含有关 O'Reilly 书籍的四个事实，由单个制表符分隔：封面上的动物、书名、出版年份和第一作者的姓名。

## 命令 #1: wc

`wc` 命令会打印文件中的行数、单词数和字符数：

```
$ wc animals.txt
  7  51 325 animals.txt
```

`wc` 报告文件 *animals.txt* 有 7 行，51 个单词和 325 个字符。如果你用眼睛数字符，包括空格和制表符，你会发现只有 318 个字符，但 `wc` 还包括每行结尾的不可见换行符。

选项 `-l`、`-w` 和 `-c` 指示 `wc` 只打印行数、单词数和字符数：

```
$ wc -l animals.txt
7 animals.txt
$ wc -w animals.txt
51 animals.txt
$ wc -c animals.txt
325 animals.txt
```

计数是一项非常有用的通用任务，`wc` 的作者设计了命令以处理管道。如果你省略文件名，它会从 stdin 读取，并将结果输出到 stdout。让我们使用 `ls` 列出当前目录的内容，并通过管道将其传递给 `wc` 来统计行数。这个管道回答了问题：“我的当前目录中有多少个文件可见？”

```
$ ls -1
animals.txt
myfile
myfile2
test.py
$ ls -1 | wc -l
4
```

选项 `-1` 告诉 `ls` 将其结果以单列方式打印，在这里并非严格必要。要了解为什么我使用它，请参阅边栏 “ls 在重定向时的行为更改”。

`wc` 是本章中你见过的第一个命令，所以你在管道中能做的事情有限。只是为了好玩，将 `wc` 的输出再次通过管道传递给 `wc`，展示同一个命令可以在管道中出现多次。这个组合命令报告说 `wc` 输出的单词数是四个：三个整数和一个文件名：

```
$ wc animals.txt
  7  51 325 animals.txt
$ wc animals.txt | wc -w
4
```

为什么要停在这里？在管道中添加第三个 `wc`，并计算输出的行数、单词数和字符数，结果是“4”：

```
$ wc animals.txt | wc -w | wc
      1       1       2
```

输出显示了一行（包含数字 4）、一个单词（数字 4 本身）和两个字符。为什么是两个？因为字符串 “4” 末尾有一个不可见的换行符。

对于 `wc` 的愚蠢管道已经足够了。随着你掌握更多命令，管道将变得更实用。

## 命令 #2: head

`head` 命令打印文件的前几行。使用 `-n` 选项，使用 `head` 打印 *animals.txt* 的前三行：

```
$ head -n3 animals.txt
python	Programming Python	2010	Lutz, Mark
snail	SSH, The Secure Shell	2005	Barrett, Daniel
alpaca	Intermediate Perl	2012	Schwartz, Randal
```

如果请求的行数超过文件包含的行数，`head` 将打印整个文件（就像 `cat` 命令一样）。如果省略 `-n` 选项，`head` 默认为 10 行（`-n10`）。

`head` 命令在你不关心文件的其余内容时，非常方便，可以快速而高效地查看文件顶部。即使是非常大的文件，也是如此，因为它无需读取整个文件。此外，`head` 将结果输出到 stdout，使其在管道中非常有用。统计 *animals.txt* 文件前三行的单词数：

```
$ head -n3 animals.txt | wc -w
20
```

`head` 还可以从 stdin 读取，用于更多的管道乐趣。一个常见的用法是在你不想看到全部输出时，从另一个命令中减少输出，比如长目录列表。例如，在 */bin* 目录中列出前五个文件名：

```
$ ls /bin | head -n5
bash
bsd-csh
bunzip2
busybox
bzcat
```

## 命令 #3: cut

`cut` 命令从文件中打印一个或多个列。例如，打印 *animals.txt* 中第二列中出现的所有书名：

```
$ cut -f2 animals.txt
Programming Python
SSH, The Secure Shell
Intermediate Perl
MySQL High Availability
Linux in a Nutshell
Cisco IOS in a Nutshell
Writing Word Macros
```

`cut` 提供两种定义“列”的方式。第一种是按字段（`-f`）分割，当输入由每个由单个制表符分隔的字符串（字段）组成时。方便的是，这正是 *animals.txt* 文件的格式。前面的 `cut` 命令通过 `-f2` 选项打印每行的第二个字段。

为了缩短输出，将其通过管道传递给 `head`，仅打印 *animals.txt* 的前三行：

```
$ cut -f2 animals.txt | head -n3
Programming Python
SSH, The Secure Shell
Intermediate Perl
```

你还可以通过用逗号分隔它们的字段号来剪切多个字段：

```
$ cut -f1,3 animals.txt | head -n3
python	2010
snail	2005
alpaca	2012
```

或者按数字范围：

```
$ cut -f2-4 animals.txt | head -n3
Programming Python	2010	Lutz, Mark
SSH, The Secure Shell	2005	Barrett, Daniel
Intermediate Perl	2012	Schwartz, Randal
```

第二种为 `cut` 定义“列”的方式是通过字符位置，使用 `-c` 选项。从文件的每行中打印前三个字符，可以使用逗号（`1,2,3`）或范围（`1-3`）来指定：

```
$ cut -c1-3 animals.txt
pyt
sna
alp
rob
hor
don
ory
```

现在您已经看到了基本功能，请尝试使用`cut`和管道来进行更实际的操作。想象一下*animals.txt*文件有数千行长，并且您需要提取作者的姓氏。首先，隔离第四个字段，作者名：

```
$ cut -f4 animals.txt
Lutz, Mark
Barrett, Daniel
Schwartz, Randal
⋮
```

然后再次将结果管道传输到`cut`，使用选项`-d`（意味着“分隔符”）将分隔字符更改为逗号而不是制表符，以隔离作者的姓氏：

```
$ cut -f4 animals.txt | cut -d, -f1
Lutz
Barrett
Schwartz
⋮
```

# 保存时间与命令历史和编辑

您正在重复输入许多命令吗？而不是重复输入，使用向上箭头键滚动前面运行过的命令。 （这个 shell 功能称为*命令历史*。）当您到达所需命令时，按 Enter 键立即运行它，或者使用左右箭头键定位光标和 Backspace 键删除以进行编辑。 （这个功能是*命令行编辑*。）

我将在第三章中讨论更强大的命令历史和编辑功能。

## 命令 #4: grep

`grep`是一个非常强大的命令，但目前我将隐藏其大部分功能，简单说它打印与给定字符串匹配的行。（更多详细信息请参见第五章。）例如，以下命令显示*animals.txt*中包含字符串`Nutshell`的行：

```
$ grep Nutshell animals.txt
horse	Linux in a Nutshell	2009	Siever, Ellen
donkey	Cisco IOS in a Nutshell	2005	Boney, James
```

您还可以使用`-v`选项打印不匹配给定字符串的行。请注意不包含“Nutshell”字符串的行：

```
$ grep -v Nutshell animals.txt
python	Programming Python	2010	Lutz, Mark
snail	SSH, The Secure Shell	2005	Barrett, Daniel
alpaca	Intermediate Perl	2012	Schwartz, Randal
robin	MySQL High Availability	2014	Bell, Charles
oryx	Writing Word Macros	1999	Roman, Steven
```

总的来说，`grep`对于在文件集合中查找文本非常有用。以下命令打印在以*.txt*结尾的文件中包含字符串`Perl`的行：

```
$ grep Perl *.txt
animals.txt:alpaca      Intermediate Perl       2012    Schwartz, Randal
essay.txt:really love the Perl programming language, which is
essay.txt:languages such as Perl, Python, PHP, and Ruby
```

在这种情况下，`grep`找到三行匹配的内容，一个在*animals.txt*中，两个在*essay.txt*中。

`grep`读取标准输入并写入标准输出，非常适合管道。假设您想知道大目录*/usr/lib*中有多少个子目录。没有单个 Linux 命令可以提供答案，因此构建一个管道。从`ls -l`命令开始：

```
$ ls -l /usr/lib
drwxrwxr-x  12 root root    4096 Mar  1  2020 4kstogram
drwxr-xr-x   3 root root    4096 Nov 30  2020 GraphicsMagick-1.4
drwxr-xr-x   4 root root    4096 Mar 19  2020 NetworkManager
-rw-r--r--   1 root root   35568 Dec  1  2017 attica_kde.so
-rwxr-xr-x   1 root root     684 May  5  2018 cnf-update-db
⋮
```

注意，`ls -l`在行首用`d`标记目录。使用`cut`来隔离第一列，可能是`d`也可能不是：

```
$ ls -l /usr/lib | cut -c1
d
d
d
-
-
⋮
```

然后使用`grep`仅保留包含`d`的行：

```
$ ls -l /usr/lib | cut -c1 | grep d
d
d
d
⋮
```

最后，用`wc`计算行数，您就得到了答案，由一个四条命令的管道产生——*/usr/lib*包含 145 个子目录：

```
$ ls -l /usr/lib | cut -c1 | grep d | wc -l
145
```

## 命令 #5: sort

`sort`命令将文件行按升序（默认）重新排序：

```
$ sort animals.txt
alpaca	Intermediate Perl	2012	Schwartz, Randal
donkey	Cisco IOS in a Nutshell	2005	Boney, James
horse	Linux in a Nutshell	2009	Siever, Ellen
oryx	Writing Word Macros	1999	Roman, Steven
python	Programming Python	2010	Lutz, Mark
robin	MySQL High Availability	2014	Bell, Charles
snail	SSH, The Secure Shell	2005	Barrett, Daniel
```

或按降序排列（使用`-r`选项）：

```
$ sort -r animals.txt
snail	SSH, The Secure Shell	2005	Barrett, Daniel
robin	MySQL High Availability	2014	Bell, Charles
python	Programming Python	2010	Lutz, Mark
oryx	Writing Word Macros	1999	Roman, Steven
horse	Linux in a Nutshell	2009	Siever, Ellen
donkey	Cisco IOS in a Nutshell	2005	Boney, James
alpaca	Intermediate Perl	2012	Schwartz, Randal
```

`sort`可以按字母顺序（默认）或数字顺序（使用`-n`选项）对行进行排序。我将演示使用管道切割*animals.txt*中第三个字段，即出版年份：

```
$ cut -f3 animals.txt                         *Unsorted*
2010
2005
2012
2014
2009
2005
1999
$ cut -f3 animals.txt | sort -n               *Ascending*
1999
2005
2005
2009
2010
2012
2014
$ cut -f3 animals.txt | sort -nr              *Descending*
2014
2012
2010
2009
2005
2005
1999
```

要了解*animals.txt*中最新一本书的年份，请将`sort`的输出通过管道传输到`head`的输入，并仅打印第一行：

```
$ cut -f3 animals.txt | sort -nr | head -n1
2014
```

# 最大值和最小值

`sort`和`head`在处理数值数据时是强大的搭档，每行一个值。您可以通过管道数据到以下命令来打印最大值：

```
... | sort -nr | head -n1
```

并打印出最小值：

```
... | sort -n | head -n1
```

作为另一个例子，让我们玩玩文件 */etc/passwd*，其中列出了可以在系统上运行进程的用户。^(4) 你将生成一个按字母顺序排列的所有用户列表。浏览前五行，你会看到像这样的内容：

```
$ head -n5 /etc/passwd
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
smith:x:1000:1000:Aisha Smith,,,:/home/smith:/bin/bash
jones:x:1001:1001:Bilbo Jones,,,:/home/jones:/bin/bash
```

每行由冒号分隔的字符串组成，第一个字符串是用户名，所以你可以用 `cut` 命令来隔离用户名：

```
$ head -n5 /etc/passwd | cut -d: -f1
root
daemon
bin
smith
jones
```

并对它们进行排序：

```
$ head -n5 /etc/passwd | cut -d: -f1 | sort
bin
daemon
jones
root
smith
```

要生成所有用户名按字母顺序排序的列表，而不只是前五个，请用 `head` 替换成 `cat`：

```
$ cat /etc/passwd | cut -d: -f1 | sort
```

要检测给定用户在您的系统上是否有账户，可以将他们的用户名与 `grep` 命令匹配。空输出意味着没有账户：

```
$ cut -d: -f1 /etc/passwd | grep -w jones
jones
$ cut -d: -f1 /etc/passwd | grep -w rutabaga         *(produces no output)*
```

`-w` 选项告诉 `grep` 只匹配完整的单词，而不是部分单词，以防您的系统也有包含“jones”的用户名，比如 `sallyjones2`。

## 命令 #6：uniq

`uniq` 命令用于检测文件中重复的相邻行。默认情况下，它会删除重复行。我将用一个包含大写字母的简单文件来演示这一点：

```
$ cat letters
A
A
A
B
B
A
C
C
C
C
$ uniq letters
A
B
A
C
```

注意，`uniq` 将前三行的 `A` 缩减为单个 `A`，但保留最后一个 `A`，因为它与前三个不是*相邻*的。

你也可以使用 `-c` 选项来计算出现次数：

```
$ uniq -c letters
      3 A
      2 B
      1 A
      4 C
```

我承认，当我第一次接触到 `uniq` 命令时，并没有看到它的多大用处，但它很快成为我最喜欢的命令之一。假设你有一个以制表符分隔的学生最终成绩文件，从 `A`（最好）到 `F`（最差）：

```
$ cat grades
C	Geraldine
B	Carmine
A	Kayla
A	Sophia
B	Haresh
C	Liam
B	Elijah
B	Emma
A	Olivia
D	Noah
F	Ava
```

你希望打印出现次数最多的等级。（如果有并列的情况，只打印其中一个获胜者。）从 `cut` 命令开始隔离等级并进行排序：

```
$ cut -f1 grades | sort
A
A
A
B
B
B
B
C
C
D
F
```

接下来，使用 `uniq` 命令来计算相邻行：

```
$ cut -f1 grades | sort | uniq -c
      3 A
      4 B
      2 C
      1 D
      1 F
```

然后以逆序的数值方式对行进行排序，将最频繁出现的等级移动到顶部行：

```
$ cut -f1 grades | sort | uniq -c | sort -nr
      4 B
      3 A
      2 C
      1 F
      1 D
```

并只保留第一行，使用 `head` 命令：

```
$ cut -f1 grades | sort | uniq -c | sort -nr | head -n1
      4 B
```

最后，因为你只想要字母等级，而不是数量，所以用 `cut` 命令来隔离等级：

```
$ cut -f1 grades | sort | uniq -c | sort -nr | head -n1 | cut -c9
B
```

至此，你的答案就出来了，多亏了一个六步骤的管道—我们迄今为止最长的管道。这种一步一步的管道构建不仅是一个教育性的练习。这是 Linux 专家实际工作的方式。第八章 就专门讨论了这种技术。

# 检测重复文件

让我们结合你所学到的内容，看一个更大的例子。假设你在一个充满 JPEG 文件的目录中，想知道是否有重复的文件：

```
$ ls
image001.jpg  image005.jpg  image009.jpg  image013.jpg  image017.jpg
image002.jpg  image006.jpg  image010.jpg  image014.jpg  image018.jpg
⋮
```

你可以用一个管道来回答这个问题。你需要另一个命令 `md5sum`，它检查文件的内容并计算一个称为*校验和*的 32 个字符的字符串：

```
$ md5sum image001.jpg
146b163929b6533f02e91bdf21cb9563  image001.jpg
```

由于数学原因，给定文件的校验和很可能是唯一的。如果两个文件具有相同的校验和，那么它们几乎肯定是重复的。在这里，`md5sum` 表示第一个和第三个文件是重复的：

```
$ md5sum image001.jpg image002.jpg image003.jpg
146b163929b6533f02e91bdf21cb9563  image001.jpg
63da88b3ddde0843c94269638dfa6958  image002.jpg
146b163929b6533f02e91bdf21cb9563  image003.jpg
```

当只有三个文件时，重复的校验和很容易用肉眼检测，但是如果有三千个文件呢？管道来拯救。计算所有的校验和，使用`cut`来分离每行的前 32 个字符，并对行进行排序以使任何重复项相邻：

```
$ md5sum *.jpg | cut -c1-32 | sort
1258012d57050ef6005739d0e6f6a257
146b163929b6533f02e91bdf21cb9563
146b163929b6533f02e91bdf21cb9563
17f339ed03733f402f74cf386209aeb3
⋮
```

现在添加`uniq`来计算重复的行数：

```
$ md5sum *.jpg | cut -c1-32 | sort | uniq -c
      1 1258012d57050ef6005739d0e6f6a257
      2 146b163929b6533f02e91bdf21cb9563
      1 17f339ed03733f402f74cf386209aeb3
      ⋮
```

如果没有重复项，`uniq`生成的所有计数都将为 1。将结果按数字从高到低排序，任何计数大于 1 的都将出现在输出的顶部：

```
$ md5sum *.jpg | cut -c1-32 | sort | uniq -c | sort -nr
      3 f6464ed766daca87ba407aede21c8fcc
      2 c7978522c58425f6af3f095ef1de1cd5
      2 146b163929b6533f02e91bdf21cb9563
      1 d8ad913044a51408ec1ed8a204ea9502
      ⋮
```

现在让我们移除非重复项。它们的校验和前面有六个空格，数字 1 和一个空格。我们将使用`grep -v`来移除这些行：^(5)

```
$ md5sum *.jpg | cut -c1-32 | sort | uniq -c | sort -nr | grep -v "      1 "
      3 f6464ed766daca87ba407aede21c8fcc
      2 c7978522c58425f6af3f095ef1de1cd5
      2 146b163929b6533f02e91bdf21cb9563
```

最后，你有一个按出现次数排序的重复校验和列表，由一个美丽的六命令管道产生。如果没有输出，那么就没有重复文件。

如果该命令能显示重复文件的文件名，那么它将更加有用，但这需要我们尚未讨论的功能。 （你将在“改进重复文件检测器”中了解它们。）现在，通过使用`grep`来搜索具有给定校验和的文件来识别文件：

```
$ md5sum *.jpg | grep 146b163929b6533f02e91bdf21cb9563
146b163929b6533f02e91bdf21cb9563  image001.jpg
146b163929b6533f02e91bdf21cb9563  image003.jpg
```

并使用`cut`来清理输出：

```
$ md5sum *.jpg | grep 146b163929b6533f02e91bdf21cb9563 | cut -c35-
image001.jpg
image003.jpg
```

# 总结

现在你已经看到了标准输入 stdin、标准输出 stdout 和管道的强大之处。它们将少量命令转变为可组合的工具集合，证明整体大于各个部分之和。*任何*能够读取标准输入或写入标准输出的命令都可以参与管道操作。^(6) 随着你学习更多命令，你可以将本章的一般概念应用到自己的强大组合中。

^(1) 在美国键盘上，管道符号位于与反斜杠（`\`）相同的键上，通常位于回车和退格键之间或左 Shift 键和 Z 键之间。

^(2) POSIX 标准将这种形式的命令称为*实用程序*。

^(3) 根据你的设置，`ls`可能还会在打印到屏幕时使用其他格式功能，例如颜色，但在重定向时不会使用。

^(4) 一些 Linux 系统将用户信息存储在其他地方。

^(5) 在技术上，你不需要在这个管道中的最后加上`sort -nr`来隔离重复项，因为`grep`会移除所有非重复项。

^(6) 一些命令不使用标准输入/输出，因此无法从管道读取或向管道写入。例如`mv`和`rm`。然而，管道可以以其他方式整合这些命令；你将在第八章中看到例子。
