# 前言

这本书将带领你的 Linux 命令行技能迈向一个新的高度，让你能够更快、更智能、更高效地工作。

如果你和大多数 Linux 用户一样，在工作中学习了你早期的命令行技能，或者通过阅读一本介绍书，或者在家里安装 Linux 然后试试各种东西。我写这本书是为了帮助你迈出下一步——在 Linux 命令行上建立中级到高级的技能。它充满了我希望能够改变你与 Linux 交互方式并提升你的生产力的技术和概念。把它看作是一本关于 Linux 使用的第二本书，让你超越基础知识。

命令行是最简单的界面，但也是最具挑战性的。它简单是因为你只看到一个提示符，等待你运行任何你可能知道的命令：^(1)

```
$
```

它具有挑战性，因为提示符之外的一切都取决于你。没有友好的图标、按钮或菜单来指导你。相反，你输入的每个命令都是一种创造性的行为。这对基本命令如列出你的文件也是如此：

```
$ ls
```

还有像这样复杂命令：

```
$ paste <(echo {1..10}.jpg | sed 's/ /\n/g') \
        <(echo {0..9}.jpg | sed 's/ /\n/g') \
  | sed 's/^/mv /' \
  | bash
```

如果你正盯着上述命令想，“那是什么*？”或者“我永远不需要这样一个复杂的命令”，那么这本书适合你。^(2)

# 你将学到什么

这本书将让你在三个关键技能上更快更有效：

+   选择或构建命令来解决手头的业务问题

+   运行这些命令的高效性

+   轻松导航 Linux 文件系统

最终，你将了解在运行命令时发生的背后情况，这样你就能更好地预测结果（而不是养成迷信）。你将看到十几种不同的启动命令的方法，并学会何时使用每种方法以取得最佳效果。你还将学到一些实用的技巧和窍门，让你的工作更加高效，例如：

+   逐步从更简单的命令构建复杂的命令，以解决实际问题，如管理密码或生成一万个测试文件

+   通过智能组织你的主目录来节省时间，这样你就不必寻找文件

+   转换文本文件并像数据库一样查询它们以实现业务目标

+   控制 Linux 命令行的点和点击功能，如使用剪贴板进行复制和粘贴，以及检索和处理 Web 数据，而无需从键盘上抬起你的手

最重要的是，你将学到通用的最佳实践，无论你运行哪些命令，你都可以在日常 Linux 使用中取得更多成功，并在职场上更有竞争力。这本书就是我学习 Linux 时希望拥有的书籍。

# 这本书不是什么

这本书不会优化你的 Linux 计算机以使其更有效地运行。它会让*你*在与 Linux 的交互中更加高效。

本书也不是命令行的全面参考资料——有数百个命令和功能我没有提到。本书侧重于专业技能。它按照实用顺序精选了一组命令行知识。要获取参考式指南，请尝试我的早前著作 [*Linux Pocket Guide*](https://oreil.ly/46N1v)（O’Reilly）。

# 受众和先决条件

本书假设您具有 Linux 使用经验；*这不是入门指南*。适合希望提升命令行技能的用户，比如学生、系统管理员、软件开发人员、站点可靠性工程师、测试工程师和 Linux 爱好者。高级 Linux 用户也许也能从中找到一些有用的资料，特别是那些希望通过实践运行命令来加深概念理解的用户。

要从本书中获得最大的收益，您应该已经对以下主题感到熟悉（如果不熟悉，请参见附录 A 进行快速复习）：

+   使用文本编辑器（如`vim`（`vi`）、`emacs`、`nano`或`pico`）创建和编辑文本文件

+   基本的文件处理命令，比如`cp`（复制）、`mv`（移动或重命名）、`rm`（删除）、`chmod`（修改文件权限）

+   基本的文件查看命令，比如`cat`（查看整个文件）和`less`（逐页查看）

+   基本的目录命令，比如`cd`（切换目录）、`ls`（列出目录中的文件）、`mkdir`（创建目录）、`rmdir`（删除目录）和`pwd`（显示当前目录名称）

+   Shell 脚本的基础知识：将 Linux 命令存储在文件中，使文件可执行（使用`chmod 755`或`chmod +x`），然后运行该文件

+   使用`man`命令查看 Linux 内置文档（称为 man 页面）（例如：`man cat` 显示关于 `cat` 命令的文档）

+   使用`sudo`命令成为超级用户，以完全访问您的 Linux 系统（例如：`sudo nano /etc/hosts` 编辑系统文件 */etc/hosts*，普通用户无法访问）

如果您还熟悉常见的命令行功能，比如用于文件名的模式匹配（使用`*`和`?`符号）、输入/输出重定向（`<`和`>`）和管道（`|`），您已经迈出了良好的开端。

# 您的 Shell

我假设您的 Linux Shell 是`bash`，这是大多数 Linux 发行版的默认 Shell。每当我提到“shell”时，我指的是`bash`。本书中提到的大多数概念也适用于其他 shell，比如`zsh`或`dash`；参见附录 B 来帮助将本书的示例翻译成其他 shell 的语法。很多内容在苹果 Mac 终端上也能无需修改地运行，苹果 Mac 终端默认运行`zsh`，也可以运行`bash`。^(3)

# 本书使用的约定

本书中使用以下排版约定：

*斜体*

指示新术语、URL、电子邮件地址、文件名和文件扩展名。

`等宽字体`

用于程序列表，以及在段落内引用程序元素，例如变量或函数名、数据库、数据类型、环境变量、语句和关键字。

**`Constant width bold`**

显示用户应按字面意思键入的命令或其他文本。也偶尔在命令输出中用于突出显示感兴趣的文本。

*`Constant width italic`*

显示应由用户提供值或由上下文确定值替换的文本。也用于代码列表右侧的简要注释。

`Constant width highlighted`

用于复杂程序列表中以引起特定文本注意的标记。

###### Tip

这个元素表示提示或建议。

###### Note

这个元素表示一般注释。

###### Warning

这个元素表示警告或注意事项。

# 使用代码示例

补充材料（代码示例、练习等）可在[*https://efficientlinux.com/examples*](https://efficientlinux.com/examples)下载。

如果你有技术问题或者在使用代码示例时遇到问题，请发送电子邮件至*bookquestions@oreilly.com*。

本书旨在帮助您完成工作。一般情况下，如果本书提供示例代码，则可以在您的程序和文档中使用它。除非您复制了代码的大部分内容，否则您无需联系我们以获取权限。例如，编写一个使用本书中几个代码块的程序不需要权限。销售或分发来自 O’Reilly 图书的示例代码则需要权限。通过引用本书并引用示例代码回答问题不需要权限。将本书中大量示例代码整合到产品文档中则需要权限。

我们欣赏，但通常不需要署名。署名通常包括标题、作者、出版商和 ISBN。例如：“*Efficient Linux at the Command Line* by Daniel J. Barrett (O’Reilly)。版权所有 2022 Daniel Barrett, 978-1-098-11340-7。”

如果您认为您使用的代码示例超出了合理使用范围或以上述许可证给出的权限，请随时通过*permissions@oreilly.com*联系我们。

# O’Reilly Online Learning

###### Note

超过 40 年来，[*O’Reilly Media*](https://oreilly.com) 提供技术和商业培训、知识和洞见，帮助公司取得成功。

我们独特的专家和创新者网络通过书籍、文章和我们的在线学习平台分享他们的知识和专长。O’Reilly 的在线学习平台为您提供按需访问的实时培训课程、深度学习路径、交互式编码环境以及来自 O’Reilly 和其他 200 多家出版商的大量文本和视频。更多信息，请访问[*https://oreilly.com*](https://oreilly.com)。

# 如何联系我们

请将关于本书的评论和问题发送至出版社：

+   O’Reilly Media, Inc.

+   1005 Gravenstein Highway North

+   Sebastopol, CA 95472

+   800-998-9938（美国或加拿大境内）

+   707-829-0515（国际或本地）

+   707-829-0104（传真）

我们有这本书的网页，列出勘误、示例和任何其他信息。您可以访问[*https://oreil.ly/efficient-linux*](https://oreil.ly/efficient-linux)。

请发送电子邮件至*bookquestions@oreilly.com*，就本书的评论或技术问题发表意见。

关于我们的书籍和课程的新闻和信息，请访问[*https://oreilly.com*](https://oreilly.com)。

在 Facebook 上找到我们：[*https://facebook.com/oreilly*](https://facebook.com/oreilly)

在 Twitter 上关注我们：[*https://twitter.com/oreillymedia*](https://twitter.com/oreillymedia)

在 YouTube 上观看我们：[*https://www.youtube.com/oreillymedia*](https://www.youtube.com/oreillymedia)

# 致谢

书写这本书是一种快乐。感谢 O'Reilly 的出色团队，特别是编辑 Virginia Wilson 和 John Devins，制作编辑 Caitlin Ghegan 和 Gregory Hyman，内容经理 Kristen Brown，副本编辑 Kim Wimpsett，索引编辑 Sue Klefstad，以及永远乐于助人的工具团队。我也非常感谢本书的技术审阅者 Paul Bayer、John Bonesio、Dan Ritter 和 Carla Schroder，他们提供了许多有见地的评论和批评。还要感谢波士顿 Linux 用户组提供的书名建议。特别感谢 Google 的 Maggie Johnson，她慷慨允许我撰写这本书。

我要深深感谢 35 年前在约翰斯·霍普金斯大学的同学 Chip Andrews、Matthew Diaz 和 Robert Strandh。他们注意到我对 Unix 的新兴兴趣，并且出乎我的意料，推荐计算机科学系聘请我作为下一任系统管理员。他们这一小小的信任举动改变了我生活的轨迹。（Robert 还因在第三章中关于打字速度提示而获得赞誉。）也感谢 Linux、GNU Emacs、Git、AsciiDoc 以及许多其他开源工具的创造者和维护者们——如果没有这些聪明和慷慨的人们，我的职业生涯将会截然不同。

一如既往，感谢我美好的家人 Lisa 和 Sophia，他们的爱和耐心。

^(1) 本书将 Linux 提示符显示为一个美元符号。你的提示符可能有所不同。

^(2) 你将在第八章中了解这个神秘命令的用途。

^(3) macOS 上的`bash`版本古老且缺少重要功能。要升级`bash`，请参阅 Daniel Weibel 的文章[“Upgrading Bash on macOS”](https://oreil.ly/35jux)。