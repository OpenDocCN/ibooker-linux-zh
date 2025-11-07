## 附录。答案键

### 第四章

> **1**
> 
> 你会如何找出你的电脑上有什么类型的无线网卡？
> 
> **答案：** `lspci -v`
> 
> **2**
> 
> 你在哪里找到你的电脑的日志文件？
> 
> **答案：** /var/log 目录或系统日志。

### 第六章

> **1**
> 
> 当你访问作业书签时会发生什么？作业文件是否随它移动？
> 
> **答案：** 文件随它移动。

**加分题**

> **1**
> 
> 你会如何查看文件的属性？
> 
> **答案：** 右键点击然后选择属性。
> 
> **2**
> 
> 你会如何查看文件夹的属性？
> 
> **答案：** 右键点击然后选择属性。
> 
> **3**
> 
> 你如何创建一个文件或文件夹的快捷方式？
> 
> **答案：** 右键点击然后创建链接。

### 第七章

> **1**
> 
> 你如何在 Synaptic 中启动搜索？
> 
> **答案：** 使用屏幕中间的搜索框。
> 
> **2**
> 
> 你如何在 Ubuntu 软件中心中启动搜索？
> 
> **答案：** 使用屏幕右侧的搜索框。

### 第八章

> **1**
> 
> 使用 LibreOffice Calc 计算 10、15、19 和 18 的平均值。
> 
> **答案：** 使用公式 `=AVERAGE(10, 15, 19, 18)`。
> 
> **2**
> 
> 下载 GIMP 并尝试从 archive.org 缩放一个图像，这次将其放大。
> 
> **答案：** 图像 → 缩放图像。
> 
> **3**
> 
> 使用 Rhythmbox 安装歌曲歌词插件。提示：它在工具菜单下。
> 
> **答案：** 工具 → 插件 → 歌词 → 关闭。

### 第九章

> **1**
> 
> 使用终端安装 Emacs。
> 
> **答案：** `sudo apt-get install emacs`

**高级实验**

> **1**
> 
> 在 Vim 中打开文件并将每一行复制粘贴，使每一行在文件中重复出现。
> 
> **答案：** 使用`:y`和`p`。

### 第十章

> **1**
> 
> 在你的文档文件夹中创建一个名为 command_line_homework 的文件夹。
> 
> **答案：** `mkdir command_line_homework`
> 
> **2**
> 
> 在 command_line_homework 文件夹中创建一个名为 homeworkfile 的文档。
> 
> **答案：**
> 
> ```
> cd command_line_homework
> touch homeworkfile
> ```
> 
> **3**
> 
> 将 homeworkfile 移动到 linux.lunches。
> 
> **答案：** `mv homeworkfile ../linux.lunches`
> 
> **4**
> 
> 进入 linux.lunches 并创建另一个名为 homework2 的文件。
> 
> **答案：**
> 
> ```
> cd ..
> cd linux.lunches/
> touch homework2
> ```
> 
> **5**
> 
> 将 homeworkfile 复制到文档中。
> 
> **答案：** `cp homeworkfile ..`
> 
> **6**
> 
> 从`linux.lunches`中删除`homework2`。
> 
> **答案：** `rm homework2`
> 
> **7**
> 
> 删除`command_line_homework`。
> 
> **答案：**
> 
> ```
> cd ..
> cd command_line_homework
> rm homeworkfile
> 
> cd ..
> rmdir command_line_homework
> ```

**高级实验**

> **1**
> 
> 在名为 recursive 的单个目录中创建三个.txt 文件。
> 
> **答案：**
> 
> ```
> mkdir recursive
> cd recursive
> touch 1.txt
> touch 2.txt
> touch 3.txt
> ```
> 
> **2**
> 
> 将递归复制到你的桌面。
> 
> **答案：**
> 
> ```
> cd ..
> cp -R recursive ../Desktop
> ```
> 
> **3**
> 
> 使用单个通配符命令一次性删除三个 recursive.txt 文件。
> 
> **答案：**
> 
> ```
> cd ..
> cd Desktop
> cd recursive
> rm *.txt
> ```

### 第十一章

> **1**
> 
> 你会使用什么命令来关闭 Firefox 的所有实例？打开 Firefox 并使用该命令强制关闭。
> 
> **答案：** `killall firefox`
> 
> **2**
> 
> 你会用什么命令来查找硬件配置中的音频信息？运行该命令以定位你的信息。
> 
> **答案：** `lspci -v | grep Audio`
> 
> **3**
> 
> 你会用什么命令从[`mng.bz/o25h`](http://mng.bz/o25h)下载美国宪法的 PDF 文件？使用`wget`下载它。
> 
> **答案：** `wget` [`constitutioncenter.org/media/files/constitution.pdf`](http://constitutioncenter.org/media/files/constitution.pdf)
> 
> **4**
> 
> 哪个命令需要进程的 PID 来关闭它？打开 Firefox 并使用该命令关闭它。
> 
> **答案：** `top`

**高级实验**

> **1**
> 
> 使用`wget`命令从 Project Gutenberg ([`www.gutenberg.org/ebooks/6527.txt.utf-8`](http://www.gutenberg.org/ebooks/6527.txt.utf-8))下载*Debian GNU/Linux：安装和使用指南*的纯文本版本。
> 
> **答案：** `wget` [`www.gutenberg.org/ebooks/6527.txt.utf-8`](http://www.gutenberg.org/ebooks/6527.txt.utf-8)
> 
> **2**
> 
> 使用`grep`在文件中搜索单词 linux，但使搜索不区分大小写。
> 
> **答案：** `grep 'linux\|Linux' 6527.txt.utf-8`

### 第十二章

> **1**
> 
> 使用命令行在你的计算机上安装 Midori 网络浏览器。
> 
> **答案：** `sudo apt-get install midori`
> 
> **2**
> 
> 使用命令行将其删除。
> 
> **答案：** `sudo apt-get remove midori`
> 
> **3**
> 
> 在 sudo 的手册中查找所有关于 root 的提及。
> 
> **答案：**
> 
> ```
> man sudo
> /root
> ```
> 
> **4**
> 
> 使用单行命令复制名为 sudo.txt 的 sudo 手册文本文件。
> 
> **答案：** `man sudo > sudo.txt`

**高级实验**

> **1**
> 
> 将上一个实验中`grep`搜索的输出（搜索 Linux 或 linux）导入名为 linux.txt 的文本文件。
> 
> **答案：** `grep 'linux\|Linux' 6527.txt.utf-8 > linux.txt`

### 第十三章

> **1**
> 
> 如果你还没有这样做，请使用命令行安装 Guake 和 Terminator。
> 
> **答案：**
> 
> ```
> sudo apt-get install guake
> sudo apt-get install terminator
> ```
> 
> **2**
> 
> 使用自动完成功能进入你的文档目录。
> 
> **答案：** `cd Doc <Tab>`
> 
> **3**
> 
> 现在，将历史记录的输出导入到文本文件中。
> 
> **答案：** `history > history.txt`
> 
> **4**
> 
> 进入 Guake 并查看你的历史记录。它与默认终端的输出相比如何？
> 
> **答案：** 历史记录相同。
> 
> **5**
> 
> 将 Guake 配置为使用 F11 命令启动。
> 
> **答案：** Guake 首选项 → 键盘快捷键
> 
> **6**
> 
> 将 Terminator 分成四个窗口，形成一个 2x2 的网格。
> 
> **答案：** Ctrl-Shift-E, Ctrl-Shift-O 在每个垂直终端中
> 
> **7**
> 
> 输入`apt`然后按 Tab 键两次。会发生什么？那个输出意味着什么？
> 
> **答案：** 这些是以`apt`开头的命令。

### 第十四章

> **1**
> 
> 将终端提示符恢复到原始状态。
> 
> **答案：** 从.bashrc 中删除`export PS1="\u@\h:\w \d\\$ \[$(tput sgr0)\]"`。
> 
> **2**
> 
> 将你的 GRUB 返回到原始状态，以便在没有消息的情况下启动和关闭。
> 
> **回答：** 将 `GRUB_CMDLINE_LINUX_DEFAULT="quiet splash"` 返回到 GRUB 文件 (/etc/default/grub)，替换 `GRUB_CMDLINE_LINUX_DEFAULT=""`。别忘了你需要以 root 用户身份执行此操作，并在更改后运行 `sudo update-grub`。
> 
> **3**
> 
> 使用终端删除你的隐藏文件（mynewhiddenfile）。
> 
> **回答：** `rm .mynewhiddenfile`
> 
> **4**
> 
> 将你的家目录中所有文件和目录的列表（包括隐藏的）管道到一个隐藏文件中。
> 
> **回答：** `ls -A > .hiddendirectories.txt`
> 
> **5**
> 
> 在你的 tmp 目录中保存一个名为 important.txt 的文件，然后重新启动你的计算机。文件会发生什么？为什么会这样？
> 
> **回答：** 它消失了，因为 tmp 目录不是持久存储——它只是用于临时文件。

### 第十五章

> **1**
> 
> 使用 Wine 卸载 Savings Bond Wizard。
> 
> **回答：** 卸载 Wine 软件 → Savings Bond Wizard → 删除。
> 
> **2**
> 
> 使用 Wine 在 USB 上创建一个 D: 驱动器，就像你想要将 Wine 程序从你的主硬盘上分离出来一样。
> 
> **回答：** 配置 Wine → 添加驱动器 → D: → 浏览到 USB 驱动器路径。
> 
> **3**
> 
> 只玩一次纸牌游戏。如果有人问你在做什么，告诉他们这是这本书的作业。
> 
> **回答：** “我不是在玩。我是在学习 Linux！天哪！”

### 第十六章

> **1**
> 
> 使用 GNOME Do 在你的桌面上创建一个名为 testfolder 的文件夹。
> 
> **回答：** GNOME Do → 输入 `testfolder` → Tab → 创建新文件夹。
> 
> **2**
> 
> 使用 Kupfer 将该文件夹移动到文档中。
> 
> **回答：** Kupfer → 输入 `testfolder` → Tab → 移动到... → Tab → 文档。
> 
> **3**
> 
> 使用 GNOME Do 在你的桌面上创建一个名为 linuxlunches.doc 的文件。
> 
> **回答：** GNOME Do → 输入 `linuxlunches.doc` → Tab → 创建新文件。
> 
> **4**
> 
> 使用 Kupfer 使用 gedit 打开 linuxlunches.doc 文件。
> 
> **回答：** Kupfer → 输入 `linuxlunches.doc` → Tab → 打开方式... → Tab → gedit
> 
> **5**
> 
> 将 gedit 映射到 Ctrl-Shift-G。
> 
> **回答：** 键盘 → 快捷键 → `+` → 名称：gedit；命令：gedit → 点击禁用以映射。

**高级实验室**

> **Q1:**
> 
> 为我们在第十一章中使用的 `xkill` 命令分配一个键组合。使用该快捷键来结束一个打开的程序。
> 
> **回答：** 键盘 → 快捷键 → `+` → 名称：xkill；命令：xkill → 点击禁用以映射到快捷键。

### 第十七章

> **1**
> 
> 仅使用命令安装 Atom 文本编辑器。它的 PPA 是 ppa:webupd8team/atom，详细信息可以在项目页面上找到 [`launchpad.net/~webupd8team/+archive/ubuntu/atom`](https://launchpad.net/~webupd8team/+archive/ubuntu/atom)
> 
> **回答：**
> 
> ```
> sudo add-apt-repository ppa:webupd8team/atom
> sudo apt-get update
> sudo apt-get install atom
> ```
> 
> **2**
> 
> Atom 有哪些依赖项？
> 
> **答案：** gconf2, gconf-service, libgtk2.0-0, libudev0, libudev1, libgcrypt11, libgcrypt20, libgnome-keyring0, gir1.2-gnomekeyring-1.0, libnotify4, libxtst6, libnss3, python, gvfs-bin, xdg-utils, libdbus-1-3, libcap2.
> 
> **3**
> 
> 删除 Kupfer 及其依赖项。
> 
> **答案：** `sudo apt-get --purge remove kupfer`

### 第十八章

> **1**
> 
> 你的系统上 Firefox 的版本是什么？Arch 存储库中的版本是什么？这告诉你关于 Ubuntu 更新 Firefox 频率的什么信息？
> 
> **答案：** 在 Ubuntu 中，Firefox 版本为 45.0.1，截至本文写作时。在 Arch 中，Firefox 版本为 45.0.1-5，截至本文写作时。Firefox 在 Arch 中更新得更频繁，因为它是滚动发布，尽管两个版本很接近，正如你可以通过版本号看到的。
> 
> **2**
> 
> `update` 命令和 `upgrade` 命令之间的区别是什么？
> 
> **答案：** `update` 下载更新，但 `upgrade` 安装它们。
> 
> **3**
> 
> 配置 Ubuntu 软件更新器以自动下载安全更新。
> 
> **答案：** 软件与更新 → 更新 → 当有安全更新时：→ 自动下载和安装。
> 
> **4**
> 
> 配置 Ubuntu 软件更新器以立即显示非安全相关更新。
> 
> **答案：** 软件与更新 → 更新 → 当有其他更新时：→ 立即显示。
> 
> **5**
> 
> 前往 Gentoo 存储库 [`packages.gentoo.org/`](https://packages.gentoo.org/)。它的 Firefox 和 LibreOffice 库与 Arch 和 Ubuntu 相比如何？它们是更旧还是更新？相差多少？
> 
> **答案：** Gentoo 的 Firefox 版本为 45.0.1，截至本文写作时，比 Arch 略旧，但与 Ubuntu 相同。Gentoo 的 LibreOffice 版本为 5.1.2.2，截至本文写作时。Arch 版本为 5.1.1-4，截至本文写作时，略旧。Ubuntu 的 LibreOffice 版本为 4.2.8.2，比两者都旧得多。

### 第十九章

> **1**
> 
> 创建一个名为 tommy 的管理员账户。这个账户有 `sudo` 访问权限吗？你怎么知道？
> 
> **答案：** 用户账户 → 解锁 → `+` → 账户类型：管理员 → tommy；tommy
> 
> 如果你使用 `groups tommy`，你会看到该账户在 `sudo` 组中。
> 
> **2**
> 
> 使用命令行删除账户。（如果你收到消息说用户正被某个进程使用，你可以使用 `sudo kill -9` 和进程号来结束该进程。）
> 
> **答案：** `sudo userdel -r tommy`
> 
> **3**
> 
> 当你尝试在未挂载的私有目录中保存文件时会发生什么？为什么会这样发生？
> 
> **答案：** 你会收到一条没有权限的消息。在没有首先解锁它的情况下，你不能访问加密目录。
> 
> **4**
> 
> 安装 Gufw（软件包名称 `gufw`）并创建一条规则拒绝从应用程序 BitTorrent（图 19.9 和 19.10）流出流量。
> 
> **答案：** `+` → 策略：拒绝 → 方向：出 → 应用程序：BitTorrent。
> 
> **5**
> 
> 关闭防火墙。
> 
> **答案：** `sudo ufw disable`

### 第二十章

> **1**
> 
> 启用你的虚拟机的防火墙，然后通过 SSH 连接到它。你能否连接到远程机器？
> 
> **答案：** 是的。
> 
> **2**
> 
> 如果你使用 `sudo ufw deny ssh/tcp` 在虚拟机中阻止 SSH，会发生什么？
> 
> **答案：** 你无法连接。
> 
> **3**
> 
> 使用 `PSCP`（或如果你的主机机器是 Linux 或 OS X 则为 `scp`）将 hi 文件移动到你的本地机器。
> 
> **答案：** `scp user@ip:Desktop/hi Desktop`（Linux/OS）或 `pscp user@ip :Desktop/hi Desktop`（Windows）
> 
> **4**
> 
> 你能否通过 SSH 更新和升级你的虚拟机？如果是的话，请这样做。
> 
> **答案：** 是的。

### 第二十二章

> **1**
> 
> 在 GitLab 中从头创建一个新的公开项目。
> 
> **答案：** 新项目 → 公开 → 创建项目
> 
> **2**
> 
> 从终端向项目中推送文件。
> 
> **答案：**
> 
> ```
> git add *your file*
> git commit -m *"your commmit message"*
> git push origin master
> ```
> 
> **3**
> 
> 通过网页界面向项目中添加文件。
> 
> **答案：** 文件 → + → 新文件 → 提交更改
> 
> **4**
> 
> 使用 Git 拉取文件。
> 
> **答案：** `git pull`

**附加问题**

> **Q1:**
> 
> 使用网页界面使你的项目私有化。
> 
> **答案：** 齿轮图标 → 编辑项目 → 可见性级别 → 私有
