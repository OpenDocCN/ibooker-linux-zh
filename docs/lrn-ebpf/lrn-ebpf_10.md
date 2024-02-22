# 第十章：eBPF 编程

到目前为止，在本书中，你已经学到了很多关于 eBPF 的知识，并看到了许多关于它如何用于各种应用的例子。但是，如果你想根据 eBPF 实现自己的想法，该怎么办呢？本章讨论了在编写自己的 eBPF 代码时的选择。

从阅读本书你知道，eBPF 编程包括两个部分：

+   编写在内核中运行的 eBPF 程序

+   编写管理和与 eBPF 程序交互的用户空间代码

本章中大多数我将讨论的库和语言都要求你作为程序员处理这两部分，并意识到在哪里处理了什么。但是`bpftrace`，也许是最简单的 eBPF 编程语言，掩盖了程序员对这种区别的认识。

# Bpftrace

正如在项目的*README*页面上所描述的，“`bpftrace`是一个用于 Linux eBPF 的高级跟踪语言……受 awk 和 C 的启发，以及前身跟踪器，如 DTrace 和 SystemTap。”

[`bpftrace`](https://oreil.ly/BZNZO)命令行工具将用高级语言编写的程序转换为 eBPF 内核代码，并为终端内的结果提供一些输出格式。作为用户，你不需要真正考虑内核-用户空间的分离。

你会在项目文档中找到几个有用的一行脚本的例子，包括一个很好的[教程](https://oreil.ly/Ah2QB)，它将带你从编写一个简单的“Hello World”脚本到编写更复杂的脚本，可以跟踪内核数据结构中读取的数据。

###### 注意

从 Brendan Gregg 的[`bpftrace`速查表](https://oreil.ly/VBwLm)中了解`bpftrace`提供的各种功能。或者，要深入了解`bpftrace`和 BCC，可以参考他的书[*BPF 性能工具*](https://oreil.ly/kjc95)。

正如其名称所示，`bpftrace`可以附加到跟踪（也称为性能相关）事件，包括 kprobes、uprobes 和 tracepoints。例如，你可以使用`-l`选项在机器上列出可用的 tracepoints 和 kprobes，就像这样：

```cpp
$ bpftrace -l "*execve*"
tracepoint:syscalls:sys_enter_execve
tracepoint:syscalls:sys_exit_execve
...
kprobe:do_execve_file
kprobe:do_execve
kprobe:__ia32_sys_execve
kprobe:__x64_sys_execve
...
```

这个例子找到了所有包含“execve”的可用附加点。从这个输出中，你可以看到可以附加到一个名为`do_execve`的 kprobe。以下是一个附加到该事件的`bpftrace`一行脚本：

```cpp
bpftrace -e 'kprobe:do_execve { @[comm] = count(); }'
Attaching 1 probe...
^C

@[node]: 6
@[sh]: 6
@[cpuUsage.sh]: 18
```

`{ @[comm] = count(); }`部分是附加到该事件的脚本。这个例子跟踪了不同可执行文件触发该事件的次数。

`bpftrace`的脚本可以协调附加到不同事件的多个 eBPF 程序。例如，考虑报告文件被打开的[*opensnoop.bt*脚本](https://oreil.ly/3HWZ2)。以下是一个摘录：

```cpp
tracepoint:syscalls:sys_enter_open,
tracepoint:syscalls:sys_enter_openat
{
    @filename[tid] = args->filename;
}

tracepoint:syscalls:sys_exit_open,
tracepoint:syscalls:sys_exit_openat
/@filename[tid]/
{
    $ret = args->ret;
    $fd = $ret > 0 ? $ret : -1;
    $errno = $ret > 0 ? 0 : - $ret;

    printf("%-6d %-16s %4d %3d %s\n", pid, comm, $fd, $errno,
        str(@filename[tid]));
    delete(@filename[tid]);
}
```

该脚本定义了两个不同的 eBPF 程序，分别附加到两个不同的内核跟踪点，即`open()`和`openat()`系统调用的进入和退出点。¹这两个系统调用都用于打开文件，并将文件名作为输入参数。由任一类型的系统调用进入触发的程序缓存了该文件名，并将其存储在一个映射中，其中键是当前线程 ID。当退出跟踪点被触发时，脚本中的`/@filename[tid]/`行检索了这个映射中缓存的文件名。

运行这个脚本会生成如下输出：

```cpp
./opensnoop.bt 
Attaching 6 probes...
Tracing open syscalls... Hit Ctrl-C to end.
PID    COMM               FD ERR PATH
297388 node               30   0 /home/liz/.vscode-server/data/User/
                                 workspaceStorage/73ace3ed015
297360 node               23   0 /proc/307224/cmdline
297360 node               23   0 /proc/305897/cmdline
297360 node               23   0 /proc/307224/cmdline
```

我刚告诉你有四个 eBPF 程序附加到跟踪点，那为什么这个输出说有六个探针呢？答案是这个程序的完整版本包括用于初始化和清理脚本的`BEGIN`和`END`子句两个“特殊探针”（与 awk 语言非常相似）。出于简洁起见，我在这里省略了这些子句，但你可以在[GitHub 的源代码](https://oreil.ly/X8wgW)中找到它们。

如果你正在使用`bpftrace`，你不需要了解底层的程序和映射，但是对于那些已经阅读了本书早期章节的人来说，这些概念现在应该是熟悉的。如果你有兴趣看到在运行`bpftrace`程序时加载到内核中的程序和映射，你可以很容易地使用`bpftool`来做到这一点（就像你在第三章中看到的那样）。这是我在运行*opensnoop.bt*时得到的输出。

```cpp
$ bpftool prog list 
...
494: tracepoint  name sys_enter_open  tag 6f08c3c150c4ce6e  gpl
        loaded_at 2022-11-18T12:44:05+0000  uid 0
        xlated 128B  jited 93B  memlock 4096B  map_ids 254
495: tracepoint  name sys_enter_opena  tag 26c093d1d907ce74  gpl
        loaded_at 2022-11-18T12:44:05+0000  uid 0
        xlated 128B  jited 93B  memlock 4096B  map_ids 254
496: tracepoint  name sys_exit_open  tag 0484b911472301f7  gpl
        loaded_at 2022-11-18T12:44:05+0000  uid 0
        xlated 936B  jited 565B  memlock 4096B  map_ids 254,255
497: tracepoint  name sys_exit_openat  tag 0484b911472301f7  gpl
        loaded_at 2022-11-18T12:44:05+0000  uid 0
        xlated 936B  jited 565B  memlock 4096B  map_ids 254,255

$ bpftool map list 
254: hash  flags 0x0
        key 8B  value 8B  max_entries 4096  memlock 331776B
255: perf_event_array  name printf  flags 0x0
        key 4B  value 4B  max_entries 2  memlock 4096B
```

你可以清楚地看到四个跟踪点程序，以及用于缓存文件名的哈希映射和用于从内核传递输出数据到用户空间的`perf_event_array`。

###### 注意

`bpftrace`实用程序是建立在 BCC 之上的，你在本书的其他地方已经见过了，我将在本章后面介绍。`bpftrace`脚本被转换为 BCC 程序，然后使用 LLVM/Clang 工具链在运行时编译。

如果你想要基于 eBPF 的性能测量的命令行工具，你可能会发现你的需求可以通过[`bpftrace`](https://oreil.ly/u5FrJ)来满足。但是，虽然`bpftrace`可以成为使用 eBPF 进行跟踪的强大工具，但它并没有打开 eBPF 所能实现的全部可能性。

要充分发挥 eBPF 的潜力，你需要直接为内核编写 eBPF 程序，并处理用户空间部分。这两个方面可以，而且通常是用完全不同的语言编写的。让我们从运行在内核中的 eBPF 代码的选择开始。

# 内核中的 eBPF 语言选择

eBPF 程序可以直接用 eBPF 字节码编写，²但实际上，大多数程序是从 C 或 Rust 编译成字节码的。这些语言有编译器支持 eBPF 字节码作为目标输出。

###### 注意

eBPF 字节码并不适合所有编译语言的目标。如果语言涉及运行时组件（比如 Go，或者 Java 的虚拟机），它可能与 eBPF 的验证器不兼容。例如，很难想象内存垃圾收集如何与验证器对内存安全使用的检查相互配合。同样，eBPF 程序要求是单线程的，因此语言中的任何并发特性都无法使用。

虽然不是真正的 eBPF，但有一个有趣的项目叫做[XDPLua](https://oreil.ly/7_3Fx)，它提出用 Lua 脚本在内核中直接运行 XDP 程序。然而，该项目的初步研究表明，eBPF 可能更具性能，随着每个内核版本的 eBPF 变得更加强大（例如，现在能够实现循环），目前还不清楚除了一些人可能更喜欢用 Lua 脚本编写代码之外，是否还有其他优势。

我敢猜测，大多数选择用 Rust 编写 eBPF 内核代码的人也会选择相同的语言来编写用户空间代码，因为共享数据结构不需要重新编写。不过这并非强制性的，你可以混合和匹配 eBPF 代码和任何你选择的用户空间语言。

选择用 C 编写内核端代码的人也可以选择用 C 编写用户空间代码（你在本书中已经看到了很多这样的例子）。但 C 是一种相当低级的语言，需要程序员自己处理很多细节，特别是内存管理。虽然有些人习惯这样做，但很多人更愿意用另一种更高级的语言来编写用户空间代码。无论你偏好哪种语言，你都希望有一个提供 eBPF 支持的库，这样你就不必直接编写到你在第三章中看到的系统调用接口。在本章的其余部分，我们将讨论一些不同语言中最受欢迎的 eBPF 库选项。

# BCC Python/Lua/C++

回到第二章，我给你的第一个“Hello World”例子是使用 BCC 库编写的 Python 程序。这个项目包括了许多有用的性能测量工具，这些工具都是使用相同的库实现的（以及基于*libbpf*的新实现，我马上就会谈到）。

除了[文档](https://oreil.ly/Elggv)描述如何使用提供的 BCC 工具来测量性能，BCC 还包括一个[参考指南](https://oreil.ly/WgeJA)和一个[Python 编程教程](https://oreil.ly/hR3xr)，帮助你在这个框架中开发自己的 eBPF 工具。

第五章包括了对 BCC 的可移植性方法的讨论，即在运行时编译 eBPF 代码，以确保它与目标机器的内核数据结构兼容。在 BCC 中，你将内核端的 eBPF 程序代码定义为一个字符串（或者 BCC 读入字符串的文件内容）。这个字符串被传递给 Clang 进行编译，但在此之前，BCC 对字符串进行了一些预处理。这使得它可以为程序员提供便利的快捷方式，其中一些你在本书中已经看到了。例如，这里是*chapter2/hello_map.py*中示例代码的一些相关行。

```cpp
#!/usr/bin/python3 // ①
from bcc import BPF

program = """                                     // ②
BPF_RINGBUF_OUTPUT(output, 1); // ③ 
...
int hello(void *ctx) {
 ...
 output.ringbuf_output(&data, sizeof(data), 0); // ④

 return 0;
}
"""

b = BPF(text=program)                             // ⑤
...

b["output"].open_ring_buffer(print_event)         // ⑥
...
```

①

这是一个在用户空间运行的 Python 程序。

②

`program`字符串保存了要编译然后加载到内核中的 eBPF 程序。

③

`BPF_RINGBUF_OUTPUT`是一个 BCC 宏，定义了一个名为`output`的环形缓冲区。这是`program`字符串的一部分，所以自然地会认为它是从内核的角度定义缓冲区。等到我们到达调用 6 时再考虑这个问题。

④

这一行看起来像是一个叫做`object`的对象上的`ringbuf_output()`方法。但等一下——对象上的方法甚至不是 C 语言的一部分！BCC 在这里做了一些繁重的工作，将这些方法扩展成底层的 BPF 辅助函数，本例中是`bpf_ringbuf_output()`。

⑤

这是程序字符串被重写为 BPF C 代码的地方，Clang 可以编译。这行还将结果程序加载到内核中。

// ⑥

代码中没有其他地方定义名为`output`的环形缓冲区，但它可以从这里的 Python 用户空间代码中访问。BCC 在预处理调用 3 中承担了双重职责，因为它为用户空间和内核部分都定义了环形缓冲区。

正如这个例子所示，BCC 基本上为 BPF 编程提供了自己的类似 C 的语言。它为程序员提供了便利，处理了内核和用户空间的共享结构定义等问题，并提供了方便的快捷方式来包装 BPF 辅助函数。这意味着，如果你是这个领域的新手，尤其是如果你已经熟悉 Python，BCC 是一个进入 eBPF 编程的可访问的方式。

###### 注意

如果你想探索 BCC 编程，这个[面向 Python 程序员的教程](https://oreil.ly/0pHKY)是一个很好的方式，可以让你了解 BCC 的许多特性和功能，这本书中没有足够的空间来介绍。

文档并不是非常清楚，但是除了支持 Python 作为 eBPF 工具用户空间的语言之外，BCC 还支持用 Lua 和 C++编写工具。在提供的[示例](https://oreil.ly/PP0cL)中有*lua*和*cpp*目录，你可以基于它们编写自己的代码，如果你想尝试这种方法的话。

BCC 可能对程序员很方便，但是由于在你的实用程序旁边分发编译器工具链的低效率（在第五章中更深入讨论），如果你想要编写用于分发的生产质量工具，我建议考虑本章中讨论的其他一些库。

# C 和 Libbpf

在本书中，你已经看到了很多用 C 编写的 eBPF 程序的例子，使用 LLVM 工具链编译成 eBPF 字节码。你也看到了添加了支持 BTF 和 CO-RE 的扩展。许多 C 程序员也熟悉另一个主要的 C 编译器 GCC，并且会很高兴地听到[从版本 10 开始的 GCC](https://oreil.ly/XAzxP)也支持编译为 eBPF 目标；然而，与 LLVM 提供的功能相比，仍然存在一些差距。

正如你在第五章中看到的，CO-RE 和*libbpf*实现了一种可移植的 eBPF 编程方法，不需要在每个 eBPF 工具旁边提供编译器工具链。BCC 项目利用了这一点，并且除了最初的一组 BCC 性能跟踪工具之外，现在还有这些工具的版本重写以利用*libbpf*。普遍的共识是，基于*libbpf*重写的 BCC 工具版本是更好的选择，因为它们具有显著较低的内存占用³，并且不涉及编译步骤的启动延迟。

如果你擅长使用 C 进行编程，使用*libbpf*是很有意义的。在本书中，你已经看到了很多这方面的例子。

###### 注意

要在 C 中编写自己的*libbpf*程序，最好的起点（现在你已经读过这本书了！）是[*libbpf-bootstrap*](https://oreil.ly/4mx81)。阅读 Andrii Nakryiko 的[关于此的博客文章](https://oreil.ly/-OW8v)是对这个项目背后动机的很好介绍。

还有一个名为[*libxdp*](https://oreil.ly/374mL)的库，它构建在*libbpf*之上，以便更轻松地开发和管理 XDP 程序。这是 xdp-tools 的一部分，也是我最喜欢的 eBPF 编程学习资源之一：[XDP 教程](https://oreil.ly/E6dvl)⁴。

但是 C 是一种相当具有挑战性的低级语言。C 程序员必须对内存管理和缓冲区处理等事项负责，很容易编写出存在安全漏洞的代码，更不用说由于错误处理指针而导致的崩溃。eBPF 验证器在内核端提供帮助，但对于用户空间代码没有类似的保护。

好消息是，还有其他编程语言的库可以与*libbpf*进行接口，或者提供类似的重定位功能，以便编写可移植的 eBPF 程序。以下是一些最受欢迎的库。

## Go

Go 语言已经被广泛应用于基础设施和云原生工具，因此在其中编写 eBPF 代码是很自然的选择。

###### 注意

[Michael Kashin 的这篇文章](https://oreil.ly/s9umt)提供了另一个视角，比较了 Go 语言的不同 eBPF 库。

## Gobpf

可能第一个严肃的 Golang 实现是[gobpf](https://oreil.ly/pC0dF)项目，它作为 Iovisor 的一部分与 BCC 并列。然而，它已经有一段时间没有得到积极的维护，就在我写这篇文章的时候，有一些[关于废弃它的讨论](https://oreil.ly/MnE79)，所以在选择库时要记住这一点。

## Ebpf-go

作为 Cilium 项目的一部分，[eBPF Go 库](https://oreil.ly/BnGyl)被广泛使用（我在 GitHub 上找到了大约 1 万个引用，该项目获得了接近 4,000 颗星）。它提供了方便的函数来管理和加载 eBPF 程序和映射，包括 CO-RE 支持，全部纯粹使用 Go 实现。

使用该库，您可以选择将您的 eBPF 程序编译为字节码，并将该字节码嵌入到 Go 源代码中，使用一个名为[bpf2go](https://oreil.ly/-kDbH)的工具。您需要 LLVM/Clang 编译器来在构建步骤中进行此生成。一旦编译了 Go 代码，您就有一个单一的 Go 二进制文件，您可以分发该文件，其中包括 eBPF 字节码，并且可以在不同的内核上运行，而不需要除 Linux 内核本身之外的任何依赖项。

*cilium/ebpf*库还支持加载和管理作为独立 ELF 文件构建的 eBPF 程序（就像您在本书中看到的**.bpf.o*示例）。

在撰写本文时，*cilium/ebpf*库支持用于跟踪的 perf 事件，包括相对较新的 fentry 事件，以及一系列广泛的网络程序类型，如 XDP 和 cgroup 套接字附件。

在这个项目的[*cilium/ebpf*下的*examples*目录](https://oreil.ly/Vuf9d)中，您会看到内核程序的 C 代码与 Go 中相应的用户空间代码位于相同的目录中：

+   C 文件以`// +build ignore`开头，告诉 Go 编译器忽略它们。在撰写本文时，正在进行[更新](https://oreil.ly/ymuyn)，以更改为较新的`//go:build`样式的构建标签。

+   用户空间文件包括以下行，告诉 Go 编译器在 C 文件上调用 bpf2go 工具：

    ```cpp
    //go:generate go run github.com/cilium/ebpf/cmd/bpf2go -cc $BPF_CLANG
                         -cflags $BPF_CFLAGS bpf <C filename> -- -I../headers
    ```

在该包上运行`go:generate`会重新构建 eBPF 程序并在单个步骤中重新生成骨架。

就像你在第五章中看到的`bpftool gen skeleton`一样，`bpf2go`生成了用于操作 eBPF 对象的骨架代码，最大程度地减少了用户空间代码的编写（除了生成 Go 代码而不是 C 代码）。输出文件还包括包含字节码的*.o*对象文件。

事实上，`bpf2go`生成了字节码*.o*文件的两个版本，用于大端和小端架构。还生成了两个相应的*.go*文件，并且在编译时使用了目标平台的正确版本。例如，[来自*cilium/ebpf*的 kprobe 示例](https://oreil.ly/CgwVd)中的自动生成文件是：

+   包含 eBPF 字节码的*bpf_bpfeb.o*和*bpf_bpfel.o* ELF 文件

+   定义 Go 结构和函数的*bpf_bpfeb.go*和*bpf_bpfel.go*文件，这些结构和函数对应于该字节码中定义的映射、程序和链接

您可以将自动生成的 Go 代码中定义的对象与生成它的 C 代码相关联。以下是该 kprobe 示例的 C 代码中定义的对象：

```cpp
struct bpf_map_def SEC("maps") kprobe_map = { `...` ``};` ``SEC``(``"kprobe/sys_execve"``)` ``int` `kprobe_execve``()` `{` ``...` ``}``````cpp
```

```cppThe auto-generated Go code includes structures representing all the maps and programs (in this case, there is only one of each):

```

typebpfMapsstruct{ `KprobeMap``*``ebpf``.``Map```cpp``
```ebpf:"kprobe_execve"`` ``}```cppThe names “KprobeMap” and “KprobeExecve” are derived from the map and program names used in the C code. These objects are grouped into a `bpfObjects` structure representing everything that’s being loaded into the kernel:

```

```cpp
```

typebpfObjectsstruct{ `bpfPrograms` ``bpfMaps` ``}```cppYou can then use these object definitions and related auto-generated functions in your user space Go code. To give you an idea of what this might involve, here’s an extract based on the main function from the same [kprobe example](https://oreil.ly/YXAjH) (omitting error handling for brevity):

```

```cpp

[// ①](#code_id_10_7)

Load all the BPF objects that were embedded in bytecode form, into the `bpfObjects` I just showed you defined by the auto-generated code.

[// ②](#code_id_10_8)

Attach the program to the `sys_execve` kprobe.

[// ③](#code_id_10_9)

Set up a ticker so that the code can poll the map once per second.

[// ④](#code_id_10_10)

Read an item out of the map.

There are several other examples in the *cilium/ebpf* directory that you can use for reference and inspiration.```

objs:=bpfObjects{}loadBpfObjects(&objs,nil)①deferobjs.Close()kp,_:=link.Kprobe("sys_execve",objs.KprobeExecve,nil)②deferkp.Close()ticker:=time.NewTicker(1*time.Second)③deferticker.Stop()forrangeticker.C{varvalueuint64objs.KprobeMap.Lookup(mapKey,&value)④log.Printf("%s called %d times\n",fn,value)}

```cpp``````## Libbpfgo

The [*libbpfgo* project](https://oreil.ly/gvbXr) by Aqua Security implements a Go wrapper around *libbpf*’s C code, providing utilities for loading and attaching programs and using Go-native features like channels for receiving events. Because it’s built on *libbpf*, it supports CO-RE.

Here’s an extract from the example from *libbpfgo*’s *README*, which gives a good high-level view of what to expect from this library:

```  ``## Libbpfgo

由 Aqua Security 实施的[*libbpfgo*项目](https://oreil.ly/gvbXr)在*libbpf*的 C 代码周围实现了一个 Go 包装器，提供了加载和附加程序的实用程序，并使用 Go 本地功能（如用于接收事件的通道）。因为它是建立在*libbpf*之上的，所以它支持 CO-RE。

以下是*libbpfgo*的*README*中的示例摘录，它很好地概述了从该库中可以期望得到的高层视图：

```

[// ①](#code_id_10_11)

Read eBPF bytecode from an object file.

[// ②](#code_id_10_12)

Load that bytecode into the kernel.

[// ③](#code_id_10_13)

Manipulate an entry in an eBPF map.

[// ④](#code_id_10_14)

Go programmers will appreciate receiving data from a ring or perf buffer on a channel, which is a language feature designed to handle asynchronous events.

This library was created for Aqua’s [Tracee](https://oreil.ly/A03zd) security project, and it’s also being used by other projects such as [Parca](https://oreil.ly/s8JP9) from Polar Signals, which provides eBPF-based CPU profiling. The only concern about this project’s approach is the CGo boundary between the *libbpf* C code and Go, which can cause performance and other issues.^([5](ch10.xhtml#ch10fn5))

While Go has been the established language for lots of infrastructure coding for around a decade, there has more recently been a growing body of developers who prefer to use Rust.```cpp

①

从对象文件中读取 eBPF 字节码。

②

将该字节码加载到内核中。

③

操作 eBPF 映射中的条目。

④

Go 程序员会喜欢通过通道从环或 perf 缓冲区接收数据，这是一种处理异步事件的语言特性。

这个库是为 Aqua 的[Tracee](https://oreil.ly/A03zd)安全项目创建的，也被其他项目使用，比如[Polar Signals](https://oreil.ly/s8JP9)的[Parca](https://oreil.ly/s8JP9)，它提供基于 eBPF 的 CPU 性能分析。这个项目方法的唯一问题是*libbpf* C 代码和 Go 之间的 CGo 边界，可能会导致性能和其他问题。⁵

尽管 Go 语言在基础设施编码方面已经有十多年的历史，但最近有越来越多的开发人员更喜欢使用 Rust。``  ``# Rust

Rust 越来越多地被用于构建基础设施工具。它允许像 C 一样进行低级访问，但又具有内存安全的优势。事实上，Linus Torvalds 在 2022 年[确认](https://oreil.ly/7fINA)，Linux 内核本身将开始纳入 Rust 代码，最近的[6.1 版本也有一些初步的 Rust 支持](https://oreil.ly/HrXy2)。

正如我在本章前面讨论的，Rust 可以编译为 eBPF 字节码，这意味着（在正确的库支持下）可以在 Rust 中编写 eBPF 实用程序的用户空间和内核代码。

Rust eBPF 开发有几个选择：*libbpf-rs*、*Redbpf*和 Aya。

## Libbpf-rs

[*Libbpf-rs*](https://oreil.ly/qBagk)是*libbpf*项目的一部分，它提供了*libbpf* C 代码的 Rust 包装，以便您可以用 Rust 编写 eBPF 代码的用户空间部分。从该项目的[示例](https://oreil.ly/6wpf8)中可以看出，eBPF 程序本身是用 C 编写的。

###### 注意

在[*libbpf-bootstrap*](https://oreil.ly/ter6c)项目中还有更多的 Rust 示例，旨在帮助您开始构建自己的代码。

这个 crate 有助于将 eBPF 程序纳入基于 Rust 的项目，但它并不能满足许多人希望在内核端用 Rust 编写代码的愿望。让我们看看其他一些可以实现这一点的项目。

## Redbpf

[*Redbpf*](https://oreil.ly/AtJod)是一组与*libbpf*接口的 Rust crate，作为[eBPF](https://oreil.ly/dwGNK)安全监控代理的一部分开发。

*Redbpf*早于 Rust 编译为 eBPF 字节码的能力，因此它使用了[多步编译过程](https://oreil.ly/DuHxE)，包括从 Rust 编译到 LLVM 位码，然后使用 LLVM 工具链生成 ELF 格式的 eBPF 字节码。*Redbpf*支持一系列程序类型，包括 tracepoints、kprobes 和 uprobes、XDP、TC 以及一些套接字事件。

随着 Rust 编译器 rustc 直接获得生成 eBPF 字节码的能力，这被一个名为 Aya 的项目利用。在撰写本文时，Aya 被认为是“新兴”的，而*Redbpf*被列为一个重要项目，但我个人的观点是，势头似乎正在向 Aya 发展。

## Aya

[Aya](https://aya-rs.dev/book)直接在 Rust 中构建到系统调用级别，因此它不依赖于*libbpf*（或者 BCC 或 LLVM 工具链）。但它支持 BTF 格式，与*libbpf*相同的重定位（如第五章中所述），因此它提供了相同的 CO-RE 能力，可以一次编译，然后在其他内核上运行。在撰写本文时，它支持比*Redbpf*更广泛的 eBPF 程序类型，包括跟踪/性能相关事件、XDP 和 TC、cgroups 以及 LSM 附件。

正如我所提到的，Rust 编译器也支持[编译为 eBPF 字节码](https://oreil.ly/a5q7M)，因此这种语言可以用于内核和用户空间的 eBPF 编程。

###### 注意

在 Rust 中原生编写内核端和用户空间端，而无需中间依赖 LLVM 的能力吸引了 Rust 程序员选择这个选项。关于为什么[lockc 项目](https://oreil.ly/_-L6z)的开发人员决定将其项目从*libbpf-rs*移植到 Aya 的有趣的[GitHub 讨论](https://oreil.ly/nls4l)。

该项目包括[aya-tool](https://oreil.ly/Kd0nf)，这是一个实用程序，用于生成与内核数据结构匹配的 Rust 结构定义，这样您就不必自己编写它们。

Aya 项目非常强调开发者体验，并且让新手很容易上手。考虑到这一点，[“Aya 书”](https://aya-rs.dev/book)是一个非常易读的介绍，其中包含一些很好的示例代码，并附有有用的解释。

为了让您对 Rust 中的 eBPF 代码是什么样子有一个简要的了解，这里是 Aya 基本 XDP 示例的一部分，允许所有流量通过：

```# Rust

Rust is increasingly being used for building infrastructure tools. It allows for the low-level access of C, but with the added benefit of memory safety. Indeed, Linus Torvalds [confirmed in 2022](https://oreil.ly/7fINA) that the Linux kernel itself will start to incorporate Rust code, and the recent [6.1 release has some initial Rust support](https://oreil.ly/HrXy2).

As I discussed earlier in this chapter, Rust can be compiled to eBPF bytecode, meaning that (with the right library support) it’s possible to write both the user space and kernel code for eBPF utilities in Rust.

There are a few options for Rust eBPF development: *libbpf-rs*, *Redbpf*, and Aya.

## Libbpf-rs

[*Libbpf-rs*](https://oreil.ly/qBagk) is part of the *libbpf* project, and provides a Rust wrapper around the *libbpf* C code so that you can write the user space parts of eBPF code in Rust. As you can see from the project’s [examples](https://oreil.ly/6wpf8), the eBPF programs themselves are written in C.

###### Note

There are further examples in Rust in the [*libbpf-bootstrap*](https://oreil.ly/ter6c) project, designed to help you get off the ground if you want to try building your own code using this crate.

This crate is helpful for incorporating eBPF programs into a Rust-based project, but it doesn’t fulfill the desire that many people have to write the kernel-side code in Rust as well. Let’s look at some other projects that enable that.

## Redbpf

[*Redbpf*](https://oreil.ly/AtJod) is a set of Rust crates that interface with *libbpf*, developed as part of [foniod](https://oreil.ly/dwGNK), an eBPF-based security monitoring agent.

*Redbpf* predates Rust’s ability to compile to eBPF bytecode, so it uses a [multistep compilation process](https://oreil.ly/DuHxE) that involves compiling from Rust to LLVM bitcode and then using the LLVM toolchain to generate eBPF bytecode in ELF format. *Redbpf* supports a range of program types including tracepoints, kprobes and uprobes, XDP, TC, and some socket events.

As the Rust compiler rustc gained the ability to generate eBPF bytecode directly, this was leveraged by a project called Aya. At the time of this writing, Aya is considered “emerging” according to the [community site at ebpf.io](https://oreil.ly/WynV6), while *Redbpf* is listed as a major project, but my personal perspective is that momentum seems to be moving toward Aya.

## Aya

[Aya](https://aya-rs.dev/book) is built in Rust directly to the syscall level, so it doesn’t depend on *libbpf* (or indeed on BCC or the LLVM toolchain). But it does support the BTF format, the same relocations that *libbpf* does (as described in [Chapter 5](ch05.xhtml#co_recomma_btfcomma_and_libbpf)), so it’s providing the same CO-RE abilities to compile once and run on other kernels. At the time of this writing, it supports a wider range of eBPF program types than *Redbpf*, including tracing/perf-related events, XDP and TC, cgroups, and LSM attachments.

As I mentioned, the Rust compiler also supports [compiling to eBPF bytecode](https://oreil.ly/a5q7M), so this language can be used for both kernel and user space eBPF programming.

###### Note

The ability to write both the kernel side and the user space side natively in Rust without the intermediate dependency on LLVM has attracted Rust programmers to this option. There’s an interesting [discussion on GitHub](https://oreil.ly/nls4l) about why the developers of the [lockc project](https://oreil.ly/_-L6z) (an eBPF-based project that enhances the security of container workloads using LSM hooks) decided to port their project from *libbpf-rs* to Aya.

The project includes [aya-tool](https://oreil.ly/Kd0nf), a utility for generating Rust structure definitions that match kernel data structures so that you don’t have to write them yourself.

The Aya project strongly emphasizes developer experience and makes it easy for newcomers to get started. With that in mind, the [“Aya book”](https://aya-rs.dev/book) is a very readable introduction with some good example code, annotated with helpful explanations.

To give you a brief idea of what eBPF code looks like in Rust, here’s an extract from Aya’s basic XDP example that permits all traffic:

```cpp

①

这一行定义了部分名称，相当于 C 语言中的`SEC("xdp/myapp")`。

②

名为`myapp`的 eBPF 程序调用`try_myapp`函数来处理在 XDP 接收到的网络数据包。

③

`try_myapp`函数记录了接收到数据包的事实，并始终返回`XDP_PASS`值，告诉内核按照通常方式处理数据包。

就像我们在本书中看到的基于 C 的示例一样，eBPF 程序被编译为 ELF 对象文件。不同之处在于 Aya 使用 Rust 编译器而不是 Clang 来创建该文件。

Aya 还为用户空间加载 eBPF 程序到内核并将其附加到事件的活动生成代码。以下是同一个基本示例的用户空间部分的一些关键行：

```

[// ①](#code_id_10_15)

This line is what defines the section name, equivalent to `SEC("xdp/myapp")` in C.

[// ②](#code_id_10_16)

The eBPF program called `myapp` calls the function `try_myapp` to process a network packet received at XDP.

[// ③](#code_id_10_17)

The `try_myapp` function logs the fact that a packet was received and always returns the `XDP_PASS` value that tells the kernel to carry on processing the packet as usual.

Just as we’ve seen in C-based examples throughout this book, the eBPF program gets compiled to an ELF object file. The difference is that Aya uses the Rust compiler instead of Clang to create that file.

Aya also generates code for the user space activities of loading the eBPF program into the kernel and attaching it to an event. Here are a few key lines from the user space side of that same basic example:

```cpp

①

从编译器生成的 ELF 对象文件中读取 eBPF 字节码。

②

在该字节码中找到名为`myapp`的程序。

③

将其加载到内核中。

④

将其附加到指定网络接口上的 XDP 事件。

如果您是 Rust 程序员，我强烈建议您更详细地探索“Aya 书”中的[其他示例](https://oreil.ly/bp_Hq)。Kong 的[博客文章](https://oreil.ly/mUVIk)也很不错，其中详细介绍了使用 Aya 编写 XDP 负载均衡器。

###### 注意

Aya 的维护者 Dave Tucker 和 Alessandro Decina 在[“eBPF 和 Cilium 办公时间”的第 25 集直播节目](https://oreil.ly/U7bRu)中与我一起，他们演示并介绍了使用 Aya 进行 eBPF 编程。

## Rust-bcc

[Rust-bcc](https://oreil.ly/prP_K)提供了模仿 BCC 项目的 Python 绑定的 Rust 绑定，以及一些 BCC 跟踪[工具](https://oreil.ly/Dd2nO)的一些 Rust 实现。

# 测试 BPF 程序

有一个`bpf()`命令，[`BPF_PROG_RUN`](https://oreil.ly/Y2xPC)，允许从用户空间运行 eBPF 程序进行测试。

`BPF_PROG_RUN`（目前）仅适用于大多数与网络相关的 BPF 程序类型的子集。

您还可以通过一些内置的统计信息获取有关 eBPF 程序性能的信息。运行以下命令以启用它：

```

[// ①](#code_id_10_19)

Read the eBPF bytecode from the ELF object file produced by the compiler.

[// ②](#code_id_10_20)

Find the program called `myapp` in that bytecode.

[// ③](#code_id_10_21)

Load it into the kernel.

[// ④](#code_id_10_22)

Attach it to the XDP event on a specified network interface.

If you’re a Rust programmer, I highly recommend you explore the [additional examples](https://oreil.ly/bp_Hq) in the “Aya book” in more detail. There’s also a nice [blog post from Kong](https://oreil.ly/mUVIk) that walks through writing an XDP load balancer using Aya.

###### Note

Aya maintainers Dave Tucker and Alessandro Decina joined me for [episode 25 of the “eBPF and Cilium Office Hours” livestream](https://oreil.ly/U7bRu) where they demonstrated and gave an introduction to eBPF programming with Aya.

## Rust-bcc

[Rust-bcc](https://oreil.ly/prP_K) provides Rust bindings that mimic the BCC project’s Python bindings, along with some Rust implementations of some of the BCC set of tracing [tools](https://oreil.ly/Dd2nO).

# Testing BPF Programs

There’s a `bpf()` command, [`BPF_PROG_RUN`](https://oreil.ly/Y2xPC), that allows for running an eBPF program from user space for test purposes.

`BPF_PROG_RUN` (currently) works only with a subset of BPF program types that are mostly networking related.

You can also get information about eBPF program performance with some built-in statistics information. Run the following command to enable it:

```cpp

这将在`bpftool`的输出中显示有关程序的其他信息，如下所示：

```

This will show additional information in `bpftool`’s output about programs, like this:

```

额外的统计信息以粗体显示，这里显示该程序已运行四次，总共耗时约 300 微秒。

###### 注意

从 Quentin Monnet 的 FOSDEM 2020 演讲中了解更多，演讲主题是[“调试 BPF 程序的工具和机制”](https://oreil.ly/I5Jhd)。

# 多个 eBPF 程序

eBPF 程序是附加到内核中事件的函数。许多应用程序需要跟踪多个事件以实现其目标。其中一个简单的例子是 opensnoop。⁶我在本章的早期部分介绍了`bpftrace`版本，您看到它将 BPF 程序附加到四个不同的系统调用跟踪点：

+   `syscall_enter_open`

+   `syscall_exit_open`

+   `syscall_enter_openat`

+   `syscall_exit_openat`

这些是内核处理`open()`和`openat()`系统调用的入口和出口点。这两个系统调用可用于打开文件，而 opensnoop 工具跟踪这两个系统调用。

但为什么需要跟踪这些系统调用的入口和出口？入口点用于系统调用参数可用的时候，这些参数包括文件名和传递给`open[at]`系统调用的任何标志。但在那个阶段，还不知道文件是否会成功打开。这就解释了为什么有必要在出口点附加 eBPF 程序。

如果您查看[*libbpf-tools*版本的 opensnoop](https://oreil.ly/IOty_)，您会发现只有一个用户空间程序，并且它将所有四个 eBPF 程序加载到内核中并将它们附加到它们的事件上。eBPF 程序本身基本上是独立的，但它们使用 eBPF 映射来相互协调。

一个复杂的应用程序甚至可能需要在很长一段时间内动态添加和删除 eBPF 程序。对于任何给定的应用程序，甚至可能没有固定数量的 eBPF 程序。例如，Cilium 将 eBPF 程序附加到每个虚拟网络接口上，在 Kubernetes 环境中，这些接口根据运行的 pod 数量而来去。

本章中的大多数库都会自动处理这种多样性的 eBPF 程序。例如，*libbpf*和*ebpf-go*会生成骨架代码，该代码将从对象文件或缓冲区中的字节码加载*所有*程序和映射，并且它们还会生成更细粒度的函数，以便您可以单独操作程序和映射。

# 总结

大多数使用基于 eBPF 的工具的人不需要自己编写 eBPF 代码，但如果您发现自己想要自己实现一些东西，您有很多选择。这是一个不断变化的领域，所以很可能在您阅读本文时，可能已经存在新的语言库和框架，或者已经就本章中我强调的一些库达成共识。您可以在[ebpf.io 的重要项目列表的基础设施页面](https://ebpf.io/infrastructure)上找到关于 eBPF 的主要语言项目的最新列表。

对于快速收集跟踪信息，`bpftrace`可能是一个非常有价值的选择。

对于更灵活和可控的操作，如果您熟悉 Python，BCC 是构建 eBPF 工具的快速方式，前提是您不在乎运行时发生的编译步骤。

如果您希望编写广泛分发和跨不同内核版本可移植的 eBPF 代码，您可能会想利用 CO-RE。目前支持 CO-RE 的用户空间框架有 C 的*libbpf*，Go 的*cilium/ebpf*和*libbpfgo*，以及 Rust 的 Aya。

对于进一步的建议，我强烈建议加入[eBPF Slack](http://ebpf.io/slack)并在那里讨论您的问题。您可能会在该社区中找到许多这些语言库的维护者。

# 练习

如果您想尝试本章讨论的一个或多个库，“Hello World”总是一个很好的开始：

1.  使用您选择的一个或多个库，编写一个输出简单跟踪消息的“Hello World”示例程序。

1.  使用`llvm-objdump`比较从第三章的“Hello World”示例生成的字节码。您会发现很多相似之处！

1.  正如你在第四章中看到的，你可以使用`strace -e bpf`来查看何时进行了`bpf()`系统调用。在你的“Hello World”程序上尝试一下，看看它是否表现如你所期望的那样。

¹ 附加到系统调用入口意味着这个脚本具有与前一章讨论的相同的 TOCTOU（Time Of Check To Time Of Use）漏洞。这并不能阻止它成为一个有用的工具；只是你不应该仅依赖它作为安全目的的唯一防线。

² 例如，可以查看 Cloudflare 的博文“eBPF，Sockets，Hop Distance and manually writing eBPF assembly”。

³ 例如，Brendan Gregg 的[观察](https://oreil.ly/fz_dQ)指出，基于*libbpf*的 opensnoop 版本需要大约 9MB，而基于 Python 的版本需要 80MB。

⁴ 在“eBPF 和 Cilium 办公时间”直播的第 13 集中，看我如何通过一些 XDP 教程示例进行演示。

⁵ Dave Cheney 在 2016 年的帖子“cgo is not Go”仍然是对与 CGo 边界相关的问题的很好的概述。

⁶ 除了这个工具的`bpftrace`版本外，在 BCC 和*libbpf-tools*中也有相应的工具。它们都做着几乎相同的事情，每当一个进程打开一个文件时生成一行跟踪。在我的报告“什么是 eBPF？”中有 BCC 版本 opensnoop 的 eBPF 代码的详细说明。
