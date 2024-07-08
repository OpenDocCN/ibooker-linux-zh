# 前言

在很久以前，我写了*Linux Cookbook*的第一版，它于 2004 年面世。它卖得很好，我收到了许多读者的喜悦反馈，其中一些至今仍是我的朋友。

对于一本 Linux 书籍来说，17 岁就已经相当古老了。2004 年 Linux 有 14 岁，是一个婴儿般的计算机操作系统。即便如此，它已经是一个流行且广泛使用的强大系统，适应任何角色，从微型嵌入式设备到大型机和超级计算机。Linux 的迅速发展部分原因是它是 Unix 的免费克隆，Unix 是历史最悠久和功能最强大的操作系统。Linux 快速增长和广泛采纳的另一个主要因素是没有障碍。任何人都可以下载和尝试使用它，源代码对任何希望使用和贡献的人都是自由开放的。

那时，它是形式服从功能的一个很好的例子，就像我的第一辆车。它能跑，很可靠，但不够漂亮，需要大量定制的摇摆和摆弄来保持运行。当时运行 Linux 系统意味着要学习各种命令、脚本和配置文件，并且需要大量的手动操作和持续学习。软件管理、存储管理、网络、音频、视频、内核管理、进程管理……所有这些都需要大量的实际操作和不断的学习。

17 年后，Linux 中的每个重要子系统都有了显著的改进和变化。现在，我们用所谓的“它只是工作的子系统”取代了所有那些基本管理的手动操作。运行 Linux 系统的每个方面都更加简单，我们可以专注于利用 Linux 做一些酷炫的事情，而不是为了维持其运行而不得不摇摆不定。

我很高兴向您介绍这本大大更新的*Linux Cookbook*第二版，希望您喜欢了解所有这些新鲜和精彩的内容。

# 适合阅读本书的人

这本书适合有一些计算机经验的人士，尽管不一定要有 Linux 经验。我已经尽力使其对 Linux 初学者尽可能易于理解。您应该理解一些基本的网络概念，如 IP 地址、以太网、WiFi、客户端和服务器。您应该了解基本的计算机硬件，并且有一定的命令行使用经验。如果您需要帮助，有大量资源可以学习这些内容；我不想陷入教授已经充分记录的材料之中。

这本书中的食谱都是实践性的。我希望读者第一次尝试就能成功，但如果没有成功也不要感到难过。一般用途的 Linux 计算机是一个非常复杂的机器，有很多东西需要学习。要有耐心，花时间多读一些。很可能你想要的答案就在几句话之内。

每个 Linux 系统都内置了命令的文档，称为*man 页*（缩写为“手册页”）。例如，*man 1 ls*记录了*ls*（或列出目录内容）命令。按照本书中显示的命令精确输入，即可打开正确的 man 页。你也可以在网上找到这些信息。

# 为什么写这本书

我长期以来一直想写一本像这样的书，集结我认为是最必要的 Linux 技能。Linux 无处不在，无论你在哪里找到它，Linux 都是 Linux，所需的技能是相同的。科技世界变化快速，我认为你会发现这本书提供了一个坚实的基础，无论你的兴趣方向如何，都可以在此基础上建立。

烹饪书籍的格式特别适合教授基础知识，因为它展示了如何解决特定的现实世界问题，并将详细的解释与完成任务所需的步骤分开。

# 浏览本书

本书不是一个正式的培训课程，你不需要从头开始，可以随时跳进去，希望能找到你所需的内容。

大致上组织如下：

+   第一章、2 章和 3 章涵盖了安装 Linux、管理引导加载程序、停止和启动，解答了“我从哪里获取 Linux 并让它运行”的问题。

+   第四章介绍了使用 systemd 管理服务，这是与旧有学习各种脚本、配置文件和命令方式相比的一大进步。

+   第五章涵盖了用户和组的管理，第六章讲述了文件和目录的管理，第七章介绍了备份和恢复。这三章对系统操作和安全至关重要。

+   第八章、9 章和 11 章都涉及分区和文件系统，这些对于数据存储管理至关重要。数据管理是计算的最重要方面。

+   第十章非常有趣。这一章节讲述了如何在不打开机箱的情况下获取计算机硬件的详细信息。现代 PC 硬件能够自动报告大量信息，Linux 通过额外信息的数据库补充这些自动报告。

+   第十二章和 13 章教授了如何设置安全的远程访问，第十四章介绍了出色的 firewalld，这是一个动态防火墙，能轻松处理各种复杂情景，如在不同网络之间漫游和管理多个网络接口。

+   第十五章介绍了 CUPS（通用 Unix 打印系统）的新功能，包括“无驱动”打印，特别适合移动设备，因为它们可以连接到打印机而无需下载大量软件。

+   第十六章展示了如何使用优秀的 Dnsmasq 控制您自己的局域网名称服务。Dnsmasq 通过支持新协议和保持旧命令和配置选项不变而保持当前。它是一个一流的名称服务器，无缝集成了 DNS 和 DHCP，用于集中管理 IP 地址和广告网络服务。

+   第十七章介绍了 chrony 和 timesyncd，这两个新的网络时间协议（NTP）实现。还包括旧的经过试验和真实检验的*ntp*服务器和客户端。

+   第十八章介绍了在树莓派上安装 Linux，这款流行的廉价单板计算机，并利用它构建互联网防火墙/网关。

+   第十九章展示了如何使用 SystemRescue 重置丢失的 Linux 和 Windows 密码，救援无法启动的系统，救援故障系统上的数据，并定制 SystemRescue 使其更加实用。

+   第二十章和 21 章介绍了基本的故障排除，重点是搜索日志文件、探测网络以及探测和监控硬件。

+   附录包含管理软件安装和维护的速查表。

# 本书使用的约定

本书中使用以下排版约定：

*斜体*

表示新术语、URL、电子邮件地址、文件名和文件扩展名，以及指向程序元素，如 Linux 变量或函数名、数据库、数据类型、环境变量、语句和关键字。

`等宽字体`

用于程序清单和某些命令选项。

**`等宽字体粗体`**

显示用户应直接输入的命令或其他文本。

*`等宽字体斜体`*

显示应由用户提供的值或由上下文确定的值应替换的文本。

###### 提示

此元素表示提示或建议。

###### 注意

此元素表示一般注意事项。

###### 警告

此元素表示警告或注意事项。

# 使用代码示例

本书旨在帮助您完成工作任务。一般情况下，如果本书提供示例代码，您可以在您的程序和文档中使用它。除非您复制了代码的大部分内容，否则无需征得我们的许可。例如，编写使用本书多个代码片段的程序无需许可。销售或分发来自奥莱利图书的示例则需要许可。引用本书并引用示例代码回答问题无需许可。将本书大量示例代码整合到您产品的文档中则需要许可。

我们感谢，但通常不要求署名。署名通常包括标题、作者、出版社和 ISBN。例如：“*Linux Cookbook*，第二版，Carla Schroder 著（奥莱利）。版权所有 2021 Carla Schroder，978-1-492-08716-8。”

如果您认为您对代码示例的使用超出了合理使用范围或上述许可，请随时通过*permissions@oreilly.com*联系我们。

# 奥莱利在线学习

###### 注意

超过 40 年来，[*奥莱利传媒*](http://oreilly.com) 提供技术和商业培训，以及知识和洞察，帮助公司取得成功。

我们独特的专家和创新者网络通过图书、文章和我们的在线学习平台分享他们的知识和专长。奥莱利的在线学习平台为您提供按需访问的实时培训课程、深入学习路径、交互式编码环境以及来自奥莱利和其他 200 多家出版商的大量文本和视频。欲了解更多信息，请访问[*http://oreilly.com*](http://oreilly.com)。

# 如何联系我们

请将有关本书的评论和问题寄送给出版商：

+   奥莱利传媒公司

+   1005 Gravenstein Highway North

+   Sebastopol, CA 95472

+   800-998-9938（美国或加拿大）

+   707-829-0515（国际或本地）

+   707-829-0104（传真）

我们为本书设置了一个网页，其中列出了勘误、示例和任何其他信息。您可以访问[*https://oreil.ly/linux-cookbook-2e*](https://oreil.ly/linux-cookbook-2e)。

发送电子邮件至*bookquestions@oreilly.com*评论或询问有关本书的技术问题。

欲了解我们的图书和课程的最新信息，请访问[*http://oreilly.com*](http://oreilly.com)。

在 Facebook 上找到我们：[*http://facebook.com/oreilly*](http://facebook.com/oreilly)。

在 Twitter 上关注我们：[*http://twitter.com/oreillymedia*](http://twitter.com/oreillymedia)。

在 YouTube 上观看我们：[*http://www.youtube.com/oreillymedia*](http://www.youtube.com/oreillymedia)。

# 致谢

我真的很幸运能得到这本书。我的编辑 Jeff Bleiel 一直非常支持和乐于助人，做出了许多改进，并使整个项目保持有序和顺利进行。如果你觉得管理猫很难，试试做一名图书编辑。

我的实习生凯特·尤尔内斯在这次冒险开始时是一个 Linux 新手，这使她成为了完美的审阅者。她测试了每一个配方，并为配方的准确性和清晰度做出了重大贡献。我们一起喝了很多咖啡，也度过了愉快的时光，这也是一个重要的贡献。

技术编辑丹尼尔·巴雷特非常注重细节，不懈地追求措辞和描述的准确性，并提供了许多改进意见。提供一本充满命令的书很容易，解释它们如何工作却很难。每个作家都应该很幸运能有这样一位技术编辑。

技术编辑乔纳森·约翰逊发现了其他人都忽视的事物，贡献了一些超酷的命令咒语，并提供了幽默感，而我告诉你，我真的需要它。

收购编辑赞·麦克奎德开始了整个疯狂的交易。多年来我们都在谈论更新《Linux Cookbook》，但赞让它成为现实。

特别感谢我的妻子特里，她喂养了骡子、猫、狗和我，给予了我很大的鼓励，并阻止我因为失去理智而同意再写一本书而逃离家庭。

特别感谢我们的猫，公爵夫人（图 P-1），斯塔什（图 P-2）和疯狂麦克斯（图 P-3），它们出现在这本书中。它们通过在我的键盘上睡觉、不让我坐椅子和经常发出神秘的巨响，给了我很大帮助。

![公爵夫人为了我自己的好压住了我的脚。](img/lcb2_p001.png)

###### 图 P-1\. 公爵夫人为了我自己的好压住了我的脚

![斯塔什猫，我们的魅力男孩。](img/lcb2_p003.png)

###### 图 P-2\. 斯塔什猫，我们的魅力男孩

![疯狂麦克斯为下一场混战休息。](img/lcb2_p002.png)

###### 图 P-3\. 疯狂麦克斯为下一场混战休息