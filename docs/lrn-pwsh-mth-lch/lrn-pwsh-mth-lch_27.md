# 27 永无止境

我们几乎到了这本书的结尾，但你的 PowerShell 探索之旅远远没有结束。在 Shell 中还有更多东西要学习，根据你在本书中学到的知识，你将能够自学其中很大一部分。这一简短章节将帮助你找到正确的方向。

## 27.1 进一步探索的想法

首先，如果你是一名数据库管理员，无论是职业还是偶然，我们强烈建议你调查 dbatools，这是一个包含超过 500 个命令的免费 PowerShell 模块，可以安全快速地自动化你所有需要经常做的任务。像 PowerShell 社区一样，dbatools 社区也是友好和吸引人的；你可以在[`dbatools.io`](https://dbatools.io)了解更多关于它们和该模块的信息。dbatools 背后的团队还写了《一个月午餐学会 dbatools》一书，你可以在[`mng.bz/4jED`](http://mng.bz/4jED)免费阅读该书的一章。

本书重点介绍了你需要成为有效的 PowerShell 工具用户的技能和技术。换句话说，你应该能够开始使用所有可用的数千个 PowerShell 命令来完成任务，无论你的需求是关于 Windows、Microsoft 365、SharePoint 还是其他什么。

你的下一步是开始组合命令以创建自动化、多步骤的过程，并以一种产生打包、可供他人使用的工具的方式来做。我们称之为*工具制作*，尽管它更像是一个非常长的脚本或函数，而且这是它自己的完整书籍的主题，即 Don Jones 和 Jeffery Hicks 所著的《一个月午餐学会 PowerShell 脚本编写》（Manning，2017）。但即使你在这本书中学到了，你也可以生成包含完成任务所需的所有命令的参数化脚本——这是工具制作的开始。工具制作还涉及哪些内容？

+   PowerShell 的简化脚本语言

+   范围

+   函数，以及将多个工具构建到单个脚本文件中的能力

+   错误处理

+   编写帮助

+   调试

+   自定义格式化视图

+   自定义类型扩展

+   脚本和清单模块

+   使用数据库

+   工作流

+   管道故障排除

+   复杂的对象层次结构

+   全球化和本地化

+   代理函数

+   限制性远程操作和委托管理

+   使用.NET

还有更多。如果你对此感兴趣并且具备正确的背景技能，你甚至可能是 PowerShell 的第三大受众：软件开发者。围绕为 PowerShell 开发、在开发中使用 PowerShell 以及更多内容存在一套技术和技术。这是一个庞大的产品！

## 27.2 “现在我读完了这本书，我应该从哪里开始？”

现在最好的做法是选择一个任务。选择你在生产环境中个人认为重复性高的任务，并使用 shell 自动化它。是的，学习如何编写脚本可能需要更长的时间，但当你第二次需要它时，你的工作已经为你准备好了。你几乎肯定会遇到不知道如何操作的事情，这正是开始学习的完美地方。以下是我们看到其他管理员解决的问题中的一些：

+   编写一个脚本，更改服务使用的登录密码，并使其针对运行该服务的多台计算机。（你可以用一条命令完成这个操作。）

+   编写一个脚本，自动化新用户配置，包括创建用户账户、邮箱和主目录。

+   编写一个脚本，以某种方式管理 Exchange 或 M635 邮箱——比如获取最大邮箱的报告，或者根据邮箱大小创建费用报告。

最重要的是要记住不要**过度思考**。一位 PowerShell 开发者曾经遇到一位管理员，他花了数周时间在 PowerShell 中编写一个健壮的文件复制脚本，以便能够在 Web 服务器群集中部署内容。“为什么不直接使用 xcopy 或 robocopy 呢？”他问道。管理员盯着他看了一分钟，然后笑了。他太专注于“在 PowerShell 中完成它”，以至于忘记了 PowerShell 可以使用所有已经存在的优秀工具。

## 27.3 你会越来越喜欢的其他资源

我们在 PowerShell 上花费了大量时间，撰写关于 PowerShell 的文章，并教授 PowerShell。问问我们的家人——有时我们几乎无法停止谈论它，以至于吃饭的时间都不够。这意味着我们已经积累了大量我们每天使用并推荐给所有学生的在线资源。希望它们也能为你提供一个良好的起点：

+   [`powershell.org`](https://powershell.org)—这应该是你的第一站。在这里，你可以找到从问答论坛到免费电子书、免费网络研讨会、现场教育活动等一切内容。它是 PowerShell 社区的一个中心聚集地，包括已经运行多年多年的播客。

+   [`youtube.com/powershellorg`](https://youtube.com/powershellorg)—PowerShell.org 的 YouTube 频道有大量的免费 PowerShell 视频，包括在 PowerShell + DevOps 全球峰会记录的会议。

+   [`jdhitsolutions.com`](https://jdhitsolutions.com)—这是 Jeff Hick 的多功能脚本和 PowerShell 博客。

+   [`devopscollective.org`](https://devopscollective.org)—这是 PowerShell.org 的母组织，专注于 IT 管理的更大图景 DevOps 方法。

学生们经常询问我们是否推荐其他 PowerShell 书籍。两本推荐的书是 Don Jones 和 Jeffery Hicks 所著的 *Learn PowerShell Scripting in a Month of Lunches*（Manning, 2017）以及 Don Jones、Jeffery Hicks 和 Richard Siddaway 所著的 *PowerShell in Depth, Second Edition*（Manning, 2014）。*Windows PowerShell in Action, Third Edition*（Manning, 2017）是由该语言的设计者之一 Bruce Payette 以及 Microsoft MVP Richard Siddaway 撰写的一部全面介绍该语言的书籍。我们还推荐 Jeffery Hicks、Richard Siddaway、Oisin Grehan 和 Aleksandar Nikolic 所著的 *PowerShell Deep Dives*（Manning, 2013），这是一本由 PowerShell MVPs 撰写的深入技术文章集（本书的收益将用于 Save the Children 慈善机构，所以请购买三本）。最后，如果你是视频培训的粉丝，Pluralsight 网站上有大量的 PowerShell 视频课程。[`Pluralsight.com`](http://Pluralsight.com)。Tyler 还有一个关于 PowerShell 的视频介绍，“如何导航 PowerShell 帮助系统”，最初是为 Twitch 录制的，现在可以在 [`mng.bz/QW6R`](http://mng.bz/QW6R) 上免费观看。
