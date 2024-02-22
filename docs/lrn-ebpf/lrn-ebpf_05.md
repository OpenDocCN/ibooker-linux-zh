# 第五章 CO-RE、BTF 和 Libbpf

在上一章中，您第一次遇到了 BTF（BPF 类型格式）。本章讨论了它存在的原因以及它如何用于使 eBPF 程序在不同版本的内核之间可移植。这是 BPF“编译一次，到处运行”（CO-RE）方法的关键部分，它解决了使 eBPF 程序在不同内核版本之间可移植的问题。

许多 eBPF 程序访问内核数据结构，eBPF 程序员需要包含相关的 Linux 头文件，以便他们的 eBPF 代码可以正确地定位这些数据结构中的字段。然而，Linux 内核正在不断发展，这意味着在不同的内核版本之间内部数据结构可能会发生变化。如果您将在一台机器上编译的 eBPF 对象文件¹加载到具有不同内核版本的机器上，就无法保证数据结构会相同。

CO-RE 方法在以高效的方式解决这个可移植性问题方面迈出了重要的一步。它允许 eBPF 程序包含有关它们编译时使用的数据结构布局的信息，并提供了一种机制，用于在目标机器上运行时调整字段的访问方式，如果数据结构布局在目标机器上是不同的。只要程序不想访问目标机器内核中根本不存在的字段或数据结构，程序就可以在不同的内核版本之间进行移植。

但在我们深入讨论 CO-RE 的工作原理之前，让我们通过查看 BCC 项目最初实施的内核可移植性的先前方法，来讨论为什么它是如此令人向往的。

# BCC 的可移植性方法

在第二章中，我使用[BCC](https://oreil.ly/ReUtn)展示了一个基本的“Hello World”eBPF 程序示例。BCC 项目是第一个流行的实现 eBPF 程序的项目，为具有较少内核经验的程序员提供了相对易于访问的用户空间和内核方面的框架。为了解决跨内核的可移植性问题，BCC 采取了在目标机器上即时编译 eBPF 代码的方法。这种方法存在一些问题：

+   编译工具链需要安装在您希望代码运行的每台目标机器上，以及内核头文件（这并不总是默认存在的）。

+   在工具启动之前，您必须等待编译完成，这可能意味着每次启动工具都会有几秒钟的延迟。

+   如果您在大量相同的机器上运行工具，那么在每台机器上重复编译是一种计算资源的浪费。

+   一些基于 BCC 的项目将它们的 eBPF 源代码和工具链打包到一个容器映像中，这样可以更容易地将其分发到每台机器上。但这并不能解决确保内核头文件存在的问题，甚至可能意味着更多的重复，如果这些 BCC 容器中的几个安装在每台机器上。

+   嵌入式设备可能没有足够的内存资源来运行编译步骤。

由于这些问题，如果您计划着手开发一个重要的新的 eBPF 项目，我不建议使用这种传统的 BCC 方法，特别是如果您计划将其分发给其他人使用。在本书中，我给出了一些基于 BCC 的例子，因为它是学习 eBPF 基本概念的一个很好的方法，特别是因为 Python 用户空间代码如此紧凑且易于阅读。如果您更喜欢它并且想快速地组合一些东西，那么它也是一个完全不错的选择。但对于严肃的现代 eBPF 开发来说，这并不是最佳的方法。

CO-RE 方法为 eBPF 程序的跨内核可移植性问题提供了一个更好的解决方案。

###### 注意

[*github.com/iovisor/bcc*](https://oreil.ly/ReUtn)上的 BCC 项目包括一系列用于观察 Linux 机器行为的命令行工具。位于[*tools*](https://oreil.ly/fI4w_)目录中的原始版本大多使用 Python 实现，使用了我在本节中描述的传统可移植性方法。

在 BCC 的[*libbpf-tools*](https://oreil.ly/ke7yq)目录中，您将找到用 C 编写的这些工具的更新版本，它们利用了*libbpf*和 CO-RE，并且不会遇到我刚列出的问题。它们是一组非常有用的实用程序！

# CO-RE 概述

CO-RE 方法包括几个元素：²^,³

BTF

[BTF](https://oreil.ly/iRCuI)是一种用于表达数据结构和函数签名布局的格式。在 CO-RE 中，它用于确定编译时和运行时结构之间的任何差异。BTF 也被`bpftool`等工具用于以人类可读的格式转储数据结构。从 5.4 版本开始的 Linux 内核支持 BTF。

内核头文件

Linux 内核源代码包括描述其使用的数据结构的头文件，这些头文件在不同版本的 Linux 之间可能会发生变化。eBPF 程序员可以选择包含单个头文件，或者，正如您将在本章中看到的，您可以使用`bpftool`从运行中的系统生成一个名为*vmlinux.h*的头文件，其中包含 BPF 程序可能需要的有关内核的所有数据结构信息。

编译器支持

[Clang 编译器进行了增强](https://oreil.ly/6xFJm)，以便在使用`-g`标志编译 eBPF 程序时，包含所谓的*CO-RE 重定位*，这些重定位来自描述内核数据结构的 BTF 信息。GCC 编译器在[12 版本](https://oreil.ly/_6PEE)中还为 BPF 目标添加了 CO-RE 支持。

数据结构重定位的库支持

在用户空间程序加载 eBPF 程序到内核时，CO-RE 方法要求调整字节码以补偿编译时存在的数据结构与即将运行的目标机器上的数据结构之间的任何差异，这是基于编译到对象中的 CO-RE 重定位信息。有几个库可以处理这个问题：[*libbpf*](https://oreil.ly/E742u)是最初包含此重定位功能的 C 库，Cilium eBPF 库为 Go 程序员提供了相同的功能，Aya 则为 Rust 提供了这个功能。

可选地，BPF 骨架

骨架可以从编译的 BPF 对象文件中自动生成，其中包含方便的函数，用户空间代码可以调用这些函数来管理 BPF 程序的生命周期——将它们加载到内核中，将它们附加到事件等等。如果您用 C 编写用户空间代码，可以使用`bpftool gen skeleton`生成骨架。这些函数是更高级的抽象，对于开发人员来说可能比直接使用底层库（*libbpf*、*cilium/ebpf*等）更方便。

###### 注意

Andrii Nakryiko 写了一篇[优秀的博客文章](https://oreil.ly/aeQJo)，描述了 CO-RE 的背景，以及它的工作原理和如何使用它。他还写了权威的[BPF CO-RE 参考指南](https://oreil.ly/lbW_T)，所以如果您要开始编写代码，请务必阅读。他的[*libbpf-bootstrap*指南](https://oreil.ly/_jet-)介绍了如何使用 CO-RE + *libbpf* +骨架从头开始构建 eBPF 应用程序，也是必读之物。

现在您已经了解了 CO-RE 的元素，让我们深入了解它们的工作原理，从探索 BTF 开始。

# BPF 类型格式

BTF 信息描述了数据结构和代码在内存中的布局。这些信息可以用于各种不同的用途。

## BTF 用例

在 CO-RE 章节中讨论 BTF 的主要原因是，了解 eBPF 程序编译的结构布局与即将运行的结构布局之间的差异，可以在程序加载到内核时进行适当的调整。我将在本章后面讨论重定位过程，但现在，让我们也考虑一下 BTF 信息可以用于的其他一些用途。

了解结构的布局方式以及结构中每个字段的类型，可以使得以人类可读的形式打印结构的内容成为可能。例如，从计算机的角度来看，字符串只是一系列字节，但将这些字节转换为字符使得字符串对人类来说更容易理解。在上一章中，您已经看到了这方面的一个例子，`bpftool`使用 BTF 信息来格式化映射转储的输出。

BTF 信息还包括行和函数信息，使得`bpftool`能够在翻译或 JIT 程序转储的输出中交错源代码，就像您在第三章中看到的那样。当您来到第六章时，您还将看到源代码信息与验证器日志输出交错，这同样来自 BTF 信息。

BTF 信息也是 BPF 自旋锁所必需的。*自旋锁*用于阻止两个 CPU 核同时访问相同的映射值。锁必须是映射值结构的一部分，就像这样：

```cpp
struct my_value {
     ... <other fields>
     struct bpf_spin_lock lock;
... <other fields>
};
```

在内核中，eBPF 程序使用`bpf_spin_lock()`和`bpf_spin_unlock()`辅助函数来获取和释放锁。只有在 BTF 信息可用以描述结构中锁字段的位置时，才能使用这些辅助函数。

###### 注意

自旋锁支持是在内核版本 5.1 中添加的。对自旋锁的使用有很多限制：它们只能用于哈希或数组映射类型，并且不能用于跟踪或套接字过滤类型的 eBPF 程序。在[lwn.net 关于 BPF 并发管理的文章](https://oreil.ly/kAyAU)中了解更多关于自旋锁的信息。

现在您知道 BTF 信息的用途，让我们通过查看一些示例来使其更具体。

## 使用 bpftool 列出 BTF 信息

与程序和映射一样，您可以使用`bpftool`实用程序显示 BTF 信息。以下命令列出了加载到内核中的所有 BTF 数据：

```cpp
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

（出于简洁起见，我省略了许多条目。）

列表中的第一个条目是`vmlinux`，它对应于我之前提到的*vmlinux*文件，该文件保存了有关当前运行内核的 BTF 信息。

###### 注意

本章早期的一些示例重用了第四章中的程序，然后在本章后期，您将找到新的示例，其源代码位于[*github.com/lizrice/learning-ebpf*](https://github.com/lizrice/learning-ebpf)的*chapter5*目录中。

为了获得此示例输出，我在运行第四章中的`hello-buffer-config`示例时运行了此命令。您可以在以`149:`开头的行上看到描述此进程正在使用的 BTF 信息的条目：

```cpp
149: name <anon>  size 4372B  prog_ids 319  map_ids 103
        pids hello-buffer-co(7660)
```

这行告诉我们的是：

+   这个 BTF 信息块的 ID 是 149。

+   这是一个大约 4KB 的 BTF 信息的匿名 blob。

+   它被具有`prog_id 319`的 BPF 程序和具有`map_id 103`的 BPF 映射使用。

+   它还被 ID 为 7660 的进程（括号内显示）使用，运行名为`hello-buffer-config`的可执行文件（其名称已被截断为 15 个字符）。

这些程序、映射和 BTF 标识符与`bpftool`显示的有关`hello-buffer-config`的名为`hello`的程序的输出相匹配：

```cpp
bpftool prog show name hello
319: kprobe  name hello  tag a94092da317ac9ba  gpl
        loaded_at 2022-08-28T14:13:35+0000  uid 0
        xlated 400B  jited 428B  memlock 4096B  map_ids 103,104
        btf_id 149
        pids hello-buffer-co(7660)
```

唯一似乎不完全匹配的是程序引用了额外的`map_id`，`104`。那是性能事件缓冲区映射，它不使用 BTF 信息；因此，它不会出现在与 BTF 相关的输出中。

就像`bpftool`可以转储程序和映射的内容一样，它也可以用来查看包含在数据块中的 BTF 类型信息。

## BTF 类型

知道 BTF 信息的 ID 后，可以使用命令`bpftool btf dump id <id>`来检查其内容。当我使用之前获得的 ID 149 运行时，我得到了 69 行输出，每行都是一个类型定义。我只描述前几行，这应该能让你很好地理解如何解释其余部分。这些前几行的 BTF 信息与在源代码中定义的`config`哈希映射相关：

```cpp
structuser_msg_t{ `char``message``[``12``];` ``};` ``BPF_HASH``(``config``,``u32``,``struct``user_msg_t``);```

```cpp

 ```这个哈希表的键的类型是`u32`，值的类型是`struct user_msg_t`。该结构包含一个 12 字节的`message`字段。让我们看看这些类型在相应的 BTF 信息中是如何定义的。

BTF 输出的前三行如下：

```cpp
[1] TYPEDEF 'u32' type_id=2
[2] TYPEDEF '__u32' type_id=3
[3] INT 'unsigned int' size=4 bits_offset=0 nr_bits=32 encoding=(none)
```

每行开头的方括号中的数字是类型 ID（因此第一行，以`[1]`开头，定义了`type_id 1`，依此类推）。让我们更详细地了解这三种类型：

+   类型 1 定义了一个名为`u32`的类型及其类型，由`type_id 2`定义，即在以`[2]`开头的行中定义的类型。你知道，哈希表中的键具有这种类型`u32`。

+   类型 2 的名称是`__u32`，由`type_id 3`定义的类型。

+   类型 3 是一个名为`unsigned int`的整数类型，长度为 4 字节。

这三种类型都是 32 位无符号整数类型的同义词。在 C 中，整数的长度取决于平台，因此 Linux 定义了像`u32`这样的类型，以明确定义特定长度的整数。在这台机器上，`u32`对应于无符号整数。引用这些的用户空间代码应该使用带下划线前缀的同义词，如`__u32`。

BTF 输出中的接下来几种类型如下：

```cpp
[4] STRUCT 'user_msg_t' size=12 vlen=1
        'message' type_id=6 bits_offset=0
[5] INT 'char' size=1 bits_offset=0 nr_bits=8 encoding=(none)
[6] ARRAY '(anon)' type_id=5 index_type_id=7 nr_elems=12
[7] INT '__ARRAY_SIZE_TYPE__' size=4 bits_offset=0 nr_bits=32 encoding=(none)
```

这些与`config`映射中的值使用的`user_msg_t`结构相关：

+   类型 4 是`user_msg_t`结构本身，总共有 12 个字节长。它包含一个名为`message`的字段，由类型 6 定义。`vlen`字段指示了这个定义中有多少个字段。

+   类型 5 的名称是`char`，是一个 1 字节的整数——这正是 C 程序员对名为“char”的类型所期望的定义。

+   类型 6 将`message`字段定义为一个具有 12 个元素的数组。每个元素的类型为 5（它是一个`char`），并且该数组由类型 7 索引。

+   类型 7 是一个 4 字节的整数。

有了这些定义，你可以完整地了解`user_msg_t`结构在内存中的布局，就像图 5-1 中所示的那样。

![user_msg_t 结构占用 12 字节内存](img/lebp_0501.png)

###### 图 5-1\. `user_msg_t`结构占用 12 字节内存

到目前为止，所有条目的`bits_offset`都设置为`0`，但是输出的下一行有一个具有多个字段的结构：

```cpp
[8] STRUCT '____btf_map_config' size=16 vlen=2
        'key' type_id=1 bits_offset=0
        'value' type_id=4 bits_offset=32
```

这是一个用于存储名为`config`的映射中的键值对的结构定义。我没有在源代码中定义这个`____btf_map_config`类型，但它是由 BCC 生成的。键的类型是`u32`，值是`user_msg_t`结构。这些对应于之前看到的类型 1 和 4。

关于这个结构的 BTF 信息的另一个重要部分是`value`字段在结构开始后 32 位开始。这完全合理，因为前 32 位需要保存`key`字段。

###### 注意

在 C 中，结构字段会自动对齐到边界，因此不能简单地假设一个字段总是直接跟在前一个字段的内存中。例如，考虑这样一个结构：

```cpp
structsomething{ `char``letter``;`
`u64``number``;` ``}``
```

```cppThere would be 7 bytes of unused memory after the field called `letter` before the `number` field so that the 64-bit number can be aligned to a memory location divisible by 8.

It’s possible in some circumstances to turn on compiler packing to avoid this unused space, but it generally results in lower performance and—at least in my experience—it’s unusual to do so. More often, C programmers will design structures by hand to make efficient use of space.``````cpp  ```## 带有 BTF 信息的映射

您刚刚看到了与映射相关的 BTF 信息。现在让我们看看在创建映射时内核如何传递此 BTF 数据。

您在第四章中看到，映射是使用`bpf(BPF_MAP_CREATE)`系统调用创建的。这需要一个`bpf_attr`结构作为参数，[在内核中定义](https://oreil.ly/PLrYG)如下（一些细节被省略）：

```cpp
struct{/* anonymous struct used by BPF_MAP_CREATE command */`__u32``map_type`;/* one of enum bpf_map_type */`__u32``key_size`;/* size of key in bytes */`__u32``value_size`;/* size of value in bytes */`__u32``max_entries`;/* max number of entries in a map */...char`map_name`[`BPF_OBJ_NAME_LEN`];...`__u32``btf_fd`;/* fd pointing to a BTF type data */`__u32``btf_key_type_id`;/* BTF type_id of the key */`__u32``btf_value_type_id`;/* BTF type_id of the value */...};
```

在引入 BTF 之前，`btf_*`字段不存在于`bpf_attr`结构中，内核对键或值的结构一无所知。`key_size`和`value_size`字段定义了它们所需的内存量，但它们只是被视为一些字节。通过另外传递定义键和值类型的 BTF 信息，内核可以内省它们，而像`bpftool`这样的实用程序可以检索类型信息以进行漂亮的打印，如前面讨论的那样。但是，有趣的是要注意为键和值分别传递了单独的 BTF `type _id`。您刚刚看到的`____btf_map_config`结构并未被内核用于映射定义；它只是由用户空间的 BCC 使用。

## 函数和函数原型的 BTF 数据

到目前为止，此示例输出中的 BTF 数据与数据类型有关，但 BTF 数据还包含有关函数和函数原型的信息。以下是描述`hello`函数的相同 BTF 数据块中的信息：

```cpp
[31] FUNC_PROTO '(anon)' ret_type_id=23 vlen=1
        'ctx' type_id=10
[32] FUNC 'hello' type_id=31 linkage=static
```

在类型 32 中，您可以看到名为`hello`的函数被定义为具有前一行中定义的类型。这是一个*函数原型*，它返回类型 ID `23`的值，并带有一个名为`ctx`的单个参数（`vlen=1`），其类型 ID 为`10`。为了完整起见，这里是输出中较早时那些类型的定义：

```cpp
[10] PTR '(anon)' type_id=0

[23] INT 'int' size=4 bits_offset=0 nr_bits=32 encoding=SIGNED
```

类型 10 是一个匿名指针，其默认类型为`0`，在 BTF 输出中没有明确包含，但被定义为 void 指针。⁴

类型 23 的返回值是一个 4 字节整数，`encoding=SIGNED`表示它是一个有符号整数；也就是说，它可以具有正值或负值。这对应于*hello-buffer-config.py*源代码中的函数定义，如下所示：

```cpp
inthello(void*ctx)
```

到目前为止，我展示的示例 BTF 信息来自于列出 BTF 数据块的内容。让我们看看如何仅获取与特定映射或程序相关的 BTF 信息。## 检查映射和程序的 BTF 数据

如果您想检查与特定映射相关的 BTF 类型，`bpftool`可以轻松实现。例如，这是`config`映射的输出：

```cpp
bpftool btf dump map name config
[1] TYPEDEF 'u32' type_id=2
[4] STRUCT 'user_msg_t' size=12 vlen=1
        'message' type_id=6 bits_offset=0
```

同样，您可以使用`bpftool btf dump prog <prog identity>`检查与特定程序相关的 BTF 信息。我会让您查看[manpage](https://oreil.ly/lCoV5)以获取更多详细信息。

###### 注意

如果您想更好地了解 BTF 类型数据是如何生成和去重的，还有另一篇[Andrii Nakryiko 的博客文章](https://oreil.ly/0-a9g)可以参考。

到目前为止，您应该已经了解了 BTF 如何描述数据结构和函数的格式。用 C 编写的 eBPF 程序需要定义类型和结构的头文件。让我们看看为 eBPF 程序可能需要的任何内核数据类型生成头文件有多容易。```cpp`  ```# 生成内核头文件

如果在启用了 BTF 的内核上运行`bpftool btf list`，您将看到许多预先存在的 BTF 数据块，看起来像这样：

```cpp
$ bpftool btf list
1: name [vmlinux]  size 5842973B
2: name [aes_ce_cipher]  size 407B
3: name [cryptd]  size 3372B
...
```

此列表中的第一项，ID 为 1，名称为`vmlinux`，是有关运行在此（虚拟）机器上的内核使用的所有数据类型、结构和函数定义的 BTF 信息。⁵

eBPF 程序需要它将要引用的任何内核数据结构和类型的定义。在 CO-RE 出现之前，您通常需要弄清楚 Linux 内核源代码中的许多个别头文件中哪一个包含了您感兴趣的结构的定义，但现在有了一个更简单的方法，因为启用了 BTF 的工具可以从内核中包含的 BTF 信息生成一个适当的头文件。

这个头文件通常被称为*vmlinux.h*，您可以使用`bpftool`生成它，就像这样：

```cpp
bpftool btf dump file /sys/kernel/btf/vmlinux format c > vmlinux.h
```

这个文件定义了所有内核的数据类型，因此在您的 eBPF 程序源代码中包含这个生成的*vmlinux.h*文件会提供您可能需要的任何 Linux 数据结构的定义。当您将源代码编译成 eBPF 对象文件时，该对象将包含与此头文件中使用的定义相匹配的 BTF 信息。稍后，在目标机器上运行程序时，将加载它到内核中的用户空间程序将对构建时的 BTF 信息和在目标机器上运行的内核的 BTF 信息之间的差异进行调整。

自 Linux 内核 5.4 版本以来，以*/sys/kernel/btf/vmlinux*文件的形式包含的 BTF 信息，⁶，但*libbpf*可以利用的原始 BTF 数据也可以为较旧的内核生成。换句话说，如果您想在目标机器上运行一个启用了 CO-RE 的 eBPF 程序，而该目标机器尚未具有 BTF 信息，您可能可以自己提供该目标的 BTF 数据。有关如何生成 BTF 文件以及各种 Linux 发行版的文件存档的信息，请访问[BTFHub](https://oreil.ly/mPSO0)。

###### 注意

BTFHub 存储库还包括有关[BTF 内部](https://oreil.ly/CfyQh)的进一步阅读，如果您想深入了解这个主题。

接下来，让我们看看如何使用这种和其他策略来编写可在使用 CO-RE 的内核之间移植的 eBPF 程序。

# CO-RE eBPF 程序

您会记得 eBPF 程序在内核中运行。在本章的后面，我将展示一些用户空间代码，这些代码将与内核中运行的代码进行交互，但在本节中，我集中讨论内核方面。

正如您已经看到的，eBPF 程序被编译成 eBPF 字节码，而（至少在撰写本文时）支持此功能的编译器是 Clang 或 gcc 用于编译 C 代码，以及 Rust 编译器。我将在第十章中讨论一些您在使用 Rust 时的选择，但在本章中，我将假设您是用 C 语言编写的，并使用 Clang，以及*libbpf*库。

在本章的其余部分，让我们考虑一个名为*hello-buffer-config*的示例应用程序。它与上一章中使用 BCC 框架的*hello-buffer-config.py*示例非常相似，但这个版本是用 C 语言编写的，以使用*libbpf*和 CO-RE。

如果您有基于 BCC 的 eBPF 代码，想要迁移到*libbpf*，请查看 Andrii Nakryiko 在他的网站上的优秀而全面的[指南](https://oreil.ly/iWDcv)。BCC 提供了一些方便的快捷方式，使用*libbpf*的方式并不完全相同；相反，*libbpf*提供了一套宏和库函数，以使 eBPF 程序员的生活更加轻松。当我演示示例时，我将指出 BCC 和*libbpf*方法之间的一些差异。

###### 注意

您将在[*github.com/lizrice/learning-ebpf*](https://github.com/lizrice/learning-ebpf)存储库的*chapter5*目录中找到本节的示例 C eBPF 程序。

首先让我们看一下*hello-buffer-config.bpf.c*，它实现了在内核中运行的 eBPF 程序。本章后面我将向您展示*hello-buffer-config.c*中的用户空间代码，该代码加载程序并显示输出，就像 Python 代码在第四章中对此示例的 BCC 实现所做的那样。

与任何 C 程序一样，eBPF 程序将需要包含一些头文件。

## 头文件

*hello-buffer-config.bpf.c*的前几行指定了它需要的头文件：

```cpp
#include "vmlinux.h"
#include <bpf/bpf_helpers.h>
#include <bpf/bpf_tracing.h>
#include <bpf/bpf_core_read.h>
#include "hello-buffer-config.h"
```

这五个文件是*vmlinux.h*文件，*libbpf*的一些头文件和我自己编写的特定于应用程序的头文件。让我们看看为*libbpf*程序所需的头文件为什么是典型的模式。

### 内核头信息

如果您正在编写引用任何内核数据结构或类型的 eBPF 程序，最简单的选择是包含本章前面描述的*vmlinux.h*文件。或者，也可以从 Linux 源中包含单个头文件，或者在自己的代码中手动定义类型，如果您真的想要这样做的话。如果要使用*libbpf*中的任何 BPF 辅助函数，您需要包含*vmlinux.h*或*linux/types.h*，以获取 BPF 辅助源引用的`u32`、`u64`等类型的定义。

*vmlinux.h*文件源自内核源头文件，但不包括其中的`#define`值。例如，如果您的 eBPF 程序解析以太网数据包，则可能需要告诉您数据包包含的协议的常量定义（例如`0x0800`表示它是 IP 数据包，或`0x0806`表示 ARP 数据包）。如果您不包括定义这些值的[*if_ether.h*文件](https://oreil.ly/hoZzP)，则需要在自己的代码中复制一系列常量值。对于*hello-buffer-config*，我不需要这些值定义，但您将在第八章中看到另一个示例，其中这是相关的。

### 来自 libbpf 的头文件

要在 eBPF 代码中使用任何 BPF 辅助函数，您需要包含*libbpf*中给出其定义的头文件。

###### 注意

关于*libbpf*可能会让人稍微困惑的一件事是，它不仅仅是一个用户空间库。您将发现自己在用户空间和 eBPF C 代码中都包含*libbpf*的头文件。

在撰写本文时，通常会看到 eBPF 项目将*libbpf*作为子模块包含并从源代码构建/安装-这是我在本书示例存储库中所做的。如果将其包含为子模块，则只需从*libbpf/src*目录运行`make install`。我认为不久之后，*libbpf*将更常见地作为常见 Linux 发行版上的软件包提供，特别是自从*libbpf*现在已经通过了[1.0 版本发布](https://oreil.ly/8BFq6)的里程碑。

### 特定于应用程序的头文件

通常会有一个特定于应用程序的头文件，定义了用户空间和 eBPF 应用程序的共同使用的任何结构。在我的示例中，*hello-buffer-config.h*头文件定义了`data_t`结构，我用它来从 eBPF 程序传递事件数据到用户空间。它几乎与您在此代码的 BCC 版本中看到的结构相同，看起来是这样的：

```cpp
structdata_t{ `int``pid``;` ``int``uid``;` ``char``command``[``16``];` ``char``message``[``12``];` ``char``path``[``16``];` ``};``````cpp
```

```cppThe only difference from the version you saw before is that I have added a field called `path`.

The reason to pull this structure definition into a separate header file is that I will also refer to it from the user space code in *hello-buffer-config.c*. In the BCC version, the kernel and user space code were both defined in a single file, and BCC did some work behind the scenes to make the structure available to the Python user space code.```  ```cpp## Defining Maps

After including the header files, the next few lines of the source code in *hello-buffer-config.bpf.c* define the structures used for maps, like this:

```

struct{ `__uint``(``type``,``BPF_MAP_TYPE_PERF_EVENT_ARRAY``);` ``__uint``(``key_size``,``sizeof``(``u32``));` ``__uint``(``value_size``,``sizeof``(``u32``));` ``}``output``SEC``(``".maps"``);` ``struct``user_msg_t``{` ``char``message``[``12``];` ``};` ``struct``{` ``__uint``(``type``,``BPF_MAP_TYPE_HASH``);` ``__uint``(``max_entries``,``10240``);` ``__type``(``key``,``u32``);` ``__type``(``value``,``struct``user_msg_t``);` ``}``my_config``SEC``(``".maps"``);```cpp```````cpp```
```

 ```cppThis requires more lines of code than I needed in the equivalent BCC example! With BCC, the map called `config` was created with the following macro:

```
BPF_HASH(config,u64,structuser_msg_t);
```cpp

 `This macro isn’t available when you’re not using BCC, so in C you have to write it out longhand. You’ll see that I have used `__uint` and `__type`. These are defined in [*bpf/bpf_helpers_def.h*](https://oreil.ly/2FgjB) along with `__array`, like this:

```
#define `__uint`(name, val) int (*name)[val]#define `__type`(name, val) `typeof`(val) *name#define `__array`(name, val) `typeof`(val) *name[]
```cpp

These macros generally seem to be used by convention in *libbpf*-based programs, and I think they make the map definitions a little easier to read.

###### Note

The name “config” clashed with a definition in *vmlinux.h*, so I renamed the map “my_config” for this example.````  ```cpp## eBPF Program Sections

Use of *libbpf* requires each eBPF program to be marked with a `SEC()` macro that defines the program type, like this:

```

SEC（“kprobe”）

```cpp

 `This results in a section called `kprobe` in the compiled ELF object, so *libbpf* knows to load this as a `BPF_PROG_TYPE_KPROBE`. We’ll discuss different program types further in [Chapter 7](ch07.html#ebpf_program_and_attachment_types).

Depending on the program type, you can also use the section name to specify what event the program will be attached to. The *libbpf* library will use this information to set up the attachment automatically, rather than leaving you to do it explicitly in your user space code. So, for example, to auto-attach to the kprobe for the `execve` syscall on an ARM-based machine, you could specify the section like this:

```

SEC("ksyscall/execve")

```cpp

 `This requires you to know the function name for the syscall on that architecture (or figure it out, perhaps by looking at the */proc/kallsyms* file on your target machine, which lists all the kernel symbols, including its function names). But *libbpf* can make life even easier for you with the `k(ret)syscall` section name, which tells the loader to attach to the kprobe in the architecture-specific function automatically:

```

这个`__builtin_preserve_access_index()`指令是对“常规”C 代码的扩展，将其添加到 eBPF 还需要对 Clang 编译器进行更改以支持它并发出这些 CO-RE 重定位条目。这些扩展是一些 C 编译器今天（至少）无法生成 eBPF 字节码的原因的例子。在[LLVM 邮件列表](https://oreil.ly/jHTHE)上阅读有关 Clang 对 eBPF CO-RE 支持所需的更改的更多信息。

```cpp

 `###### Note

The valid section names and formats are listed in the [*libbpf* documentation](https://oreil.ly/FhHrm). In the past, the requirements for section names were much looser, so you may come across eBPF programs written before *libbpf 1.0* with section names that don’t match the valid set. Don’t let them confuse you!

The section definition declares where the eBPF program should be attached, and then the program itself follows. As before, the eBPF program itself is written as a C function. In the example code it’s called `hello()`, and it’s extremely similar to the `hello()` function you saw in [Chapter 4](ch04.html#the_bpfleft_parenthesisright_parenthesi). Let’s consider the differences between that previous version and the version here:

```

```  ```cpp## 对象文件中的 BTF 信息

```cpp

①

I’ve taken advantage of a [`BPF_KPROBE_SYSCALL`](https://oreil.ly/pgI1B) macro defined in *libbpf* that makes it easy to access the arguments to a syscall by name. For `execve()`, the first argument is the pathname for the program that’s going to be executed. The eBPF program name is `hello`.

②

Since the macro has made it so easy to access that pathname argument to `execve()`, I’m including it in the data sent to the perf buffer output. Notice that copying memory requires the use of a BPF helper function.

③

Here, `bpf_map_lookup_elem()` is the BPF helper function for looking up values in a map, given a key. BCC’s equivalent of this would be `p = my_config.lookup(&data.uid)`. BCC rewrites this to use the underlying `bpf_map_lookup_elem()` function before it passes the C code to the compiler. When you’re using *libbpf*, there is no rewriting of the code before compilation,⁷ so you have to write directly to the helper functions.

④

Here’s another similar example where I have written directly to the helper function `bpf_perf_event_output()`, where BCC gave me the convenient equivalent `output.perf_submit(ctx, &data, sizeof(data))`.

The only other difference is that in the BCC version, I defined the message string as a local variable within the `hello()` function. BCC doesn’t (at least at the time of this writing) support global variables. In this version I have defined it as a global variable, like this:

```

正如您稍后在本章中将看到的，CO-RE 重定位条目告诉*libbpf*在将 eBPF 程序加载到内核时重新编写地址，以考虑任何 BTF 差异。如果`src`在其包含结构中的偏移在目标内核上不同，重新编写的指令将考虑到这一点。

```cpp

 `In *chapter4/hello-buffer-config.py* the `hello` function was defined rather differently, like this:

```

在你看到的`bpf_core_read()`中，直接调用`bpf_probe_read_kernel()`，唯一的区别是它用`__builtin_preserve_access_index()`包装了`src`字段。这告诉 Clang 在访问内存中的这个地址时发出 CO-RE 重定位条目以及 eBPF 指令。

```cpp

 `The `BPF_KPROBE_SYSCALL` macro is one of the convenient additions from *libbpf* that I mentioned. You’re not required to use the macro, but it makes life easier. It does all the heavy lifting to provide named arguments for all the parameters passed to a syscall. In this case, it supplies a `pathname` argument that points to a string holding the path of the executable that is about to be run, which is the first argument to the `execve()` syscall.

If you’re paying very close attention you might notice that the `ctx` variable isn’t visibly defined in my source code for *hello-buffer-config.bpf.c*, but nevertheless, I’ve been able to use it when submitting data to the output perf buffer, like this:

```

`然后您可以使用`bpf_probe_read_kernel()`辅助函数从点`d`读取。

Andrii 的[指南](https://oreil.ly/tU0Gb)中有一个很好的描述。

```cpp
structb_t*b; `struct``c_t``*``c``;` ``bpf_core_read``(``&``b``,``8``,``&``a``->``b``);` ``bpf_core_read``(``&``c``,``8``,``&``b``->``c``);` ``bpf_core_read``(``&``d``,``8``,``&``c``->``d``);````

charmessage[12]="Hello World";

```cpp
#define bpf_core_read(dst, sz, src)                        \
 bpf_probe_read_kernel(dst, sz,                         \
 (const void *)__builtin_preserve_access_index(src))
```

```cpp

 `The `ctx` variable does exist, hidden within the `BPF_KPROBE_SYSCALL` macro definition inside [*bpf/bpf_tracing.h*](https://oreil.ly/pgI1B), in *libbpf*, where you’ll also find some commentary about this. It can be slightly confusing to use a variable that’s not visibly defined, but it’s very helpful that it can be accessed.``````cpp  ```## CO-RE 内存访问

###### *libbpf*库提供了围绕`bpf_probe_read_*()`辅助函数的 CO-RE 包装，以利用 BTF 信息并使内存访问调用在不同的内核版本中可移植。以下是其中一个这些包装的示例，定义在[*bpf_core_read.h*头文件](https://oreil.ly/XWWyc)中。

SEC("ksyscall/execve")intBPF_KPROBE_SYSCALL(hello,constchar*pathname)①{structdata_tdata={};structuser_msg_t*p;data.pid=bpf_get_current_pid_tgid()>>32;data.uid=bpf_get_current_uid_gid()&0xFFFFFFFF;bpf_get_current_comm(&data.command,sizeof(data.command));bpf_probe_read_user_str(&data.path,sizeof(data.path),pathname);②p=bpf_map_lookup_elem(&my_config,&data.uid);③if(p!=0){bpf_probe_read_kernel(&data.message,sizeof(data.message),p->message);}else{bpf_probe_read_kernel(&data.message,sizeof(data.message),message);}bpf_perf_event_output(ctx,&output,BPF_F_CURRENT_CPU,④&data,sizeof(data));return0;}

*libbpf*库提供了`BPF_CORE_READ()`宏，以便您可以在一行中写入多个`bpf_core_read()`调用，而不需要为每个指针解引用调用一个单独的辅助函数。例如，如果您想要做类似`d = a->b->c->d`的事情，您可以编写以下代码：

`您现在已经看到了*hello-buffer-config.bpf.c*示例中的所有代码。现在让我们将其编译成一个对象文件。

```cpp```````cpp```  ```# Compiling eBPF Programs for CO-RE

In [Chapter 3](ch03.html#anatomy_of_an_ebpf_program) you saw an extract from a Makefile that compiles C to eBPF bytecode. Let’s dig into the options used and see why they are necessary for CO-RE/*libbpf* programs.

## Debug Information

You have to pass the `-g` flag to Clang so that it includes debug information, which is necessary for BTF. However, the `-g` flag also adds DWARF debugging information to the output object file, but that’s not needed by eBPF programs, so you can reduce the size of the object by running the following command to strip it out:

```cpp
llvm-strip -g <object file>
```

## Optimization

The `-O2` optimization flag (level 2 or higher) is required for Clang to produce BPF bytecode that will pass the verifier. One example of this being necessary is that, by default, Clang will output `callx <register>` to call helper functions, but eBPF doesn’t support calling addresses from registers.

## Target Architecture

If you’re using certain macros defined by *libbpf*, you’ll need to specify the target architecture at compile time. The *libbpf* header file *bpf/bpf_tracing.h* defines several macros that are platform specific, such as `BPF_KPROBE` and `BPF_KPROBE_SYSCALL` that I’ve used in this example. The `BPF_KPROBE` macro can be used for eBPF programs that are being attached to kprobes, and `BPF_KPROBE_SYSCALL` is a variant specifically for syscall kprobes.

The argument to a kprobe is a `pt_regs` structure that holds a copy of the contents of the CPU registers. Since registers are architecture specific, the `pt_regs` structure definition depends on the architecture you’re running on. This means that if you want to use these macros, you’ll need to also tell the compiler what the target architecture is. You can do this by setting `-D __TARGET_ARCH_($ARCH)` where `$ARCH` is an architecture name like arm64, amd64, and so on.

Also note that if you didn’t use the macro, you’d need architecture-specific code to access the register information anyway for a kprobe.

Perhaps “compile once *per architecture*, run everywhere” would have been a bit of a mouthful!

## Makefile

The following is an example Makefile instruction for compiling CO-RE objects (taken from the Makefile in the *chapter5* directory of the GitHub repo for this book):

```cpp
hello-buffer-config.bpf.o: %.o: %.c
   clang \ -target bpf \ `-D __TARGET_ARCH_`$(``ARCH``)` \ `-I/usr/include/`$(``shell` `uname` -`m``)`-linux-gnu \ `-Wall \ `-O2 -g \ `-c `$<` -o `$@` `llvm-strip -g `$@``````cpp`

```cpp

 ```但使用起来更紧凑：

```cpp
d=BPF_CORE_READ(a,b,c,d);
```

用于跟踪的 eBPF 程序通过`bpf_probe_read_*()`家族的 BPF 辅助函数对内存的访问受到限制。⁸（还有一个`bpf_probe_write_user()`辅助函数，但它只是[“用于实验”](https://oreil.ly/ibcy1)）。问题在于，正如您将在下一章中看到的，eBPF 验证器通常不会让您像在 C 中那样简单地通过指针读取内存（例如，`x = p->y`）。⁹

```cpp`  ```## 许可证定义

SEC("kprobe/__arm64_sys_execve")

```cpp
charLICENSE[]SEC("license")="Dual BSD/GPL";
```

正如您从第三章中已经知道的，eBPF 程序必须声明其许可证。示例代码是这样做的：

```

 ```cpp 如果您使用示例代码，应该能够通过在*chapter5*目录中运行`make`来构建 eBPF 对象文件*hello-buffer-config.bpf.o*（以及我将很快描述的伴随的用户空间可执行文件）。让我们检查该对象文件，看看它是否包含 BTF 信息。

[BTF 的内核文档](https://oreil.ly/5QrBy)描述了 BTF 数据如何在 ELF 对象文件中以两个部分进行编码：*.BTF*，其中包含数据和字符串信息，以及*.BTF.ext*，其中包含函数和行信息。您可以使用`readelf`来查看这些部分是否已添加到对象文件中，就像这样：

```
$ readelf -S hello-buffer-config.bpf.o | grep BTF
  [10] .BTF              PROGBITS         0000000000000000  000002c0
  [11] .rel.BTF          REL              0000000000000000  00000e50
  [12] .BTF.ext          PROGBITS         0000000000000000  00000b18
  [13] .rel.BTF.ext      REL              0000000000000000  00000ea0
```cpp

`bpftool`实用程序允许我们检查对象文件中的 BTF 数据，就像这样：

```
bpftool btf dump file hello-buffer-config.bpf.o
```cpp

输出看起来就像您从加载的程序和映射中转储 BTF 信息时获得的输出，就像您在本章前面看到的那样。

让我们看看如何使用这些 BTF 信息来允许程序在具有不同内核版本和不同数据结构的另一台机器上运行。```  ```cpp# BPF 重定位

*libbpf*库将 eBPF 程序适配到目标内核上的数据结构布局，即使此布局与编译代码的内核不同。为此，*libbpf*需要 Clang 在编译过程中生成的 BPF CO-RE 重定位信息。

您可以从[*linux/bpf.h*](https://elixir.bootlin.com/linux/v5.19.17/source/include/uapi/linux/bpf.h#L6711)头文件中`struct bpf_core_relo`的定义中了解有关重定位工作原理的更多信息：

```
struct`bpf_core_relo`{`__u32``insn_off`;`__u32``type_id`;`__u32``access_str_off`;enum`bpf_core_relo_kind``kind`;};
```cpp

eBPF 程序的 CO-RE 重定位数据由每个需要重定位的指令的这些结构之一组成。假设该指令正在将寄存器设置为结构中字段的值。该指令的`bpf_core_relo`结构（由`insn_off`字段标识）对该结构的 BTF 类型（`type_id`字段）进行编码，并且还指示相对于该结构的字段如何被访问（`access_str_off`）。

正如您刚刚看到的，Clang 会自动生成内核数据结构的重定位数据，并将其编码到 ELF 对象文件中。就是下面这行，您会在*vmlinux.h*文件的开头附近找到，它导致 Clang 执行此操作：

```
#pragma clang attribute push (__attribute__((preserve_access_index)), \
 apply_to = record)
```cpp

`preserve_access_index`属性告诉 Clang 为类型定义生成 BPF CO-RE 重定位。`clang attribute push`部分表示该属性应应用于所有定义，直到出现`clang attribute pop`，该语句出现在文件末尾。这意味着 Clang 为*vmlinux.h*中定义的所有类型生成重定位信息。

当您使用`bpftool`加载 BPF 程序并使用`-d`标志打开调试信息时，您可以看到重定位正在进行：

```
bpftool -d prog load hello.bpf.o /sys/fs/bpf/hello
```cpp

这会生成大量输出，但与重定位相关的部分看起来像这样：

```
libbpf: CO-RE relocating [24] struct user_pt_regs: found target candidate [205]
struct user_pt_regs in [vmlinux]
libbpf: prog 'hello': relo #0: <byte_off> [24] struct user_pt_regs.regs[0]
(0:0:0 @ offset 0)
libbpf: prog 'hello': relo #0: matching candidate #0 <byte_off> [205] struct
user_pt_regs.regs[0] (0:0:0 @ offset 0)
libbpf: prog 'hello': relo #0: patched insn #1 (LDX/ST/STX) off 0 -> 0
```cpp

在这个例子中，您可以看到`hello`程序的 BTF 信息中的类型 ID 24 指的是名为`user_pt_regs`的结构。*libbpf*库已将此与内核结构匹配，该结构也称为`user_pt_regs`，在*vmlinux* BTF 数据集中的类型 ID 为 205。实际上，因为我在同一台机器上编译和加载了程序，所以类型定义是相同的，因此在这个例子中，从结构开始的偏移量仍然保持不变，并且对指令#1 的“修补”也保持不变。

在许多应用程序中，您不希望要求用户运行`bpftool`来加载 eBPF 程序。相反，您希望将此功能构建到一个专用的用户空间程序中，该程序作为可执行文件提供。让我们考虑如何编写这个用户空间代码。

# CO-RE 用户空间代码

不同编程语言中有一些不同的框架支持 CO-RE，它们通过在将 eBPF 程序加载到内核时实现重定位来支持 CO-RE。在本章中，我将展示使用*libbpf*的 C 代码；其他选项包括 Go 包*cilium/ebpf*和*libbpfgo*，以及 Rust 的 Aya。我将在第十章中进一步讨论这些选项。

# 用户空间的 Libbpf 库

*libbpf*库是一个用户空间库，如果你的应用程序的用户空间部分是用 C 编写的，你可以直接使用它。如果愿意，你可以在不使用 CO-RE 的情况下使用这个库。在[Andrii Nakryiko 的*libbpf-bootstrap*博客文章](https://oreil.ly/b3v7B)中有一个例子。

这个库提供了包装`bpf()`和相关系统调用的函数，你在第四章中遇到过，用于执行加载程序到内核并将其附加到事件，或者从用户空间访问映射信息。使用这些抽象的传统和最简单的方法是通过自动生成的 BPF 骨架代码。

## BPF 骨架

你可以使用`bpftool`从现有的以 ELF 文件格式存在的 eBPF 对象自动生成这个骨架代码，就像这样：

```
bpftool gen skeleton hello-buffer-config.bpf.o > hello-buffer-config.skel.h
```cpp

查看这个骨架头文件，你会发现它包含了 eBPF 程序和映射的结构定义，以及一些以`hello_buffer_config_bpf__`开头的函数（根据对象文件的名称）。这些函数管理 eBPF 程序和映射的生命周期。你不一定要使用骨架代码，如果愿意，你可以直接调用*libbpf*，但是自动生成的代码通常会节省一些输入。

在生成的骨架文件的末尾，你会看到一个名为`hello_buffer_config_bpf__elf_bytes`的函数，它返回*hello-buffer-config.bpf.o*的 ELF 对象文件的字节内容。一旦骨架被生成，我们实际上不再需要那个对象文件。你可以通过运行`make`来生成`hello-buffer-config`可执行文件，然后删除*.o*文件来测试；可执行文件中包含了 eBPF 字节码。

###### 注意

如果愿意，你可以使用*libbpf*函数`bpf_object__open_file`从 ELF 文件中加载 eBPF 程序和映射，而不是使用骨架文件中的字节。

这是管理本示例中 eBPF 程序和映射生命周期的用户空间代码的概要，使用了生成的骨架代码。为了清晰起见，我省略了一些细节和错误处理，但你可以在*chapter5/hello-buffer-config.c*中找到完整的源代码。

```
...[other#includes]#include"hello-buffer-config.h"①#include"hello-buffer-config.skel.h"...[somecallbackfunctions]intmain(){structhello_buffer_config_bpf*skel;structperf_buffer*pb=NULL;interr;libbpf_set_print(libbpf_print_fn);②skel=hello_buffer_config_bpf__open_and_load();③...err=hello_buffer_config_bpf__attach(skel);④...pb=perf_buffer__new(bpf_map__fd(skel->maps.output),8,handle_event,lost_event,NULL,NULL);⑤...while(true){![6](img/6.png)err=perf_buffer__poll(pb,100);...}perf_buffer__free(pb);![7](img/7.png)hello_buffer_config_bpf__destroy(skel);return-err;}
```cpp

①

这个文件包括了自动生成的骨架头文件，以及我手动编写的用于用户空间和内核代码之间共享的数据结构的头文件。

②

这段代码设置了一个回调函数，用于打印*libbpf*生成的任何日志消息。

③

这里创建了一个`skel`结构，代表了 ELF 字节中定义的所有映射和程序，并将它们加载到内核中。

④

程序会自动附加到适当的事件上。

⑤

这个函数创建了一个用于处理 perf 缓冲区输出的结构。

![6](img/6.png)

这里 perf 缓冲区被持续轮询。

![7](img/7.png)

这是清理代码。

让我们更详细地了解其中的一些步骤。

### 将程序和映射加载到内核中

第一个调用自动生成的函数是这个：

```
skel=hello_buffer_config_bpf__open_and_load();
```cpp

`正如其名称所示，这个函数涵盖了两个阶段：打开和加载。 “打开”阶段涉及读取 ELF 数据并将其部分转换为代表 eBPF 程序和映射的结构。 “加载”阶段将这些映射和程序加载到内核中，并在必要时执行任何 CO-RE 修复。

这两个阶段可以很容易地分开处理，因为骨架代码提供了单独的`name__open()`和`name__load()`函数。这样你就有选择在加载之前操作 eBPF 信息的选项。这通常用于在加载之前配置程序。例如，我可以将计数器全局变量`c`初始化为某个值，就像这样：

```
skel=hello_buffer_config_bpf__open(); `if``(``!``skel``)``{` ``// Error ...`
`}`
`skel``->``data``->``c``=``10``;` ``err``=``hello_buffer_config_bpf__load``(``skel``);```cpp

```

 ```由`hello_buffer_config_bpf__open()`和`hello_buffer_config_bpf__load()`返回的数据类型是一个名为`hello_buffer_config_bpf`的结构，在骨架头文件中定义，包括有关对象文件中定义的所有映射、程序和数据的信息。

###### 注意

在这个示例中，骨架对象（例如`hello_buffer_config_bpf`）只是来自 ELF 字节的用户空间表示。一旦它被加载到内核中，如果你在对象中更改一个值，它不会对内核端的数据产生任何影响。所以，例如，在加载后更改`skel->data->c`将不会产生任何影响。````  ```cpp### Accessing existing maps

By default, *libbpf* will also create any maps that are defined in the ELF bytes, but sometimes you might want to write an eBPF program that reuses an existing map. You already saw an example of this in the previous chapter, where you saw `bpftool` iterating through all the maps, looking for the one that matched a specified name. Another common reason to use a map is to share information between two different eBPF programs, so only one program should create the map. The `bpf_map__set_autocreate()` function allows you to override *libbpf*’s auto-creation.

So how do you access an existing map? Maps can be pinned, and if you know the pinned path, you can get a file descriptor to an existing map with `bpf_obj_get()`. Here’s a very simple example (available in the GitHub repository as *chapter5/find-map.c*):

```
structbpf_map_infoinfo={}; `unsigned``int``len``=``sizeof``(``info``);` ``int``findme``=``bpf_obj_get``(``"/sys/fs/bpf/findme"``);` ``if``(``findme``<=``0``)``{` ``printf``(``"No FD``\n``"``);` ``}``else``{` ``bpf_obj_get_info_by_fd``(``findme``,``&``info``,``&``len``);` ``printf``(``"Name: %s``\n``"``,``info``.``name``);` ``}```cpp````

```cpp

 ```你可以使用`bpftool`创建一个映射，就像这样：

```cpp
$ bpftool map create /sys/fs/bpf/findme type array key 4 value 32 entries 4
name findme
```

运行`find-map`可执行文件将打印出：

```cpp
Name: findme
```

让我们回到`hello-buffer-config`示例和骨架代码。```cpp  ```### 附着到事件

示例中的下一个骨架函数将程序附着到`execve`系统调用函数：

```cpp
err=hello_buffer_config_bpf__attach(skel);
```

`libbpf`库会自动从`SEC()`定义中获取程序的附着点。如果你没有完全定义附着点，那么有一系列`libbpf`函数，比如`bpf_program__attach_kprobe`，`bpf_program__attach_xdp`等，用于附着不同类型的程序。### 管理事件缓冲区

设置性能缓冲区使用的是`libbpf`中定义的函数，而不是在骨架中定义的函数。

```cpp
pb=perf_buffer__new(bpf_map__fd(skel->maps.output),8,handle_event, `lost_event``,``NULL``,``NULL``);`
```

你可以看到`perf_buffer__new()`函数将“输出”映射的文件描述符作为第一个参数。`handle_event`参数是一个回调函数，当新数据到达性能缓冲区时会被调用，`lost_event`在性能缓冲区没有足够的空间让内核写入数据条目时会被调用。在我的示例中，这些函数只是将消息写入屏幕。

最后，程序必须重复轮询性能缓冲区：

```cpp
while(true){ `err``=``perf_buffer__poll``(``pb``,``100``);` ``...` ``}```

```cpp

 ```100 是毫秒级的超时时间。之前设置的回调函数在数据到达或缓冲区满时会被调用。

最后，为了清理，我释放了性能缓冲区，并在内核中销毁了 eBPF 程序和映射，就像这样：

```cpp
perf_buffer__free(pb); `hello_buffer_config_bpf__destroy``(``skel``);`
```

``libbpf`中有一整套与`perf_buffer_*`和`ring_buffer_*`相关的函数，帮助你管理事件缓冲区。

如果你制作并运行这个`hello-buffer-config`示例程序，你会看到以下输出（与第四章中看到的非常相似）：

```cpp
23664  501    bash             Hello World
23665  501    bash             Hello World
23667  0      cron             Hello World
23668  0      sh               Hello World
``````cpp```````cpp````  ```cpp## Libbpf Code Examples

There are lots of great examples of *libbpf*-based eBPF programs available that you can use as inspiration and guidance for writing your own:

*   The [*libbpf-bootstrap*](https://oreil.ly/zB0Co) project is intended to help you get off the ground with a set of example programs.

*   The BCC project has many of the original BCC-based tools migrated to a *libbpf* version. You’ll find them in the [*libbpf-tools* directory](https://oreil.ly/Z9xDX).```  ```cpp# Summary

CO-RE enables eBPF programs that can run on kernel versions different from the versions on which they were built. This massively improves the portability of eBPF and makes life much easier for tool developers who want to deliver production-ready tooling to their users and customers.

In this chapter you saw how CO-RE achieves this by encoding type information into the compiled object file and using relocations to rewrite instructions as they are loaded into the kernel. You also had an introduction to writing code in C that uses *libbpf*: both the eBPF programs that run in the kernel and the user space programs that manage the lifecycle of those programs, based on auto-generated BPF skeleton code. In the next chapter you’ll learn how the kernel verifies that eBPF programs are safe to run.

# Exercises

Here are a few things you can do to further explore BTF, CO-RE, and *libbpf*:

1.  Experiment with `bpftool btf dump map` and `bpftool btf dump prog` to see the BTF information associated with maps and programs, respectively. Remember that you can specify individual maps and programs in more than one way.

2.  Compare the output from `bpftool btf dump file` and `bpftool btf dump prog` for the same program in its ELF object file form and after it has been loaded into the kernel. They should be identical.

3.  Examine the debug output from *bpftool -d prog load hello-buffer-config.bpf.o /sys/fs/bpf/hello*. You’ll see each section being loaded, checks on the license, and relocations taking place, as well as output describing each BPF program instruction.

4.  Try building a BPF program against a different *vmlinux* header file from BTFHub, and look in the debug output from `bpftool` for relocations that change offsets.

5.  Modify the *hello-buffer-config.c* program so that you can configure different messages for different user IDs using the map (similar to the *hello-buffer-config.py* example in [Chapter 4](ch04.html#the_bpfleft_parenthesisright_parenthesi)).

6.  Try changing the section name in the `SEC();`, perhaps to your own name. When you come to load the program into the kernel you should see an error because *libbpf* doesn’t recognize the section name. This illustrates how *libbpf* uses the section name to work out what kind of BPF program this is. You could try writing your own attachment code to explicitly attach to an event of your choice rather than relying on *libbpf*’s auto-attachment.

¹ Strictly speaking, the data structure definitions come from kernel header files, and you could choose to compile based on a set of these header files that is different from what was used to build the kernel running on that machine. To work correctly (without the CO-RE mechanisms described in this chapter), the kernel headers have to be compatible with the kernel on the target machine where the eBPF program will run.

² Part of this section is adapted from “What Is eBPF?” by Liz Rice. Copyright © 2022 O’Reilly Media. Used with permission.

³ A small and unscientific survey suggests that most people pronounce this the same as the word *core* rather than in two syllables.

⁴ See the kernel documentation at [*https://docs.kernel.org/bpf/btf.html#type-encoding*](https://docs.kernel.org/bpf/btf.html#type-encoding).

⁵ The kernel needs to have been built with the `CONFIG_DEBUG_INFO_BTF` option enabled.

⁶ Which is the oldest Linux kernel version that can support BTF? See [*https://oreil.ly/HML9m*](https://oreil.ly/HML9m).

⁷ Well, normal C preprocessing applies so that you can do things like `#define`. But there’s no *special* rewriting like there is when you use BCC.

⁸ eBPF programs handling network packets don’t get to use this helper function and can only access the network packet memory.

⁹ It is permitted in certain BTF-enabled program types such as `tp_btf`, `fentry`, and `fexit`.``````cpp```
