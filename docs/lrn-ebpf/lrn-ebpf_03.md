# 第三章：eBPF 程序解剖

在前一章中，您看到了使用 BCC 框架编写的简单 eBPF “Hello World” 程序。本章还展示了一个完全用 C 编写的“Hello World” 程序版本，以便查看 BCC 在幕后处理的一些细节。

本章还展示了 eBPF 程序从源代码到执行过程中的各个阶段，如 图 3-1 所示。

![C（或 Rust）源代码编译成 eBPF 字节码，然后 JIT 编译或解释成本地机器码指令](img/lebp_0301.png)

###### 图 3-1\. C（或 Rust）源代码编译成 eBPF 字节码，然后 JIT 编译或解释成本地机器码指令

一个 eBPF 程序是一组 eBPF 字节码指令。可以直接在此字节码中编写 eBPF 代码，就像可以使用汇编语言编程一样。人类通常更喜欢使用高级编程语言来处理，至少在撰写本文时，我可以说绝大多数 eBPF 代码是用 C^(1) 编写并编译成 eBPF 字节码。

在概念上，这些字节码在内核中的 eBPF 虚拟机中运行。

# eBPF 虚拟机

与任何虚拟机一样，eBPF 虚拟机是计算机的软件实现。它接收 eBPF 字节码指令形式的程序，并将其转换为在 CPU 上运行的本地机器指令。

在 eBPF 的早期实现中，内核内部解释了字节码指令——也就是说，每次运行 eBPF 程序时，内核都会检查指令并将其转换为机器码，然后执行。出于性能和避免 eBPF 解释器中一些与 Spectre 相关的漏洞的考虑，解释已大部分被 JIT（即时编译）替代。*编译*意味着程序加载到内核时，将字节码转换为本地机器指令，仅需进行一次。

eBPF 字节码包含一组指令，这些指令作用于（虚拟的）eBPF 寄存器。eBPF 指令集和寄存器模型设计得非常符合常见的 CPU 架构，因此从字节码到机器码的编译或解释步骤相对直接。

## eBPF 寄存器

eBPF 虚拟机使用 10 个通用寄存器，编号从 0 到 9。此外，寄存器 10 用作堆栈帧指针（只能读取，不能写入）。在执行 BPF 程序时，这些寄存器中存储的值用于跟踪状态。

理解的重点是，eBPF 虚拟机中的这些 eBPF 寄存器是通过软件实现的。你可以在 Linux 内核源代码的 [*include/uapi/linux/bpf.h* 头文件](https://oreil.ly/_ZhU2)中看到它们从 `BPF_REG_0` 到 `BPF_REG_10` 的枚举。

在执行开始之前，eBPF 程序的上下文参数被加载到寄存器 1 中。函数的返回值存储在寄存器 0 中。

在从 eBPF 代码调用函数之前，该函数的参数被放置在寄存器 1 到寄存器 5 中（如果少于五个参数，则不使用所有寄存器）。

## eBPF 指令

同样的 [*linux/bpf.h* 头文件](https://oreil.ly/_ZhU2)定义了一个称为 `bpf_insn` 的结构，表示一个 BPF 指令：

```
struct `bpf_insn` {
    `__u8` `code`;          /* opcode */                      ![1](img/1.png) 
    `__u8` `dst_reg`:4;     /* dest register */               ![2](img/2.png)
    `__u8` `src_reg`:4;     /* source register */
    `__s16` `off`;       /* signed offset */                  ![3](img/3.png)
    `__s32` `imm`;       /* signed immediate constant */
};
```

![1](img/#code_id_3_1)

每个指令都有一个操作码，定义了指令要执行的操作，例如将一个值添加到寄存器的内容中，或者跳转到程序中的另一个指令。^(2) Iovisor 项目的[“非官方 eBPF 规范”](https://oreil.ly/FXcPu)列出了有效指令的列表。

![2](img/#code_id_3_2)

不同的操作可能涉及最多两个寄存器。

![3](img/#code_id_3_3)

根据操作的不同，可能存在偏移值和/或“立即”整数值。

这个 `bpf_insn` 结构长达 64 位（或 8 字节）。然而，有时一条指令可能需要跨越超过 8 字节。如果要将寄存器设置为 64 位值，不可能将所有 64 位值与操作码和寄存器信息一起挤入结构中。在这些情况下，该指令使用了 *宽指令编码*，总长为 16 字节。你将在本章看到一个例子。

当加载到内核中时，eBPF 程序的字节码由一系列 `bpf_insn` 结构表示。验证器对此信息执行多项检查，以确保代码可以安全运行。你将在第六章了解更多关于验证过程的信息。

大多数不同的操作码属于以下类别之一：

+   将值加载到寄存器中（可以是立即值，也可以是从内存或另一个寄存器中读取的值）

+   将寄存器中的值存储到内存中

+   执行算术操作，例如将一个值添加到寄存器的内容中

+   如果满足特定条件，跳转到另一个指令

###### 注意

关于 eBPF 架构的概述，我推荐阅读 [BPF 和 XDP 参考指南](https://oreil.ly/rvm1i)，它作为 Cilium 项目文档的一部分包含在内。如果你想要更多细节，[内核文档](https://oreil.ly/_2XDT)清楚地描述了 eBPF 指令和编码。

让我们以另一个简单的 eBPF 程序为例，从 C 源代码开始，跟随它的旅程，经过 eBPF 字节码，最终到达机器码指令。

###### 注意

如果你想自行构建和运行这段代码，你可以在[*github.com/lizrice/learning-ebpf*](https://github.com/lizrice/learning-ebpf)找到该代码以及设置环境的说明。本章的代码位于*chapter3*目录下。

本章的示例使用 C 语言编写，使用了名为*libbpf*的库。你将在第五章详细了解这个库。

# eBPF“Hello World”适用于网络接口

上一章的示例通过系统调用的 kprobe 触发了跟踪“Hello World”；这次我将展示一个 eBPF 程序，它在接收到网络数据包时触发并写入一行跟踪信息。

数据包处理是 eBPF 的一个非常常见的应用。我将在第八章中详细讨论这一点，但现在知道一个 eBPF 程序的基本概念可能会对你有所帮助，该程序会在网络接口上到达的每个数据包上触发。程序可以检查甚至修改数据包的内容，并对内核对该数据包的处理做出决策（或*评判*）。评判可能告诉内核继续像往常一样处理，丢弃或重定向到其他位置。

在我这里展示的简单示例中，程序不会处理网络数据包；每次接收到网络数据包时，它只是将*Hello World*和一个计数器写入跟踪管道。

示例程序位于*chapter3/hello.bpf.c*。把 eBPF 程序放在以*bpf.c*结尾的文件名中是一个相当普遍的约定，以区分可能存放在同一源代码目录中的用户空间 C 代码。以下是整个程序：

```
#include <linux/bpf.h>                           ![1](img/1.png)
#include <bpf/bpf_helpers.h>

int counter = 0;                                 ![2](img/2.png)

SEC("xdp")                                       ![3](img/3.png)
int hello(void *ctx) {                           ![4](img/4.png)
    bpf_printk("Hello World %d", counter);
    counter++;
    return XDP_PASS;
}

char LICENSE[] SEC("license") = "Dual BSD/GPL";  ![5](img/5.png)
```

![1](img/#code_id_3_4)

这个示例首先包含一些头文件。以防你不熟悉 C 编码，每个程序都必须包含定义程序要使用的任何结构或函数的头文件。从名称可以猜到，这些头文件与 BPF 相关。

![2](img/#code_id_3_5)

这个示例展示了 eBPF 程序如何使用全局变量。每次程序运行时，这个计数器都会递增。

![3](img/#code_id_3_6)

宏`SEC()`定义了一个名为`xdp`的段，在编译后的目标文件中可见。我将在第五章再回到段名是如何用于 eBPF 程序的讨论，但现在你可以简单地将其视为定义了一种称为 eXpress 数据路径（XDP）的 eBPF 程序。

![4](img/#code_id_3_7)

这里你可以看到实际的 eBPF 程序。在 eBPF 中，程序名称即为函数名称，因此该程序称为`hello`。它使用一个辅助函数`bpf_printk`来写入文本字符串，增加全局变量`counter`的值，然后返回值`XDP_PASS`，这是告诉内核应像往常一样处理这个网络数据包的评判结果。

![5](img/#code_id_3_8)

最后还有另一个`SEC()`宏定义了一个许可字符串，这是 eBPF 程序的一个关键要求。内核中的一些 BPF 辅助函数被定义为“仅限 GPL 使用”。如果你想使用这些函数中的任何一个，你的 BPF 代码必须声明为具有 GPL 兼容许可证。验证器（我们将在第六章讨论）会检查声明的许可证是否与程序使用的函数兼容。包括使用 BPF LSM 的某些 eBPF 程序类型，你将在第九章了解到，也需要[兼容 GPL](https://oreil.ly/ItahV)。

###### 注意

你可能会想为什么上一章使用了 `bpf_trace_printk()`，而这个版本使用了 `bpf_printk()`。简短的答案是，BCC 的版本称为 `bpf_trace_printk()`，*libbpf* 的版本是 `bpf_printk()`，但这两者都是对内核函数 `bpf_trace_printk()` 的包装。Andrii Nakryiko 在他的博客上写了一篇[很好的文章](https://oreil.ly/9mNSY)来解释这一点。

这是一个附加到网络接口上 XDP 钩点的 eBPF 程序示例。你可以将 XDP 事件看作是在网络接口上入站时触发的。

###### 注意

一些网络卡支持将 XDP 程序卸载到网络卡本身以便执行。这意味着每个到达的网络数据包可以在卡上处理，而不需要接近计算机的 CPU。XDP 程序可以检查甚至修改每个网络数据包，因此这对于进行 DDoS 保护、防火墙或负载均衡是非常有用的。你将在第八章详细了解这些内容。

你已经看过了 C 源代码，下一步是将其编译成内核能理解的对象。

# 编译一个 eBPF 对象文件

我们的 eBPF 源代码需要编译成 eBPF 虚拟机可以理解的机器指令：eBPF 字节码。如果你指定了 `-target bpf`，LLVM 项目的 Clang 编译器将会完成这项工作。以下是一个 Makefile 的片段，用于执行编译：

```
hello.bpf.o: %.o: %.c
   clang \
       -target bpf \
       -I/usr/include/$(shell uname -m)-linux-gnu \
       -g \
       -O2 -c $< -o $@
```

这从源代码 *hello.bpf.c* 生成了一个名为 *hello.bpf.o* 的对象文件。这里 `-g` 标志是可选的，^(3) 但它生成调试信息，这样你可以在检查对象文件时看到源代码和字节码。让我们检查一下这个对象文件，以更好地理解它包含的 eBPF 代码。

# 检查一个 eBPF 对象文件

文件工具通常用于确定文件的内容：

```
$ file hello.bpf.o
hello.bpf.o: ELF 64-bit LSB relocatable, eBPF, version 1 (SYSV), with debug_info,
not stripped
```

这显示了它是一个 ELF（可执行和可链接格式）文件，包含 eBPF 代码，适用于 LSB（最低有效位）架构的 64 位平台。如果在编译步骤中使用了 `-g` 标志，则包括调试信息。

你可以使用 `llvm-objdump` 进一步检查这个对象以查看 eBPF 指令：

```
$ llvm-objdump -S hello.bpf.o
```

即使你对反汇编不熟悉，这个命令的输出也不难理解：

```
hello.bpf.o:    file format elf64-bpf               ![1](img/1.png)

Disassembly of section xdp:                         ![2](img/2.png)

0000000000000000 <hello>:                           ![3](img/3.png)
;  bpf_printk("Hello World %d", counter");          ![4](img/4.png) 
    0:   18 06 00 00 00 00 00 00 00 00 00 00 00 00 00 00 r6 = 0 ll
    2:   61 63 00 00 00 00 00 00 r3 = *(u32 *)(r6 + 0)
    3:   18 01 00 00 00 00 00 00 00 00 00 00 00 00 00 00 r1 = 0 ll
    5:   b7 02 00 00 0f 00 00 00 r2 = 15
    6:   85 00 00 00 06 00 00 00 call 6
;  counter++;                                       ![5](img/5.png)
    7:   61 61 00 00 00 00 00 00 r1 = *(u32 *)(r6 + 0)
    8:   07 01 00 00 01 00 00 00 r1 += 1
    9:   63 16 00 00 00 00 00 00 *(u32 *)(r6 + 0) = r1
;  return XDP_PASS;                                 ![6](img/6.png)
   10:   b7 00 00 00 02 00 00 00 r0 = 2
   11:   95 00 00 00 00 00 00 00 exit
```

![1](img/#code_id_3_9)

第一行进一步确认了 *hello.bpf.o* 是一个具有 eBPF 代码的 64 位 ELF 文件（某些工具使用术语 *BPF* 而另一些使用 *eBPF* 并无特定的差异；正如我之前所说，这些术语现在几乎是可互换的）。

![2](img/#code_id_3_10)

接下来是标记为 `xdp` 的部分的反汇编，与 C 源代码中的 `SEC()` 定义相匹配。

![3](img/#code_id_3_11)

这一节是一个名为`hello`的函数。

![4](img/#code_id_3_12)

有五行 eBPF 字节码指令对应于源代码行`bpf_printk("Hello World %d", counter");`。

![5](img/#code_id_3_13)

三行 eBPF 字节码指令增加了`counter`变量。

![6](img/#code_id_3_14)

从源代码 `return XDP_PASS;` 生成了另外两行字节码。

除非你特别想这么做，没有必要完全理解每条字节码与源代码的关系。编译器负责生成字节码，你不需要去考虑它！但是让我们更详细地检查一下输出，这样你可以感受一下这个输出如何与本章前面学到的 eBPF 指令和寄存器相关联。

在每条字节码的左边，你可以看到该指令与内存中的`hello`位置的偏移量。如本章前所述，eBPF 指令通常是 8 字节长，由于在 64 位平台上每个内存位置可以容纳 8 字节，因此每条指令的偏移量通常递增一个。但是，该程序的第一条指令恰好是一个广泛的指令编码，需要 16 字节以设置寄存器 6 为`0`的 64 位值。这将该指令放在输出的第二行，偏移量为`2`。接着又有另一条 16 字节的指令，将寄存器 1 设置为`0`的 64 位值。然后，剩余的指令每个占用 8 字节，因此偏移量每行递增一。

每行的第一个字节是操作码，告诉内核执行什么操作，在每条指令行的右侧是指令的人类可读解释。在撰写本文时，Iovisor 项目有关 eBPF 操作码的文档是最全面的，但官方的 Linux 内核文档正在迎头赶上，而 eBPF 基金会正在制作不限于特定操作系统的[标准文档](https://oreil.ly/7ZWzj)。

例如，让我们看看偏移量为`5`的指令：

```
    5:   b7 02 00 00 0f 00 00 00 r2 = 15
```

操作码是 `0xb7`，文档告诉我们，对应此操作码的伪代码是 `dst = imm`，可以理解为“将目标设置为立即值”。目标由第二个字节定义，`0x02` 意味着“寄存器 2”。这里的“立即值”（或字面值）是 `0x0f`，即十进制的 15。因此，我们可以理解此指令告诉内核“将寄存器 2 设置为值 15”。这对应我们在指令右侧看到的输出：`r2 = 15`。

偏移量为 `10` 的指令类似：

```
   10:   b7 00 00 00 02 00 00 00 r0 = 2
```

此行的操作码也是 `0xb7`，这次它将寄存器 0 的值设置为 `2`。当 eBPF 程序运行结束时，寄存器 0 包含返回码，而 `XDP_PASS` 的值为 `2`。这与源代码匹配，源代码总是返回 `XDP_PASS`。

现在您知道 *hello.bpf.o* 包含一个字节码中的 eBPF 程序。下一步是将其加载到内核中。

# 将程序加载到内核中

对于本示例，我们将使用一个名为 `bpftool` 的实用程序。您还可以以编程方式加载程序，稍后在本书中将看到示例。

###### 注意

一些 Linux 发行版提供了一个包，其中包含 `bpftool`，或者您可以[从源代码编译它](https://github.com/libbpf/bpftool)。您可以在[Quentin Monnet 的博客](https://oreil.ly/Yqepv)上找到有关安装或构建此工具的更多详细信息，以及在[Cilium 网站](https://oreil.ly/rnTIg)上的附加文档和用法。

以下是使用 `bpftool` 将程序加载到内核的示例。请注意，您可能需要 root 权限（或使用 `sudo`）以获取 `bpftool` 需要的 BPF 权限。

```
$ bpftool prog load hello.bpf.o /sys/fs/bpf/hello
```

这将从我们编译的对象文件中加载 eBPF 程序，并将其“pin”到位置 */sys/fs/bpf/hello*。^(4) 此命令的无输出响应表示成功，但您可以使用 `ls` 确认该程序已经就位：

```
$ ls /sys/fs/bpf
hello
```

已成功加载 eBPF 程序。让我们使用 `bpftool` 实用程序了解有关程序及其在内核中的状态的更多信息。

# 检查已加载的程序

`bpftool` 实用程序可以列出加载到内核中的所有程序。如果您自己尝试，可能会在输出中看到几个预先存在的 eBPF 程序，但为了清晰起见，我只会显示与我们的“Hello World”示例相关的行：

```
$ bpftool prog list 
...
540: xdp  name hello  tag d35b94b4c0c10efb  gpl
        loaded_at 2022-08-02T17:39:47+0000  uid 0
        xlated 96B  jited 148B  memlock 4096B  map_ids 165,166
        btf_id 254
```

该程序已被分配 ID 540。此标识是在加载每个程序时分配的编号。知道了这个 ID，您可以要求 `bpftool` 显示关于此程序的更多信息。这次，让我们以格式化的 JSON 格式获取输出，以便可见字段名以及值：

```
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

给定字段名，这些内容大部分都很容易理解：

+   程序的 ID 是 540。

+   `type`字段告诉我们，此程序可以使用 XDP 事件附加到网络接口。还有几种其他类型的 BPF 程序可以附加到不同类型的事件上，我们将在第七章中进一步讨论。

+   该程序的名称是`hello`，这是源代码中的函数名称。

+   `tag`是该程序的另一个标识符，稍后我会详细描述它。

+   该程序以 GPL 兼容许可证定义。

+   显示程序加载的时间戳。

+   用户 ID 0（即 root 用户）加载了该程序。

+   该程序中有 96 字节的已翻译 eBPF 字节码，稍后我将向您展示。

+   该程序已经进行了 JIT 编译，并且编译结果是 148 字节的机器码。我稍后也会详细讨论这个。

+   `bytes _memlock`字段告诉我们，该程序保留了 4,096 字节的内存，不会被分页出去。

+   此程序引用了 ID 为 165 和 166 的 BPF 映射。这可能令人惊讶，因为在源代码中并没有明显的映射引用。稍后在本章中，您将看到如何使用映射语义来处理 eBPF 程序中的全局数据。

+   您将在第五章了解有关 BTF 的内容，但现在只需知道`btf_id`指示此程序有一个 BTF 信息块。仅当您使用`-g`标志编译时，此信息才包含在对象文件中。

## BPF 程序标签

`tag`是程序指令的 SHA（安全哈希算法）摘要，可以用作程序的另一个标识符。每次加载或卸载程序时，ID 可能会变化，但标签将保持不变。`bpftool`实用程序接受通过 ID、名称、标签或固定路径引用 BPF 程序，因此在这里的示例中，以下所有内容将产生相同的输出：

+   `bpftool prog show id 540`

+   `bpftool prog show name hello`

+   `bpftool prog show tag d35b94b4c0c10efb`

+   `bpftool prog show pinned /sys/fs/bpf/hello`

您可以具有相同名称的多个程序，甚至具有相同标签的多个程序实例，但 ID 和固定路径将始终是唯一的。

## 翻译后的字节码

`bytes_xlated`字段告诉我们经过验证器的 eBPF 字节码中有多少字节的“翻译”代码。这是 eBPF 字节码，在本书后面我将讨论内核可能因为我稍后会讨论的原因对其进行修改。

让我们使用`bpftool`来显示我们“Hello World”代码的翻译版本：

```
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

这看起来与您之前从`llvm-objdump`的输出中看到的反汇编代码非常相似。偏移地址相同，指令看起来也很相似，例如我们可以看到偏移量为`5`的指令是`r2=15`。

## JIT 编译的机器码

转换后的字节码相当低级，但还不是机器码。eBPF 使用 JIT 编译器将 eBPF 字节码转换为在目标 CPU 上本地运行的机器码。`bytes_jited` 字段显示，在此转换后程序长度为 108 字节。

###### 注意

为了更高的性能，eBPF 程序通常是 JIT 编译的。另一种方法是在运行时解释 eBPF 字节码。eBPF 指令集和寄存器设计得相当接近本机机器指令，使得解释变得直观且相对快速，但编译后的程序将更快，并且大多数体系结构现在支持 JIT。^(5)

`bpftool` 实用程序可以生成这些 JITed 代码的汇编语言转储。如果你对汇编语言不熟悉，这可能看起来完全难以理解！我只是为了说明从源代码到可执行机器指令的所有转换过程。以下是命令及其输出：

```
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

一些打包的 `bpftool` 发行版尚未包含支持转储 JIT 输出的功能，如果是这种情况，你将看到“错误：没有 libbfd 支持。”你可以按照[*https://github.com/libbpf/bpftool*](https://github.com/libbpf/bpftool)上的说明自行构建 `bpftool`。

你已经看到“Hello World”程序已加载到内核中，但此时它尚未与事件关联，因此没有任何触发器来运行它。它需要附加到一个事件上。

# 附加到事件

程序类型必须与其附加的事件类型匹配；你将在第七章中了解更多。在这种情况下，它是一个 XDP 程序，你可以使用 `bpftool` 将示例 eBPF 程序附加到网络接口的 XDP 事件上，如下所示：

```
$ bpftool net attach xdp id 540 dev eth0
```

###### 注意

此时，`bpftool` 实用程序不支持附加所有程序类型的能力，但已被[最近扩展](https://oreil.ly/Tt99p)以自动附加 k(ret)probes、u(ret)probes 和 tracepoints。

在这里我使用了程序的 ID 540，但你也可以使用名称（假设它是唯一的）或标签来标识被附加的程序。在本例中，我已将程序附加到网络接口 `eth0` 上。

你可以使用 `bpftool` 查看所有已附加到网络的 eBPF 程序：

```
$ bpftool net list 
xdp:
eth0(2) driver id 540

tc:

flow_dissector:
```

ID 为 540 的程序已附加到 `eth0` 接口的 XDP 事件上。此输出还提供了关于网络堆栈中其他潜在事件的一些线索，你可以将 eBPF 程序附加到其中：`tc` 和 `flow_dissector`。更多信息请参见第七章。

你还可以使用 `ip link` 检查网络接口，你将看到类似以下输出（为清晰起见，已删除了部分细节）：

```
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT
group default qlen 1000
    ...
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 xdp qdisc fq_codel state UP
mode DEFAULT group default qlen 1000
    ...
    prog/xdp id 540 tag 9d0e949f89f1a82c jited 
    ...
```

在这个例子中有两个接口：回环接口 `lo`，用于向本机上的进程发送流量；以及 `eth0` 接口，连接本机与外部世界。该输出还显示 `eth0` 有一个 JIT 编译的 eBPF 程序，其标识为 `540`，标签为 `9d0e949f89f1a82c`，已附加到其 XDP 钩子上。

###### 注意

你也可以使用 `ip link` 命令将 XDP 程序附加到或从网络接口中分离出来。我已经在本章末尾包含了这个作为练习，并且在 第七章 中有进一步的例子。

此时，*hello* eBPF 程序应该在每次接收到网络数据包时产生跟踪输出。你可以通过运行 `cat /sys/kernel/debug/tracing/trace_pipe` 来验证这一点。应该会显示类似于以下内容的大量输出：

```
<idle>-0       [003] d.s.. 655370.944105: bpf_trace_printk: Hello World 4531
<idle>-0       [003] d.s.. 655370.944587: bpf_trace_printk: Hello World 4532
<idle>-0       [003] d.s.. 655370.944896: bpf_trace_printk: Hello World 4533
```

如果你记不住跟踪管道的位置，你可以使用命令 `bpftool prog tracelog` 来获取相同的输出。

与你在 第二章 中看到的输出相比，这一次每个事件的开头都没有与其相关联的命令或进程 ID；相反，你会看到每行跟踪的开头是 `<idle>-0`。在 第二章 中，每个系统调用事件发生是因为在用户空间执行命令的进程调用了系统调用 API。该进程 ID 和命令是执行 eBPF 程序时的上下文的一部分。但在这个例子中，XDP 事件发生是由于网络数据包的到达。与该数据包相关联的没有用户空间进程——在触发 *hello* eBPF 程序时，系统仅仅是在内存中接收到该数据包，并且并不知道该数据包是什么，它要去往何处。

你可以看到追踪输出中的计数器值每次递增一个，这是预期的。在源代码中，`counter` 是一个全局变量。让我们看看如何在 eBPF 中使用映射来实现它。

# 全局变量

正如你在前一章学到的，eBPF 映射是一种数据结构，可以从 eBPF 程序或用户空间访问。由于同一个映射可以被同一程序的不同运行重复访问，它可以用来在不同执行之间保存状态。多个程序也可以访问同一个映射。由于这些特性，映射的语义可以被重新用作全局变量。

###### 注意

在 [2019 年添加全局变量支持之前](https://oreil.ly/IDftt)，eBPF 程序员必须显式编写映射来执行相同的任务。

之前你看到 `bpftool` 显示了这个示例程序使用了两个映射，它们的标识分别是 165 和 166。（如果你自己尝试，可能会看到不同的标识，因为这些标识是在内核中创建映射时分配的。）让我们来探索一下这些映射中有什么。

`bpftool`实用程序可以显示加载到内核中的映射。为了清晰起见，我将仅显示与示例“Hello World”程序相关的条目 165 和 166：

```
$ bpftool map list
165: array  name hello.bss  flags 0x400
        key 4B  value 4B  max_entries 1  memlock 4096B
        btf_id 254
166: array  name hello.rodata  flags 0x80
        key 4B  value 15B  max_entries 1  memlock 4096B
        btf_id 254  frozen
```

从 C 程序编译的对象文件中的 bss^(6)部分通常保存全局变量，您可以使用`bpftool`检查其内容，如下所示：

```
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

我也可以使用`bpftool map dump id 165`来检索相同的信息。如果再次运行任何一个命令，我会看到计数器增加了，因为程序在每次接收网络数据包时都会运行。

正如您将在第五章中了解到的那样，只有在可用 BTF 信息时，`bpftool`才能漂亮地打印映射（这里是变量名`counter`）的字段名，并且仅当您在编译时使用`-g`标志时才包含该信息。如果在编译步骤中省略了该标志，您将看到类似以下的输出：

```
$ bpftool map dump name hello.bss
key: 00 00 00 00  value: 19 01 00 00
Found 1 element
```

没有 BTF 信息，`bpftool`无法知道源代码中使用的变量名。你可以推断，由于此映射中只有一个项目，十六进制值`19 01 00 00`必须是`counter`的当前值（281 的十进制，因为字节按最低有效字节起始排序）。

您在这里看到 eBPF 程序使用映射的语义来读取和写入全局变量。映射还用于保存静态数据，如您可以通过检查其他映射看到的那样。

另一个映射命名为`hello.rodata`表明这可能是与我们的*hello*程序相关的只读数据。您可以转储此映射的内容以查看它包含的 eBPF 程序用于跟踪的字符串：

```
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

如果您没有使用`-g`标志编译对象，则会看到类似以下的输出：

```
$ bpftool map dump id 166
key: 00 00 00 00  value: 48 65 6c 6c 6f 20 57 6f  72 6c 64 20 25 64 00
Found 1 element
```

此映射中有一个键值对，该值包含 12 个字节的数据，以 0 结尾。这些字节很可能是字符串`"Hello World %d"`的 ASCII 表示。

现在我们已经完成了检查此程序及其映射的工作，是时候清理它了。我们将从触发它的事件中开始分离它。

# 分离程序

您可以像这样将程序从网络接口分离：

```
$ bpftool net detach xdp dev eth0
```

如果此命令成功运行，则不会输出任何内容，但您可以通过`bpftool net list`输出中缺少 XDP 条目来确认程序已不再附加：

```
$ bpftool net list 
xdp:

tc:

flow_dissector:
```

但是，程序仍加载在内核中：

```
$ bpftool prog show name hello 
395: xdp  name hello  tag 9d0e949f89f1a82c  gpl
        loaded_at 2022-12-19T18:20:32+0000  uid 0
        xlated 48B  jited 108B  memlock 4096B  map_ids 4
```

# 卸载程序

在撰写本文时，`bpftool prog load`没有相反操作（至少没有），但您可以通过删除固定的伪文件从内核中删除程序：

```
$ rm /sys/fs/bpf/hello
$ bpftool prog show name hello
```

由于程序不再加载到内核中，因此此`bpftool`命令不会输出任何内容。

# BPF 到 BPF 调用

在上一章中，您看到了尾调用的实际应用，并且我提到现在还可以从 eBPF 程序内调用函数的能力。让我们看一个简单的例子，就像尾调用示例一样，它可以附加到 `sys_enter` 跟踪点，但这次它将跟踪系统调用的操作码。您将在 *chapter3/hello-func.bpf.c* 中找到这段代码。

为了说明问题，我编写了一个非常简单的函数，用于从跟踪点参数中提取系统调用操作码：

```
static __attribute((noinline)) int get_opcode(struct bpf_raw_tracepoint_args 
                                                                         *ctx) {
   return ctx->args[1];
}
```

如果可以选择，编译器可能会内联这个非常简单的函数，我只会从一个地方调用它。由于这样会破坏这个示例的目的，我添加了 `__attribute((noinline))` 来强制编译器的行为。在正常情况下，您应该省略这一点，允许编译器根据需要进行优化。

调用此函数的 eBPF 函数如下所示：

```
SEC("raw_tp")
int hello(struct bpf_raw_tracepoint_args *ctx) {
   int opcode = get_opcode(ctx);
   bpf_printk("Syscall: %d", opcode);
   return 0;
}
```

将其编译为 eBPF 对象文件后，您可以将其加载到内核中，并使用 `bpftool` 确认已加载：

```
$ bpftool prog load hello-func.bpf.o /sys/fs/bpf/hello
$ bpftool prog list name hello 
893: raw_tracepoint  name hello  tag 3d9eb0c23d4ab186  gpl
        loaded_at 2023-01-05T18:57:31+0000  uid 0
        xlated 80B  jited 208B  memlock 4096B  map_ids 204
        btf_id 302
```

此练习的有趣部分是检查 eBPF 字节码以查看 `get_opcode()` 函数：

```
$ bpftool prog dump xlated name hello 
int hello(struct bpf_raw_tracepoint_args * ctx):
; int opcode = get_opcode(ctx);                            ![1](img/1.png)
   0: (85) call pc+7#bpf_prog_cbacc90865b1b9a5_get_opcode
; bpf_printk("Syscall: %d", opcode);
   1: (18) r1 = map[id:193][0]+0
   3: (b7) r2 = 12
   4: (bf) r3 = r0
   5: (85) call bpf_trace_printk#-73584
; return 0;
   6: (b7) r0 = 0
   7: (95) exit
int get_opcode(struct bpf_raw_tracepoint_args * ctx):      ![2](img/2.png)
; return ctx->args[1];
   8: (79) r0 = *(u64 *)(r1 +8)
; return ctx->args[1];
   9: (95) exit
```

![1](img/#code_id_3_15)

在这里，您可以看到 `hello()` eBPF 程序调用了 `get_opcode()`。偏移量为 `0` 的 eBPF 指令是 `0x85`，根据指令集文档，它对应于“函数调用”。而不是执行下一个指令（在偏移量 1 处），执行将跳过七个指令（`pc+7`），这意味着在偏移量 `8` 处的指令。

![2](img/#code_id_3_16)

这是 `get_opcode()` 的字节码，正如您希望的那样，第一条指令位于偏移量 `8` 处。

函数调用指令要求将当前状态放入 eBPF 虚拟机的堆栈中，以便在调用的函数退出时，执行可以继续在调用函数中进行。由于堆栈大小限制为 512 字节，BPF 到 BPF 的调用不能太深嵌套。

###### 注意

有关尾调用和 BPF 到 BPF 调用的更多详细信息，请参阅 Cloudflare 博客上 Jakub Sitnicki 的优秀文章：“[在内！x86 和 ARM 上的 BPF 尾调用](https://oreil.ly/6kOp3)”。

# 摘要

在本章中，您看到了一些示例 C 源代码转换为 eBPF 字节码，然后编译为机器代码，以便在内核中执行。您还学习了如何使用 `bpftool` 检查加载到内核中的程序和映射，并附加到 XDP 事件。

此外，您还看到了由不同类型事件触发的不同类型的 eBPF 程序示例。XDP 事件是由数据包在网络接口上的到达触发的，而 kprobe 和 tracepoint 事件是由命中内核代码的某个特定点触发的。我将在第七章讨论一些其他 eBPF 程序类型。

您还学习了如何使用映射来实现 eBPF 程序的全局变量，并看到了 BPF 到 BPF 函数调用。

下一章将进一步详细地介绍当`bpftool`或任何其他用户空间代码加载程序并将其附加到事件时，系统调用级别发生了什么。

# 练习

如果你想进一步探索 BPF 程序，可以尝试以下几件事情：

1.  尝试使用像下面这样的`ip link`命令来附加和分离 XDP 程序：

    ```
    $ ip link set dev eth0 xdp obj hello.bpf.o sec xdp
    $ ip link set dev eth0 xdp off
    ```

1.  运行来自第二章的任何 BCC 示例。在程序运行时，使用第二个终端窗口使用`bpftool`检查加载的程序。以下是我运行*hello-map.py*示例时看到的示例：

    ```
    $ bpftool prog show name hello 
    197: kprobe  name hello  tag ba73a317e9480a37  gpl
            loaded_at 2022-08-22T08:46:22+0000  uid 0
            xlated 296B  jited 328B  memlock 4096B  map_ids 65
            btf_id 179
            pids hello-map.py(2785)
    ```

    你也可以使用`bpftool prog dump`命令来查看这些程序的字节码和机器码版本。

1.  在*chapter2*目录中运行*hello-tail.py*，同时它在运行时查看加载的程序。你会看到每个尾调用程序都单独列出，就像这样：

    ```
    $ bpftool prog list 
    ...
    120: raw_tracepoint  name hello  tag b6bfd0e76e7f9aac  gpl
            loaded_at 2023-01-05T14:35:32+0000  uid 0
            xlated 160B  jited 272B  memlock 4096B  map_ids 29
            btf_id 124
            pids hello-tail.py(3590)
    121: raw_tracepoint  name ignore_opcode  tag a04f5eef06a7f555  gpl
            loaded_at 2023-01-05T14:35:32+0000  uid 0
            xlated 16B  jited 72B  memlock 4096B
            btf_id 124
            pids hello-tail.py(3590)
    122: raw_tracepoint  name hello_exec  tag 931f578bd09da154  gpl
            loaded_at 2023-01-05T14:35:32+0000  uid 0
            xlated 112B  jited 168B  memlock 4096B
            btf_id 124
            pids hello-tail.py(3590)
    123: raw_tracepoint  name hello_timer  tag 6c3378ebb7d3a617  gpl
            loaded_at 2023-01-05T14:35:32+0000  uid 0
            xlated 336B  jited 356B  memlock 4096B
            btf_id 124
            pids hello-tail.py(3590)
    ```

    你也可以使用`bpftool prog dump xlated`来查看字节码指令，并将其与“BPF to BPF Calls”中看到的内容进行比较。

1.  *要小心这个问题，最好的做法可能是简单地思考为什么会发生这种情况，而不是试图去做！* 如果从一个 XDP 程序返回`0`值，这对应于`XDP_ABORTED`，告诉内核放弃进一步处理这个数据包。这可能有点违反直觉，因为在 C 语言中，`0`值通常表示成功，但事实就是如此。所以，如果你尝试修改程序返回`0`并将其附加到虚拟机的`eth0`接口，所有网络数据包将会被丢弃。如果你正在使用 SSH 连接到这台机器，这将会有些不幸，你可能需要重新启动机器来恢复访问！

    你可以在容器内运行程序，这样 XDP 程序就会附加到仅影响该容器而不是整个虚拟机的虚拟以太网接口上。在[*https://github.com/lizrice/lb-from-scratch*](https://github.com/lizrice/lb-from-scratch)有一个实现这一点的示例。

^(1) 越来越多的 eBPF 程序也开始使用 Rust 编写，因为 Rust 编译器支持 eBPF 字节码作为目标。

^(2) 有几条指令的操作是通过指令中其他字段的值“修改”的。例如，在内核 5.12 中引入了一组原子指令，包括一个在`imm`字段中指定的算术操作（`ADD`、`AND`、`OR`、`XOR）。

^(3) 生成 BTF 信息需要使用`-g`标志，这对于 CO-RE eBPF 程序是必需的，我将在第五章中讨论。

^(4) 通常情况下，这是可选的——eBPF 程序可以加载到内核中而不被固定在文件位置上——但对于`bpftool`来说不是可选的，它加载的程序必须固定。这个原因在“BPF 程序和映射引用”中进一步讨论。

^(5) 要利用 JIT 编译，需要启用内核设置`CONFIG_BPF_JIT`，可以通过`net.core.bpf_jit_enable sysctl`设置在运行时启用或禁用。有关不同芯片架构上 JIT 支持的更多信息，请参阅[文档](https://oreil.ly/4-xi6)。

^(6) 在这里，*bss*代表“block started by symbol”。
