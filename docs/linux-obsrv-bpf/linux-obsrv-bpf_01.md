# 前言

2015 年，David 作为 Docker 的核心开发者，这家公司使容器变得流行。他的日常工作分为帮助社区和推动项目增长。他的工作之一是审查社区成员发送的大量拉取请求；他还必须确保 Docker 适用于各种场景，包括在任何时候运行和配置数千个容器的高性能工作负载。

在 Docker 诊断性能问题时，我们使用了*火焰图*，这是高级可视化工具，可帮助您轻松地浏览数据。Go 编程语言通过嵌入式 HTTP 端点使得测量和提取应用性能数据变得非常容易，并基于这些数据生成图表。David 撰写了一篇关于 Go 分析器功能的文章，以及如何使用其数据生成火焰图。关于 Docker 收集性能数据的一个主要问题是分析器默认情况下是禁用的，因此如果您尝试调试性能问题，首要操作是重新启动 Docker。这种策略的主要问题在于通过重新启动服务，您可能会丢失正在尝试收集的相关数据，然后需要等待直到再次发生您要跟踪的事件。在 David 关于 Docker 火焰图的文章中，他提到这是测量 Docker 性能的一个必要步骤，但不一定需要这样做。这一认识使他开始研究不同的技术来收集和分析任何应用程序的性能，这导致他发现了 BPF。

与此同时，远离 David，Lorenzo 正在寻找一个理由来更好地研究 Linux 内核的内部机制，他发现通过学习 BPF 时能够轻松了解许多内核子系统。几年后，他能够在他在 InfluxData 的工作中应用 BPF，以了解如何使 InfluxCloud 中的数据摄取更快。现在 Lorenzo 参与了 BPF 社区和 IOVisor，并在 Sysdig 工作，负责 Falco，这是一个使用 BPF 进行容器和 Linux 运行时安全性的工具。

在过去的几年中，我们在多种场景中使用了 BPF，从收集 Kubernetes 集群的利用率数据到管理网络流量策略。通过使用它和阅读像 Brendan Gregg 和 Alexei Starovoitov 这样的技术领袖以及 Cilium 和 Facebook 等公司的许多博客文章，我们深入了解了它的方方面面。他们的文章和出版物在过去对我们帮助很大，并且它们也是本书开发的重要参考。

阅读了许多这些资源后，我们意识到，每次我们需要学习有关 BPF 的东西时，我们都需要在许多博客文章、手册页面和互联网的其他地方之间跳转。这本书是我们试图将分散在网络上的知识集中到一个中心位置，供下一代 BPF 爱好者学习这一美妙技术之用。

我们将我们的工作分为九个不同的章节，向您展示使用 BPF 可以实现的功能。您可以单独阅读一些章节作为参考指南，但如果您是 BPF 的新手，我们建议您按顺序阅读它们。这将使您了解 BPF 的核心概念，并指导您在前进的道路上的可能性。

无论您是已经是可观察性和性能分析的专家，还是正在研究新的可能性来回答您之前无法解决的生产系统问题，我们希望您能在本书中找到新的知识。

# 本书使用的约定

本书使用以下排版约定：

*斜体*

表示新术语、URL、电子邮件地址、文件名和文件扩展名。

`Constant width`

用于程序清单，以及在段落中用于引用程序元素，如变量或函数名称、数据库、数据类型、环境变量、语句和关键字。

**`Constant width bold`**

显示用户应该按照字面意义输入的命令或其他文本。

*`Constant width italic`*

显示应由用户提供值或由上下文确定值的文本。

###### 提示

此元素表示提示或建议。

###### 注意

此元素表示一般注释。

###### 警告

此元素表示警告或注意事项。

# 使用代码示例

补充材料（代码示例、练习等）可在[*https://oreil.ly/lbpf-repo*](https://oreil.ly/lbpf-repo)下载。

本书旨在帮助您完成工作。一般而言，如果本书提供示例代码，则可以在您的程序和文档中使用它。除非您重现了代码的重要部分，否则无需联系我们以获得许可。例如，编写一个使用本书中几个代码片段的程序不需要许可。销售或分发 O'Reilly 书籍的示例需要许可。通过引用本书并引用示例代码来回答问题不需要许可。将本书中大量示例代码合并到您产品的文档中需要许可。

我们感谢，但不要求署名。通常包括标题、作者、出版商和 ISBN 号的署名。例如：“*Linux Observability with BPF* by David Calavera and Lorenzo Fontana (O’Reilly). Copyright 2020 David Calavera and Lorenzo Fontana, 978-1-492-05020-9.”

如果您认为您使用的代码示例超出了公平使用范围或这里给出的许可，请随时通过*permissions@oreilly.com*与我们联系。

# O’Reilly 在线学习

###### 注意

40 多年来，[*O’Reilly Media*](http://oreilly.com)提供技术和商业培训、知识和见解，帮助公司取得成功。

我们独特的专家和创新者网络通过书籍、文章、会议以及我们的在线学习平台分享他们的知识和专业知识。O’Reilly 的在线学习平台为您提供按需访问实时培训课程、深入学习路径、交互式编码环境以及来自 O’Reilly 和其他 200 多家出版商的大量文本和视频。欲了解更多信息，请访问[*http://oreilly.com*](http://oreilly.com)。

# 如何联系我们

请将有关本书的评论和问题发送至出版商：

+   O’Reilly Media, Inc.

+   1005 Gravenstein Highway North

+   CA 95472，Sebastopol

+   800-998-9938（美国或加拿大）

+   707-829-0515（国际或本地）

+   707-829-0104（传真）

我们为本书创建了一个网页，列出勘误、示例和任何额外信息。您可以访问[*https://oreil.ly/linux-bpf*](https://oreil.ly/linux-bpf)查看这个页面。

通过电子邮件*bookquestions@oreilly.com*对本书提出评论或技术问题。

欲了解有关我们的图书、课程、会议和新闻的更多信息，请访问我们的网站：[*http://www.oreilly.com*](http://www.oreilly.com)。

在 Facebook 上找到我们：[*http://facebook.com/oreilly*](http://facebook.com/oreilly)

在 Twitter 上关注我们：[*http://twitter.com/oreillymedia*](http://twitter.com/oreillymedia)

在 YouTube 上观看我们：[*http://www.youtube.com/oreillymedia*](http://www.youtube.com/oreillymedia)

# 致谢

写一本书比我们想象的更困难，但这可能是我们生活中最有价值的活动之一。这本书耗费了许多日夜，如果没有我们的合作伙伴、家人、朋友和狗的帮助，这将是不可能完成的。我们要感谢 Debora Pace、Lorenzo 的女朋友以及他的儿子 Riccardo，在长时间的写作过程中对他们的耐心。同时也感谢 Lorenzo 的朋友 Leonardo Di Donato 提供的所有建议，特别是关于 XDP 和测试的部分。

我们对 Robin Means 和 David 的妻子在早期草稿和开始本书的初步概述的校对表示永远的感激，以及在多年来帮助他撰写许多文章并笑着听他编造的听起来比实际更可爱的英语单词。

我们都想向那些使 eBPF 和 BPF 变得可能的人表示衷心的感谢。感谢 David Miller 和 Alexei Starovoitov 不断贡献改进 Linux 内核，最终改进了 eBPF 及其周围的社区。感谢 Brendan Gregg 乐于分享、他的热情以及他在使 eBPF 更加易于访问的工具方面的工作。感谢 IOVisor 团队为 bpftrace、gobpf、kubectl-trace 和 BCC 所付出的努力、邮件以及所有工作。感谢 Daniel Borkmann 在 libbpf 和工具基础设施方面的所有激励性工作。感谢 Jessie Frazelle 撰写前言并对我们俩以及数千名开发者产生了启发作用。感谢 Jérôme Petazzoni 成为我们最想要的最佳技术评审者；他的问题让我们重新思考了本书的许多部分及其代码示例的处理方式。

感谢所有数千名 Linux 内核贡献者，特别是那些在 BPF 邮件列表中活跃的人，感谢他们的问题/回答、补丁和倡议。最后，感谢所有参与 O'Reilly 出版此书的人员，包括我们的编辑 John Devins 和 Melissa Potter，以及所有在幕后制作封面、审核页面的人员，使本书比我们职业生涯中的任何其他作品更加专业。
