# 第十一章：eBPF 的未来发展

eBPF 还没有完成！像大多数软件一样，它在 Linux 内核中不断发展，并且也正在被添加到 Windows 操作系统中。在本章中，我们将探讨这项技术的未来发展路径。

自从在 Linux 内核中引入以来，BPF 已经发展成为一个拥有自己的子系统、自己的邮件列表和维护者的独立实体。随着 eBPF 的流行和兴趣超出了 Linux 内核社区，创建一个中立的机构来协调各方的利益是有意义的。这个机构就是 eBPF 基金会。

# eBPF 基金会

[eBPF 基金会](https://ebpf.io/foundation)是由 Google、Isovalent、Meta（当时称为 Facebook）、微软和 Netflix 在 Linux 基金会的支持下于 2021 年成立的。该基金会作为一个中立的机构，可以持有资金和知识产权，以便各个商业公司之间可以进行合作。

基金会的活动由 BPF 指导委员会负责，该委员会完全由构建技术的技术专家组成，包括 Linux 内核 BPF 维护者和其他核心 eBPF 项目的代表。

eBPF 基金会专注于 eBPF 作为技术平台以及支持 eBPF 开发的工具生态系统。寻求中立所有权的基于 eBPF 的项目可能会在其他基金会中找到更好的归属地。例如，Cilium、Pixie 和 Falco 都是 CNCF 的一部分，这是有道理的，因为它们都旨在在云原生环境中使用。

除了现有的 Linux 维护者之外，这种合作的一个关键推动因素是微软对在 Windows 操作系统中开发 eBPF 的兴趣。这带来了一个需要定义 eBPF 标准的需求，以便可以在一个操作系统上编写的程序可以在另一个操作系统上使用。这项工作是在 eBPF 基金会的支持下进行的。

# Windows 上的 eBPF

微软正在积极开展支持[Windows 上的 eBPF](https://oreil.ly/ArwkR)的工作。截至我在 2022 年年底写下这篇文章时，已经有[功能演示](https://oreil.ly/H-0dv)显示 Cilium 第 4 层负载平衡和基于 eBPF 的连接跟踪在 Windows 上运行。

我之前说过 eBPF 编程是内核编程，乍一看，一个在 Linux 内核中运行并且可以访问 Linux 内核数据结构的程序在任何其他操作系统中都能运行似乎有些不合常理。但实际上，特别是在网络方面，所有操作系统都有很多共同之处。无论是在 Windows 还是 Linux 机器上创建的网络数据包都具有相同的结构，网络堆栈的层次也必须以相同的方式处理。

你可能还记得，eBPF 程序由一组字节码指令组成，这些指令由内核中实现的虚拟机（VM）处理。这个虚拟机也可以在 Windows 中实现！

图 11-1 显示了 eBPF for Windows 的架构概述，摘自该项目的 GitHub 存储库。正如您从这个图表中看到的，eBPF for Windows 重用了现有 eBPF 生态系统中的一些开源组件，比如*libbpf*，以及 Clang 对生成 eBPF 字节码的支持。Linux 内核是根据 GPL 许可的，而 Windows 是专有的，因此 Windows 项目无法重用 Linux 内核实现的任何部分。相反，它使用了[PREVAIL 验证器](https://vbpf.github.io)和[uBPF JIT 编译器](https://oreil.ly/btrkJ)（这两者都是宽松许可，因此可以被更广泛的项目和组织使用）。

![eBPF for Windows 的架构概述，摘自 https://oreil.ly/HxKsu](img/lebp_1101.png)

###### 图 11-1：eBPF for Windows 的架构概述，摘自[*https://oreil.ly/HxKsu*](https://oreil.ly/HxKsu)

一个有趣的区别是，eBPF 代码是在 Windows 安全环境中的用户空间中进行验证和 JIT 编译，而不是在内核中进行（内核中显示的 uBPF 解释器仅用于调试构建，而不是生产环境）。

期望每个在 Linux 上运行的 eBPF 程序都能在 Windows 上运行是不现实的。但这与让 eBPF 程序在不同的 Linux 内核版本上运行的挑战并没有太大不同：即使有 CO-RE 支持，内部内核数据结构也可能在版本之间发生更改、添加或删除。eBPF 程序员的工作是优雅地处理这些可能性。

说到 Linux 内核的变化，未来几年我们可以期待在 eBPF 中看到哪些变化呢？

# Linux eBPF 的发展

自 3.15 版以来，eBPF 的功能已经随着几乎每个内核版本的发布而发展。如果您想知道任何给定版本中可用的功能，BCC 项目维护了一个[有用的列表](https://oreil.ly/4H5hU)。我当然期待在未来几年中会有更多的添加。

预测即将发生的事情的最好方法就是听取正在从事该工作的人的意见。例如，在 2022 年 Linux Plumbers Conference 上，eBPF 维护者 Alexei Starovoitov 发表了一次演讲，讨论了他对 eBPF 程序中 C 语言的使用预期会如何发展。我们已经看到 eBPF 从支持几千条指令发展到几乎无限的复杂性，增加了对循环的支持以及不断增加的 BPF 辅助函数集。随着支持的 C 语言中添加了额外的功能，并且有了验证器的支持，eBPF C 可能会发展到允许所有内核模块开发的灵活性，但具有 eBPF 的安全性和动态加载特性。

一些正在讨论和开发的新 eBPF 功能和能力的想法包括：

签名的 eBPF 程序

软件供应链安全是过去几年的热门话题，其中一个关键要素是能够检查您打算运行的程序是否来自预期的来源，并且没有被篡改。一种实现这一点的方法是通常验证伴随程序的加密签名。您可能认为这是内核可以为 eBPF 程序执行的一种操作，也许作为验证步骤的一部分，但不幸的是这并不简单！正如您在本书中所看到的，用户空间加载程序会动态调整程序，提供有关地图位置的信息，以及用于 CO-RE 目的的信息，从签名的角度来看，这很难与恶意修改区分开来。这是 eBPF 社区急于找到解决方案的问题。

长期内核指针

eBPF 程序可以使用辅助函数或 kfunc 检索内核对象的指针，但指针仅在程序执行期间有效。指针不能存储在地图中以供以后检索。[类型指针支持](https://oreil.ly/fWVdo)的想法将在这个领域提供更多的灵活性。

内存分配

对于 eBPF 程序来说，简单调用`kmalloc()`等内存分配函数是不安全的，但有[一个提议建议](https://oreil.ly/Yxxc5)使用 eBPF 特定的替代方法。

你什么时候能够利用新的 eBPF 功能？作为最终用户，你能够利用的功能取决于你在生产中运行的内核版本，正如我在第一章中所讨论的，内核发布需要数年时间才能到达 Linux 的稳定发行版。作为个人，你可能会选择最前沿的内核，但绝大多数运行服务器部署的组织使用稳定、受支持的版本。eBPF 程序员必须考虑到，如果他们编写的代码利用了内核新增的最新功能，这些功能在未来几年内可能无法在大多数生产环境中使用。一些组织可能有足够紧急的需求，值得更快地推出更新的内核版本，以便更早地采用新的 eBPF 功能。

例如，在另一场关于[构建明天的网络](https://oreil.ly/IvPgd)的前瞻性演讲中，Daniel Borkmann 讨论了一个名为*Big TCP*的功能。这是在 Linux 5.19 版本中添加的，以便通过批处理网络数据包在内核中处理，实现 100 GBit/s（甚至更快）的网络速度。大多数 Linux 发行版在未来几年内不会支持这么新的内核，但对于处理大量网络流量的专业组织来说，更早升级可能是值得的。今天将 Big TCP 支持添加到 eBPF 和 Cilium 中，这意味着对于这些大规模用户来说，即使大多数人暂时无法启用它，它也是可用的。

由于 eBPF 允许内核代码动态调整，合理地期望它被用于解决“现场”问题。在第九章中，你了解到使用 eBPF 来缓解内核漏洞；还有工作正在进行，使用 eBPF 来帮助支持硬件设备，比如[人机接口设备](https://oreil.ly/JVYcY)，如鼠标、键盘和游戏控制器。这是在我在第七章中提到的解码红外控制器使用的协议的基础上构建的。

# eBPF 是一个平台，而不是一个功能

将近十年前，热门的新技术是容器，似乎每个人都在谈论它们以及它们将带来的优势。今天，我们对 eBPF 也处于类似的阶段，有很多会议演讲和博客文章——本书中我提到的几篇——赞扬 eBPF 的好处。今天，对许多开发人员来说，容器已经成为日常生活的一部分，无论是使用 Docker 或其他容器运行时在本地运行代码，还是将代码部署到 Kubernetes 环境中。eBPF 是否也会成为每个人的常规工具包的一部分呢？

我认为答案是否定的，或者至少不是直接的。大多数用户不会直接编写 eBPF 程序，也不会使用`bpftool`等工具手动操作它们。但他们将经常与使用 eBPF 构建的工具进行交互，无论是用于性能测量、调试、网络、安全、跟踪，还是其他许多尚未使用 eBPF 实现的功能。用户可能不知道他们在使用 eBPF，就像他们可能不知道当他们使用容器时，他们在使用像命名空间和 cgroups 这样的内核功能一样。

今天，具有 eBPF 知识的项目和供应商强调他们的使用，因为它非常强大，并意味着许多优势。随着基于 eBPF 的项目和产品获得市场份额，eBPF 正在成为基础设施工具的事实上的默认技术平台。

eBPF 编程知识是一种受欢迎但相对罕见的技能，就像今天内核开发比开发商业应用或游戏要少得多一样。如果您喜欢深入系统的底层并且想要构建基本的基础设施工具，eBPF 技能将对您有所帮助。我希望本书对您的 eBPF 之旅有所帮助！

# 结论

恭喜您完成了本书的阅读！

我希望阅读《学习 eBPF》能让您了解 eBPF 的强大之处。也许它已经激发了您自己编写 eBPF 代码或尝试我讨论过的一些工具的兴趣。如果您决定进行一些 eBPF 编程，我希望本书能让您对如何入门有些信心。如果您在阅读本书的过程中完成了练习，那太棒了！

如果你对 eBPF 感兴趣，有很多参与社区的方式。最好的起点是网站[ebpf.io](http://ebpf.io)。这将指引你了解最新的新闻、项目、事件和发生的事情，还有[eBPF Slack](http://ebpf.io/slack)频道，你可能会找到有专业知识的人来回答你可能有的任何问题。

我欢迎您对这篇文章提出反馈、评论和任何更正意见。您可以通过伴随本书的 GitHub 存储库[*github.com/lizrice/learning-ebpf*](https://github.com/lizrice/learning-ebpf)提供您的意见。我也很乐意直接听取您的评论。您可以在互联网的许多地方找到我，用户名为@lizrice。

1. 致谢 Meta 的 Alexei Starovoitov 和 Andrii Nakryiko，以及 Isovalent 的 Daniel Borkmann，他们在 Linux 内核中维护 BPF 子树。

2. [Dave Thaler 在 Linux Plumbers Conference 上介绍了这项标准化工作的进展](https://oreil.ly/4bo6Y)。

3. 嗯，*可能*，但这样做需要微软也以 GPL 许可证发布 Windows 源代码。

4. Alexei Starovoitov 在[这个视频](https://oreil.ly/xunKW)中讨论了 BPF 从受限的 C 语言到扩展和安全 C 的发展历程。
