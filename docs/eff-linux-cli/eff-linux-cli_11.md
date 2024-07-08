# 第九章：利用文本文件

在许多 Linux 系统上，纯文本是最常见的数据格式。大多数管道中从命令到命令发送的内容都是文本。程序员的源代码文件、系统配置文件（位于 */etc*）、HTML 和 Markdown 文件都是文本文件。电子邮件消息是文本；即使是附件也以文本形式内部存储以进行传输。你甚至可以将像购物清单和个人笔记这样的日常文件存储为文本。

与今天的互联网形成对比，后者充斥着流媒体音频和视频、社交媒体帖子、Google Docs 和 Office 365 中的浏览器文档、PDF 和其他富媒体。 （更不用说移动应用程序处理的数据，这些应用程序已经将“文件”的概念隐藏起来，成为整个一代人的隐喻。）在这样的背景下，纯文本文件似乎显得有些古老。

然而，任何文本文件都可以成为你可以用精心设计的 Linux 命令挖掘的丰富数据源，特别是如果文本是结构化的。例如，文件 */etc/passwd* 中的每一行代表一个 Linux 用户，具有七个字段，包括用户名、数字用户 ID、主目录等。这些字段由冒号分隔，使得文件可以轻松被 `cut -d:` 或 `awk -F:` 解析。这里有一个按字母顺序打印所有用户名（第一个字段）的命令：

```
$ cut -d: -f1 /etc/passwd | sort
avahi
backup
daemon
⋮
```

这里有一个例子，通过数字用户 ID 将人类用户与系统帐户分开，并向用户发送欢迎电子邮件。让我们逐步构建这个大胆的单行代码。首先使用 `awk` 打印用户名（第一个字段），当数字用户 ID（第三个字段）为 1000 或更大时：

```
$ awk -F: '$3>=1000 {print $1}' /etc/passwd
jones
smith
```

然后通过管道传递给 `xargs` 生成问候语：

```
$ awk -F: '$3>=1000 {print $1}' /etc/passwd \
  | xargs -I@ echo "Hi there, @!"
Hi there, jones!
Hi there, smith!
```

然后生成命令（字符串），通过 `mail` 命令将每个问候发送给指定用户，带有给定的主题行（`-s`）：

```
$ awk -F: '$3>=1000 {print $1}' /etc/passwd \
  | xargs -I@ echo 'echo "Hi there, @!" | mail -s greetings @'
echo "Hi there, jones!" | mail -s greetings jones
echo "Hi there, smith!" | mail -s greetings smith
```

最后，将生成的命令通过管道传递给 `bash` 发送电子邮件：

```
$ awk -F: '$3>=1000 {print $1}' /etc/passwd \
  | xargs -I@ echo 'echo "Hi there, @!" | mail -s greetings @' \
  | bash
echo "Hi there, jones!" | mail -s greetings jones
echo "Hi there, smith!" | mail -s greetings smith
```

像本书中的许多其他解决方案一样，这些解决方案从现有的文本文件开始，并使用命令操作其内容。现在是时候扭转这种方法，有意*设计新的文本文件*，以便与 Linux 命令良好合作。这是在 Linux 系统上高效完成工作的一种成功策略，只需四个步骤：

1.  注意你想解决的涉及数据的业务问题。

1.  将数据以便捷的格式存储在文本文件中。

1.  发明 Linux 命令来处理文件以解决问题。

1.  （*可选*。）将这些命令捕获在脚本、别名或函数中，以简化运行。

在本章中，你将构建各种结构化文本文件，并创建命令来处理它们，以解决几个业务问题。

# 第一个例子：查找文件

假设你的主目录包含数万个文件和子目录，而且经常会忘记其中一个文件的位置。`find` 命令可以按文件名（例如 *animals.txt*）定位文件：

```
$ find $HOME -name animals.txt -print
/home/smith/Work/Writing/Books/Lists/animals.txt
```

但`find`很慢，因为它搜索整个主目录，而且你需要定期定位文件。这是第 1 步，注意到涉及数据的业务问题：通过名称快速在你的主目录中找到文件。

第 2 步是以方便的格式将数据存储在文本文件中。运行`find`一次，以构建所有文件和目录的列表，每行一个文件路径，并将其存储在一个隐藏文件中：

```
$ find $HOME -print > $HOME/.ALLFILES
$ head -n3 $HOME/.ALLFILES
/home/smith
/home/smith/Work
/home/smith/Work/resume.pdf
⋮
```

现在你有数据：文件的逐行索引。第 3 步是发明 Linux 命令以加快文件搜索，为此使用`grep`。在大型文件中进行`grep`要比在大型目录树中运行`find`快得多：

```
$ grep animals.txt $HOME/.ALLFILES
/home/smith/Work/Writing/Books/Lists/animals.txt
```

第 4 步是使命令更易于运行。编写一个名为`ff`的单行脚本，表示“查找文件”，该脚本运行`grep`与任何用户提供的选项和搜索字符串，如示例 9-1 中所示。

##### 示例 9-1\. `ff`脚本

```
#!/bin/bash
# $@ means all arguments provided to the script
grep "$@" $HOME/.ALLFILES
```

使脚本可执行，并将其放入搜索路径中的任何目录，例如您的个人*bin*子目录：

```
$ chmod +x ff
$ echo $PATH                                        *Check your search path*
/home/smith/bin:/usr/local/bin:/usr/bin:/bin
$ mv ff ~/bin
```

随时运行`ff`来快速定位文件，当你忘记放置它们的位置时。

```
$ ff animal
/home/smith/Work/Writing/Books/Lists/animals.txt
$ ff -i animal | less                               *Case-insensitive grep*
/home/smith/Work/Writing/Books/Lists/animals.txt
/home/smith/Vacations/Zoos/Animals/pandas.txt
/home/smith/Vacations/Zoos/Animals/tigers.txt
⋮
$ ff -i animal | wc -l                              *How many matches?*
16
```

定期重新运行`find`命令以更新索引。（或者更好的办法是使用`cron`创建定期作业；参见“学习 cron、crontab 和 at”。）Voilà——你已经利用两个小命令构建了一个快速灵活的文件搜索实用程序。Linux 系统提供了其他快速索引和搜索文件的应用程序，如`locate`命令以及 GNOME、KDE Plasma 和其他桌面环境中的搜索实用程序，但这不是重点。看看你自己创建它*多么容易*。成功的关键是创建一个简单格式的文本文件。

# 检查域名过期

对于下一个示例，假设你拥有一些互联网域名，并希望跟踪它们的到期时间以便续订。这是第 1 步，确定业务问题。第 2 步是创建这些域名的文件，例如*domains.txt*，每行一个域名：

```
example.com
oreilly.com
efficientlinux.com
⋮
```

第 3 步是发明利用这个文本文件来确定到期日期的命令。从查询域名注册商提供有关域名信息的`whois`命令开始：

```
$ whois example.com | less
Domain Name: EXAMPLE.COM
Registry Domain ID: 2336799_DOMAIN_COM-VRSN
Registrar WHOIS Server: whois.iana.org
Updated Date: 2021-08-14T07:01:44Z
Creation Date: 1995-08-14T04:00:00Z
Registry Expiry Date: 2022-08-13T04:00:00Z
⋮
```

该过期日期之前的字符串“Registry Expiry Date”可通过`grep`和`awk`分离出来：

```
$ whois example.com | grep 'Registry Expiry Date:'
Registry Expiry Date: 2022-08-13T04:00:00Z
$ whois example.com | grep 'Registry Expiry Date:' | awk '{print $4}'
2022-08-13T04:00:00Z
```

可通过`date --date`命令使日期更易读，该命令可以将一个日期字符串从一种格式转换为另一种格式：

```
$ date --date 2022-08-13T04:00:00Z
Sat Aug 13 00:00:00 EDT 2022
$ date --date 2022-08-13T04:00:00Z +'%Y-%m-%d'       *Year-month-day format*
2022-08-13
```

使用命令替换将`whois`的日期字符串提供给`date`命令：

```
$ echo $(whois example.com | grep 'Registry Expiry Date:' | awk '{print $4}')
2022-08-13T04:00:00Z
$ date \
  --date $(whois example.com \
           | grep 'Registry Expiry Date:' \
	   | awk '{print $4}') \
  +'%Y-%m-%d'
2022-08-13
```

现在你有一个命令，查询注册商并打印到期日期。创建一个名为`check-expiry`的脚本，如示例 9-2 所示，运行前述命令并打印到期日期、一个制表符和域名：

```
$ ./check-expiry example.com
2022-08-13	example.com
```

##### 示例 9-2\. `check-expiry`脚本

```
#!/bin/bash
expdate=$(date \
            --date $(whois "$1" \
	             | grep 'Registry Expiry Date:' \
		     | awk '{print $4}') \
            +'%Y-%m-%d')
echo "$expdate	$1"		# Two values separated by a tab
```

现在，使用循环检查文件*domains.txt*中的所有域。创建一个新脚本`check-expiry-all`，如示例 9-3 所示。

##### 示例 9-3\. `check-expiry-all`脚本

```
#!/bin/bash
cat domains.txt | while read domain; do
    ./check-expiry "$domain"
    sleep 5			# Be kind to the registrar's server
done
```

将脚本在后台运行，因为如果你有很多域名的话，可能需要一段时间，并将所有输出（stdout 和 stderr）重定向到文件：

```
$ ./check-expiry-all &> expiry.txt &
```

脚本完成后，文件*expiry.txt*包含所需的信息：

```
$ cat expiry.txt
2022-08-13	example.com
2022-05-26	oreilly.com
2022-09-17	efficientlinux.com
⋮
```

万岁！但不要停在这里。文件*expiry.txt*本身结构良好，便于进一步处理，有两列标签。例如，对日期进行排序，找到下一个需要续订的域名：

```
$ sort -n expiry.txt | head -n1
2022-05-26	oreilly.com
```

或者，使用`awk`查找今天到期或将要到期的域名，即其到期日期（字段 1）小于或等于今天的日期（使用`date +%Y-%m-%d`打印）：

```
$ awk "\$1<=\"$(date +%Y-%m-%d)\"" expiry.txt
```

关于前述`awk`命令的几点说明：

+   在`awk`之前，我转义了美元符号（在`$1`前）和日期字符串周围的双引号，这样 shell 在`awk`执行之前就不会对它们进行评估。

+   我稍微作弊了，使用字符串运算符`<=`来比较日期。这不是数学比较，只是字符串比较，但它能够工作，因为日期格式*`YYYY-MM-DD`*按字母顺序和时间顺序排序。

更多努力的话，你可以在`awk`中进行日期数学运算，报告到期日期，比如提前两周，然后创建一个定时作业，每晚运行脚本并通过电子邮件发送报告。随时进行实验。然而，这里的重点是，再次用几个命令，你已经构建了一个由文本文件驱动的有用工具。

# 构建区号数据库

下一个示例使用一个包含三个字段的文件，你可以用多种方式处理它。文件名为*areacodes.txt*，包含美国的电话区号。从本书的[补充材料](https://efficientlinux.com/examples)中的目录*chapter09/build_area_code_database*获取文件，或者自己创建一个文件，比如从[Wikipedia](https://oreil.ly/yz2M1)获取：^(2)

```
201	NJ	Hackensack, Jersey City
202	DC	Washington
203	CT	New Haven, Stamford
⋮
989	MI	Saginaw
```

###### 提示

先排列长度可预测的字段，这样列就会整齐地对齐。如果把城市名放在第一列，文件看起来会多么凌乱：

```
Hackensack, Jersey City	201	NJ
Washington	202	DC
⋮
```

一旦这个文件就位，你可以做很多事情。使用`grep`按州查找区号，添加`-w`选项以匹配完整的单词（以防其他文本恰好包含“NJ”）：

```
$ grep -w NJ areacodes.txt
201	NJ	Hackensack, Jersey City
551	NJ	Hackensack, Jersey City
609	NJ	Atlantic City, Trenton, southeast and central west
⋮
```

或者按区号查找城市：

```
$ grep -w 202 areacodes.txt
202	DC	Washington
```

或者通过文件中的任何字符串查找：

```
$ grep Washing areacodes.txt
202	DC	Washington
227	MD	Silver Spring, Washington suburbs, Frederick
240	MD	Silver Spring, Washington suburbs, Frederick
⋮
```

使用`wc`计算区号：

```
$ wc -l areacodes.txt
375 areacodes.txt
```

找到区号最多的州（冠军是加利福尼亚州，有 38 个）：

```
$ cut -f2 areacodes.txt | sort | uniq -c | sort -nr | head -n1
     38 CA
```

将文件转换为 CSV 格式，以导入电子表格应用程序。打印第三个字段时用双引号括起来，以防止其逗号被解释为 CSV 分隔符：

```
$ awk -F'\t' '{printf "%s,%s,\"%s\"\n", $1, $2, $3}' areacodes.txt \
  > areacodes.csv
$ head -n3 areacodes.csv
201,NJ,"Hackensack, Jersey City"
202,DC,"Washington"
203,CT,"New Haven, Stamford"
```

将给定州的所有区号汇总到一行上：

```
$ awk '$2~/^NJ$/{ac=ac FS $1} END {print "NJ:" ac}' areacodes.txt
NJ: 201 551 609 732 848 856 862 908 973
```

或者为每个州使用数组和`for`循环汇总，如“提高重复文件检测器”中所示：

```
$ awk '{arr[$2]=arr[$2] " " $1} \
         END {for (i in arr) print i ":" arr[i]}' areacodes.txt \
  | sort
AB: 403 780
AK: 907
AL: 205 251 256 334 659
⋮
WY: 307
```

将任何前述命令转换为别名、函数或脚本，方便使用。一个简单的例子是示例 9-4 中的`areacode`脚本。

##### 示例 9-4\. `areacode` 脚本

```
#!/bin/bash
if [ -n "$1" ]; then
  grep -iw "$1" areacodes.txt
fi
```

`areacode`脚本搜索*areacodes.txt*文件中的任何整词，如区号、州名缩写或城市名：

```
$ areacode 617
617	MA	Boston
```

# 构建密码管理器

作为最后深入的示例，让我们将用户名、密码和备注存储在加密的文本文件中，以结构化格式便于在命令行上进行检索。生成的命令是一个基本的密码管理器，一个简化记忆大量复杂密码负担的应用程序。

###### 警告

密码管理在计算机安全中是一个复杂的主题。这个例子创建了一个非常基本的密码管理器作为教育练习。不要将其用于关键任务应用程序。

密码文件，命名为*vault*，有三个字段，由单个制表符分隔：

+   用户名

+   密码

+   注意（任何文本）

创建*vault*文件并添加数据。文件尚未加密，所以现在只插入虚假密码。

```
$ touch vault                                     *Create an empty file*
$ chmod 600 vault                                 *Set file permissions*
$ emacs vault                                     *Edit the file*
$ cat vault
sally	fake1	google.com account
ssmith	fake2	dropbox.com account for work
s999	fake3	Bank of America account, bankofamerica.com
smith2	fake4	My blog at wordpress.org
birdy	fake5	dropbox.com account for home
```

将保险库存储在已知位置：

```
$ mkdir ~/etc
$ mv vault ~/etc
```

这个想法是使用像`grep`或`awk`这样的模式匹配程序打印匹配给定字符串的行。这种简单而强大的技术可以匹配任何行的任何部分，而不仅仅是用户名或网站。例如：

```
$ cd ~/etc
$ grep sally vault                            *Match a username*
sally	fake1	google.com account
$ grep work vault                             *Match the notes*
ssmith	fake2	dropbox.com account for work
$ grep drop vault                             *Match multiple lines*
ssmith	fake2	dropbox.com account for work
birdy	fake5	dropbox.com account for home
```

在脚本中捕获这个简单的功能；然后，我们一步步改进它，包括最终对*vault*文件进行加密。将脚本命名为`pman`，表示“密码管理器”，并在[示例 9-5](https://example.org/ex_pman_1)中创建最简单的版本。

##### 示例 9-5\. `pman` 版本 1：尽可能简单

```
#!/bin/bash
# Just print matching lines
grep "$1" $HOME/etc/vault
```

将脚本存储在您的搜索路径中：

```
$ chmod 700 pman
$ mv pman ~/bin
```

尝试这个脚本：

```
$ pman goog
sally	fake1	google.com account
$ pman account
sally	fake1	google.com account
ssmith	fake2	dropbox.com account for work
s999	fake3	Bank of America account, bankofamerica.com
birdy	fake5	dropbox.com account for home
$ pman facebook                                        *(produces no output)*
```

[示例 9-6](https://example.org/ex_pman_2)中的下一个版本添加了一些错误检查和一些值得记住的变量名。

##### 示例 9-6\. `pman` 版本 2：添加一些错误检查

```
#!/bin/bash
# Capture the script name.
# $0 is the path to the script, and basename prints the final filename.
PROGRAM=$(basename $0)
# Location of the password vault
DATABASE=$HOME/etc/vault

# Ensure that at least one argument was provided to the script.
# The expression >&2 directs echo to print on stderr instead of stdout.
if [ $# -ne 1 ]; then
    >&2 echo "$PROGRAM: look up passwords by string"
    >&2 echo "Usage: $PROGRAM string"
    exit 1
fi
# Store the first argument in a friendly, named variable
searchstring="$1"

# Search the vault and print an error message if nothing matches
grep "$searchstring" "$DATABASE"
if [ $? -ne 0 ]; then
    >&2 echo "$PROGRAM: no matches for '$searchstring'"
    exit 1
fi
```

运行脚本：

```
$ pman
pman: look up passwords by string
Usage: pman string
$ pman smith
ssmith	fake2	dropbox.com account for work
smith2	fake4	My blog at wordpress.org
$ pman xyzzy
pman: no matches for 'xyzzy'
```

这种技术的一个缺点是它无法扩展。如果*vault*包含数百行，而`grep`匹配并打印了其中的 63 行，你将不得不靠眼睛寻找你需要的密码。通过在第三列中的每行添加一个唯一的键（一个字符串），并更新`pman`来首先搜索该唯一键来改进脚本。现在，*vault*文件，用粗体标记的第三列，看起来像这样：

```
sally	fake1	google	google.com account
ssmith	fake2	dropbox	dropbox.com account for work
s999	fake3	bank	Bank of America account, bankofamerica.com
smith2	fake4	blog	My blog at wordpress.org
birdy	fake5	dropbox2	dropbox.com account for home
```

[示例 9-7](https://example.org/ex_pman_3)展示了使用`awk`而不是`grep`的更新脚本。它还使用命令替换来捕获输出并检查是否为空（测试`-z`表示“零长度字符串”）。请注意，如果搜索一个在*vault*中不存在的键，`pman`会回到其原始行为，并打印所有匹配搜索字符串的行。

##### 示例 9-7\. `pman` 版本 3：优先搜索第三列的关键字

```
#!/bin/bash
PROGRAM=$(basename $0)
DATABASE=$HOME/etc/vault

if [ $# -ne 1 ]; then
    >&2 echo "$PROGRAM: look up passwords"
    >&2 echo "Usage: $PROGRAM string"
    exit 1
fi
searchstring="$1"

# Look for exact matches in the third column
match=$(awk '$3~/^'$searchstring'$/' "$DATABASE")

# If the search string doesn't match a key, find all matches
if [ -z "$match" ]; then
    match=$(awk "/$searchstring/" "$DATABASE")
fi

# If still no match, print an error message and exit
if [ -z "$match" ]; then
    >&2 echo "$PROGRAM: no matches for '$searchstring'"
    exit 1
fi

# Print the match
echo "$match"
```

运行脚本：

```
$ pman dropbox
ssmith	fake2	dropbox	dropbox.com account for work
$ pman drop
ssmith	fake2	dropbox	dropbox.com account for work
birdy	fake5	dropbox2	dropbox.com account for home
```

明文文件*vault*是一个安全风险，所以用标准的 Linux 加密程序 GnuPG 对其进行加密，调用`gpg`。如果您已经设置好了 GnuPG 供使用，那很好。否则，用以下命令设置它，提供您的电子邮件地址：^(3)

```
$ gpg --quick-generate-key *your_email_address* default default never
```

在为密钥输入密码时会提示您输入密码（两次）。提供一个强密码。当 `gpg` 完成后，您就可以使用公钥加密来加密密码文件，生成 *vault.gpg* 文件：

```
$ cd ~/etc
$ gpg -e -r *your_email_address* vault
$ ls vault* 
vault   vault.gpg
```

作为测试，将 *vault.gpg* 文件解密到标准输出：^(4)

```
$ gpg -d -q vault.gpg
Passphrase: xxxxxxxx
sally	fake1	google	google.com account
ssmith	fake2	dropbox	dropbox.com account for work
⋮
```

接下来，更新您的脚本以使用加密的 *vault.gpg* 文件而不是明文的 *vault* 文件。这意味着将 *vault.gpg* 解密到标准输出，并将其内容管道传输到 `awk` 进行匹配，如 示例 9-8 中所示。

##### 示例 9-8\. `pman` 版本 4：使用加密保险库

```
#!/bin/bash
PROGRAM=$(basename $0)
# Use the encrypted file
DATABASE=$HOME/etc/vault.gpg

if [ $# -ne 1 ]; then
    >&2 echo "$PROGRAM: look up passwords"
    >&2 echo "Usage: $PROGRAM string"
    exit 1
fi
searchstring="$1"

# Store the decrypted text in a variable
decrypted=$(gpg -d -q "$DATABASE")
# Look for exact matches in the third column
match=$(echo "$decrypted" | awk '$3~/^'$searchstring'$/')

# If the search string doesn't match a key, find all matches
if [ -z "$match" ]; then
    match=$(echo "$decrypted" | awk "/$searchstring/")
fi

# If still no match, print an error message and exit
if [ -z "$match" ]; then
    >&2 echo "$PROGRAM: no matches for '$searchstring'"
    exit 1
fi

# Print the match
echo "$match"
```

脚本现在显示来自加密文件的密码：

```
$ pman dropbox
Passphrase: xxxxxxxx
ssmith	fake2	dropbox	dropbox.com account for work
$ pman drop
Passphrase: xxxxxxxx
ssmith	fake2	dropbox	dropbox.com account for work
birdy	fake5	dropbox2	dropbox.com account for home
```

所有的部件现在都准备就绪，用于您的密码管理器。一些最后的步骤是：

+   当您确信可以可靠地解密 *vault.gpg* 文件时，请删除原始的 *vault* 文件。

+   如果需要的话，用真实的密码替换虚假密码。参见 “直接编辑加密文件” 了解如何编辑加密文本文件的建议。

+   支持密码保险库中的注释——以井号 (`#`) 开头的行——以便您可以对条目做出备注。为此，请更新脚本将解密内容传输到 `grep -v` 来过滤任何以井号开头的行：

    ```
    decrypted=$(gpg -d -q "$DATABASE" | grep -v '^#')
    ```

将密码打印到标准输出不利于安全性。“改进密码管理器” 将更新脚本，用复制粘贴代替打印密码。

# 概要

文件路径、域名、区号和登录凭据等数据在结构化文本文件中表现良好。比如：

+   您的音乐文件？（使用类似 `id3tool` 的 Linux 命令从 MP3 文件中提取 ID3 信息并将其放入文件中。）

+   您手机上的联系人？（使用应用程序将联系人导出为 CSV 格式，将其上传到云存储，然后从 Linux 机器下载以进行处理。）

+   您在学校的成绩？（使用 `awk` 跟踪您的平均绩点。）

+   您看过的电影或读过的书籍清单，包括额外的数据（评分、作者、演员等）？

通过这种方式，您可以构建一个由个人有意义或有助于工作的节省时间命令组成的生态系统，仅受您的想象力限制。

^(1) 这种方法类似于设计一个数据库模式，以适应已知的查询。

^(2) [CSV 格式中的官方区号列表](https://oreil.ly/SptWL)，由北美编号计划管理员维护，缺少城市名称。

^(3) 此命令使用所有默认选项和“永不”过期日期生成公钥/私钥对。要了解更多信息，请查看 `man gpg` 以了解 `gpg` 选项，或在线查找 GnuPG 教程。

^(4) 如果 `gpg` 在不提示输入密码的情况下继续进行，它已暂时缓存（保存）了您的密码。
