# 第三章：eBPF 程序的解剖

在上一章中，您看到了使用 BCC 框架编写的简单 eBPF“Hello World”程序。在本章中，有一个完全用 C 编写的“Hello World”程序的示例版本，以便您可以看到 BCC 在幕后处理的一些细节。

本章还向您展示了 eBPF 程序从源代码到执行的过程，如图 3-1 所示。

![C（或 Rust）源代码被编译成 eBPF 字节码，然后被即时编译或解释成本机机器代码指令](img/lebp_0301.png)

###### 图 3-1：C（或 Rust）源代码被编译成 eBPF 字节码，然后被即时编译或解释成本机机器代码指令

eBPF 程序是一组 eBPF 字节码指令。可以直接在这个字节码中编写 eBPF 代码，就像可以用汇编语言编程一样。人们通常发现使用高级编程语言更容易处理，至少在撰写本文时，我会说绝大多数 eBPF 代码是用 C 编写的¹，然后编译成 eBPF 字节码。

从概念上讲，这个字节码在内核中的 eBPF 虚拟机中运行。

# eBPF 虚拟机

eBPF 虚拟机，像任何虚拟机一样，是计算机的软件实现。它接收以 eBPF 字节码指令形式的程序，并且这些指令必须转换为在 CPU 上运行的本机机器指令。

在早期的 eBPF 实现中，字节码指令在内核中被解释执行 - 也就是说，每次 eBPF 程序运行时，内核都会检查指令并将其转换为机器代码，然后执行。出于性能原因和避免 eBPF 解释器中一些 Spectre 相关漏洞的可能性，解释已经在很大程度上被 JIT（即时）编译所取代。*编译*意味着将转换为本机机器指令的过程只发生一次，当程序加载到内核时。

eBPF 字节码由一组指令组成，这些指令作用于（虚拟）eBPF 寄存器。eBPF 指令集和寄存器模型的设计旨在与常见的 CPU 体系结构相匹配，以便从字节码到机器代码的编译或解释步骤相对简单。

## eBPF 寄存器

eBPF 虚拟机使用 10 个通用寄存器，编号从 0 到 9。此外，寄存器 10 用作堆栈帧指针（只能读取，不能写入）。随着 BPF 程序的执行，值被存储在这些寄存器中以跟踪状态。

重要的是要理解，eBPF 虚拟机中的这些 eBPF 寄存器是在软件中实现的。您可以在 Linux 内核源代码的[*include/uapi/linux/bpf.h*头文件](https://oreil.ly/_ZhU2)中从`BPF_REG_0`到`BPF_REG_10`中看到它们的枚举。

在 eBPF 程序的执行开始之前，上下文参数被加载到寄存器 1 中。函数的返回值存储在寄存器 0 中。

在从 eBPF 代码调用函数之前，该函数的参数被放置在寄存器 1 到寄存器 5 中（如果参数少于五个，则不会使用所有寄存器）。

## eBPF 指令

同样的[*linux/bpf.h*头文件](https://oreil.ly/_ZhU2)定义了一个称为`bpf_insn`的结构，它表示一个 BPF 指令：

```cpp
struct `bpf_insn` {
    `__u8` `code`;          /* opcode */                      ![1](assets/1.png) 
    `__u8` `dst_reg`:4;     /* dest register */               ![2](assets/2.png)
    `__u8` `src_reg`:4;     /* source register */
    `__s16` `off`;       /* signed offset */                  ![3](assets/3.png)
    `__s32` `imm`;       /* signed immediate constant */
};
```

①

每个指令都有一个操作码，它定义了指令要执行的操作：例如，将一个值添加到寄存器的内容中，或者跳转到程序中的另一个指令。² Iovisor 项目的[“非官方 eBPF 规范”](https://oreil.ly/FXcPu)中列出了有效指令的列表。

②

不同的操作可能涉及最多两个寄存器。

③

根据操作的不同，可能会有一个偏移值和/或一个“立即”整数值。

这个`bpf_insn`结构是 64 位（或 8 字节）长。然而，有时一条指令可能需要跨越超过 8 个字节。如果你想将一个寄存器设置为 64 位值，你不能以某种方式将该值的所有 64 位挤入结构中，同时还包括操作码和寄存器信息。在这些情况下，指令使用总长度为 16 字节的*宽指令编码*。您将在本章中看到一个例子。

当加载到内核中时，eBPF 程序的字节码由一系列这些`bpf_insn`结构表示。验证器对这些信息执行几项检查，以确保代码可以安全运行。您将在第六章中了解更多关于验证过程的信息。

大多数不同的操作码属于以下类别：

+   将一个值加载到寄存器中（可以是立即值，也可以是从内存或另一个寄存器中读取的值）

+   将寄存器中的值存储到内存中

+   执行算术运算，例如将一个值添加到寄存器的内容中

+   如果满足特定条件，跳转到不同的指令

###### 注意

关于 eBPF 架构的概述，我推荐[BPF 和 XDP 参考指南](https://oreil.ly/rvm1i)，这是 Cilium 项目文档的一部分。如果您想了解更多细节，[内核文档](https://oreil.ly/_2XDT)清楚地描述了 eBPF 指令和编码。

让我们使用另一个 eBPF 程序的简单示例，并跟随它从 C 源代码到 eBPF 字节码再到机器代码指令的过程。

###### 注意

如果您想自己构建和运行这段代码，您将在[*github.com/lizrice/learning-ebpf*](https://github.com/lizrice/learning-ebpf)找到代码以及设置环境的说明。本章的代码在*chapter3*目录中。

本章中的示例是使用一个名为*libbpf*的库以 C 语言编写的。您将在第五章中了解更多关于这个库的信息。

# eBPF“Hello World”用于网络接口

上一章中的示例通过系统调用 kprobe 触发了跟踪“Hello World”；这次我将展示一个 eBPF 程序，当网络数据包到达时触发时，它会写出一行跟踪。

数据包处理是 eBPF 的一个非常常见的应用。我将在第八章中更详细地介绍这一点，但现在了解每个数据包到达网络接口时都会触发的 eBPF 程序的基本思想可能会有所帮助。该程序可以检查甚至修改数据包的内容，并对内核应该如何处理该数据包做出决定（或*verdict*）。裁决可以告诉内核继续像往常一样处理它，丢弃它，或者将其重定向到其他地方。

在我这里展示的简单示例中，程序不对网络数据包进行任何操作；它只是在每次接收到网络数据包时向跟踪管道写出*Hello World*和一个计数器。

示例程序在*chapter3/hello.bpf.c*中。将 eBPF 程序放入以*bpf.c*结尾的文件名中是一个相当常见的约定，以区分它们与可能存在于同一源代码目录中的用户空间 C 代码。这是整个程序：

```cpp
#include <linux/bpf.h>                           ![1](assets/1.png)
#include <bpf/bpf_helpers.h>

int counter = 0;                                 ![2](assets/2.png)

SEC("xdp")                                       ![3](assets/3.png)
int hello(void *ctx) {                           ![4](assets/4.png)
    bpf_printk("Hello World %d", counter);
    counter++;
    return XDP_PASS;
}

char LICENSE[] SEC("license") = "Dual BSD/GPL";  ![5](assets/5.png)
```

①

这个例子首先包括一些头文件。以防您不熟悉 C 编码，每个程序都必须包括定义程序将使用的任何结构或函数的头文件。您可以从这些头文件的名称猜出，这些头文件与 BPF 相关。

②

这个例子展示了 eBPF 程序如何使用全局变量。每次程序运行时，这个计数器都会增加。

③

宏`SEC()`定义了一个名为`xdp`的部分，您将能够在编译后的对象文件中看到它。我将在第五章中回到部分名称的用法，但现在您可以简单地将其视为定义为 eXpress Data Path (XDP)类型的 eBPF 程序。

④

在这里，您可以看到实际的 eBPF 程序。在 eBPF 中，程序名称是函数名称，因此此程序称为`hello`。它使用一个辅助函数`bpf_printk`来写入一串文本，增加全局变量`counter`，然后返回值`XDP_PASS`。这是向内核指示应像通常一样处理此网络数据包的判决。

⑤

最后还有另一个`SEC()`宏，它定义了一个许可字符串，这是 eBPF 程序的一个关键要求。内核中的一些 BPF 辅助函数被定义为“仅限 GPL”。如果您想使用这些函数中的任何一个，您的 BPF 代码必须声明为具有 GPL 兼容许可。验证器（我们将在第六章中讨论）将在声明的许可与程序使用的函数不兼容时提出异议。某些 eBPF 程序类型，包括使用 BPF LSM 的程序（您将在第九章中了解），也需要[兼容 GPL](https://oreil.ly/ItahV)。

###### 注意

也许您会想为什么上一章使用了`bpf_trace_printk()`，而这个版本使用了`bpf_printk()`。简短的答案是 BCC 的版本称为`bpf_trace_printk()`，而*libbpf*的版本称为`bpf_printk()`，但这两者都是内核函数`bpf_trace_printk()`的包装器。Andrii Nakryiko 在他的博客上写了一篇[很好的文章](https://oreil.ly/9mNSY)。

这是一个附加到网络接口上的 XDP 挂钩点的 eBPF 程序示例。您可以将 XDP 事件视为在（物理或虚拟）网络接口上入站到达网络数据包的瞬间触发。

###### 注意

一些网络卡支持卸载 XDP 程序，以便它们可以在网络卡上执行。这意味着每个到达的网络数据包都可以在接近机器 CPU 之前在卡上处理。XDP 程序可以检查甚至修改每个网络数据包，因此这对于执行诸如 DDoS 保护、防火墙或负载平衡等操作非常有用。您将在第八章中了解更多信息。

你已经看到了 C 源代码，所以下一步是将其编译成内核可以理解的对象。

# 编译 eBPF 对象文件

我们的 eBPF 源代码需要编译成 eBPF 虚拟机可以理解的机器指令：eBPF 字节码。如果您指定了`-target bpf`，则来自[LLVM 项目](https://llvm.org)的 Clang 编译器将执行此操作。以下是一个 Makefile 的摘录，它将执行编译：

```cpp
hello.bpf.o: %.o: %.c
   clang \ -target bpf \ `-I/usr/include/`$(``shell` `uname` -`m``)`-linux-gnu \ `-g \ `-O2 -c `$<` -o `$@````

```cpp

 ```这将从*hello.bpf.c*中的源代码生成一个名为*hello.bpf.o*的对象文件。这里的`-g`标志是可选的，但它会生成调试信息，这样当您检查对象文件时，您可以看到源代码和字节码。让我们检查一下这个对象文件，以更好地理解它包含的 eBPF 代码。```cpp  ```#检查 eBPF 对象文件

文件实用程序通常用于确定文件的内容：

```cpp
$ file hello.bpf.o
hello.bpf.o: ELF 64-bit LSB relocatable, eBPF, version 1 (SYSV), with debug_info,
not stripped
```

这显示它是一个 ELF（可执行和可链接格式）文件，包含 64 位平台的 eBPF 代码，具有 LSB（最低有效位）架构。如果您在编译步骤中使用了`-g`标志，则包括调试信息。

您可以使用`llvm-objdump`进一步检查此对象，以查看 eBPF 指令：

```cpp
$ llvm-objdump -S hello.bpf.o
```

即使您不熟悉反汇编，此命令的输出也不难理解：

```cpp
hello.bpf.o:    file format elf64-bpf               ![1](assets/1.png)

Disassembly of section xdp:                         ![2](assets/2.png)

0000000000000000 <hello>:                           ![3](assets/3.png)
;  bpf_printk("Hello World %d", counter");          ![4](assets/4.png) 
    0:   18 06 00 00 00 00 00 00 00 00 00 00 00 00 00 00 r6 = 0 ll
    2:   61 63 00 00 00 00 00 00 r3 = *(u32 *)(r6 + 0)
    3:   18 01 00 00 00 00 00 00 00 00 00 00 00 00 00 00 r1 = 0 ll
    5:   b7 02 00 00 0f 00 00 00 r2 = 15
    6:   85 00 00 00 06 00 00 00 call 6
;  counter++;                                       ![5](assets/5.png)
    7:   61 61 00 00 00 00 00 00 r1 = *(u32 *)(r6 + 0)
    8:   07 01 00 00 01 00 00 00 r1 += 1
    9:   63 16 00 00 00 00 00 00 *(u32 *)(r6 + 0) = r1
;  return XDP_PASS;                                 ![6](assets/6.png)
   10:   b7 00 00 00 02 00 00 00 r0 = 2
   11:   95 00 00 00 00 00 00 00 exit
```

①

第一行进一步确认*hello.bpf.o*是一个包含 eBPF 代码的 64 位 ELF 文件（一些工具使用术语*BPF*，而另一些使用*eBPF*并没有特定的原因；正如我之前所说，这些术语现在几乎可以互换使用）。

②

接下来是标记为`xdp`的部分的反汇编，与 C 源代码中的`SEC()`定义相匹配。

③

这部分是一个名为`hello`的函数。

④

有五行 eBPF 字节码指令对应于源代码行`bpf_printk("Hello World %d", counter");`。

⑤

三行 eBPF 字节码指令增加了`counter`变量。

![6](img/6.png)

另外两行字节码是从源代码`return XDP_PASS;`生成的。

除非你特别想这样做，否则没有真正的必要去理解每一行字节码与源代码的关系。编译器负责生成字节码，这样你就不必考虑它！但让我们更详细地检查输出，这样你就可以感受到这个输出与本章前面学到的 eBPF 指令和寄存器的关系。

在每行字节码的左侧，你可以看到该指令与内存中`hello`的偏移量。正如本章前面描述的那样，eBPF 指令通常是 8 个字节长，在 64 位平台上，每个内存位置可以容纳 8 个字节，因此每条指令的偏移量通常会递增 1。然而，这个程序中的第一条指令恰好是一个宽指令编码，需要 16 个字节来将寄存器 6 设置为 64 位值`0`。这将该指令放在输出的第二行，偏移量为`2`。之后是另一条 16 字节的指令，将寄存器 1 设置为 64 位值`0`。之后，剩下的指令每条都占 8 个字节，因此偏移量在每行递增 1。

每行的第一个字节是操作码，告诉内核要执行什么操作，而指令行的右侧是指令的人类可读解释。在撰写本文时，Iovisor 项目拥有最完整的[eBPF 操作码文档](https://oreil.ly/nLbLp)，但官方的[Linux 内核文档](https://oreil.ly/yp-jW)正在迎头赶上，而 eBPF 基金会正在制定[标准文档](https://oreil.ly/7ZWzj)，这些文档不与特定操作系统绑定。

例如，让我们看看偏移量为`5`的指令，看起来是这样的：

```cpp
    5:   b7 02 00 00 0f 00 00 00 r2 = 15
```

操作码是`0xb7`，文档告诉我们对应的伪代码是`dst = imm`，可以理解为“将目的地设置为立即值”。目的地由第二个字节定义，为`0x02`，意思是“寄存器 2”。这里的“立即”（或文字）值是`0x0f`，在十进制中是 15。所以我们可以理解这个指令告诉内核“将寄存器 2 设置为值 15”。这对应于我们在指令右侧看到的输出：`r2 = 15`。

偏移量为`10`的指令类似：

```cpp
   10:   b7 00 00 00 02 00 00 00 r0 = 2
```

这行指令的操作码也是`0xb7`，这次是将寄存器 0 的值设置为`2`。当一个 eBPF 程序运行结束时，寄存器 0 保存返回码，而`XDP_PASS`的值为`2`。这与源代码相匹配，源代码总是返回`XDP_PASS`。

你现在知道*hello.bpf.o*包含了一个字节码的 eBPF 程序。下一步是将它加载到内核中。

# 将程序加载到内核中

在这个例子中，我们将使用一个叫做`bpftool`的实用工具。你也可以以编程方式加载程序，你将在本书的后面看到相关示例。

###### 注意

一些 Linux 发行版提供了包含`bpftool`的软件包，或者您可以[从源代码编译](https://github.com/libbpf/bpftool)。您可以在[Quentin Monnet 的博客](https://oreil.ly/Yqepv)上找到有关安装或构建此工具的更多详细信息，以及[Cilium 网站](https://oreil.ly/rnTIg)上的其他文档和用法。

以下是使用`bpftool`将程序加载到内核的示例。请注意，您可能需要 root 权限（或使用`sudo`）来获取`bpftool`所需的 BPF 权限。

```cpp
$ bpftool prog load hello.bpf.o /sys/fs/bpf/hello
```

这将从我们编译的对象文件中加载 eBPF 程序，并将其“固定”到位置*/sys/fs/bpf/hello*。⁴此命令没有输出响应表示成功，但您可以使用`ls`确认程序已经就位：

```cpp
$ ls /sys/fs/bpf
hello
```

eBPF 程序已成功加载。让我们使用`bpftool`实用程序来了解更多关于程序及其在内核中的状态的信息。

# 检查已加载的程序

`bpftool`实用程序可以列出加载到内核中的所有程序。如果您自己尝试这样做，您可能会在输出中看到几个预先存在的 eBPF 程序，但为了清晰起见，我只会显示与我们的“Hello World”示例相关的行：

```cpp
$ bpftool prog list 
...
540: xdp  name hello  tag d35b94b4c0c10efb  gpl
        loaded_at 2022-08-02T17:39:47+0000  uid 0
        xlated 96B  jited 148B  memlock 4096B  map_ids 165,166
        btf_id 254
```

该程序已被分配 ID 540。这个标识是在加载每个程序时分配的一个数字。知道了 ID，您可以要求`bpftool`显示有关该程序的更多信息。这一次，让我们以美化的 JSON 格式获取输出，以便字段名称和值都是可见的：

```cpp
$ bpftool prog show id 540 --pretty
{
    "id": 540,
    "type": "xdp",
    "name": "hello",
    "tag": "d35b94b4c0c10efb",
    "gpl_compatible": true,
    "loaded_at": 1659461987,
    "uid": 0,
    "bytes_xlated": 96,
    "jited": true,
    "bytes_jited": 148,
    "bytes_memlock": 4096,
    "map_ids": [165,166
    ],
    "btf_id": 254
}
```

鉴于字段名称，这些内容大部分都很容易理解：

+   程序的 ID 是 540。

+   `type`字段告诉我们，该程序可以使用 XDP 事件附加到网络接口。还有其他几种类型的 BPF 程序可以附加到不同类型的事件上，我们将在第七章中更详细地讨论这一点。

+   程序的名称是`hello`，这是源代码中的函数名。

+   `tag`是这个程序的另一个标识符，我将在稍后更详细地描述。

+   该程序是使用 GPL 兼容许可证定义的。

+   有一个时间戳显示程序加载的时间。

+   用户 ID 0（即 root）加载了该程序。

+   该程序中有 96 字节的已翻译的 eBPF 字节码，我马上会向您展示。

+   该程序已经进行了 JIT 编译，编译结果是 148 字节的机器代码。我稍后也会介绍这个。

+   `bytes _memlock`字段告诉我们，该程序保留了 4096 字节的内存，不会被分页出去。

+   该程序引用了 ID 为 165 和 166 的 BPF 映射。这可能令人惊讶，因为源代码中并没有明显的映射引用。稍后在本章中，您将看到如何使用映射语义来处理 eBPF 程序中的全局数据。

+   您将在第五章中了解 BTF，但现在只需知道`btf_id`表示该程序有一块 BTF 信息。只有在使用`-g`标志进行编译时，此信息才包含在对象文件中。

## BPF 程序标签

`tag`是程序指令的 SHA（安全哈希算法）总和，可以用作程序的另一个标识符。ID 每次加载或卸载程序时都可能会有所变化，但标签将保持不变。`bpftool`实用程序接受对 BPF 程序的引用，可以通过 ID、名称、标签或固定路径进行引用，因此在这个例子中，以下所有内容都将给出相同的输出：

+   `bpftool prog show id 540`

+   `bpftool prog show name hello`

+   `bpftool prog show tag d35b94b4c0c10efb`

+   `bpftool prog show pinned /sys/fs/bpf/hello`

您可以拥有多个具有相同名称的程序，甚至可以拥有具有相同标签的程序的多个实例，但 ID 和固定路径将始终是唯一的。

## 已翻译的字节码

`bytes_xlated`字段告诉我们“翻译”eBPF 代码有多少字节。这是通过验证器后的 eBPF 字节码（可能已经被内核修改，我将在本书后面讨论原因）。

让我们使用`bpftool`来显示我们的“Hello World”代码的翻译版本：

```cpp
$ bpftool prog dump xlated name hello 
int hello(struct xdp_md * ctx):
; bpf_printk("Hello World %d", counter);
   0: (18) r6 = map[id:165][0]+0
   2: (61) r3 = *(u32 *)(r6 +0)
   3: (18) r1 = map[id:166][0]+0
   5: (b7) r2 = 15
   6: (85) call bpf_trace_printk#-78032
; counter++; 
   7: (61) r1 = *(u32 *)(r6 +0)
   8: (07) r1 += 1
   9: (63) *(u32 *)(r6 +0) = r1
; return XDP_PASS;
  10: (b7) r0 = 2
  11: (95) exit
```

这看起来与您之前从`llvm-objdump`的输出中看到的反汇编代码非常相似。偏移地址相同，指令看起来相似——例如，我们可以看到偏移`5`处的指令是`r2=15`。

## JIT 编译的机器码

翻译后的字节码非常低级，但还不是机器码。eBPF 使用 JIT 编译器将 eBPF 字节码转换为在目标 CPU 上本地运行的机器码。`bytes_jited`字段显示，在此转换后，程序长度为 108 字节。

###### 注意

为了获得更高的性能，eBPF 程序通常是 JIT 编译的。另一种选择是在运行时解释 eBPF 字节码。eBPF 指令集和寄存器设计得相当接近本机机器指令，使得这种解释相对简单且相对快速，但编译的程序将更快，大多数架构现在都支持 JIT。

`bpftool`实用程序可以生成汇编语言中的 JIT 代码转储。如果您对汇编语言不熟悉，这看起来可能完全难以理解！我只是为了说明 eBPF 代码从源代码到可执行机器指令经历的所有转换。以下是命令及其输出：

```cpp
$ bpftool prog dump jited name hello 
int hello(struct xdp_md * ctx):
bpf_prog_d35b94b4c0c10efb_hello:
; bpf_printk("Hello World %d", counter);
   0:   hint    #34
   4:   stp     x29, x30, [sp, #-16]!
   8:   mov     x29, sp
   c:   stp     x19, x20, [sp, #-16]!
  10:   stp     x21, x22, [sp, #-16]!
  14:   stp     x25, x26, [sp, #-16]!
  18:   mov     x25, sp
  1c:   mov     x26, #0
  20:   hint    #36
  24:   sub     sp, sp, #0
  28:   mov     x19, #-140733193388033
  2c:   movk    x19, #2190, lsl #16
  30:   movk    x19, #49152
  34:   mov     x10, #0
  38:   ldr     w2, [x19, x10]
  3c:   mov     x0, #-205419695833089
  40:   movk    x0, #709, lsl #16
  44:   movk    x0, #5904
  48:   mov     x1, #15
  4c:   mov     x10, #-6992
  50:   movk    x10, #29844, lsl #16
  54:   movk    x10, #56832, lsl #32
  58:   blr     x10
  5c:   add     x7, x0, #0
; counter++; 
  60:   mov     x10, #0
  64:   ldr     w0, [x19, x10]
  68:   add     x0, x0, #1
  6c:   mov     x10, #0
  70:   str     w0, [x19, x10]
; return XDP_PASS;
  74:   mov     x7, #2
  78:   mov     sp, sp
  7c:   ldp     x25, x26, [sp], #16
  80:   ldp     x21, x22, [sp], #16
  84:   ldp     x19, x20, [sp], #16
  88:   ldp     x29, x30, [sp], #16
  8c:   add     x0, x7, #0
  90:   ret
```

###### 注意

一些打包的`bpftool`发行版尚未包含支持转储 JIT 输出的功能，如果是这种情况，您将看到“错误：没有 libbfd 支持”。您可以按照[*https://github.com/libbpf/bpftool*](https://github.com/libbpf/bpftool)上的说明自行构建`bpftool`。

您已经看到“Hello World”程序已加载到内核中，但此时它尚未与事件关联，因此不会触发运行。它需要附加到一个事件上。

# 附加到事件

程序类型必须与其附加的事件类型匹配；您将在第七章中了解更多信息。在本例中，这是一个 XDP 程序，您可以使用`bpftool`将示例 eBPF 程序附加到网络接口上的 XDP 事件，如下所示：

```cpp
$ bpftool net attach xdp id 540 dev eth0
```

###### 注意

在撰写本文时，`bpftool`实用程序不支持附加所有程序类型的能力，但它已经[最近扩展](https://oreil.ly/Tt99p)以自动附加 k(ret)probes、u(ret)probes 和 tracepoints。

在这里，我使用了 ID 为 540 的程序，但您也可以使用名称（前提是唯一的）或标签来标识要附加的程序。在此示例中，我已将程序附加到网络接口`eth0`。

您可以使用`bpftool`查看所有网络连接的 eBPF 程序：

```cpp
$ bpftool net list 
xdp:
eth0(2) driver id 540

tc:

flow_dissector:
```

ID 为 540 的程序附加到`eth0`接口的 XDP 事件上。此输出还提供了有关网络堆栈中其他一些潜在事件的一些线索，您可以将 eBPF 程序附加到`tc`和`flow_dissector`上。在第七章中会详细介绍。

您还可以使用`ip link`检查网络接口，您将看到类似以下内容的输出（为了清晰起见，已删除了一些细节）：

```cpp
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT
group default qlen 1000
    ...
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 xdp qdisc fq_codel state UP
mode DEFAULT group default qlen 1000
    ...
    prog/xdp id 540 tag 9d0e949f89f1a82c jited 
    ...
```

在此示例中有两个接口：用于将流量发送到此计算机上的进程的环回接口`lo`；以及连接此计算机与外部世界的`eth0`接口。此输出还显示`eth0`有一个 JIT 编译的 eBPF 程序，其标识为`540`，标签为`9d0e949f89f1a82c`，附加到其 XDP 挂钩上。

###### 注意

您还可以使用`ip link`将 XDP 程序附加到网络接口并将其分离。我已经在本章末尾包括了这个作为一个练习，并且在第七章中还有更多的例子。

此时，*hello* eBPF 程序应该在每次接收到网络数据包时产生跟踪输出。您可以通过运行`cat /sys/kernel/debug/tracing/trace_pipe`来检查这一点。这应该显示很多类似于这样的输出：

```cpp
<idle>-0       [003] d.s.. 655370.944105: bpf_trace_printk: Hello World 4531
<idle>-0       [003] d.s.. 655370.944587: bpf_trace_printk: Hello World 4532
<idle>-0       [003] d.s.. 655370.944896: bpf_trace_printk: Hello World 4533
```

如果您忘记了跟踪管道的位置，您可以使用命令`bpftool prog tracelog`来获得相同的输出。

与您在第二章中看到的输出相比，这一次没有与每个事件相关的命令或进程 ID；相反，您在跟踪的每一行开头看到`<idle>-0`。在第二章中，每个系统调用事件发生是因为一个在用户空间执行命令的进程调用了系统调用 API。该进程 ID 和命令是 eBPF 程序执行的上下文的一部分。但在这个例子中，XDP 事件是由网络数据包的到达引起的。与这个数据包相关的没有用户空间进程—在触发*hello* eBPF 程序时，系统对数据包除了在内存中接收它之外还没有做任何事情，也不知道数据包是什么或者它要去哪里。

您可以看到被跟踪的计数器值每次增加 1，这是预期的。在源代码中，`counter`是一个全局变量。让我们看看在 eBPF 中如何实现这一点。

# 全局变量

正如您在上一章中学到的，eBPF 映射是一种数据结构，可以从 eBPF 程序或用户空间访问。由于同一个映射可以被同一个程序的不同运行重复访问，因此它可以用于保存从一次执行到下一次执行的状态。多个程序也可以访问同一个映射。由于这些特性，映射语义可以被重新用作全局变量。

###### 注

在[2019 年添加对全局变量的支持之前](https://oreil.ly/IDftt)，eBPF 程序员必须显式编写映射来执行相同的任务。

您之前看到`bpftool`显示了这个示例程序，使用了两个具有标识符 165 和 166 的映射。(如果您自己尝试，可能会看到不同的标识符，因为这些标识符是在内核中创建映射时分配的。) 让我们来探索这些映射中的内容。

`bpftool`实用程序可以显示加载到内核中的映射。为了清晰起见，我只会显示与示例“Hello World”程序相关的条目 165 和 166：

```cpp
$ bpftool map list
165: array  name hello.bss  flags 0x400
        key 4B  value 4B  max_entries 1  memlock 4096B
        btf_id 254
166: array  name hello.rodata  flags 0x80
        key 4B  value 15B  max_entries 1  memlock 4096B
        btf_id 254  frozen
```

从 C 程序编译的对象文件中的 bss⁶部分通常保存全局变量，您可以使用`bpftool`来检查其内容，就像这样：

```cpp
$ bpftool map dump name hello.bss
[{
        "value": {
            ".bss": [{
                    "counter": 11127
                }
            ]
        }
    }
]
```

我也可以使用`bpftool map dump id 165`来检索相同的信息。如果我再次运行这些命令中的任何一个，我会看到计数器已经增加，因为每次接收到网络数据包时程序都会运行。

正如您将在第五章中了解到的，`bpftool`能够在映射中漂亮地打印字段名称(这里是变量名`counter`)，只有在 BTF 信息可用时才能打印出来，而这些信息只有在使用`-g`标志进行编译时才包含在内。如果在编译步骤中省略了该标志，您将看到更像这样的东西：

```cpp
$ bpftool map dump name hello.bss
key: 00 00 00 00  value: 19 01 00 00
Found 1 element
```

没有 BTF 信息，`bpftool`无法知道源代码中使用的变量名。您可以推断，由于这个映射中只有一个项目，十六进制值`19 01 00 00`必须是`counter`的当前值(281 的十进制，因为字节是从最不重要的字节开始排序的)。

You’ve seen here that the eBPF program uses the semantics of a map to read and write to a global variable. Maps are also used to hold static data, as you can see by inspecting the other map.

The fact that the other map is named `hello.rodata` gives a hint that this could be read-only data related to our *hello* program. You can dump the contents of this map to see that it holds the string used by the eBPF program for tracing:

```cpp
$ bpftool map dump name hello.rodata
[{
        "value": {
            ".rodata": [{
                    "hello.____fmt": "Hello World %d"
                }
            ]
        }
    }
]
```

If you didn’t compile the object with the `-g` flag, you’ll see output that looks like this:

```cpp
$ bpftool map dump id 166
key: 00 00 00 00  value: 48 65 6c 6c 6f 20 57 6f  72 6c 64 20 25 64 00
Found 1 element
```

There is one key–value pair in this map, and the value contains 12 bytes of data ending with a 0\. It probably won’t surprise you that those bytes are the ASCII representation of the string `"Hello World %d"`.

Now that we’ve finished inspecting this program and its maps, it’s time to clean it up. We’ll start by detaching it from the event that triggers it.

# Detaching the Program

You can detach the program from the network interface like this:

```cpp
$ bpftool net detach xdp dev eth0
```

There is no output if this command runs successfully, but you can confirm that the program is no longer attached by the lack of XDP entries in the output from `bpftool net list`:

```cpp
$ bpftool net list 
xdp:

tc:

flow_dissector:
```

However, the program is still loaded into the kernel:

```cpp
$ bpftool prog show name hello 
395: xdp  name hello  tag 9d0e949f89f1a82c  gpl
        loaded_at 2022-12-19T18:20:32+0000  uid 0
        xlated 48B  jited 108B  memlock 4096B  map_ids 4
```

# Unloading the Program

There’s no inverse of `bpftool prog load` (at least not at the time of this writing), but you can remove the program from the kernel by deleting the pinned pseudofile:

```cpp
$ rm /sys/fs/bpf/hello
$ bpftool prog show name hello
```

There is no output from this `bpftool` command because the program is no longer loaded in the kernel.

# BPF to BPF Calls

In the previous chapter you saw tail calls in action, and I mentioned that now there is also the ability to call functions from within an eBPF program. Let’s take a look at a simple example, which, like the tail call example, can be attached to the `sys_enter` tracepoint, except this time it will trace out the opcode for the syscall. You’ll find the code in *chapter3/hello-func.bpf.c*.

For illustrative purposes I have written a very simple function that extracts the syscall opcode from the tracepoint arguments:

```cpp
static __attribute((noinline)) int get_opcode(struct bpf_raw_tracepoint_args 
                                                                         *ctx) { `return` `ctx``->``args``[``1``];` ``}``
```

Given the choice, the compiler would probably inline this very simple function that I’m only going to call from one place. Since that would defeat the point of this example, I have added `__attribute((noinline))` to force the compiler’s hand. In normal circumstances you should probably omit this and allow the compiler to optimize as it sees fit.

The eBPF function that calls this function looks like this:

```cppGiven the choice, the compiler would probably inline this very simple function that I’m only going to call from one place. Since that would defeat the point of this example, I have added `__attribute((noinline))` to force the compiler’s hand. In normal circumstances you should probably omit this and allow the compiler to optimize as it sees fit.

The eBPF function that calls this function looks like this:

```

After compiling this to an eBPF object file, you can load it into the kernel and confirm that it is loaded with `bpftool`:

```cpp``
```

The interesting part of this exercise is inspecting the eBPF bytecode to see the `get_opcode()` function:

```cppAfter compiling this to an eBPF object file, you can load it into the kernel and confirm that it is loaded with `bpftool`:

```

①

Here you can see the `hello()` eBPF program making a call to `get_opcode()`. The eBPF instruction at offset `0` is `0x85`, which from the instruction set documentation corresponds to “Function call.” Instead of executing the next instruction, which would be at offset 1, execution will jump seven instructions ahead (`pc+7`), which means the instruction at offset `8`.

②

Here’s the bytecode for `get_opcode()`, and as you might hope, the first instruction is at offset `8`.

The function call instruction necessitates putting the current state on the eBPF virtual machine’s stack so that when the called function exits, execution can continue in the calling function. Since the stack size is limited to 512 bytes, BPF to BPF calls can’t be very deeply nested.

###### Note

For a lot more detail on tail calls and BPF to BPF calls, there’s an excellent post by Jakub Sitnicki on Cloudflare’s blog: [“Assembly within! BPF tail calls on x86 and ARM”](https://oreil.ly/6kOp3).

# Summary

In this chapter you saw some example C source code transformed into eBPF bytecode and then compiled to machine code so that it’s ready to be executed in the kernel. You also learned how to use `bpftool` to inspect programs and maps loaded into the kernel, and to attach to XDP events.

In addition, you saw examples of different types of eBPF programs triggered by different kinds of events. An XDP event is triggered by the arrival of a packet of data on a network interface, whereas kprobe and tracepoint events are triggered by hitting some particular point in kernel code. I’ll discuss some other eBPF program types in [Chapter 7](ch07.html#ebpf_program_and_attachment_types).

You also learned how maps are used to implement global variables for eBPF programs, and you saw BPF to BPF function calls.

下一章将更详细地介绍当`bpftool`—或任何其他用户空间代码—加载程序并将其附加到事件时，在系统调用级别发生了什么。

# 练习

如果您想进一步探索 BPF 程序，可以尝试以下几种方法：

1.  尝试使用类似以下的`ip link`命令来附加和分离 XDP 程序：

```cpp

The interesting part of this exercise is inspecting the eBPF bytecode to see the `get_opcode()` function:

```

1.  运行第二章中的任何 BCC 示例。在程序运行时，使用第二个终端窗口使用`bpftool`检查加载的程序。以下是我通过运行*hello-map.py*示例看到的示例：

```cpp

[![1](assets/1.png)](#code_id_3_15)

Here you can see the `hello()` eBPF program making a call to `get_opcode()`. The eBPF instruction at offset `0` is `0x85`, which from the instruction set documentation corresponds to “Function call.” Instead of executing the next instruction, which would be at offset 1, execution will jump seven instructions ahead (`pc+7`), which means the instruction at offset `8`.

[![2](assets/2.png)](#code_id_3_16)

Here’s the bytecode for `get_opcode()`, and as you might hope, the first instruction is at offset `8`.

The function call instruction necessitates putting the current state on the eBPF virtual machine’s stack so that when the called function exits, execution can continue in the calling function. Since the stack size is limited to 512 bytes, BPF to BPF calls can’t be very deeply nested.

###### Note

For a lot more detail on tail calls and BPF to BPF calls, there’s an excellent post by Jakub Sitnicki on Cloudflare’s blog: [“Assembly within! BPF tail calls on x86 and ARM”](https://oreil.ly/6kOp3).```

您还可以使用`bpftool prog dump`命令来查看这些程序的字节码和机器码版本。

1.  运行*chapter2*目录中的*hello-tail.py*，在其运行时，查看它加载的程序。您会看到每个尾调用程序都单独列出，就像这样：

```cpp  ```

你还可以使用`bpftool prog dump xlated`来查看字节码指令，并将其与“BPF to BPF Calls”中所见的进行比较。

1.  *小心使用这个，最好只是考虑为什么会发生这种情况而不是尝试它！*如果从 XDP 程序返回`0`值，则对应于`XDP_ABORTED`，这告诉内核中止对此数据包的任何进一步处理。这可能看起来有点违反直觉，因为在 C 中`0`值通常表示成功，但事实就是如此。因此，如果尝试修改程序以返回`0`并将其附加到虚拟机的`eth0`接口，所有网络数据包都将被丢弃。如果您正在使用 SSH 连接到该机器，这将有些不幸，您可能需要重新启动机器才能恢复访问！

您可以在容器中运行程序，以便 XDP 程序附加到仅影响该容器而不是整个虚拟机的虚拟以太网接口。在[*https://github.com/lizrice/lb-from-scratch*](https://github.com/lizrice/lb-from-scratch)中有一个示例。

¹随着 Rust 编译器支持 eBPF 字节码作为目标，越来越多的 eBPF 程序也是用 Rust 编写的。

²有一些指令的操作会受到指令中其他字段值的“修改”的影响。例如，在内核 5.12 中引入了一组[原子指令](https://oreil.ly/oyTI7)，其中包括一个在`imm`字段中指定的算术操作（`ADD`、`AND`、`OR`、`XOR）。

³使用`-g`标志是为了生成 CO-RE eBPF 程序所需的 BTF 信息，我将在第五章中介绍。

⁴一般来说，这是可选的——eBPF 程序可以加载到内核中而不被固定到文件位置，但对于`bpftool`来说是不可选的，它总是必须固定它加载的程序。这样做的原因在[“BPF Program and Map References”](ch04.html#bpf_program_and_map_references)中有进一步介绍。

⁵内核设置`CONFIG_BPF_JIT`需要启用 JIT 编译才能发挥作用，并且可以使用`net.core.bpf_jit_enable sysctl`设置在运行时启用或禁用。有关不同芯片架构上 JIT 支持的更多信息，请参见[文档](https://oreil.ly/4-xi6)。

⁶这里，“bss”代表“由符号开始的块”。
