# 第十章：eBPF 编程

在本书中，你已经学到了很多关于 eBPF 的知识，并看到了许多如何将其用于各种应用程序的示例。但是如果你想基于 eBPF 实现自己的想法，该章节将讨论你在编写自己的 eBPF 代码时的选择。

正如你从本书中了解到的那样，eBPF 编程包括两个部分：

+   编写在内核中运行的 eBPF 程序

+   编写管理和与 eBPF 程序交互的用户空间代码

我将在本章讨论的大多数库和语言都要求你作为程序员处理这两部分，并意识到处理的具体位置。但是`bpftrace`，也许是最简单的 eBPF 编程语言，掩盖了程序员对这种区别的感知。

# Bpftrace

正如项目的*README*页面所描述的，“`bpftrace`是 Linux eBPF 的高级跟踪语言…灵感来自 awk 和 C，以及前身跟踪器如 DTrace 和 SystemTap。”

[`bpftrace`](https://oreil.ly/BZNZO)命令行工具将用高级语言编写的程序转换为 eBPF 内核代码，并为终端中的结果提供一些输出格式。作为用户，你不需要真正考虑内核与用户空间的分离。

在项目文档中，您会找到几个有用的单行示例，包括一个很好的[教程](https://oreil.ly/Ah2QB)，该教程将引导您从编写简单的“Hello World”脚本到编写更复杂的脚本，可以跟踪内核数据结构中读取的数据。

###### 注意

从 Brendan Gregg 的[`bpftrace`速查表](https://oreil.ly/VBwLm)中了解`bpftrace`提供的各种能力。或者，深入了解`bpftrace`和 BCC，参见他的书籍[*BPF 性能工具*](https://oreil.ly/kjc95)。

如其名所示，`bpftrace`可以附加到跟踪（也称为与性能相关的）事件，包括 kprobe、uprobe 和 tracepoint。例如，您可以使用`-l`选项列出机器上可用的 tracepoint 和 kprobe，就像这样：

```
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

此示例查找所有包含“execve”的可用附加点。从此输出中，您可以看到可以附加到名为`do_execve`的 kprobe。以下是附加到该事件的`bpftrace`单行脚本：

```
bpftrace -e 'kprobe:do_execve { @[comm] = count(); }'
Attaching 1 probe...
^C

@[node]: 6
@[sh]: 6
@[cpuUsage.sh]: 18
```

`{ @[comm] = count(); }`部分是附加到该事件的脚本。此示例跟踪了由不同可执行文件触发该事件的次数。

`bpftrace`的脚本可以协调附加到不同事件的多个 eBPF 程序。例如，考虑报告被打开文件的[*opensnoop.bt*脚本](https://oreil.ly/3HWZ2)。以下是摘录：

```
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

此脚本定义了两个不同的 eBPF 程序，分别附加到两个不同的内核跟踪点，即进入和退出`open()`和`openat()`系统调用时。^(1)这两个系统调用都用于打开文件，并将文件名作为输入参数。由任一系统调用进入触发的程序会缓存该文件名，并将其存储在映射中，其中键是当前线程 ID。当触发退出跟踪点时，脚本中的`/@filename[tid]/`行将从该映射中检索缓存的文件名。

运行此脚本将生成如下输出：

```
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

我刚刚告诉过你有四个 eBPF 程序附加到跟踪点，那么为什么这个输出中说有六个探针？答案是这里有两个“特殊探针”用于该程序的`BEGIN`和`END`子句，这类似于 awk 语言的完整版本。出于简洁起见，我在这里省略了这些子句，但你可以在[GitHub 上的源代码中](https://oreil.ly/X8wgW)找到它们。

如果你正在使用`bpftrace`，你不需要了解底层的程序和映射，但如果你已经阅读了本书的前几章，这些概念对你来说应该已经很熟悉了。如果你有兴趣查看在运行`bpftrace`程序时加载到内核中的程序和映射，你可以轻松使用`bpftool`来完成（就像你在第三章中看到的那样）。这是我在运行*opensnoop.bt*时得到的输出：

```
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

你可以清楚地看到四个跟踪点程序，以及用于缓存文件名的哈希映射和用于将内核输出数据传递到用户空间的`perf_event_array`。

###### 注意

`bpftrace`实用程序是建立在 BCC 之上的，你在本书的其他地方也遇到过，并且我将在本章后面进行详细介绍。`bpftrace`脚本被转换为 BCC 程序，然后使用 LLVM/Clang 工具链在运行时编译。

如果你希望使用基于 eBPF 的性能测量的命令行工具，你可能会发现[`bpftrace`](https://oreil.ly/u5FrJ)能够满足你的需求。但尽管`bpftrace`可以成为利用 eBPF 进行跟踪的强大工具，它并未打开 eBPF 能够实现的所有可能性。

要充分发挥 eBPF 的潜力，你需要直接为内核编写 eBPF 程序，还需要处理用户空间部分。这两个方面通常可以用完全不同的语言编写。让我们从运行在内核中的 eBPF 代码的选择开始。

# 内核中的 eBPF 语言选择

eBPF 程序可以直接用 eBPF 字节码编写，^(2)但在实践中，大多数情况下是从 C 或 Rust 编译为字节码。这些语言的编译器支持将 eBPF 字节码作为目标输出。

###### 注意

eBPF 字节码并不适合所有编译语言。如果语言涉及运行时组件（如 Go 或 Java 的虚拟机），它可能与 eBPF 的验证器不兼容。例如，很难想象内存垃圾回收如何与验证器对内存安全使用的检查协同工作。同样，eBPF 程序必须是单线程的，因此语言中的任何并发特性都无法使用。

尽管不完全是 eBPF，但有一个有趣的项目叫做[XDPLua](https://oreil.ly/7_3Fx)，提议使用 Lua 脚本在内核中直接运行 XDP 程序。然而，该项目的初步研究表明，eBPF 可能更具性能，随着每个内核版本发布，eBPF 变得越来越强大（例如现在能够实现循环），并不清楚除了个人偏好外是否有其他优势。

我敢猜测，大多数选择用 Rust 编写 eBPF 内核代码的人也会选择同样的语言编写用户空间代码，因为共享数据结构不需要重新编写。当然，这并非强制性的——你可以将 eBPF 代码与任何你选择的用户空间语言混合使用。

那些选择用 C 语言编写内核端代码的人也可以选择在用户空间用 C 语言编写代码（本书中已有多个例子）。但是 C 语言是一种相当底层的语言，需要程序员自己处理许多细节，特别是内存管理。虽然有些人能够轻松应对这些，但许多人更愿意用另一种更高级的语言编写用户空间代码。无论你偏爱哪种语言，你都希望有一个提供 eBPF 支持的库，这样你就不必直接写系统调用接口了（你在第三章看到过） 。在本章的其余部分，我们将讨论一些流行的 eBPF 库选项，涵盖多种语言。

# BCC Python/Lua/C++

回到第二章，我给你的第一个“Hello World”示例是使用 BCC 库编写的 Python 程序。该项目包括许多有用的性能测量工具，使用了同一库（以及稍后我将介绍的基于*libbpf*的新实现）。

除了描述如何使用提供的 BCC 工具来测量性能的[文档](https://oreil.ly/Elggv)，BCC 还包括一个[参考指南](https://oreil.ly/WgeJA)和一个[Python 编程教程](https://oreil.ly/hR3xr)，帮助您在这个框架中开发自己的 eBPF 工具。

第五章 讨论了 BCC 在可移植性方面的方法，即在运行时编译 eBPF 代码，以确保其与目标机器的内核数据结构兼容。在 BCC 中，您将内核端 eBPF 程序代码定义为字符串（或 BCC 读入字符串的文件内容）。该字符串被传递给 Clang 进行编译，但在此之前，BCC 对字符串进行了一些预处理。这使其能够为程序员提供方便的快捷方式，其中一些已经在本书中看到了。例如，以下是来自 *chapter2/hello_map.py* 示例代码的相关行：

```
#!/usr/bin/python3 ![1](img/1.png)
from bcc import BPF

program = """                                     ![2](img/2.png)
BPF_RINGBUF_OUTPUT(output, 1); ![3](img/3.png) 
...
int hello(void *ctx) {
 ...
 output.ringbuf_output(&data, sizeof(data), 0); ![4](img/4.png)

 return 0;
}
"""

b = BPF(text=program)                             ![5](img/5.png)
...

b["output"].open_ring_buffer(print_event)         ![6](img/6.png)
...
```

![1](img/#code_id_10_1)

这是一个运行在用户空间的 Python 程序。

![2](img/#code_id_10_2)

`program` 字符串包含要编译并加载到内核中的 eBPF 程序。

![3](img/#code_id_10_3)

`BPF_RINGBUF_OUTPUT` 是一个 BCC 宏，用于定义名为 `output` 的环形缓冲区。这是 `program` 字符串的一部分，因此可以自然地认为它从内核的角度定义了缓冲区。在我们到达 callout 6 之前，请暂时保留这个想法。

![4](img/#code_id_10_4)

此行看起来像是 `object` 对象上的 `ringbuf_output()` 方法。但等一下 —— 对象的方法甚至不是 C 语言的一部分！BCC 在这里进行了一些繁重的工作，像这样扩展方法到底层的 BPF 助手函数，例如在这种情况下是 `bpf_ringbuf_output()`。

![5](img/#code_id_10_5)

这是程序字符串被重写为 Clang 可以编译的 BPF C 代码的地方。这行还将结果程序加载到内核中。

![6](img/#code_id_10_6)

代码中没有定义名为`output`的环形缓冲区的其他位置，但在此处的 Python 用户空间代码中可以访问它。在调用 3 中，BCC 在预处理行时起到了双重作用，因为它为用户空间和内核部分都定义了环形缓冲区。

正如这个例子所示，BCC 实际上为 BPF 编程提供了自己的类似 C 的语言。它为程序员简化了生活，处理了诸如共享结构定义（供内核和用户空间使用）等事务，并提供了方便的快捷方式来包装 BPF 助手函数。这意味着如果您对 Python 已经感到满意，那么 BCC 是进入 eBPF 编程的一种可访问方式，尤其是如果您对该领域还不熟悉的话。

###### 注意

如果您想探索 BCC 编程，这个[面向 Python 程序员的教程](https://oreil.ly/0pHKY)是了解 BCC 的更多特性和能力的好方法，比本书所能包含的内容要多得多。

文档并没有非常清楚，但除了支持 Python 作为 eBPF 工具用户空间的语言外，BCC 还支持用 Lua 和 C++编写工具。如果你有兴趣尝试这种方法，提供的[示例](https://oreil.ly/PP0cL)中有*lua*和*cpp*目录，你可以基于它们编写自己的代码。

BCC 对程序员来说可能很方便，但由于在你的实用程序旁边分发编译器工具链的低效率（在第五章中更深入讨论），如果你打算编写可分发的生产质量工具，我建议考虑本章讨论的其他一些库。

# C 和 Libbpf

你在本书中已经看到了很多用 C 编写的 eBPF 程序的示例，使用 LLVM 工具链编译成 eBPF 字节码。你还看到了添加了支持 BTF 和 CO-RE 的扩展。许多 C 程序员也熟悉另一种主要的 C 编译器 GCC，并且将很高兴听到，从版本 10 开始，[GCC 也支持编译为 eBPF 目标](https://oreil.ly/XAzxP)；然而，与 LLVM 提供的功能相比，仍然存在一些差距。

正如你在第五章中看到的那样，CO-RE 和*libbpf*实现了一种便携式 eBPF 编程的方法，不需要在每个 eBPF 工具旁边附带编译器工具链。BCC 项目利用了这一点，并且除了最初的一套 BCC 性能跟踪工具之外，现在还有这些工具的版本重写以利用*libbpf*。普遍的共识是，基于*libbpf*重写的 BCC 工具版本是更好的选择，因为它们具有显著更低的内存占用^(3)，并且不涉及在编译步骤进行时的启动延迟。

如果你熟悉 C 编程，使用*libbpf*会非常合理。在本书的多个示例中已经看到了很多这样的例子。

###### 注意

要用 C 编写你自己的*libbpf*程序，现在（现在你已经读完本书！）最好的起点是[*libbpf-bootstrap*](https://oreil.ly/4mx81)。阅读 Andrii Nakryiko 的[关于该项目的博客文章](https://oreil.ly/-OW8v)，作为该项目背后动机的良好介绍。

也有一个名为[*libxdp*](https://oreil.ly/374mL)的库，它建立在*libbpf*之上，以便更轻松地开发和管理 XDP 程序。这是 xdp-tools 的一部分，该工具集还包含我喜欢的 eBPF 编程学习资源之一：[XDP 教程](https://oreil.ly/E6dvl)。^(4)

但是 C 是一门相当具有挑战性的低级语言。C 程序员必须对内存管理和缓冲区处理等事项负责，并且很容易因为处理指针不当而导致编写具有安全漏洞的代码，更不用说由于这些问题而导致崩溃。eBPF 验证器在内核端有所帮助，但对于用户空间代码没有类似的保护机制。

好消息是，还有其他编程语言的库与 *libbpf* 接口，或提供类似重定位功能，以便于可移植的 eBPF 程序。以下是一些最流行的库。

## Go

Go 语言已被广泛应用于基础设施和云原生工具，因此在其中编写 eBPF 代码是很自然的选择。

###### 注意

[Michael Kashin 的这篇文章](https://oreil.ly/s9umt) 提供了比较不同 Go eBPF 库的另一视角。

## Gobpf

可能第一个严肃的 Golang 实现是 [gobpf](https://oreil.ly/pC0dF) 项目，作为 Iovisor 的一部分与 BCC 并列。然而，它已经有一段时间没有得到积极维护，在我写作此文时，有一些 [讨论计划弃用它](https://oreil.ly/MnE79)，因此在选择库时请记住这一点。

## Ebpf-go

[Cilium 项目的一部分，包含的 eBPF Go 库](https://oreil.ly/BnGyl) 被广泛使用（我在 GitHub 上找到约 10,000 个引用，并且该项目接近 4,000 个星）。它提供了方便的函数来管理和加载 eBPF 程序和映射，包括 CO-RE 支持，全部纯 Go 实现。

使用此库，您可以选择将您的 eBPF 程序编译为字节码，并将该字节码嵌入到 Go 源代码中，使用名为 [bpf2go](https://oreil.ly/-kDbH) 的提供的工具。在构建步骤中，您需要 LLVM/Clang 编译器来生成此字节码。一旦 Go 代码编译完成，您将得到一个单一的 Go 二进制文件，其中包含 eBPF 字节码，并且可以在不同内核之间进行移植，除了 Linux 内核本身之外没有任何依赖项。

*cilium/ebpf* 库还支持加载和管理构建为独立 ELF 文件的 eBPF 程序（例如您在本书中看到的 **.bpf.o** 示例）。

在撰写本文时，*cilium/ebpf* 库支持用于跟踪的 perf 事件，包括比较新的 fentry 事件，以及一系列广泛的网络程序类型，如 XDP 和 cgroup 套接字附加。

在本项目的 [*cilium/ebpf* 目录下的 *examples* 目录中](https://oreil.ly/Vuf9d)，您会看到内核程序的 C 代码与相应的 Go 用户空间代码位于相同的目录中：

+   C 文件以 `// +build ignore` 开头，告诉 Go 编译器忽略它们。在撰写本文时，正在进行一个 [更新](https://oreil.ly/ymuyn)，以改用更新的 `//go:build` 样式的构建标签。

+   用户空间文件包括如下一行，告诉 Go 编译器在 C 文件上调用 bpf2go 工具：

    ```
    //go:generate go run github.com/cilium/ebpf/cmd/bpf2go -cc $BPF_CLANG
                         -cflags $BPF_CFLAGS bpf <C filename> -- -I../headers
    ```

    运行`go:generate`命令对包进行重建，以一步完成 eBPF 程序的重新生成和骨架的更新。

类似于`bpftool gen skeleton`，您在第五章中看到的，`bpf2go`为操作 eBPF 对象生成骨架代码，减少了您需要自己编写的用户空间代码（不过它生成的是 Go 代码而不是 C 代码）。输出文件还包括包含字节码的*.o*对象文件。

实际上，`bpf2go`生成了字节码的*.o*文件的两个版本，用于大端和小端架构。还有两个相应生成的*.go*文件，在编译时根据目标平台选择正确的版本。例如，在[*cilium/ebpf*中的 kprobe 示例](https://oreil.ly/CgwVd)中自动生成的文件包括：

+   包含 eBPF 字节码的*bpf_bpfeb.o*和*bpf_bpfel.o* ELF 文件

+   *bpf_bpfeb.go*和*bpf_bpfel.go*文件定义了对应于字节码中定义的映射、程序和链接的 Go 结构和函数。

您可以将自动生成的 Go 代码中定义的对象与生成它的 C 代码相关联。以下是为该 kprobe 示例中的 C 代码定义的对象：

```
struct bpf_map_def SEC("maps") kprobe_map = {
...
};

SEC("kprobe/sys_execve")
int kprobe_execve() {
...
}
```

自动生成的 Go 代码包括表示所有映射和程序的结构（在本例中只有一个映射和一个程序）：

```
type bpfMaps struct {
    KprobeMap *ebpf.Map `ebpf:"kprobe_map"`
}

type bpfPrograms struct {
    KprobeExecve *ebpf.Program `ebpf:"kprobe_execve"`
}
```

名称“KprobeMap”和“KprobeExecve”源自用于 C 代码中的映射和程序名称。这些对象被分组到一个`bpfObjects`结构中，代表着加载到内核中的所有内容：

```
type bpfObjects struct {
    bpfPrograms
    bpfMaps
}
```

然后，您可以在用户空间的 Go 代码中使用这些对象定义和相关的自动生成函数。为了让您了解可能涉及的内容，这里有一个基于相同[kprobe 示例](https://oreil.ly/YXAjH)主函数的摘录（为简洁起见省略了错误处理）：

```
objs := bpfObjects{}                                   
loadBpfObjects(&objs, nil)                             ![1](img/1.png) 
defer objs.Close()

kp, _ := link.Kprobe("sys_execve", 
                     objs.KprobeExecve, nil)           ![2](img/2.png)
defer kp.Close()

ticker := time.NewTicker(1 * time.Second)              ![3](img/3.png)
defer ticker.Stop()

for range ticker.C {
    var value uint64
    objs.KprobeMap.Lookup(mapKey, &value)              ![4](img/4.png)
    log.Printf("%s called %d times\n", fn, value)
}
```

![1](img/#code_id_10_7)

将所有以字节码形式嵌入的 BPF 对象加载到`bpfObjects`中，刚才我向您展示了由自动生成的代码定义的内容。

![2](img/#code_id_10_8)

将程序附加到`sys_execve` kprobe 上。

![3](img/#code_id_10_9)

设置一个定时器，以便代码每秒钟轮询映射。

![4](img/#code_id_10_10)

从映射中读取一个项目。

在*cilium/ebpf*目录中还有其他几个示例，可以用作参考和灵感。

## Libbpfgo

[*libbpfgo*项目](https://oreil.ly/gvbXr)由 Aqua Security 实现了围绕*libbpf*的 C 代码的 Go 包装器，提供了加载和附加程序的实用工具，并使用 Go 本地特性（如通道）接收事件。因为它构建在*libbpf*上，所以支持 CO-RE。

这里是从*libbpfgo*的*README*中的示例摘录，它很好地高层次地展示了这个库的预期效果：

```
bpfModule := bpf.NewModuleFromFile(bpfObjectPath)         ![1](img/1.png)
bpfModule.BPFLoadObject()                                 ![2](img/2.png)

mymap, _ := bpfModule.GetMap("mymap")                     ![3](img/3.png)
mymap.Update(key, value)

rb, _ := bpfModule.InitRingBuffer("events", eventsChannel, buffSize)
rb.Start()
e := <-eventsChannel                                      ![4](img/4.png)
```

![1](img/#code_id_10_11)

从对象文件中读取 eBPF 字节码。

![2](img/#code_id_10_12)

将这段字节码加载到内核中。

![3](img/#code_id_10_13)

操纵 eBPF 映射中的条目。

![4](img/#code_id_10_14)

Go 程序员将喜欢通过通道从环或性能缓冲区接收数据，这是一种处理异步事件的语言特性。

这个库是为 Aqua 的[Tracee](https://oreil.ly/A03zd)安全项目创建的，也被其他项目如 Polar Signals 的[Parca](https://oreil.ly/s8JP9)所使用，它提供基于 eBPF 的 CPU 性能分析。对于这个项目的一个关注点是*libbpf* C 代码和 Go 之间的 CGo 边界可能会引起性能和其他问题^(5)。

虽然在过去十年中，Go 语言一直是许多基础设施编码的首选语言，但最近有越来越多的开发者更倾向于使用 Rust。

# Rust

Rust 在构建基础设施工具方面的使用越来越广泛。它允许像 C 语言一样进行低级别访问，但又具有内存安全性的附加好处。确实，Linus Torvalds 在 2022 年[确认](https://oreil.ly/7fINA)，Linux 内核本身将开始整合 Rust 代码，最近的[6.1 版本已经开始支持 Rust](https://oreil.ly/HrXy2)。

正如我在本章前面讨论的那样，Rust 可以编译成 eBPF 字节码，这意味着（通过正确的库支持）可以用 Rust 编写 eBPF 实用程序的用户空间和内核代码。

对于 Rust eBPF 开发，有几个选择：*libbpf-rs*，*Redbpf*和 Aya。

## Libbpf-rs

[*Libbpf-rs*](https://oreil.ly/qBagk)是*libbpf*项目的一部分，提供了围绕*libbpf* C 代码的 Rust 包装器，使您可以用 Rust 编写 eBPF 代码的用户空间部分。正如您可以从该项目的[示例](https://oreil.ly/6wpf8)中看到的那样，eBPF 程序本身是用 C 语言编写的。

###### 注意

在[*libbpf-bootstrap*](https://oreil.ly/ter6c)项目中还有更多的 Rust 示例，旨在帮助您开始构建自己的代码。

这个 crate 对于将 eBPF 程序整合到基于 Rust 的项目中很有帮助，但它不能满足许多人希望在内核端用 Rust 编写代码的愿望。让我们看看其他一些支持这一功能的项目。

## Redbpf

[*Redbpf*](https://oreil.ly/AtJod)是一组与*libbpf*接口的 Rust 包，作为[eBPF](https://oreil.ly/dwGNK)安全监控代理的一部分进行开发。

*Redbpf*在 Rust 能够编译成 eBPF 字节码之前就已存在，因此它使用一个[多步编译过程](https://oreil.ly/DuHxE)，包括从 Rust 编译到 LLVM 比特码，然后使用 LLVM 工具链生成 eBPF 字节码的 ELF 格式。*Redbpf*支持一系列程序类型，包括跟踪点、kprobes 和 uprobes、XDP 以及一些套接字事件。

随着 Rust 编译器 rustc 直接获得生成 eBPF 字节码的能力，这一能力被一个名为 Aya 的项目所利用。在撰写本文时，根据[ebpf.io 社区网站](https://oreil.ly/WynV6)，Aya 被认为是“新兴”项目，而*Redbpf*被列为一个主要项目，但我个人认为动力似乎正在向 Aya 方向发展。

## Aya

[Aya](https://aya-rs.dev/book) 直接在 Rust 中构建到系统调用级别，因此它不依赖于*libbpf*（或者 BCC 或 LLVM 工具链）。但它支持 BTF 格式，与*libbpf*相同的重定位（如第五章中描述的），因此它提供了相同的 CO-RE 能力，可以编译一次并在其他内核上运行。在撰写本文时，它支持比*Redbpf*更广泛的 eBPF 程序类型，包括跟踪/性能相关事件、XDP 和 TC、cgroups 以及 LSM 附件。

正如我提到的，Rust 编译器也支持[编译为 eBPF 字节码](https://oreil.ly/a5q7M)，因此这种语言可以用于内核和用户空间的 eBPF 编程。

###### 注意

能够在 Rust 中原生编写内核端和用户空间端，而无需中间依赖于 LLVM，吸引了 Rust 程序员选择这个选项。关于为什么[lockc 项目](https://oreil.ly/_-L6z)的开发人员决定将他们的项目从 *libbpf-rs* 迁移到 Aya 的有趣[讨论](https://oreil.ly/nls4l)在 GitHub 上进行。

该项目包括 [aya-tool](https://oreil.ly/Kd0nf)，一个用于生成与内核数据结构匹配的 Rust 结构定义的实用程序，这样你就不必自己编写它们。

Aya 项目非常强调开发者体验，并且让新手很容易上手。考虑到这一点，[“Aya 书”](https://aya-rs.dev/book)是一个非常易读的介绍，附有一些很好的示例代码，并附有有用的解释。

为了让你简要了解 Rust 中的 eBPF 代码是什么样子，这里是 Aya 基本 XDP 示例的一部分，允许所有流量通过：

```
#[xdp(name="myapp")]                                         ![1](img/1.png)
pub fn myapp(ctx: XdpContext) -> u32 {
    match unsafe { try_myapp(ctx) } {                        ![2](img/2.png)
        Ok(ret) => ret,
        Err(_) => xdp_action::XDP_ABORTED,
    }
}

unsafe fn try_myapp(ctx: XdpContext) -> Result<u32, u32> {   ![3](img/3.png)
    info!(&ctx, "received a packet");
    Ok(xdp_action::XDP_PASS)
}
```

![1](img/#code_id_10_15)

这一行定义了部分名称，相当于 C 中的 `SEC("xdp/myapp")`。

![2](img/#code_id_10_16)

名为 `myapp` 的 eBPF 程序调用函数 `try_myapp` 来处理在 XDP 接收到的网络数据包。

![3](img/#code_id_10_17)

`try_myapp` 函数记录了接收到数据包的事实，并始终返回`XDP_PASS`值，告诉内核继续按照通常方式处理数据包。

正如我们在本书中看到的基于 C 的示例一样，eBPF 程序被编译为 ELF 对象文件。不同之处在于 Aya 使用 Rust 编译器而不是 Clang 来创建该文件。

Aya 还为将 eBPF 程序加载到内核并将其附加到事件的用户空间活动生成代码。以下是该基本示例的用户空间关键行：

```
let mut bpf = Bpf::load(include_bytes_aligned!(
   "../../target/bpfel-unknown-none/release/myapp"
))?;                                                                     ![1](img/1.png)

let program: &mut Xdp = bpf.program_mut("myapp").unwrap().try_into()?;   ![2](img/2.png) 

program.load()?;                                                         ![3](img/3.png)
program.attach(&opt.iface, XdpFlags::default())                          ![4](img/4.png)
```

![1](img/#code_id_10_19)

从编译器生成的 ELF 对象文件中读取 eBPF 字节码。

![2](img/#code_id_10_20)

查找名为 `myapp` 的程序的字节码。

![3](img/#code_id_10_21)

将其加载到内核中。

![4](img/#code_id_10_22)

将其附加到指定网络接口上的 XDP 事件。

如果你是 Rust 程序员，我强烈建议你更详细地探索“Aya 书籍”中的[其他示例](https://oreil.ly/bp_Hq)。Kong 的[博客文章](https://oreil.ly/mUVIk)也很好地介绍了使用 Aya 编写 XDP 负载均衡器。

###### 注意

Aya 的维护者 Dave Tucker 和 Alessandro Decina 在[“eBPF 和 Cilium 办公室时间”直播第 25 集](https://oreil.ly/U7bRu)中加入了我，他们演示并介绍了使用 Aya 进行 eBPF 编程。

## Rust-bcc

[Rust-bcc](https://oreil.ly/prP_K) 提供了模仿 BCC 项目 Python 绑定的 Rust 绑定，并附带一些 BCC 跟踪工具的 Rust 实现。

# 测试 BPF 程序

有一个 `bpf()` 命令，[`BPF_PROG_RUN`](https://oreil.ly/Y2xPC)，允许从用户空间运行 eBPF 程序进行测试。

`BPF_PROG_RUN`（目前）仅适用于大多数与网络相关的 BPF 程序类型。

您还可以通过一些内置统计信息获取有关 eBPF 程序性能的信息。运行以下命令以启用它：

```
$ sysctl -w kernel.bpf_stats_enabled=1
```

这将显示`bpftool`的输出中关于程序的额外信息，例如：

```
$ bpftool prog list 
...
2179: raw_tracepoint  name raw_tp_exec  tag 7f6d182e48b7ed38  gpl
        run_time_ns 316876 run_cnt 4
        loaded_at 2023-01-09T11:07:31+0000  uid 0
        xlated 216B  jited 264B  memlock 4096B  map_ids 780,777
        btf_id 953
        pids hello(19173)
```

额外的统计信息显示为粗体，这里显示该程序已运行四次，总共大约耗时 300 微秒。

###### 注意

从 Quentin Monnet 的 FOSDEM 2020 演讲“工具和机制来调试 BPF 程序”中了解更多信息。

# 多个 eBPF 程序

一个 eBPF 程序是附加到内核中事件的函数。许多应用程序需要跟踪多个事件以实现其目标。这方面的一个简单示例是 opensnoop。^(6) 本章早期我介绍了 `bpftrace` 版本，并展示了它如何将 BPF 程序附加到四个不同的系统调用跟踪点上：

+   `syscall_enter_open`

+   `syscall_exit_open`

+   `syscall_enter_openat`

+   `syscall_exit_openat`

这些是内核处理 `open()` 和 `openat()` 系统调用的入口和出口点。这两个系统调用可用于打开文件，而 opensnoop 工具会跟踪这两个调用。

但是为什么需要同时跟踪这些系统调用的进入和退出呢？使用进入点是因为此时系统调用参数是可用的，这些参数包括要传递给`open[at]`系统调用的文件名和任何标志。但在这个阶段，还不知道文件是否会成功打开。这解释了为什么有必要在退出点也附加 eBPF 程序。

如果你查看[*libbpf-tools*版本的 opensnoop](https://oreil.ly/IOty_)，你会看到只有一个用户空间程序，它将所有四个 eBPF 程序加载到内核并将它们附加到它们的事件上。这些 eBPF 程序本身基本上是独立的，但它们使用 eBPF 映射来在彼此之间协调。

在一个复杂的应用程序中，甚至可能需要在很长一段时间内动态地添加和删除 eBPF 程序。对于任何给定的应用程序，甚至可能没有固定数量的 eBPF 程序。例如，Cilium 将 eBPF 程序附加到每个虚拟网络接口上，在 Kubernetes 环境中，这些接口的存在与否取决于运行的 Pod 数量。

本章中的大多数库都会自动处理这些 eBPF 程序的多样性。例如，*libbpf*和*ebpf-go*会生成加载所有程序和映射的骨架代码，这可以通过一个函数调用完成。它们还生成更细粒度的函数，以便你可以单独操作程序和映射。

# 摘要

大多数使用基于 eBPF 的工具的人不需要自己编写 eBPF 代码，但如果你确实希望自己实现一些东西，你有很多选择。这是一个不断变化的领域，所以很可能在你阅读这篇文章时，会有新的语言库和框架存在，或者已经在一些我在本章中强调的库周围形成了共识。你可以在[ebpf.io 重要项目列表的基础设施页面](https://ebpf.io/infrastructure)找到关于 eBPF 主要语言项目的最新列表。

为了快速收集跟踪信息，`bpftrace`可以是一个非常有价值的选择。

对于更灵活和控制性更强的需求，如果你熟悉 Python，并且不介意运行时发生的编译步骤，BCC 是构建 eBPF 工具的快速方式。

如果你正在编写 eBPF 代码，希望在不同的内核版本间广泛分发和移植，你可能会想要利用 CO-RE。目前支持 CO-RE 的用户空间框架有 C 语言的*libbpf*，Go 语言的*cilium/ebpf*和*libbpfgo*，以及 Rust 语言的 Aya。

如果需要进一步的建议，我强烈建议加入[eBPF Slack](http://ebpf.io/slack)，在那里讨论你的问题。你很可能会在这个社区找到许多这些语言库的维护者。

# 练习

如果您想尝试本章讨论的一个或多个库，那么“Hello World”总是一个很好的开始：

1.  使用您选择的一个或多个库，编写一个“Hello World”程序，输出一个简单的跟踪消息。

1.  使用`llvm-objdump`比较从第三章的“Hello World”示例生成的字节码。你会发现很多相似之处！

1.  正如您在第四章中看到的那样，您可以使用`strace -e bpf`来查看何时进行`bpf()`系统调用。尝试在您的“Hello World”程序上执行此操作，看看它是否按预期行事。

^(1) 附加到系统调用入口点意味着这个脚本具有与上一章讨论的 TOCTOU（时间检查到使用时间）漏洞相同的漏洞。这并不能阻止它成为一个有用的工具；只是你不应该将其作为安全目的的唯一防线。

^(2) 例如，查看 Cloudflare 的博文[“eBPF，Sockets，Hop Distance and manually writing eBPF assembly”](https://oreil.ly/2GjuK)。

^(3) 例如，Brendan Gregg 的[观察](https://oreil.ly/fz_dQ)显示，*libbpf*版本的 opensnoop 大约需要 9 MB，而基于 Python 的版本需要 80 MB。

^(4) 在[第 13 集 eBPF 和 Cilium 办公时间的直播](https://oreil.ly/9SaKn)中，看我演示一些 XDP 教程示例。

^(5) 戴夫·陈尼在 2016 年的文章“[cgo is not Go](https://oreil.ly/mxThs)”仍然是对与 CGo 边界相关问题的很好概述。

^(6) 除了`bpftrace`版本的工具之外，在 BCC 和*libbpf-tools*中也有等效的工具。它们都做着相同的事情，每当一个进程打开文件时生成一行跟踪。在我的报告[“什么是 eBPF？”](https://www.oreilly.com/library/view/what-is-ebpf/9781492097266)中有 BCC 版本 opensnoop 的 eBPF 代码演示。
