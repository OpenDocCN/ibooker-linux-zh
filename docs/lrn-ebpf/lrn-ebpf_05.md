# 第五章：CO-RE、BTF 和 Libbpf

在上一章中，您首次遇到了 BTF（BPF 类型格式）。本章讨论了它存在的原因以及如何使用它使 eBPF 程序在不同内核版本间可移植。这是 BPF“编译一次，到处运行”（CO-RE）方法的关键部分，解决了在不同内核版本间实现 eBPF 程序可移植性的问题。

许多 eBPF 程序访问内核数据结构，eBPF 程序员需要包含相关的 Linux 头文件，以便他们的 eBPF 代码可以正确地定位这些数据结构中的字段。然而，Linux 内核在持续开发中，这意味着内部数据结构在不同内核版本间可能会发生变化。如果您将在一台机器上编译的 eBPF 对象文件加载到具有不同内核版本的机器上，不能保证数据结构会保持一致。

CO-RE 方法在有效解决这一可移植性问题方面迈出了重要的一步。它允许 eBPF 程序包含有关它们编译时所用数据结构布局的信息，并提供了一种机制，用于调整在目标机器上运行时数据结构布局不同情况下字段访问的方式。只要程序不需要访问目标机器内核中根本不存在的字段或数据结构，程序就能在不同内核版本间实现可移植性。

但在深入讨论 CO-RE 如何工作之前，让我们看看为什么它是如此令人向往的，通过观察最初在 BCC 项目中实现的内核可移植性的先前方法。

# BCC 对可移植性的方法

在第二章中，我使用[BCC](https://oreil.ly/ReUtn)展示了 eBPF 程序的基本“Hello World”示例。BCC 项目是第一个流行的用于实现 eBPF 程序的项目，为没有太多内核经验的程序员提供了一个相对易于使用的用户空间和内核方面的框架。为了解决跨内核的可移植性问题，BCC 采用了在目标机器上即时编译 eBPF 代码的方法。然而，这种方法存在一些问题：

+   编译工具链需要安装在每台希望运行代码的目标机器上，还需要内核头文件（这些文件通常不会默认存在）。

+   在工具启动之前，您必须等待编译完成，这可能意味着每次启动工具都会有几秒钟的延迟。

+   如果您在大量相同的机器群上运行工具，每台机器上都重复编译是一种计算资源的浪费。

+   一些基于 BCC 的项目将它们的 eBPF 源代码和工具链打包到一个容器映像中，这使得将其分发到每台机器更加容易。但这并不能解决确保内核头文件存在的问题，甚至可能意味着如果安装了多个这些 BCC 容器，则会有更多的重复。

+   嵌入式设备可能没有足够的内存资源来运行编译步骤。

因为这些问题，如果你计划开始开发一个重要的新 eBPF 项目，我不建议使用这种传统的 BCC 方法，尤其是如果你计划将其分发给其他人使用。在本书中，我给出了一些基于 BCC 的示例，因为它是了解 eBPF 基本概念的好方法，特别是因为 Python 用户空间代码如此紧凑且易于阅读。如果你更喜欢使用它，并且想要快速组装一些东西，那么它也是一个完全不错的选择。但是对于严肃的现代 eBPF 开发来说，它并不是最佳选择。

CO-RE 方法为 eBPF 程序的跨内核可移植性问题提供了一个更好的解决方案。

###### 注意

BCC 项目位于[*github.com/iovisor/bcc*](https://oreil.ly/ReUtn)，包含广泛的命令行工具，用于观察 Linux 机器行为的各种信息。位于[*tools*](https://oreil.ly/fI4w_)目录中的原始版本大多使用 Python 实现，使用了我在本节中描述的传统可移植性方法。

在 BCC 的[*libbpf-tools*](https://oreil.ly/ke7yq)目录中，你会找到使用 C 编写的更新版本的这些工具，利用了*libbpf*和 CO-RE，并且不会遇到我刚刚列出的问题。它们是一套非常有用的实用工具集！

# CO-RE 概述

CO-RE 方法由几个元素组成：^(2)^,^(3)

BTF

[BTF](https://oreil.ly/iRCuI) 是用于表示数据结构布局和函数签名的格式。在 CO-RE 中，它用于确定编译时和运行时使用的结构之间的任何差异。像`bpftool`这样的工具也使用 BTF 来以人类可读的格式转储数据结构。从 Linux 内核 5.4 版本开始支持 BTF。

内核头文件

Linux 内核源代码包括描述其使用的数据结构的头文件，这些头文件在不同版本的 Linux 之间可能会发生变化。eBPF 程序员可以选择包含单独的头文件，或者如本章所示，可以使用`bpftool`从运行中的系统生成一个名为*vmlinux.h*的头文件，其中包含 BPF 程序可能需要的有关内核的所有数据结构信息。

编译器支持

当 Clang 编译器使用 `-g` 标志编译带有 *CO-RE 重定位* 的 eBPF 程序时，它包含了从描述内核数据结构的 BTF 信息中导出的内容。GCC 编译器也在版本 12 中为 BPF 目标添加了 CO-RE 支持。

用于数据结构重定位的库支持

在用户空间程序将 eBPF 程序加载到内核时，CO-RE 方法要求调整字节码以补偿编译时存在的数据结构与目标机器上实际运行时的差异，这是基于编译到对象中的 CO-RE 重定位信息。有几个库可以处理这个问题：*libbpf* 是最初的 C 库，包含这种重定位能力；Cilium 的 eBPF 库为 Go 程序员提供了同样的功能；Aya 则为 Rust 提供了支持。

可选地，BPF 骨架

骨架可以从编译后的 BPF 对象文件自动生成，其中包含了方便用户空间代码调用的实用函数，用于管理 BPF 程序的生命周期——将它们加载到内核中，将它们附加到事件上等。如果您使用 C 编写用户空间代码，可以使用 `bpftool gen skeleton` 生成这些骨架。这些函数是比直接使用底层库（*libbpf*、*cilium/ebpf* 等）更方便的高级抽象。

###### 注意

Andrii Nakryiko 写了一篇关于 CO-RE 背景的[卓越博客文章](https://oreil.ly/aeQJo)，并详细介绍了其工作原理及如何使用。他还撰写了权威的[BPF CO-RE 参考指南](https://oreil.ly/lbW_T)，如果您准备自行编写代码，请务必阅读。他的 *libbpf-bootstrap* 指南则介绍了使用 CO-RE + *libbpf* + 骨架从头开始构建 eBPF 应用的方法，也是另一个必读资源。

现在您已经对 CO-RE 的元素有了概述，让我们深入了解它们的工作原理，从探索 BTF 开始。

# BPF 类型格式

BTF 信息描述了数据结构和代码在内存中的布局方式。这些信息可以用于各种不同的用途。

## BTF 使用案例

讨论 BTF 在本章关于 CO-RE 中的主要原因是，了解在编译 eBPF 程序的结构布局与将要运行的结构布局之间的差异，允许在程序加载到内核时进行适当的调整。我将在本章后面讨论重定位过程，但现在，让我们也考虑一些 BTF 信息可以用于的其他用途。

知道结构体的布局方式及其每个字段的类型，可以使结构体的内容以人类可读的形式进行漂亮打印。例如，从计算机的角度来看，字符串只是一系列字节，但将这些字节转换为字符可以使字符串更易于人类理解。您在前一章已经看到了一个例子，其中 `bpftool` 使用 BTF 信息来格式化映射转储的输出。

BTF 信息还包括行和函数信息，使 `bpftool` 能够在从翻译或 JIT 编译的程序转储的输出中插入源代码，正如您在 第三章 中看到的那样。当您阅读 第六章 时，您还将看到源代码信息与验证器日志输出交错，这同样来自于 BTF 信息。

BTF 信息还需要用于 BPF 自旋锁。*自旋锁* 用于阻止两个 CPU 核同时访问相同的映射值。锁必须是映射值结构体的一部分，如下所示：

```
struct my_value {
     ... <other fields>
     struct bpf_spin_lock lock;
... <other fields>
};
```

在内核内部，eBPF 程序使用 `bpf_spin_lock()` 和 `bpf_spin_unlock()` 辅助函数来获取和释放锁。只有在有可用的 BTF 信息描述锁字段所在位置时，才能使用这些辅助函数。

###### 注意

自旋锁支持是在内核版本 5.1 中添加的。对于自旋锁的使用有很多限制：它们只能用于哈希或数组映射类型，并且不能在跟踪或套接字过滤类型的 eBPF 程序中使用。更多关于自旋锁的信息，请参阅 [lwn.net 上有关 BPF 并发管理的文章](https://oreil.ly/kAyAU)。

现在您知道 BTF 信息的用处，让我们通过查看一些示例来更具体化。

## 使用 `bpftool` 列出 BTF 信息

与程序和地图一样，您可以使用 `bpftool` 实用程序显示 BTF 信息。以下命令列出加载到内核中的所有 BTF 数据：

```
bpftool btf list
1: name [vmlinux]  size 5843164B
2: name [aes_ce_cipher]  size 407B
3: name [cryptd]  size 3372B
...
149: name <anon>  size 4372B  prog_ids 319  map_ids 103
        pids hello-buffer-co(7660)
155: name <anon>  size 37100B
        pids bpftool(7784)
```

（为了简洁起见，我省略了结果中的许多条目。）

列表中的第一项是 `vmlinux`，它对应我之前提到的 *vmlinux* 文件，其中包含有关当前运行内核的 BTF 信息。

###### 注意

本章早期的一些示例重用了 第四章 的程序，然后在本章后期，您将找到新的示例，这些示例的源代码位于 [*github.com/lizrice/learning-ebpf*](https://github.com/lizrice/learning-ebpf) 的 *chapter5* 目录中。

要获取此示例输出，我在运行 第四章 的 `hello-buffer-config` 示例时运行了此命令。您可以在以 `149:` 开头的行上看到描述此过程正在使用的 BTF 信息的条目：

```
149: name <anon>  size 4372B  prog_ids 319  map_ids 103
        pids hello-buffer-co(7660)
```

这一行告诉我们以下内容：

+   此 BTF 信息块的 ID 是 149。

+   这是一个大约 4 KB 的匿名 BTF 信息块。

+   它被具有 `prog_id 319` 的 BPF 程序和具有 `map_id 103` 的 BPF 映射使用。

+   它还被进程 ID 7660（括号内显示）运行的 `hello-buffer-config` 可执行文件使用（其名称已截断为 15 个字符）。

这些程序、映射和 BTF 标识符与 `bpftool` 显示的有关 `hello-buffer-config` 的名为 `hello` 的程序的输出匹配：

```
bpftool prog show name hello
319: kprobe  name hello  tag a94092da317ac9ba  gpl
        loaded_at 2022-08-28T14:13:35+0000  uid 0
        xlated 400B  jited 428B  memlock 4096B  map_ids 103,104
        btf_id 149
        pids hello-buffer-co(7660)
```

唯一似乎完全不匹配的是程序引用了额外的 `map_id`，`104`。这是性能事件缓冲区映射，并且不使用 BTF 信息；因此，它不会出现在与 BTF 相关的输出中。

就像 `bpftool` 可以转储程序和映射的内容一样，它也可以用来查看数据块中包含的 BTF 类型信息。

## BTF 类型

知道 BTF 信息的 ID 后，您可以使用命令 `bpftool btf dump id <id>` 检查其内容。当我使用之前获取的 ID 149 运行这个命令时，我得到了 69 行输出，每行都是一个类型定义。我将描述前几行，这应该能让您了解如何解释其余部分。这些首几行的 BTF 信息与在源代码中如此定义的 `config` 哈希映射相关：

```
struct user_msg_t {
  char message[12];
};

BPF_HASH(config, u32, struct user_msg_t);
```

此哈希表的键类型为 `u32`，值类型为 `struct user_msg_t`。该结构包含一个 12 字节的 `message` 字段。让我们看看这些类型在对应的 BTF 信息中是如何定义的。

BTF 输出的前三行如下：

```
[1] TYPEDEF 'u32' type_id=2
[2] TYPEDEF '__u32' type_id=3
[3] INT 'unsigned int' size=4 bits_offset=0 nr_bits=32 encoding=(none)
```

每行开头的方括号中的数字是类型 ID（因此第一行，以 `[1]` 开头的行定义了 `type_id 1`，依此类推）。让我们更详细地探讨这三种类型：

+   类型 1 定义了名为 `u32` 的类型及其类型，由以 `[2]` 开头的行定义，即哈希表中键的类型为 `u32`。

+   类型 2 的名称为 `__u32`，类型由 `type_id 3` 定义。

+   类型 3 是一个名为 `unsigned int` 的整数类型，长度为 4 字节。

这三种类型都是指 32 位无符号整数类型的同义词。在 C 中，整数的长度取决于平台，因此 Linux 定义了诸如 `u32` 这样的类型，明确地定义了特定长度的整数。在此机器上，`u32` 对应于无符号整数。引用这些的用户空间代码应该使用前缀带下划线的同义词，如 `__u32`。

BTF 输出的接下来几个类型如下：

```
[4] STRUCT 'user_msg_t' size=12 vlen=1
        'message' type_id=6 bits_offset=0
[5] INT 'char' size=1 bits_offset=0 nr_bits=8 encoding=(none)
[6] ARRAY '(anon)' type_id=5 index_type_id=7 nr_elems=12
[7] INT '__ARRAY_SIZE_TYPE__' size=4 bits_offset=0 nr_bits=32 encoding=(none)
```

这些关联到 `config` 映射中值为 `user_msg_t` 结构的类型：

+   类型 4 是 `user_msg_t` 结构本身，总长度为 12 字节。它包含一个名为 `message` 的字段，由类型 6 定义。`vlen` 字段指示这个定义中有多少个字段。

+   类型 5 名为 `char`，是一个 1 字节的整数——这正是 C 程序员对名为`char`类型的定义期望的内容。

+   类型 6 将`message`字段的类型定义为具有 12 个元素的数组。每个元素的类型为 5（它是一个`char`），并且数组由类型 7 索引。

+   类型 7 是一个 4 字节整数。

带着这些定义，你可以完整地了解`user_msg_t`结构在内存中的布局，如图 5-1 所示。

![一个`user_msg_t`结构占用 12 字节内存](img/lebp_0501.png)

###### 图 5-1\. 一个`user_msg_t`结构占用 12 字节内存

到目前为止，所有的条目都将`bits_offset`设置为`0`，但是下一行输出的结构具有多个字段：

```
[8] STRUCT '____btf_map_config' size=16 vlen=2
        'key' type_id=1 bits_offset=0
        'value' type_id=4 bits_offset=32
```

这是存储在名为`config`的映射中的键值对的结构定义。我没有在源代码中定义这种`____btf_map_config`类型，但它是由 BCC 生成的。键的类型是`u32`，值是`user_msg_t`结构。这对应于您之前看到的类型 1 和类型 4。

关于这个结构的 BTF 信息的另一个重要部分是，`value`字段从结构的起始位置后 32 位开始。这完全是有道理的，因为前 32 位用于保存`key`字段。

###### 注意

在 C 语言中，结构字段会自动对齐到边界，因此不能简单地假设一个字段总是紧跟在前一个字段的后面。例如，考虑这样一个结构：

```
struct something {
    char letter; 
    u64 number;
}
```

在字段称为`letter`之后，在`number`字段之前有 7 字节的未使用内存，以便 64 位数字可以对齐到可以被 8 整除的内存位置。

在某些情况下，可以启用编译器的紧凑排列来避免这些未使用的空间，但通常会导致性能下降，并且——至少在我的经验中——很少这样做。更常见的是，C 程序员会手动设计结构以有效利用空间。

## 带有 BTF 信息的映射

您刚刚看到了与映射关联的 BTF 信息。现在让我们看看在创建映射时，这些 BTF 数据如何传递给内核。

您在第四章中看到，映射是使用`bpf(BPF_MAP_CREATE)`系统调用创建的。这需要一个`bpf_attr`结构作为参数，[在内核中定义](https://oreil.ly/PLrYG)如下（省略了一些细节）：

```
struct { /* anonymous struct used by BPF_MAP_CREATE command */
    `__u32`   `map_type`;             /* one of enum bpf_map_type */
    `__u32`   `key_size`;             /* size of key in bytes */
    `__u32`   `value_size`;           /* size of value in bytes */
    `__u32`   `max_entries`;          /* max number of entries in a map */
    ...
    char    `map_name`[`BPF_OBJ_NAME_LEN`];
    ...
    `__u32`   `btf_fd`;               /* fd pointing to a BTF type data */
    `__u32`   `btf_key_type_id`;      /* BTF type_id of the key */
    `__u32`   `btf_value_type_id`;    /* BTF type_id of the value */
    ...
};
```

在引入 BTF 之前，`btf_*`字段不存在于`bpf_attr`结构中，内核不了解键或值的结构。`key_size`和`value_size`字段定义了它们所需的内存量，但它们只是被视为一些字节。通过额外传入定义键和值类型的 BTF 信息，内核可以内省它们，像`bpftool`这样的工具可以检索用于漂亮打印的类型信息，正如前面讨论的那样。但是有趣的是，为键和值分别传入了单独的 BTF `type _id`。刚才看到的`____btf_map_config`结构并不被内核用于映射定义；它只是由用户空间的 BCC 使用。

## 函数和函数原型的 BTF 数据

到目前为止，在这个示例输出中，BTF 数据与数据类型有关，但 BTF 数据还包含有关函数和函数原型的信息。以下是描述`hello`函数的同一 BTF 数据块中的信息：

```
[31] FUNC_PROTO '(anon)' ret_type_id=23 vlen=1
        'ctx' type_id=10
[32] FUNC 'hello' type_id=31 linkage=static
```

在类型 32 中，您可以看到名为`hello`的函数被定义为具有前一行中定义的类型。这是一个*函数原型*，它返回类型 ID `23`，并采用单个参数(`vlen=1`)称为`ctx`，其类型 ID 为`10`。为了完整起见，这里是前面输出中那些类型的定义：

```
[10] PTR '(anon)' type_id=0

[23] INT 'int' size=4 bits_offset=0 nr_bits=32 encoding=SIGNED
```

类型 10 是一个匿名指针，其默认类型为`0`，在 BTF 输出中没有明确包含，但定义为 void 指针。^(4)

返回值类型为 23，是一个 4 字节整数，`encoding=SIGNED`表示它是一个有符号整数；也就是说，它可以是正数或负数。这对应于源代码中*hello-buffer-config.py*中的函数定义，如下所示：

```
int hello(void *ctx)
```

到目前为止，我展示的示例 BTF 信息来自于列出 BTF 数据块内容。让我们看看如何获取与特定映射或程序相关的 BTF 信息。

## 检查映射和程序的 BTF 数据

如果您想检查与特定映射相关的 BTF 类型，`bpftool`使这变得容易。例如，这是`config`映射的输出：

```
bpftool btf dump map name config
[1] TYPEDEF 'u32' type_id=2
[4] STRUCT 'user_msg_t' size=12 vlen=1
        'message' type_id=6 bits_offset=0
```

类似地，您可以使用`bpftool btf dump prog <prog identity>`检查与特定程序相关的 BTF 信息。我会让您查看[manpage](https://oreil.ly/lCoV5)以获取更多详细信息。

###### 注意

如果您想更好地理解如何生成和去重 BTF 类型数据，请参阅 Andrii Nakryiko 的另一篇[优秀博客文章](https://oreil.ly/0-a9g)。

到此为止，您应该已经了解了 BTF 如何描述数据结构和函数的格式。一个用 C 编写的 eBPF 程序需要定义类型和结构的头文件。让我们看看为 eBPF 程序可能需要的任何内核数据类型生成头文件是多么简单。

# 生成内核头文件

如果您在启用了 BTF 的内核上运行`bpftool btf list`，您将看到许多类似以下的预存在 BTF 数据块：

```
$ bpftool btf list
1: name [vmlinux]  size 5842973B
2: name [aes_ce_cipher]  size 407B
3: name [cryptd]  size 3372B
...
```

此列表中的第一项，ID 为 1，名称为`vmlinux`，是关于所有数据类型、结构和函数定义的 BTF 信息，这些定义由运行在此（虚拟）机器上的内核使用。^(5)

一个 eBPF 程序需要任何它将引用的内核数据结构和类型的定义。在 CO-RE 出现之前，您通常需要弄清楚 Linux 内核源代码中的许多单独头文件中哪些包含您感兴趣的结构的定义，但现在有了一个更简单的方法，因为启用 BTF 的工具可以从内核中包含的 BTF 信息生成一个适当的头文件。

这个头文件通常称为*vmlinux.h*，您可以像这样使用`bpftool`生成它：

```
bpftool btf dump file /sys/kernel/btf/vmlinux format c > vmlinux.h
```

此文件定义了所有内核的数据类型，因此，在您的 eBPF 程序源代码中包含此生成的*vmlinux.h*文件将提供您可能需要的任何 Linux 数据结构的定义。当您将源代码编译为 eBPF 对象文件时，该对象将包含与此头文件中使用的定义匹配的 BTF 信息。稍后，在目标机器上运行程序时，将加载它到内核中的用户空间程序将调整以解决此构建时 BTF 信息与运行在目标机器上的内核的 BTF 信息之间的差异。

自 Linux 内核版本 5.4 以来，以*/sys/kernel/btf/vmlinux*文件形式的 BTF 信息已被包含在 Linux 内核中。^(6)但是，libbpf 可以利用的原始 BTF 数据也可以为旧内核生成。换句话说，如果您希望在目标机器上运行支持 CO-RE 的 eBPF 程序，而该目标机器没有 BTF 信息，您可能可以自己提供该目标机器的 BTF 数据。关于如何生成 BTF 文件以及各种 Linux 发行版的文件存档，请访问[BTFHub](https://oreil.ly/mPSO0)获取更多信息。

###### 注意

BTFHub 仓库还包括有关[BTF internals](https://oreil.ly/CfyQh)的进一步阅读，如果您希望深入了解此主题。

接下来，让我们看看如何使用这些策略以及其他策略来编写可通过 CO-RE 在各种内核间移植的 eBPF 程序。

# CO-RE eBPF 程序

您将回忆起 eBPF 程序在内核中运行。本章稍后将展示一些与内核中运行代码进行交互的用户空间代码，但本节集中在内核端。

正如你已经看到的那样，eBPF 程序被编译成 eBPF 字节码，（至少在撰写本文时）支持此功能的编译器有 Clang 或 gcc 用于编译 C 代码，以及 Rust 编译器。在第十章中，我将讨论一些使用 Rust 的选项，但在本章的目的上，我将假设你是在 C 中编写并使用 Clang，以及*libbpf*库。

在本章的其余部分，让我们考虑一个名为*hello-buffer-config*的示例应用程序。它与前一章中使用 BCC 框架的*hello-buffer-config.py*示例非常相似，但此版本是用 C 编写的，以使用*libbpf*和 CO-RE。

如果你有基于 BCC 的 eBPF 代码想要迁移到*libbpf*，请查看 Andrii Nakryiko 在他的网站上提供的出色且全面的[指南](https://oreil.ly/iWDcv)。BCC 提供了一些便捷的快捷方式，而使用*libbpf*并不完全相同；反之，*libbpf*提供了一套宏和库函数，使 eBPF 程序员的生活更加轻松。当我讲解示例时，我将指出 BCC 和*libbpf*方法之间的一些区别。

###### 注意

你可以在[*github.com/lizrice/learning-ebpf*](https://github.com/lizrice/learning-ebpf) repo 的*chapter5*目录中找到本节的示例 C eBPF 程序。

首先让我们看看*hello-buffer-config.bpf.c*，它实现了在内核中运行的 eBPF 程序。本章后面我会展示用户空间代码*hello-buffer-config.c*，它加载程序并显示输出，类似于 Python 代码在第四章中 BCC 实现的例子。

就像任何 C 程序一样，eBPF 程序也需要包含一些头文件。

## 头文件

*hello-buffer-config.bpf.c*的前几行指定了它所需要的头文件：

```
#include "vmlinux.h"
#include <bpf/bpf_helpers.h>
#include <bpf/bpf_tracing.h>
#include <bpf/bpf_core_read.h>
#include "hello-buffer-config.h"
```

这些五个文件是*vmlinux.h*文件，来自*libbpf*的几个头文件，以及我自己编写的一个应用程序特定的头文件。让我们看看为*libbpf*程序所需的头文件为什么是一个典型模式。

### 内核头信息

如果你编写的 eBPF 程序涉及任何内核数据结构或类型，最简单的选择是在本章前面描述的*vmliux.h*文件中包含它。或者，可以从 Linux 源中包含单独的头文件，或者如果你真的愿意的话，可以在自己的代码中手动定义类型。如果你将使用*libbpf*中的任何 BPF 助手函数，你需要包含*vmlinux.h*或*linux/types.h*以获取 BPF 助手源引用的`u32`、`u64`等类型的定义。

*vmlinux.h*文件源自内核源码头文件，但不包括来自它们的`#define`值。例如，如果你的 eBPF 程序解析以太网数据包，你可能需要常量定义来确定数据包的协议（比如`0x0800`表示 IP 数据包，或者`0x0806`表示 ARP 数据包）。有一系列常量值需要在你自己的代码中复制，如果你没有包含定义这些值的[*if_ether.h*文件](https://oreil.ly/hoZzP)。我在*hello-buffer-config*中不需要任何这些值定义，但在第八章中你会看到另一个例子，这里是相关的。

### 来自 libbpf 的头文件

要在你的 eBPF 代码中使用任何 BPF 辅助函数，你需要包含*libbpf*提供的头文件，这些头文件给出了它们的定义。

###### 注意

*libbpf*可能会让人稍感困惑的一点是，它不仅仅是一个用户空间库。你会发现自己在用户空间和 eBPF C 代码中都包含*libbpf*的头文件。

在撰写本文时，看到将*libbpf*作为子模块包含并从源代码构建/安装是很常见的——这是我在本书示例库中所做的。如果你将其作为子模块包含，你只需从*libbpf/src*目录运行`make install`即可。我认为不久之后，*libbpf*将更普遍地作为常见 Linux 发行版上的一个包提供，特别是因为*libbpf*现在已经发布了 1.0 版本的里程碑。

### 特定于应用程序的头文件

拥有一个特定于应用程序的头文件是非常常见的，该头文件定义了用户空间和 eBPF 应用程序部分都使用的任何结构。在我的示例中，*hello-buffer-config.h*头文件定义了我用来从 eBPF 程序传递事件数据到用户空间的`data_t`结构。它几乎与这段代码的 BCC 版本中看到的结构相同，如下所示：

```
struct data_t {
  int pid;
  int uid;
  char command[16];
  char message[12];
  char path[16];
};
```

与之前版本唯一的区别是我添加了一个名为`path`的字段。

将这个结构定义拉入一个单独的头文件的原因是，我将在*hello-buffer-config.c*用户空间代码中引用它。在 BCC 版本中，内核和用户空间代码都定义在一个文件中，BCC 在幕后做了一些工作，使得这个结构对 Python 用户空间代码可用。

## 定义映射

在包含头文件之后，源代码*hello-buffer-config.bpf.c*中的接下来几行定义了用于映射的结构，如下所示：

```
struct {
   __uint(type, BPF_MAP_TYPE_PERF_EVENT_ARRAY);
   __uint(key_size, sizeof(u32));
   __uint(value_size, sizeof(u32));
} output SEC(".maps");

struct user_msg_t {
  char message[12];
};

struct {
   __uint(type, BPF_MAP_TYPE_HASH);
   __uint(max_entries, 10240);
   __type(key, u32);
   __type(value, struct user_msg_t);
} my_config SEC(".maps");
```

这比我在等效的 BCC 示例中所需的代码行数更多！在 BCC 中，称为`config`的映射是用以下宏创建的：

```
BPF_HASH(config, u64, struct user_msg_t);
```

在不使用 BCC 时，此宏不可用，因此在 C 语言中，您必须手动编写它。您会看到我使用了`__uint`和`__type`。这些与[*bpf/bpf_helpers_def.h*](https://oreil.ly/2FgjB)中定义的`__array`一起使用，如下所示：

```
#define `__uint`(name, val) int (*name)[val]
#define `__type`(name, val) `typeof`(val) *name
#define `__array`(name, val) `typeof`(val) *name[]
```

在*libbpf*基于程序中通常按惯例使用这些宏，我认为它们使地图定义变得更容易阅读。

###### 注意

名称“config”与*vmlinux.h*中的定义冲突，因此我将地图重命名为“my_config”以供此示例使用。

## eBPF 程序部分

使用*libbpf*需要每个 eBPF 程序都标记为使用`SEC()`宏定义程序类型，如下所示：

```
SEC("kprobe")
```

这将在编译后的 ELF 对象中生成一个名为`kprobe`的部分，因此*libbpf*知道将其加载为`BPF_PROG_TYPE_KPROBE`。我们将在第七章中进一步讨论不同的程序类型。

根据程序类型的不同，您还可以使用部分名称来指定程序将附加到的事件。*libbpf*库将使用此信息自动设置附加项，而不是让您在用户空间代码中显式执行。例如，在基于 ARM 的机器上自动附加到`execve`系统调用的 kprobe，您可以像这样指定部分名称：

```
SEC("kprobe/__arm64_sys_execve")
```

这要求您知道该体系结构上该系统调用的函数名（或通过查看目标机器上的*/proc/kallsyms*文件找出，该文件列出所有内核符号，包括其函数名）。但是*libbpf*可以通过`k(ret)syscall`部分名称让您的生活更轻松，该名称告诉加载器自动附加到体系结构特定函数中的 kprobe：

```
SEC("ksyscall/execve")
```

###### 注意

[*libbpf*文档](https://oreil.ly/FhHrm)列出了有效的部分名称和格式。过去，部分名称的要求要宽松得多，因此您可能会遇到在*libbpf 1.0*之前编写的部分名称不匹配有效集的 eBPF 程序。不要让它们让您困惑！

部分定义声明了 eBPF 程序应附加的位置，然后是程序本身。与以前一样，eBPF 程序本身被编写为 C 函数。在示例代码中，它称为`hello()`，与您在第四章中看到的`hello()`函数非常相似。让我们考虑前一个版本与这里版本之间的区别：

```
SEC("ksyscall/execve")
int BPF_KPROBE_SYSCALL(hello, const char *pathname)                   ![1](img/1.png)
{
  struct data_t data = {};
  struct user_msg_t *p;

  data.pid = bpf_get_current_pid_tgid() >> 32;
  data.uid = bpf_get_current_uid_gid() & 0xFFFFFFFF;

  bpf_get_current_comm(&data.command, sizeof(data.command));
  bpf_probe_read_user_str(&data.path, sizeof(data.path), pathname);  ![2](img/2.png)

  p = bpf_map_lookup_elem(&my_config, &data.uid);                    ![3](img/3.png)
  if (p != 0) {
     bpf_probe_read_kernel(&data.message, sizeof(data.message), p->message);      
  } else {
     bpf_probe_read_kernel(&data.message, sizeof(data.message), message);
  }

  bpf_perf_event_output(ctx, &output, BPF_F_CURRENT_CPU,             ![4](img/4.png)
                        &data, sizeof(data));  
  return 0;
}
```

![1](img/#code_id_5_1)

我利用了在*libbpf*中定义的[`BPF_KPROBE_SYSCALL`](https://oreil.ly/pgI1B)宏，通过名字轻松访问系统调用的参数。对于`execve()`来说，第一个参数是将要执行的程序的路径名。eBPF 程序的名称是`hello`。

![2](img/#code_id_5_2)

由于宏使得访问`execve()`的路径名参数变得如此容易，我将其包含在发送到性能缓冲区输出的数据中。请注意，复制内存需要使用 BPF 辅助函数。

![3](img/#code_id_5_3)

在这里，`bpf_map_lookup_elem()`是 BPF 地图中查找值的辅助函数，给定一个键。BCC 的等效操作将是`p = my_config.lookup(&data.uid)`。在将 C 代码传递给编译器之前，BCC 会重写此操作以使用底层的`bpf_map_lookup_elem()`函数。当您使用*libbpf*时，在编译之前不会对代码进行任何重写，^(7)因此您必须直接写入辅助函数。

![4](img/#code_id_5_4)

这里是另一个类似的例子，我直接向辅助函数`bpf_perf_event_output()`写入了数据，在这里 BCC 给了我方便的等效操作`output.perf_submit(ctx, &data, sizeof(data))`。

唯一的另一个区别是，在 BCC 版本中，我将消息字符串定义为`hello()`函数内的局部变量。BCC 不支持（至少在撰写本文时不支持）全局变量。在此版本中，我将其定义为全局变量，如下所示：

```
char message[12] = "Hello World";
```

在*chapter4/hello-buffer-config.py*中，`hello`函数的定义方式略有不同，如下所示：

```
int hello(void *ctx)
```

`BPF_KPROBE_SYSCALL`宏是我提到的*libbpf*中的一个方便的新增功能之一。您不必使用该宏，但它会让生活变得更容易。它负责为传递给系统调用的所有参数提供命名参数。在这种情况下，它提供了一个`pathname`参数，该参数指向即将运行的可执行文件的路径字符串，这是传递给`execve()`系统调用的第一个参数。

如果您非常仔细地注意，您可能会注意到在我的*hello-buffer-config.bpf.c*源代码中，`ctx`变量在可见定义中并不存在，但尽管如此，我仍然能够在向输出性能缓冲区提交数据时使用它：

```
bpf_perf_event_output(ctx, &output, BPF_F_CURRENT_CPU, &data, sizeof(data));
```

`ctx`变量确实存在，隐藏在*libbpf*中[*bpf/bpf_tracing.h*](https://oreil.ly/pgI1B)内的`BPF_KPROBE_SYSCALL`宏定义中，您也会在其中找到一些关于此的评论。使用一个不明确定义的变量可能会让人感到有些困惑，但它非常有用，因为它可以被访问。

## CO-RE 下的内存访问

用于跟踪的 eBPF 程序对内存的访问受到限制，通过`bpf_probe_read_*()`家族的 BPF 辅助函数。^(8)（还有一个`bpf_probe_write_user()`辅助函数，但它仅用于[“实验”](https://oreil.ly/ibcy1)）。问题在于，正如您将在下一章看到的，eBPF 验证器通常不会允许您像在 C 中通常可以那样简单地通过指针读取内存（例如，`x = p->y`）。^(9)

*libbpf*库提供了围绕`bpf_probe_read_*()`助手函数的 CO-RE 包装器，以利用 BTF 信息并使内存访问调用跨不同内核版本可移植。以下是其中一个包装器的示例，如[*bpf_core_read.h*头文件](https://oreil.ly/XWWyc)中所定义：

```
#define bpf_core_read(dst, sz, src)                        \
 bpf_probe_read_kernel(dst, sz,                         \
 (const void *)__builtin_preserve_access_index(src))
```

如您所见，`bpf_core_read()`直接调用`bpf_probe_read_kernel()`，唯一的区别在于它用`__builtin_preserve_access_index()`包装了`src`字段。这告诉 Clang 在访问内存中这个地址时同时发出 CO-RE 重定位条目与 eBPF 指令。

###### 注意

此`__builtin_preserve_access_index()`指令是对“常规”C 代码的扩展，在 eBPF 中添加它也需要 Clang 编译器的更改以支持并发出这些 CO-RE 重定位条目。这些扩展是为什么一些 C 编译器（至少目前是这样）不能生成 eBPF 字节码的示例。有关为 eBPF CO-RE 支持所需的 Clang 更改的更多信息，请阅读[LLVM 邮件列表](https://oreil.ly/jHTHE)。

正如您稍后在本章中将看到的那样，CO-RE 重定位条目告诉*libbpf*在将 eBPF 程序加载到内核时重写地址，以考虑任何 BTF 差异。如果`src`在其包含结构中的偏移在目标内核上不同，重写的指令将考虑这一点。

*libbpf*库提供了`BPF_CORE_READ()`宏，这样您可以在单行中编写多个`bpf_core_read()`调用，而不需要为每个指针解引用编写单独的辅助函数调用。例如，如果您想要做类似`d = a->b->c->d`的操作，可以编写以下代码：

```
struct b_t *b;
struct c_t *c;

bpf_core_read(&b, 8, &a->b);
bpf_core_read(&c, 8, &b->c);
bpf_core_read(&d, 8, &c->d);
```

但使用以下方式要紧凑得多：

```
d = BPF_CORE_READ(a, b, c, d);
```

您可以使用`bpf_probe_read_kernel()`助手函数从`d`点读取。

关于这一点，Andrii 的[指南](https://oreil.ly/tU0Gb)有很好的描述。

## 许可证定义

正如您从第三章已经了解到的那样，eBPF 程序必须声明其许可证。示例代码如下所示：

```
char LICENSE[] SEC("license") = "Dual BSD/GPL";
```

您现在已经看到了*hello-buffer-config.bpf.c*示例中的所有代码。现在让我们将其编译成一个对象文件。

# 为 CO-RE 编译 eBPF 程序

在第三章中，您看到了一个从 Makefile 中提取出来的内容，用于将 C 编译为 eBPF 字节码。让我们深入研究所使用的选项，并了解它们对 CO-RE/*libbpf*程序的必要性。

## 调试信息

您必须向 Clang 传递`-g`标志，以便包含调试信息，这对于 BTF 是必要的。然而，`-g`标志也会向输出的对象文件添加 DWARF 调试信息，但 eBPF 程序不需要这些，因此您可以通过运行以下命令将其剥离以减少对象文件的大小：

```
llvm-strip -g <object file>
```

## 优化

`-O2`优化标志（2 级或更高）是 Clang 生成将通过验证器的 BPF 字节码所必需的。一个需要这样做的例子是，默认情况下，Clang 将输出`callx <register>`来调用辅助函数，但 eBPF 不支持从寄存器调用地址。

## 目标架构

如果您使用*libbpf*定义的某些宏，您需要在编译时指定目标架构。*libbpf*头文件*bpf/bpf_tracing.h*定义了几个特定于平台的宏，例如`BPF_KPROBE`和`BPF_KPROBE_SYSCALL`，我在本示例中使用了这些宏。`BPF_KPROBE`宏可用于将 eBPF 程序附加到 kprobes 上，而`BPF_KPROBE_SYSCALL`是专门用于系统调用 kprobes 的变体。

kprobe 的参数是一个`pt_regs`结构，其中保存了 CPU 寄存器内容的副本。由于寄存器是与架构相关的，`pt_regs`结构定义取决于您正在运行的架构。这意味着，如果您想要使用这些宏，您还需要告诉编译器目标架构是什么。您可以通过设置`-D __TARGET_ARCH_($ARCH)`来实现这一点，其中`$ARCH`是一个架构名称，如 arm64、amd64 等。

还要注意，如果您没有使用该宏，您仍然需要针对 kprobe 访问寄存器信息的特定于架构的代码。

或许“每个架构编译一次，到处运行”会有点啰嗦！

## Makefile

以下是一个用于编译 CO-RE 对象的示例 Makefile 指令（取自本书 GitHub 存储库中*chapter5*目录中的 Makefile）：

```
hello-buffer-config.bpf.o: %.o: %.c
   clang \
       -target bpf \
       -D __TARGET_ARCH_$(ARCH) \
       -I/usr/include/$(shell uname -m)-linux-gnu \
       -Wall \
       -O2 -g \
       -c $< -o $@
   llvm-strip -g $@
```

如果您正在使用示例代码，您应该能够通过在*chapter5*目录中运行`make`来构建 eBPF 目标文件*hello-buffer-config.bpf.o*（以及我将很快描述的其伴随的用户空间可执行文件）。让我们检查该目标文件，看看它是否包含 BTF 信息。

## 目标文件中的 BTF 信息

[BTF 的内核文档](https://oreil.ly/5QrBy)描述了 BTF 数据如何在 ELF 目标文件中以两个部分进行编码：*.BTF*，其中包含数据和字符串信息，以及*.BTF.ext*，其中包含函数和行信息。您可以使用`readelf`查看这些部分已添加到目标文件中，如下所示：

```
$ readelf -S hello-buffer-config.bpf.o | grep BTF
  [10] .BTF              PROGBITS         0000000000000000  000002c0
  [11] .rel.BTF          REL              0000000000000000  00000e50
  [12] .BTF.ext          PROGBITS         0000000000000000  00000b18
  [13] .rel.BTF.ext      REL              0000000000000000  00000ea0
```

`bpftool`实用程序允许我们检查来自目标文件的 BTF 数据，如下所示：

```
bpftool btf dump file hello-buffer-config.bpf.o
```

输出看起来就像您在本章前面看到的从加载的程序和映射中转储 BTF 信息时获得的输出。

让我们看看如何使用这些 BTF 信息来使程序能够在具有不同内核版本和不同数据结构的另一台机器上运行。

# BPF 重定位

*libbpf*库使 eBPF 程序适应它们运行的目标内核上的数据结构布局，即使此布局与编译代码的内核不同。为此，*libbpf*需要 Clang 在编译过程中生成的 BPF CO-RE 重定位信息。

您可以从[*linux/bpf.h*](https://elixir.bootlin.com/linux/v5.19.17/source/include/uapi/linux/bpf.h#L6711)头文件中`struct bpf_core_relo`的定义中了解重定位是如何工作的：

```
struct `bpf_core_relo` {
    `__u32` `insn_off`;
    `__u32` `type_id`;
    `__u32` `access_str_off`;
    enum `bpf_core_relo_kind` `kind`;
};
```

一个 eBPF 程序的 CO-RE 重定位数据由每个需要重定位的指令的一个结构体组成。假设指令将寄存器设置为结构体中字段的值，则该指令的`bpf_core_relo`结构体（由`insn_off`字段标识）编码该结构体的 BTF 类型（`type_id`字段）并指示如何相对于该结构体访问字段（`access_str_off`字段）。

正如您刚才看到的，Clang 会自动生成内核数据结构的重定位数据，并编码到 ELF 对象文件中。是以下代码，您会在*vmlinux.h*文件的开头附近找到，导致 Clang 执行此操作：

```
#pragma clang attribute push (__attribute__((preserve_access_index)), \
 apply_to = record)
```

`preserve_access_index`属性告诉 Clang 为类型定义生成 BPF CO-RE 重定位。`clang attribute push`部分表示该属性应用于直到文件末尾的所有定义的情况下，会有`clang attribute pop`。这意味着 Clang 为*vmlinux.h*中定义的所有类型生成重定位信息。

当您加载 BPF 程序时，可以使用`bpftool`并通过`-d`标志打开调试信息，如下所示，查看重定位的发生：

```
bpftool -d prog load hello.bpf.o /sys/fs/bpf/hello
```

这会生成大量输出，但与重定位相关的部分如下所示：

```
libbpf: CO-RE relocating [24] struct user_pt_regs: found target candidate [205]
struct user_pt_regs in [vmlinux]
libbpf: prog 'hello': relo #0: <byte_off> [24] struct user_pt_regs.regs[0]
(0:0:0 @ offset 0)
libbpf: prog 'hello': relo #0: matching candidate #0 <byte_off> [205] struct
user_pt_regs.regs[0] (0:0:0 @ offset 0)
libbpf: prog 'hello': relo #0: patched insn #1 (LDX/ST/STX) off 0 -> 0
```

在这个例子中，您可以看到`hello`程序的 BTF 信息中类型 ID 为 24 的结构体称为`user_pt_regs`。*libbpf*库将其与内核结构体匹配，也称为`user_pt_regs`，该结构体在*vmlinux* BTF 数据集中的类型 ID 为 205。实际上，因为我在同一台机器上编译和加载了程序，类型定义是相同的，所以在这个例子中，从结构体开始的偏移量保持不变，并且对第 1 条指令的“补丁”也保持不变。

在许多应用程序中，您可能不希望要求用户运行`bpftool`来加载 eBPF 程序。相反，您希望将此功能构建到您提供的专用用户空间程序中作为可执行文件。让我们考虑如何编写这个用户空间代码。

# CO-RE 用户空间代码

不同编程语言中有几种不同的框架支持 CO-RE，通过在加载 eBPF 程序到内核时实现重定位。在本章中，我将展示使用*libbpf*的 C 代码；其他选项包括 Go 包*cilium/ebpf*和*libbpfgo*，以及 Rust 的 Aya。我将在第十章进一步讨论这些选项。

# 用户空间的 Libbpf 库

*libbpf*库是一个用户空间库，如果你用 C 语言编写应用程序的用户空间部分，可以直接使用它。如果愿意，你可以在不使用 CO-RE 的情况下使用这个库。[Andrii Nakryiko 在*libbpf-bootstrap*上有一个例子](https://oreil.ly/b3v7B)。

此库提供的函数封装了你在第四章中遇到的`bpf()`和相关系统调用，用于像将程序加载到内核中并将其附加到事件，或者从用户空间访问映射信息的操作。使用这些抽象的传统和最简单的方式是通过自动生成的 BPF 骨架代码。

## BPF 骨架

你可以使用`bpftool`从现有的以 ELF 文件格式存在的 eBPF 对象中自动生成这些骨架代码，如下所示：

```
bpftool gen skeleton hello-buffer-config.bpf.o > hello-buffer-config.skel.h
```

查看这个骨架头文件，你会看到它包含了 eBPF 程序和映射的结构定义，以及几个以`hello_buffer_config_bpf__`开头的函数（根据对象文件的名称）。这些函数管理 eBPF 程序和映射的生命周期。如果你喜欢，你可以直接调用*libbpf*，而不必使用骨架代码，但通常自动生成的代码可以节省一些输入。

在生成的骨架文件末尾，你会看到一个名为`hello_buffer_config_bpf__elf_bytes`的函数，它返回 ELF 对象文件*hello-buffer-config.bpf.o*的字节内容。一旦生成了骨架，我们实际上不再需要该对象文件。你可以通过运行`make`生成`hello-buffer-config`可执行文件，并删除*.o*文件来测试这一点；可执行文件中包含 eBPF 字节码。

###### 注意

如果你愿意，你可以使用*libbpf*函数`bpf_object__open_file`从 ELF 文件加载 eBPF 程序和映射，而不是使用骨架文件的字节。

下面是管理此示例中 eBPF 程序和映射生命周期的用户空间代码概述，使用生成的骨架代码。为了清晰起见，我省略了部分细节和错误处理，但你可以在*chapter5/hello-buffer-config.c*中找到完整的源代码。

```
... [other #includes]
#include "hello-buffer-config.h"                                       ![1](img/1.png)
#include "hello-buffer-config.skel.h"

... [some callback functions]

int main()
{
   struct hello_buffer_config_bpf *skel;
   struct perf_buffer *pb = NULL;
   int err;

   libbpf_set_print(libbpf_print_fn);                                 ![2](img/2.png)

   skel = hello_buffer_config_bpf__open_and_load();                   ![3](img/3.png)
...
   err = hello_buffer_config_bpf__attach(skel);                       ![4](img/4.png)
...
   pb = perf_buffer__new(bpf_map__fd(skel->maps.output), 8, handle_event,
                                                         lost_event, NULL, NULL);                                              
                                                                      ![5](img/5.png)
...
   while (true) {                                                     ![6](img/6.png)
       err = perf_buffer__poll(pb, 100);
...}

   perf_buffer__free(pb);                                             ![7](img/7.png)
   hello_buffer_config_bpf__destroy(skel);
   return -err;
}
```

![1](img/#code_id_5_5)

此文件包括自动生成的骨架头文件，以及我手动编写的用于用户空间和内核代码共享的头文件。

![2](img/#code_id_5_6)

此代码设置了一个回调函数，用于打印由*libbpf*生成的任何日志消息。

![3](img/#code_id_5_7)

在这里创建了一个 `skel` 结构，表示 ELF 字节中定义的所有映射和程序，并将它们加载到内核中。

![4](img/#code_id_5_8)

程序会自动附加到适当的事件上。

![5](img/#code_id_5_9)

此函数创建用于处理性能缓冲区输出的结构。

![6](img/#code_id_5_10)

在此连续轮询性能缓冲区。

![7](img/#code_id_5_11)

这是清理代码。

让我们更详细地探讨其中的一些步骤。

### 将程序和映射加载到内核中

对自动生成的第一个调用是这个函数：

```
skel = hello_buffer_config_bpf__open_and_load();
```

正如其名称所示，此函数涵盖了两个阶段：打开和加载。 “打开” 阶段涉及读取 ELF 数据并将其节转换为表示 eBPF 程序和映射的结构。 “加载” 阶段将这些映射和程序加载到内核中，并根据需要执行任何 CO-RE 修复。

这两个阶段可以轻松分开处理，因为骨架代码提供了分开的 `name__open()` 和 `name__load()` 函数。这使您可以在加载之前操作 eBPF 信息。例如，我可以将一个计数器全局变量 `c` 初始化为某个值，如下所示：

```
skel = hello_buffer_config_bpf__open();
if (!skel) {
    // Error ...
}   
skel->data->c = 10;
err = hello_buffer_config_bpf__load(skel);
```

`hello_buffer_config_bpf__open()` 返回的数据类型，以及 `hello_buffer_config_bpf__load()` 返回的数据类型，都是一个名为 `hello_buffer_config_bpf` 的结构体，在骨架头文件中定义了有关对象文件中定义的所有映射、程序和数据的信息。

###### 注意

骨架对象（在本例中为 `hello_buffer_config_bpf`）只是来自 ELF 字节信息的用户空间表示。一旦加载到内核中，如果在加载后更改 `skel->data->c` 的值，它不会对内核端数据产生任何影响。

### 访问现有映射

默认情况下，*libbpf* 也会创建在 ELF 字节中定义的任何映射，但有时您可能希望编写一个 eBPF 程序，该程序重用现有的映射。在上一章中已经看到了一个例子，您看到 `bpftool` 遍历所有映射，查找与指定名称匹配的映射。使用映射的另一个常见原因是在两个不同的 eBPF 程序之间共享信息，因此只有一个程序应该创建映射。`bpf_map__set_autocreate()` 函数允许您覆盖 *libbpf* 的自动创建。

那么如何访问现有映射呢？映射可以被固定，如果您知道固定路径，可以使用 `bpf_obj_get()` 获取现有映射的文件描述符。以下是一个非常简单的示例（在 GitHub 存储库中作为 *chapter5/find-map.c* 可用）：

```
struct bpf_map_info info = {};
unsigned int len = sizeof(info);

int findme = bpf_obj_get("/sys/fs/bpf/findme");
if (findme <= 0) {
    printf("No FD\n");
} else {
    bpf_obj_get_info_by_fd(findme, &info, &len);
    printf("Name: %s\n", info.name);
}
```

您可以使用 `bpftool` 创建一个映射来尝试这一点，就像这样：

```
$ bpftool map create /sys/fs/bpf/findme type array key 4 value 32 entries 4
name findme
```

运行 find-map 可执行文件将输出：

```
Name: findme
```

让我们回到 *hello-buffer-config* 示例和骨架代码。

### 附加到事件

下面的示例中的下一个骨架函数将程序附加到`execve`系统调用函数：

```
err = hello_buffer_config_bpf__attach(skel);
```

*libbpf*库会自动从该程序的`SEC()`定义中获取附加点。如果您没有完全定义附加点，那么还有一系列*libbpf*函数，例如`bpf_program__attach_kprobe`、`bpf_program__attach_xdp`等，用于附加不同类型的程序。

### 管理事件缓冲区

设置 perf 缓冲区使用的函数是在*libbpf*库本身中定义的，而不是在骨架中：

```
pb = perf_buffer__new(bpf_map__fd(skel->maps.output), 8, handle_event,
                                                         lost_event, NULL, NULL);
```

您可以看到`perf_buffer__new()`函数将“输出”映射的文件描述符作为第一个参数。`handle_event`参数是一个回调函数，当 perf 缓冲区中有新数据到达时会调用它，如果内核没有足够的空间写入数据条目，则会调用`lost_event`。在我的示例中，这些函数只是将消息写入屏幕。

最后，程序必须重复轮询 perf 缓冲区：

```
while (true) {
   err = perf_buffer__poll(pb, 100);
   ...
}
```

100 是毫秒级的超时时间。以前设置的回调函数将在数据到达或缓冲区已满时适时调用。

最后，为了清理，我释放 perf 缓冲区并销毁内核中的 eBPF 程序和映射，如下所示：

```
perf_buffer__free(pb);
hello_buffer_config_bpf__destroy(skel);
```

*libbpf*中有一整套与`perf_buffer_*`和`ring_buffer_*`相关的函数，帮助您管理事件缓冲区。

如果您制作并运行此示例`hello-buffer-config`程序，您将看到以下输出（与您在第四章中看到的非常相似）：

```
23664  501    bash             Hello World
23665  501    bash             Hello World
23667  0      cron             Hello World
23668  0      sh               Hello World
```

## Libbpf 代码示例

有许多基于*libbpf*的 eBPF 程序的优秀示例可供使用，可以作为编写自己程序的灵感和指导：

+   [*libbpf-bootstrap*](https://oreil.ly/zB0Co)项目旨在帮助您通过一组示例程序入门。

+   BCC 项目已经将许多原始的基于 BCC 的工具迁移到*libbpf*版本。您可以在[*libbpf-tools*目录](https://oreil.ly/Z9xDX)中找到它们。

# 摘要

CO-RE 使得 eBPF 程序能够在与其构建时不同的内核版本上运行。这极大地提高了 eBPF 的可移植性，并且为希望向用户和客户交付生产就绪工具的工具开发人员带来了极大的便利。

在本章中，您看到了 CO-RE 是如何通过将类型信息编码到编译后的对象文件中，并使用重定位来在加载到内核时重写指令来实现的。您还介绍了如何在 C 中编写使用*libbpf*的代码：既在内核中运行的 eBPF 程序，又在用户空间中管理这些程序的生命周期，基于自动生成的 BPF 骨架代码。在下一章中，您将学习内核如何验证 eBPF 程序是否安全可运行。

# 练习

以下是您可以进一步探索 BTF、CO-RE 和*libbpf*的一些事项：

1.  尝试使用`bpftool btf dump map`和`bpftool btf dump prog`来查看与映射和程序相关的 BTF 信息。请记住，你可以用多种方式指定单独的映射和程序。

1.  比较在 ELF 对象文件形式及加载到内核后的相同程序的`bpftool btf dump file`和`bpftool btf dump prog`的输出。它们应该是相同的。

1.  检查来自*bpftool -d prog load hello-buffer-config.bpf.o /sys/fs/bpf/hello*的调试输出。你将看到每个节被加载，对许可证的检查以及正在进行的重定位，以及描述每条 BPF 程序指令的输出。

1.  尝试使用来自 BTFHub 的不同*vmlinux*头文件构建 BPF 程序，并查看`bpftool`的调试输出，以查找更改偏移量的重定位。

1.  修改*hello-buffer-config.c*程序，以便可以使用映射为不同用户 ID 配置不同的消息（类似于第四章中的*hello-buffer-config.py*示例）。

1.  尝试更改`SEC();`中的节名称，也许改成你自己的名字。当你加载程序到内核时，你应该会看到一个错误，因为*libbpf*不认识节名称。这说明了*libbpf*如何使用节名称来确定这是什么类型的 BPF 程序。你可以尝试编写自己的附加代码，显式地附加到你选择的事件，而不依赖于*libbpf*的自动附加。

^(1) 严格来说，数据结构定义来自内核头文件，你可以选择基于一组不同于用于构建运行在目标机器上的内核的头文件编译的这些头文件。要正确工作（没有本章描述的 CO-RE 机制），内核头文件必须与将运行 eBPF 程序的目标机器上的内核兼容。

^(2) 本节部分内容改编自 Liz Rice 的“What Is eBPF?”。版权所有 © 2022 O’Reilly Media。已获授权使用。

^(3) 根据一项小规模且非科学的调查，大多数人发音与单词*core*相同，而不是分成两个音节。

^(4) 请参阅内核文档[*https://docs.kernel.org/bpf/btf.html#type-encoding*](https://docs.kernel.org/bpf/btf.html#type-encoding)。

^(5) 内核需要启用`CONFIG_DEBUG_INFO_BTF`选项进行构建。

^(6) 哪个是支持 BTF 的最早的 Linux 内核版本？参见[*https://oreil.ly/HML9m*](https://oreil.ly/HML9m)。

^(7) 好吧，正常的 C 预处理适用于你可以做`#define`等事情。但是没有像使用 BCC 时那样的*特殊*重写。

^(8) 处理网络数据包的 eBPF 程序无法使用这个帮助函数，只能访问网络数据包的内存。

^(9) 在某些启用了 BTF 的程序类型中是允许的，比如 `tp_btf`、`fentry` 和 `fexit`。
