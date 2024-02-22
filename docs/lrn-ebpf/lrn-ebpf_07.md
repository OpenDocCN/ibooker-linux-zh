# 第七章：eBPF 程序和附加类型

在前面的章节中，你看到了许多 eBPF 程序的示例，你可能已经注意到它们附加到不同类型的事件上。我展示的一些示例附加到 kprobes，但在其他示例中，我演示了处理新到达的网络数据包的 XDP 程序。这只是内核中许多附加点中的两个。

###### 注意

你可以使用[*github.com/lizrice/learning-ebpf*](https://github.com/lizrice/learning-ebpf)上的代码和说明构建和运行本章的示例。本章的代码位于*chapter7*目录中。

在撰写本文时，一些示例在 ARM 处理器上不受支持。查看*chapter7*目录中的*README*文件以获取更多详细信息和建议。

[*uapi/linux/bpf.h*](https://oreil.ly/6dNIW)目前列举了大约 30 种程序类型和 40 多种附加类型。附加类型更具体地定义了程序的附加位置；对于许多程序类型，附加类型可以从程序类型中推断出来，但是一些程序类型可以附加到内核中的多个不同点，因此还必须指定附加类型。

正如你所知，这本书并不是参考手册，所以我不会涵盖每一种 eBPF 程序类型。在你阅读本书时，很可能已经添加了新的类型！

# 程序上下文参数

所有 eBPF 程序都接受一个上下文参数，这是一个指针，但它指向的结构取决于触发它的事件类型。eBPF 程序员需要编写接受适当类型上下文的程序；如果事件是 tracepoint，那么假装上下文参数指向网络数据包是没有意义的。定义不同类型的程序允许验证器确保上下文信息得到适当处理，并强制执行有关允许的辅助函数的规则。

###### 注意

要深入了解传递给不同 BPF 程序类型的上下文数据的详细信息，请查看[Alan Maguire 在 Oracle 博客上的这篇文章](https://oreil.ly/6dNIW)。

# 辅助函数和返回代码

正如你在上一章中看到的，验证器会检查程序使用的所有辅助函数是否与其程序类型兼容。上一章的示例表明，`bpf_get_current_pid_tgid()`辅助函数在 XDP 程序中是不允许的。在接收数据包并触发 XDP 钩子的地方，没有涉及用户空间进程或线程，因此在这种情况下调用发现当前进程和线程 ID 是没有意义的。

程序类型还确定了程序的返回代码的含义。再次以 XDP 为例，返回代码值告诉内核在 eBPF 程序完成处理后该如何处理数据包——可能涉及将其传递到网络堆栈，丢弃它，或将其重定向到不同的接口。当 eBPF 程序由于命中特定的 tracepoint 而触发时，这些返回代码就没有任何意义，因为这时没有涉及网络数据包。

有一个[辅助函数的 manpage](https://oreil.ly/e8K73)（合理地指出，由于 BPF 子系统的持续开发，它可能不完整）。

你可以使用`bpftool feature`命令获取你的内核版本中每种程序类型可用的辅助函数列表。这显示了系统配置，并列出了所有可用的程序类型和映射类型，甚至列出了每种程序类型支持的所有辅助函数。

辅助函数被视为*UAPI*的一部分，即 Linux 内核的外部稳定接口。因此，一旦在内核中定义了一个辅助函数，即使内核的内部函数和数据结构发生变化，它也不应该改变。

尽管内核版本之间的变化风险很大，但 eBPF 程序员们希望能够从 eBPF 程序中访问一些内部函数。这可以通过称为*BPF 内核函数*或[*kfuncs*](https://oreil.ly/gKSEx)的机制来实现。

# Kfuncs

Kfuncs 允许将内部内核函数注册到 BPF 子系统中，以便验证程序允许从 eBPF 程序中调用它们。对于每种允许调用给定 kfunc 的 eBPF 程序类型，都有一个注册。

与辅助函数不同，kfuncs 不提供兼容性保证，因此 eBPF 程序员必须考虑内核版本之间的变化可能性。

有一组[“核心”BPF kfuncs](https://oreil.ly/06qoi)，在撰写本文时，这些函数允许 eBPF 程序获取和释放对任务和 cgroups 的内核引用。

总之，eBPF 程序的类型决定了它可以附加到哪些事件上，从而定义了它接收的上下文信息的类型。程序类型还定义了它可以调用的辅助函数和 kfuncs 的集合。

程序类型通常被广泛认为分为两类：跟踪（或 perf）程序类型和与网络相关的程序类型。让我们看一些例子。

# 跟踪

附加到 kprobes、tracepoints、原始 tracepoints、fentry/fexit probes 和 perf events 的程序都旨在为内核中的 eBPF 程序提供一种有效的方式，将有关事件的跟踪信息报告到用户空间。这些与跟踪相关的类型并不会影响内核对它们附加的事件的响应方式（尽管正如您将在第九章中看到的那样，在这个领域已经有了一些创新！）。

有时这些被称为“与 perf 相关”的程序。例如，`bpftool perf`子命令允许您查看附加到与 perf 相关事件的程序，如下所示：

```cpp
$ sudo bpftool perf show
pid 232272  fd 16: prog_id 392  kprobe  func __x64_sys_execve  offset 0
pid 232272  fd 17: prog_id 394  kprobe  func do_execve  offset 0
pid 232272  fd 19: prog_id 396  tracepoint  sys_enter_execve
pid 232272  fd 20: prog_id 397  raw_tracepoint  sched_process_exec
pid 232272  fd 21: prog_id 398  raw_tracepoint  sched_process_exec
```

前面的输出是我在*chapter7*目录中的*hello.bpf.c*文件中运行示例代码时看到的，它将不同的程序附加到与`execve()`相关的各种事件上。我将在本节中讨论所有这些类型，但总的来说，这些程序是：

+   一个附加到`execve()`系统调用入口点的 kprobe。

+   一个附加到内核函数`do_execve()`的 kprobe。

+   一个放置在`execve()`系统调用入口处的 tracepoint。

+   在处理`execve()`时调用的两个版本的原始 tracepoint。其中一个，正如您将在本节中看到的那样，是启用了 BTF 的版本。

您需要`CAP_PERFMON`和`CAP_BPF`或`CAP_SYS_ADMIN`权限才能使用任何与跟踪相关的 eBPF 程序类型。

## Kprobes 和 Kretprobes

我在第一章中讨论了 kprobes 的概念。您可以将 kprobe 程序附加到内核中的几乎任何地方。通常，它们是使用 kprobes 附加到函数的入口处，使用 kretprobes 附加到函数的出口处，但是您可以使用 kprobes 将其附加到函数入口之后的某个指定偏移处的指令。如果选择这样做，您需要确信您运行的内核版本具有您想要附加到的指令的位置！附加到内核函数的入口和出口点可能相对稳定，但是任意代码行可能很容易在一个版本发布后被修改。

###### 注意

在`bpftool perf list`的示例输出中，您可以看到两个 kprobe 的偏移量都为 0。

当内核编译时，编译器还可能选择“内联”任何给定的内核函数；也就是说，编译器可能会发出机器代码来实现被调用函数内部的任何操作，而不是从调用函数处跳转。如果函数恰好被内联，那么您的 eBPF 程序将无法附加到 kprobe 入口点。

### 附加 kprobes 到系统调用入口点

本章的第一个 eBPF 程序示例称为`kprobe_sys_execve`，它是附加到`execve（）`系统调用的 kprobe。函数及其部分定义如下：

```cpp
SEC("ksyscall/execve") `int` `BPF_KPROBE_SYSCALL``(``kprobe_sys_execve``,` `char` `*``pathname``)`
```

``这与您在第五章中看到的内容相同。

附加到系统调用的一个原因是它们是稳定的接口，在内核版本之间不会更改（跟踪点也是如此，我们很快就会谈到）。但是，不应该依赖系统调用 kprobes 进行安全工具编程，我将在第九章中详细介绍原因。``  ``### 将 kprobes 附加到其他内核函数

您可以找到许多示例，其中基于 eBPF 的工具使用 kprobes 附加到系统调用，但是，如前所述，kprobes 也可以附加到内核中的任何非内联函数。我在*hello.bpf.c*中提供了一个示例，该示例将 kprobe 附加到函数`do_execve（）`，并且定义如下：

```cpp
SEC("kprobe/do_execve") `int` `BPF_KPROBE``(``kprobe_do_execve``,` `struct` `filename` `*``filename``)`
```

``因为`do_execve（）`不是系统调用，所以与前一个示例之间存在一些差异：

+   SEC 名称的格式与附加到系统调用入口点的先前版本相同，但无需定义特定于平台的变体，因为`do_execve（）`与大多数内核函数一样，适用于所有平台。

+   我使用了`BPF_KPROBE`宏，而不是`BPF_KPROBE_SYSCALL`。意图完全相同，只是后者处理系统调用参数。

+   还有一个重要的区别：系统调用的`pathname`参数是指向字符串的指针（`char *`），但对于此函数，参数称为`filename`，它是指向`struct filename`的指针，这是内核中使用的数据结构。

您可能想知道我是如何知道要为此参数使用此类型的。我会告诉你。内核中的`do_execve（）`函数具有以下签名：

```cpp
int `do_execve`(struct `filename` *`filename`,
    const char `__user` *const `__user` *__argv,
    const char `__user` *const `__user` *__envp)
```

我选择忽略`do_execve（）`参数`__argv`和`__envp`，并且只声明`filename`参数，使用`struct filename *`类型来匹配内核函数的定义。鉴于参数在内存中是顺序排列的，忽略最后的*n*个参数是可以的，但如果您想使用后面的参数，则不能忽略列表中的较早的参数。

这个`filename`结构在内核内部定义，它说明了 eBPF 编程是内核编程的一个例子：我必须查找`do_execve（）`的定义以找到其参数，以及`struct filename`的定义。即将运行的可执行文件的名称由`filename->name`指向。在示例代码中，我使用以下行检索此名称：

```cpp
const char *name = BPF_CORE_READ(filename, name); `bpf_probe_read_kernel``(``&``data``.``command``,` `sizeof``(``data``.``command``),` `name``);`
```

``因此，总结一下：系统调用 kprobe 的上下文参数是表示用户空间传递给系统调用的值的结构。“常规”（非系统调用）kprobe 的上下文参数是表示由调用它的任何内核代码传递给被调用函数的参数的结构，因此结构取决于函数定义。

Kretprobes 与 kprobes 非常相似，不同之处在于它们在函数返回时触发，并且可以访问返回值而不是参数。

Kprobes 和 kretprobes 是挂接到内核函数的合理方式，但是，如果您正在运行最新的内核，您应该考虑一个更新的选项。```cpp```  ```## Fentry/Fexit

A more efficient mechanism for tracing the entry to and exit from kernel functions was introduced along with the idea of *BPF trampoline* in kernel version 5.5 (on x86 processors; BPF trampoline support doesn’t arrive for [ARM processors until Linux 6.0](https://oreil.ly/ccuz1)). If you’re using a recent enough kernel, fentry/fexit is now the preferred method for tracing the entry to or exit from a kernel function. You can write the same code inside a kprobe or fentry type program.

There’s an example fentry program called `fentry_execve()` in *chapter7/hello.bpf.c*. I declared the eBPF program for this kprobe using *libbpf*’s macro `BPF_PROG`, which is another convenient wrapper giving access to typed parameters rather than the generic context pointer, but this version is used for fentry, fexit, and tracepoint program types. The definition looks like this:

```cpp

SEC（“fentry/do_execve”）`int``BPF_PROG``（``fentry_execve``,``struct``filename``*``filename``）`

```

 ``The section name tells *libbpf* to attach to the fentry hook at the start of the `d⁠o⁠_​e⁠x⁠e⁠c⁠v⁠e⁠(⁠)` kernel function. Just as in the kprobe example, the context parameters reflect the parameters passed to the kernel function where you want to attach this eBPF program.

Fentry and fexit attachment points were designed to be more efficient than kprobes, but there’s another advantage when you want to generate an event at the end of a function: the fexit hook has access to the input parameters to the function, which kretprobe does not. You can see an example of this in [*libbpf-bootstrap*’s examples](https://oreil.ly/6HDh_). Both *kprobe.bpf.c* and *fentry.bpf.c* are equivalent examples that hook into the `do_unlinkat()` kernel function. The eBPF program attached to the kretprobe has the following signature:

```cpp

SEC("kretprobe/do_unlinkat") `int``BPF_KRETPROBE``(``do_unlinkat_exit``,``long``ret``)`

```

 ``The `BPF_KRETPROBE` macro expands to make this a kretprobe program on exit from `do_unlinkat()`. The only parameter the eBPF program receives is `ret`, which holds the return value from `do_unlinkat()`. Compare this to the fexit version:

```cpp

SEC("fexit/do_unlinkat") `int``BPF_PROG``(``do_unlinkat_exit``,``int``dfd``,``struct``filename``*``name``,``long``ret``)`

```

 ``In this version the program gets access not just to the return value `ret`, but also to the input parameters to `do_unlinkat()`, which are `dfd` and `name`.```cpp```  ```## Tracepoints

[Tracepoints](https://oreil.ly/yXk_L)是内核代码中标记的位置（我们将在本章后面介绍用户空间 tracepoints）。它们绝不是 eBPF 的专属，长期以来一直用于生成内核跟踪输出，并被诸如[SystemTap](https://oreil.ly/bLmQL)之类的工具使用。与使用 kprobes 附加到任意指令不同，tracepoints 在内核版本之间是稳定的（尽管旧内核可能没有新内核中添加的全部 tracepoints）。

您可以通过查看*/sys/kernel/tracing/available_events*来查看内核上可用的跟踪子系统，如下所示：

```cpp
$ cat /sys/kernel/tracing/available_events 
tls:tls_device_offload_set
tls:tls_device_decrypted
...
syscalls:sys_exit_execveat
syscalls:sys_enter_execveat
syscalls:sys_exit_execve
syscalls:sys_enter_execve
...
```

我的内核版本 5.15 在此列表中定义了超过 1400 个 tracepoint。tracepoint eBPF 程序的部分定义应该与这些项目中的一个匹配，以便*libbpf*可以自动将其附加到 tracepoint。定义的形式为`SEC("tp/tracing subsystem/tracepoint name")`。

您将在*chapter7/hello.bpf.c*文件中找到一个示例，该示例与`syscalls:sys_enter_execve` tracepoint 匹配，当内核开始处理`execve()`调用时会触发该 tracepoint。部分定义告诉*libbpf*这是一个 tracepoint 程序，并且应该附加到哪里，就像这样：

```cpp
SEC("tp/syscalls/sys_enter_execve")
```

`tracepoint 的上下文参数呢？我马上就会讲到，BTF 可以在这里帮助我们，但首先让我们考虑在 BTF 不可用时需要什么。每个 tracepoint 都有一个格式，描述从中跟踪出的字段。例如，这是`execve()`系统调用入口的格式：

```cpp
$ cat /sys/kernel/tracing/events/syscalls/sys_enter_execve/format
name: sys_enter_execve
ID: 622
format:
  field:unsigned short common_type;         offset:0;  size:2; signed:0;
  field:unsigned char common_flags;         offset:2;  size:1; signed:0;
  field:unsigned char common_preempt_count; offset:3;  size:1; signed:0;
  field:int common_pid;                     offset:4;  size:4; signed:1;

  field:int __syscall_nr;                   offset:8;  size:4; signed:1;
  field:const char * filename;              offset:16; size:8; signed:0;
  field:const char *const * argv;           offset:24; size:8; signed:0;
  field:const char *const * envp;           offset:32; size:8; signed:0;

print fmt: "filename: 0x%08lx, argv: 0x%08lx, envp: 0x%08lx", 
((unsigned long)(REC->filename)), ((unsigned long)(REC->argv)), 
((unsigned long)(REC->envp))
```

我使用这些信息在*chapter7/hello.bpf.c*中定义了一个匹配的结构，称为`m⁠y⁠_⁠s⁠y⁠s⁠c⁠a⁠l⁠l⁠s⁠_⁠e⁠n⁠t⁠e⁠r⁠_​e⁠x⁠e⁠c⁠v⁠e`：

```cpp
struct my_syscalls_enter_execve { `unsigned` `short` `common_type``;` ``unsigned` `char` `common_flags``;` ``unsigned` `char` `common_preempt_count``;` ``int` `common_pid``;` ``long` `syscall_nr``;` ``long` `filename_ptr``;` ``long` `argv_ptr``;` ``long` `envp_ptr``;` ``};``````cpp```

```

 ```cppeBPF 程序不允许访问这四个字段中的前四个。如果尝试访问它们，程序将因`invalid bpf_context access`错误而无法验证。

我的示例 eBPF 程序可以附加到这个 tracepoint，并将这种类型的指针用作其上下文参数，就像这样：

```
int tp_sys_enter_execve(struct my_syscalls_enter_execve *ctx) {
```cpp

`然后您可以访问此结构的内容。例如，您可以按如下方式获取文件名指针：

```
bpf_probe_read_user_str(&data.command, sizeof(data.command), ctx->filename_ptr);
```cpp

`当您使用 tracepoint 程序类型时，传递给 eBPF 程序的结构已经从一组原始参数映射出来。为了获得更好的性能，您可以直接访问这些原始参数，使用原始 tracepoint eBPF 程序类型。部分定义应该以`raw_tp`（或`raw_tracepoint`）开头，而不是`tp`。您需要将参数从`__u64`转换为 tracepoint 结构使用的任何类型（当 tracepoint 是系统调用的入口时，这些参数取决于芯片架构）。``````cpp  ```## BTF-Enabled Tracepoints

In the previous example I wrote a structure called `my_syscalls_enter_execve` to define the context parameter for my eBPF program. But when you define a structure in your eBPF code or parse the raw arguments, there’s a risk that your code might not match the kernel it’s running on. The good news is that BTF, which you met in [Chapter 5](ch05.xhtml#co_recomma_btfcomma_and_libbpf), also solves this problem.

With BTF support, there will be a structure defined in *vmlinux.h* that matches the context structure passed to a tracepoint eBPF program. Your eBPF program should use the section definition `SEC("tp_btf/*tracepoint name*")` where the tracepoint name is one of the available events listed in */sys/kernel/tracing/available_events*. The example program in *chapter7/hello.bpf.c* looks like this:

```cpp
SEC("tp_btf/sched_process_exec") `int` `handle_exec``(``struct` `trace_event_raw_sched_process_exec` `*``ctx``)`
```

 ``As you can see, the structure name matches the tracepoint name, prefixed with `trace_event_raw_`.``  ``## User Space Attachments

So far I have shown examples of eBPF programs attaching to events defined within the kernel’s source code. There are similar attachment points within user space code: uprobes and uretprobes for attaching to the entry and exit of user space functions, and user statically defined tracepoints (USDTs) for attaching to specified tracepoints within application code or user space libraries. These all use the `BPF_PROG_TYPE_KPROBE` program type.

###### Note

There are lots of public examples of programs attached to user space events. Here are a few from the BCC project:

*   The [bashreadline](https://oreil.ly/gDkaQ) and [funclatency tools](https://oreil.ly/zLT54) attach to u(ret)probe.

*   [USDT sample](https://oreil.ly/o894f) in BCC.

If you’re using *libbpf*, the `SEC()` macro lets you define the auto-attachment point for these user space probes. You’ll find the format required for the section name in the [*libbpf* documentation](https://oreil.ly/o0CBQ). For example, to attach a uprobe to the start of the `SSL_write()` function in OpenSSL, you would define the section for the eBPF program with the following:

```cpp
SEC("uprobe/usr/lib/aarch64-linux-gnu/libssl.so.3/SSL_write")
```

 `There are a few gotchas to be aware of when instrumenting user space code:

*   Notice that the path to this shared library in this example is architecture specific, so you may need corresponding architecture-specific definitions.

*   Unless you control the machine you’re running the code on, you can’t know what user space libraries and applications will be installed.

*   An application might be built as a standalone binary, so it won’t hit any probes you might attach within shared libraries.

*   Containers typically run with their own copy of a filesystem, with their own set of dependencies installed in it. The path to a shared library used by a container won’t be the same as the path to a shared library on the host machine.

*   Your eBPF program might need to be aware of the language in which an application was written. For example, in C the arguments to a function are generally passed using registers, but in Go they are passed using the stack,^([3](ch07.xhtml#ch07fn3)) so the `pt_args` structure holding register information may be of less use.

That said, there are lots of useful tools that instrument user space applications with eBPF. For example, you can hook into the SSL library to trace out decrypted versions of encrypted information—we’ll explore this in more detail in the next chapter. Another example is continuous profiling of your applications, using tools such as [Parca](https://www.parca.dev).`  `## LSM

`BPF_PROG_TYPE_LSM` programs are attached to the *Linux Security Module (LSM) API*, which is a stable interface within the kernel originally intended for kernel modules to use to enforce security policies. As you’ll see in [Chapter 9](ch09.xhtml#ebpf_for_security), where I’ll discuss this in more detail, eBPF security tooling can now use this interface too.

`BPF_PROG_TYPE_LSM` programs are attached using `bpf(BPF_RAW_TRACEPOINT_OPEN)`, and in many ways they are treated like tracing programs. One interesting characteristic of `BPF_PROG_TYPE_LSM` programs is that the return value affects the way the kernel behaves. A nonzero return code indicates that the security check wasn’t passed, so the kernel won’t proceed with whatever operation it was asked to complete. This is a significant difference from perf-related program types where the return code is ignored.

###### Note

The Linux kernel documentation covers [LSM BPF programs](https://oreil.ly/vcPHY).

The LSM program type isn’t the only one with a role to play in security. Many of the networking-related program types that you’ll see in the next section can be used for network security to permit or deny networking traffic or networking-related operations. You’ll also see more about eBPF being used for security purposes in [Chapter 9](ch09.xhtml#ebpf_for_security).

So far in this chapter you have seen how a set of kernel and user space tracing program types enable visibility over the whole system. The next set of eBPF program types to consider are those that let us hook into the network stack, with the option not merely to observe but also to affect how it handles data being sent and received.```cpp```````cpp``  ```# Networking

There are lots of different eBPF program types intended to process network messages as they pass through various points in the network stack. [Figure 7-1](#bpf_program_types_hook_into_various_poi) shows where some of the commonly used program types attach. These program types all require `CAP_NET_ADMIN` and `CAP_BPF`, or `CAP_SYS_ADMIN`, capabilities to be permitted.

The context passed to these types of programs is the network message in question, although the type of structure depends on the data the kernel has at the relevant point in the network stack. At the bottom of the stack, data is held in the form of Layer 2 network packets, which are essentially a series of bytes that have been or are ready to be transmitted “on the wire.” At the top of the stack, applications use sockets, and the kernel creates socket buffers to handle data being sent and received from these sockets.

![BPF program types hook into various points in the network stack](assets/lebp_0701.png)

###### Figure 7-1\. BPF program types hook into various points in the network stack

###### Note

The network layer model is beyond the scope of this book, but it’s covered in many other books, posts, and training courses. I discussed it in [Chapter 10](ch10.xhtml#ebpf_programming) of [*Container Security*](https://www.oreilly.com/library/view/container-security/9781492056690/) (O’Reilly). For the purposes of this book, it’s sufficient to know that Layer 7 covers formats intended for applications to use, such as HTTP, DNS, or gRPC; TCP is at Layer 4; IP is at Layer 3; and Ethernet and WiFi are at Layer 2\. One of the roles of the networking stack is to convert messages between these different formats.

One big difference between the networking program types and the tracing-related types you saw earlier in this chapter is that they are generally intended to allow for the customization of networking behaviors. That involves two main characteristics:

1.  Using a return code from the eBPF program to tell the kernel what to do with a network packet—which could involve processing it as usual, dropping it, or redirecting it to a different destination

2.  Allowing the eBPF program to modify network packets, socket configuration parameters, and so on

You’ll see some examples of how these characteristics are used to build powerful networking capabilities in the next chapter, but for now, here’s an overview of the eBPF program types.

## Sockets

At the top of the stack, a subset of these network-related program types relates to sockets and socket operations:

*   `BPF_PROG_TYPE_SOCKET_FILTER` was the first program type to be added to the kernel. You probably guessed from the name that this is used for socket filtering, but what’s less obvious is that this doesn’t mean filtering data being sent to or from an application. It’s used to filter a *copy* of socket data that can be sent to an observability tool such as tcpdump.

*   A socket is specific to a Layer 4 (TCP) connection. `BPF_PROG_TYPE_SOCK_OPS` allows eBPF programs to intercept various operations and actions that take place on a socket, and to set for that socket parameters such as TCP timeout values. Sockets only exist at the endpoints for a connection, and not on any middleboxes that they might pass through.

*   `BPF_PROG_TYPE_SK_SKB` programs are used in conjunction with a special map type that holds a set of references to sockets to provide what’s known as [*sockmap* operations](https://oreil.ly/0Enuo): redirecting traffic to different destinations at the socket layer.

## Traffic Control

Further down the network stack comes “TC” or traffic control. There is a whole subsystem in the Linux kernel related to TC, and a glance at the [manpage for the `tc` command](https://oreil.ly/kfyg5) will give you an idea of how complex it is and how important it is to computing in general to have deep levels of flexibility and configuration over the way network packets are handled.

eBPF programs can be attached to provide custom filters and classifiers for network packets for both ingress and egress traffic. This is one of the building blocks of the Cilium project, and I’ll cover some examples in the next chapter. If you can’t wait until then, there are some good examples on [Quentin Monnet’s blog](https://oreil.ly/heQ2D). This can be done programmatically, but you also have the option to use the `tc` command to manipulate these kinds of eBPF programs.

## XDP

You briefly met XDP (eXpress Data Path) eBPF programs in [Chapter 3](ch03.xhtml#anatomy_of_an_ebpf_program). In that example I loaded the eBPF program and attached it to the `eth0` interface using the following commands:

```cpp

bpftool prog load hello.bpf.o /sys/fs/bpf/hello

bpftool net attach xdp id 540 dev eth0

```

It’s worth noting that XDP programs attach to a specific interface (or virtual interface), and you may very well have different XDP programs attached to different interfaces. In [Chapter 8](ch08.xhtml#ebpf_for_networking) you’ll learn more about how XDP programs can be offloaded to network cards or executed by network drivers.

XDP programs are another example of programs that can be managed using Linux network utilities—in this case, the `link` subcommand of [iproute2’s ip](https://oreil.ly/8Isau). The roughly equivalent command for loading and attaching the program to `eth0` would be this:

```cpp

$ ip link set dev eth0 xdp obj hello.bpf.o sec xdp

```

This command reads the eBPF program marked as section `xdp` from the `hello.bpf.o` object and attaches it to the `eth0` network interface. The `ip link show` command for this interface now includes some information about the XDP program that’s attached to it:

```cpp

2：eth0：<BROADCAST，MULTICAST，UP，LOWER_UP> mtu 1500 xdpgeneric qdisc fq_codel

状态 UP 模式默认组默认 qlen 1000

link/ether 52:55:55:3a:1b:a2 brd ff:ff:ff:ff:ff:ff

prog/xdp id 1255 tag 9d0e949f89f1a82c jited

```

Removing the XDP program with `ip link` can be done like this:

```cpp

$ ip link set dev eth0 xdp off

```

You’ll see a lot more about XDP programs and their applications in the next chapter.

## Flow Dissector

A flow dissector is used at various points in the network stack to extract details from a packet’s headers. eBPF programs of type `BPF_PROG_TYPE_FLOW_DISSECTOR` can implement custom packet dissection. There’s a nice write-up in this LWN article on [writing network flow dissectors in BPF](https://oreil.ly/nFKLV).

## Lightweight Tunnels

The family of `BPF_PROG_TYPE_LWT_*` program types can be used to implement network encapsulation in eBPF programs. These program types can also be manipulated using the `ip` command, but this time it’s the `route` subcommand that’s involved. In practice, these are used infrequently.

## Cgroups

eBPF programs can be attached to cgroups (short for “control groups”). *Cgroups* are a concept in the Linux kernel that restricts the set of resources a given process or group of processes can have access to. Cgroups are one of the mechanisms that isolate one container (or one Kubernetes pod) from another. Attaching eBPF programs to a cgroup allows for custom behavior that only applies to that cgroup’s processes. All processes are associated with a cgroup, including processes that are not running inside a container.

There are several cgroup-related program types, and even more hooks where they can be attached. At least at the time of this writing, they are nearly all networking related, although there is also a `BPF_CGROUP_SYSCTL` program type that can be attached to sysctl commands affecting a particular cgroup.

As an example, there are socket-related program types specific to cgroups `BPF_PROG_TYPE_CGROUP_SOCK` and `BPF_PROG_TYPE_CGROUP_SKB`. eBPF programs can determine whether a given cgroup is permitted to perform a requested socket operation or data transmission. This is useful for network security policy enforcement (which I’ll cover in the next chapter). Socket programs can also trick the calling process into thinking they are connecting to a particular destination address.

## Infrared Controllers

Programs of type [BPF_PROG_TYPE_LIRC_MODE2](https://oreil.ly/AwG1C) can be attached to the file descriptor for an infrared controller device to provide decoding for infrared protocols. At the time of this writing, this program type requires `CAP_NET_ADMIN`, but I think this illustrates that the division of program types into tracing related and networking related doesn’t fully express the range of different applications that eBPF can address.

# BPF Attachment Types

The attachment type offers more fine-grained control over where a program can be attached in the system. For some program types there is a one-to-one correlation to the type of hook that it can be attached to, so the attachment type is implicitly defined by the program type. For example, XDP programs are attached to XDP hooks in the network stack. For a few program types, an attachment type also has to be specified.

The attachment type is involved in deciding which helper functions are valid, and it also restricts access to parts of the context information in some cases. There was an example of this earlier in this chapter where the verifier gives an `invalid bpf_context access` error.

You can also see which program types need an attachment type to be specified, and which attachment types are valid, in the kernel function [bpf_prog_load_check_attach](https://oreil.ly/0LqCQ) (defined in [*bpf/syscall.c*](https://oreil.ly/7OrYS)).

For example, here is the code that checks the attachment type for a program of type `CGROUP_SOCK`:

```cpp

case`BPF_PROG_TYPE_CGROUP_SOCK`:`switch`(`expected_attach_type`){case`BPF_CGROUP_INET_SOCK_CREATE`:case`BPF_CGROUP_INET_SOCK_RELEASE`:case`BPF_CGROUP_INET4_POST_BIND`:case`BPF_CGROUP_INET6_POST_BIND`:return0;default:return-`EINVAL`;}

```

This program type can be attached in multiple places: at socket creation, at socket release, or after a bind is completed in IPv4 or IPv6.

Another place to find a listing of the valid attachment types for programs is the [*libbpf* documentation](https://oreil.ly/jraLh), where you’ll also find the section names that *libbpf* understands for each program and attachment type.

# Summary

In this chapter you saw that various eBPF program types are used to attach into different hook points in the kernel. If you want to write code that responds to a particular event, you’ll need to determine the program type(s) that are appropriate for hooking onto that event. The context passed into the program depends on the program type, and the kernel may also respond differently to the return code from your program, depending on its type.

The example code for this chapter mostly focused on perf-related (tracing) events. In the next two chapters you’ll see more details on different eBPF program types used for networking and security applications.

# Exercises

The example code for this chapter includes kprobe, fentry, tracepoint, raw tracepoint, and BTF-enabled tracepoint programs that are all attached to the entry to the same system call. As you know, eBPF tracing programs can be attached to many other places besides syscalls.

1.  Run the example code using `strace` to capture the `bpf()` system calls, like this:

    ```cpp

strace -e bpf -o outfile ./hello

```

    This will record information about each `bpf()` syscall into a file called *outfile*. Look for the `BPF_PROG_LOAD` instructions in that file, and see how the `prog_type` file varies for different programs. You can identify which program is which by the `prog_name` field in the trace, and match them to the source code in *chapter7/hello.bpf.c*.

2.  The example user space code in *hello.c* loads all the program objects defined in `hello.bpf.o`. As an exercise in writing *libbpf* user space code, modify the example code load and attach just one of the eBPF programs (pick whichever one you like), without removing those programs from *hello.bpf.c*.

3.  Write a kprobe and/or fentry program that is triggered when some other kernel function is called. You can find the available functions in your kernel version by looking at */proc/kallsyms*.

4.  Write a regular, raw or BTF-enabled tracepoint program that attaches to some other kernel tracepoint. You can find the available tracepoints in `/sys/kernel/tracing/available_events`.

5.  Try to attach more than one XDP program to a given interface, and confirm that you can’t! You should see an error that looks something like this:

    ```cpp

libbpf：内核错误消息：XDP 程序已附加

错误：接口 xdpgeneric 附加失败：设备或资源忙

```

^([1](ch07.xhtml#ch07fn1-marker)) Except for a few parts of the kernel where kprobes aren’t permitted for security reasons. These are listed in `/sys/kernel/debug/kprobes/blacklist`.

^([2](ch07.xhtml#ch07fn2-marker)) The only example I have seen so far is in the [cilium/ebpf test suite](https://oreil.ly/rL5E8).

^([3](ch07.xhtml#ch07fn3-marker)) Up to Go version 1.17, when a new register-based calling convention was introduced. Nevertheless, I think there will be Go executables built with older versions circulating for some time to come.```
