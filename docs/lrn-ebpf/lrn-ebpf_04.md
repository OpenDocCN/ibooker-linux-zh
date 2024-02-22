# 第四章：bpf()系统调用

正如您在第一章中看到的，当用户空间应用程序希望内核代表它们执行某些操作时，它们会使用系统调用 API 发出请求。因此，如果用户空间应用程序希望将 eBPF 程序加载到内核中，必须涉及一些系统调用。实际上，有一个名为`bpf()`的系统调用，在本章中我将向您展示如何使用它来加载和与 eBPF 程序和映射进行交互。

值得注意的是，运行在内核中的 eBPF 代码不使用系统调用来访问映射。系统调用接口仅由用户空间应用程序使用。相反，eBPF 程序使用辅助函数来读取和写入映射；您在前两章中已经看到了这方面的示例。

如果您继续自己编写 eBPF 程序，您很有可能不会直接调用这些`bpf()`系统调用。本书后面将讨论的库提供了更高级的抽象，使事情变得更容易。也就是说，这些抽象通常与您在本章中看到的底层系统调用命令相当直接地对应。无论您使用哪个库，您都需要掌握底层操作——加载程序，创建和访问映射等——这些操作您将在本章中看到。

在我向您展示`bpf()`系统调用的示例之前，让我们考虑一下[`bpf()`的 manpage](https://oreil.ly/NJdIM)所说的，即`bpf()`用于“对扩展 BPF 映射或程序执行命令”。它还告诉我们，`bpf()`的签名如下：

```cpp
int bpf(int cmd, union bpf_attr *attr, unsigned int size);
```

`bpf()`的第一个参数`cmd`指定要执行的命令。`bpf()`系统调用不仅仅执行一项任务——有许多不同的命令可用于操作 eBPF 程序和映射。图 4-1 概述了用户空间代码可能用于加载 eBPF 程序、创建映射、将程序附加到事件以及访问映射中键-值对的一些常见命令。

![用户空间程序使用 syscalls 与内核中的 eBPF 程序和映射进行交互](img/lebp_0401.png)

###### 图 4-1：用户空间程序使用 syscalls 与内核中的 eBPF 程序和映射进行交互

`bpf()`系统调用的`attr`参数保存着指定命令参数所需的任何数据，`size`指示`attr`中有多少字节的数据。

您在第一章中已经遇到了`strace`，当时我用它来展示用户空间代码如何通过系统调用 API 发出许多请求。在本章中，我将使用它来演示`bpf()`系统调用的使用。`strace`的输出包括每个系统调用的参数，但为了避免本章中的示例输出过于混乱，我将省略`attr`参数中的许多细节，除非它们特别有趣。

###### 注意

您将在[*github.com/lizrice/learning-ebpf*](https://github.com/lizrice/learning-ebpf)找到代码，以及设置运行环境的说明。本章的代码位于*chapter4*目录中。

对于本示例，我将使用一个名为*hello-buffer-config.py*的 BCC 程序，它在您在第二章中看到的示例基础上进行了扩展。与*hello-buffer.py*示例一样，该程序在每次运行时都向 perf 缓冲区发送消息，将内核中关于`execve()`系统调用事件的信息传递给用户空间。这个版本的新功能是允许为每个用户 ID 配置不同的消息。

以下是 eBPF 源代码：

```cpp
struct user_msg_t {                                          // ①
  char message[12];
};

BPF_HASH(config, u32, struct user_msg_t);                    // ②

BPF_PERF_OUTPUT(output);                                     // ③

struct data_t {                                              // ④
  int pid;
  int uid;
  char command[16];
  char message[12];
};

int hello(void *ctx) {                                       // ⑤
  struct data_t data = {};
  struct user_msg_t *p;
  char message[12] = "Hello World";

  data.pid = bpf_get_current_pid_tgid() >> 32;
  data.uid = bpf_get_current_uid_gid() & 0xFFFFFFFF;

  bpf_get_current_comm(&data.command, sizeof(data.command));

  p = config.lookup(&data.uid);                              // ⑥
  if (p != 0) {
     bpf_probe_read_kernel(&data.message, sizeof(data.message), p->message);      
  } else {
     bpf_probe_read_kernel(&data.message, sizeof(data.message), message);
  }

  output.perf_submit(ctx, &data, sizeof(data));
  return 0;
}
```

①

这一行表明有一个结构定义`user_msg_t`，用于保存一个 12 字符的消息。

②

BCC 宏`BPF_HASH`用于定义一个名为`config`的哈希表映射。它将保存`user_msg_t`类型的值，由`u32`类型的键索引，这是一个合适的用户 ID 大小。（如果您不指定键和值的类型，BCC 默认为两者都是`u64`。）

③

性能缓冲区输出的定义方式与第二章中完全相同。您可以向缓冲区提交任意数据，因此这里不需要指定任何数据类型...

④

...尽管在实践中，在这个例子中程序总是提交一个`data_t`结构。这也与第二章的例子没有变化。

⑤

大部分其他的 eBPF 程序与您之前看到的`hello()`版本没有变化。

// ⑥

唯一的区别是，使用了一个辅助函数来获取用户 ID 后，代码会查找`config`哈希映射中具有该用户 ID 的条目。如果有匹配的条目，该值包含一个消息，该消息将用于替代默认的“Hello World”。

Python 代码还有两行额外的内容：

```cpp
b["config"][ct.c_int(0)] = ct.create_string_buffer(b"Hey root!")
b["config"][ct.c_int(501)] = ct.create_string_buffer(b"Hi user 501!")
```

这些定义了`config`哈希表中用户 ID 为 0 和 501 的消息，对应于根用户和我在这个虚拟机上的用户 ID。这段代码使用 Python 的`ctypes`包来确保键和值与`user_msg_t`的 C 定义中使用的类型相同。

这是这个例子的一些说明性输出，以及我在第二个终端中运行的命令来获取它：

```cpp
Terminal 1                             Terminal 2
$ ./hello-buffer-config.py 
37926 501 bash Hi user 501!            ls 
37927 501 bash Hi user 501!            sudo ls
37929 0 sudo Hey root!
37931 501 bash Hi user 501!            sudo -u daemon ls
37933 1 sudo Hello World
```

现在您已经对这个程序做了解，我想向您展示运行时使用的`bpf()`系统调用。我将再次使用`strace`运行它，指定`-e bpf`以指示我只对`bpf()`系统调用感兴趣：

```cpp
$ strace -e bpf ./hello-buffer-config.py
```

如果您自己尝试这个例子，您将看到几次对这个系统调用的调用。对于每个调用，您将看到指示`bpf()`系统调用应该执行什么的命令。大致轮廓如下：

```cpp
bpf(BPF_BTF_LOAD, ...) = 3
bpf(BPF_MAP_CREATE, {map_type=BPF_MAP_TYPE_PERF_EVENT_ARRAY…) = 4
bpf(BPF_MAP_CREATE, {map_type=BPF_MAP_TYPE_HASH...) = 5
bpf(BPF_PROG_LOAD, {prog_type=BPF_PROG_TYPE_KPROBE,...prog_name="hello",...) = 6
bpf(BPF_MAP_UPDATE_ELEM, ...}
...
```

让我们逐一检查它们。你，读者，我都没有无限的耐心，所以我不会讨论每次调用的每个参数！我会专注于我认为真正有助于讲述用户空间程序与 eBPF 程序交互时发生的事情的部分。

# 加载 BTF 数据

我看到的第一个`bpf()`调用看起来像这样：

```cpp
bpf(BPF_BTF_LOAD, {btf="\237\353\1\0...}, 128) = 3
```

在这种情况下，您在输出中看到的命令是`BPF_BTF_LOAD`。这只是一组有效命令中的一个（至少在我写作时是如此），最全面的文档在内核源代码中进行了记录。¹

如果您使用的是相对较旧的 Linux 内核，可能不会看到带有这个命令的调用，因为它涉及到 BTF 或 BPF 类型格式。² BTF 允许 eBPF 程序在不同的内核版本之间可移植，这样您就可以在一台机器上编译程序，并在另一台机器上使用它，后者可能使用不同的内核版本，因此具有不同的内核数据结构。我将在第五章中更详细地讨论这一点。

这个`bpf()`调用正在将一块 BTF 数据加载到内核中，`bpf()`系统调用的返回代码（在我的例子中是 3）是一个引用该数据的文件描述符。

###### 注意

*文件描述符*是打开文件（或类似文件对象）的标识符。如果您打开一个文件（使用`open()`或`openat()`系统调用），返回的代码是一个文件描述符，然后将其作为参数传递给其他系统调用，如`read()`或`write()`，以执行对该文件的操作。这里的数据块并不完全是一个文件，但它被赋予了一个文件描述符作为标识符，可以用于将来引用它的操作。

# 创建映射

下一个`bpf()`创建`output`性能缓冲区映射：

```cpp
bpf(BPF_MAP_CREATE, {map_type=BPF_MAP_TYPE_PERF_EVENT_ARRAY, , key_size=4, 
value_size=4, max_entries=4, ... map_name="output", ...}, 128) = 4
```

您可能可以从命令名称`BPF_MAP_CREATE`猜到，这个调用创建了一个 eBPF 映射。您可以看到这个映射的类型是`PERF_EVENT_ARRAY`，它被称为`output`。这个 perf 事件映射中的键和值都是 4 个字节长。这个映射中还有一个限制，最多可以容纳四对键值对，由字段`max_entries`定义；我将在本章后面解释为什么这个映射中有四个条目。返回值`4`是用户空间代码访问`output`映射的文件描述符。

输出中的下一个`bpf()`系统调用创建了`config`映射：

```cpp
bpf(BPF_MAP_CREATE, {map_type=BPF_MAP_TYPE_HASH, key_size=4, value_size=12,
max_entries=10240... map_name="config", ...btf_fd=3,...}, 128) = 5
```

这个映射被定义为一个哈希表映射，其键长为 4 个字节（对应于可以用于保存用户 ID 的 32 位整数），值长为 12 个字节（与`msg_t`结构的长度相匹配）。我没有指定表的大小，所以它被赋予了 BCC 的默认大小，即 10240 个条目。

这个`bpf()`系统调用还返回一个文件描述符`5`，将用于将来的系统调用中引用这个`config`映射。

您还可以看到字段`btf_fd=3`，它告诉内核使用之前获取的 BTF 文件描述符`3`。正如您将在第五章中看到的，BTF 信息描述了数据结构的布局，并且在映射的定义中包含这些信息意味着有关该映射中使用的键和值类型的布局信息。这被像`bpftool`这样的工具用来漂亮地打印映射转储，使它们可读性强——您在第三章中看到了一个例子。

# 加载程序

到目前为止，您已经看到了使用系统调用将 BTF 数据加载到内核并创建一些 eBPF 映射的示例程序。它接下来要做的是使用以下`bpf()`系统调用将要加载到内核中的 eBPF 程序加载到内核中：

```cpp
bpf(BPF_PROG_LOAD, {prog_type=BPF_PROG_TYPE_KPROBE, insn_cnt=44,
insns=0xffffa836abe8, license="GPL", ... prog_name="hello", ... 
expected_attach_type=BPF_CGROUP_INET_INGRESS, prog_btf_fd=3,...}, 128) = 6
```

这里有很多有趣的字段：

+   `prog_type`字段描述了程序类型，这里表示它是要附加到 kprobe 上的。您将在第七章中了解更多关于程序类型的信息。

+   `insn_cnt`字段表示“指令计数”。这是程序中字节码指令的数量。

+   构成这个 eBPF 程序的字节码指令保存在`insns`字段中指定的地址中。

+   这个程序被指定为 GPL 许可，以便可以使用 GPL 许可的 BPF 辅助函数。

+   程序名称是`hello`。

+   `expected_attach_type`为`BPF_CGROUP_INET_INGRESS`可能看起来令人惊讶，因为这听起来像是与入口网络流量有关的事情，但是您知道这个 eBPF 程序将被附加到一个 kprobe 上。实际上，`expected_attach_type`字段只用于一些程序类型，而`BPF_PROG_TYPE_KPROBE`不是其中之一。`BPF_CGROUP_INET_INGRESS`只是 BPF 附加类型列表中的第一个，³所以它的值为`0`。

+   `prog_btf_fd`字段告诉内核使用先前加载的 BTF 数据块与这个程序一起使用。这里的值`3`对应于您从`BPF_BTF_LOAD`系统调用中看到的文件描述符（它是用于`config`映射的相同 BTF 数据块）。

如果程序未能通过验证（我将在第六章中讨论），这个系统调用将返回一个负值，但在这里您可以看到它返回了文件描述符 6。回顾一下，在这一点上，文件描述符具有表 4-1 中显示的含义。

表 4-1。加载程序后运行 hello-buffer-config.py 时的文件描述符

| 文件描述符 | 表示 |
| --- | --- |
| `3` | BTF 数据 |
| `4` | `output` perf 缓冲区映射 |
| `5` | `config`哈希表映射 |
| `6` | `hello` eBPF 程序 |

# 从用户空间修改映射

您已经在 Python 用户空间源代码中看到了配置特殊消息的行，这些消息将显示给用户 ID 0 的根用户和用户 ID 501 的用户：

```cpp
b["config"][ct.c_int(0)] = ct.create_string_buffer(b"Hey root!")
b["config"][ct.c_int(501)] = ct.create_string_buffer(b"Hi user 501!")
```

您可以通过这样的系统调用看到映射中定义的这些条目：

```cpp
bpf(BPF_MAP_UPDATE_ELEM, {map_fd=5, key=0xffffa7842490, value=0xffffa7a2b410,
flags=BPF_ANY}, 128) = 0
```

`BPF_MAP_UPDATE_ELEM` 命令更新映射中的键值对。`BPF_ANY` 标志表示，如果键在此映射中不存在，它将被创建。有两个这样的调用，对应于为两个不同的用户 ID 配置的两个条目。

`map_fd` 字段标识正在操作的映射。您可以看到在这种情况下它是 `5`，这是之前创建 `config` 映射时返回的文件描述符值。

文件描述符由内核为特定进程分配，因此 `5` 的值仅对 Python 程序运行的特定用户空间进程有效。然而，多个用户空间程序（和内核中的多个 eBPF 程序）都可以访问同一个映射。访问内核中相同映射结构的两个用户空间程序可能会被分配不同的文件描述符值；同样，两个用户空间程序可能对完全不同的映射具有相同的文件描述符值。

键和值都是指针，因此您无法从这个 `strace` 输出中得知键或值的数值。但是，您可以使用 `bpftool` 查看映射的内容，并看到类似这样的内容：

```cpp
$ bpftool map dump name config
[{
        "key": 0,
        "value": {
            "message": "Hey root!"
        }
    },{
        "key": 501,
        "value": {
            "message": "Hi user 501!"
        }
    }
]
```

`bpftool` 如何知道如何格式化这个输出？例如，它如何知道值是一个包含字符串的结构，名为 `message`？答案是它使用了在 `BPF_MAP_CREATE` 系统调用中包含的 BTF 信息中的定义来传达这些信息。您将在下一章中看到有关 BTF 如何传达这些信息的更多细节。

您已经看到了用户空间如何与内核交互以加载程序和映射，并更新映射中的信息。在您到目前为止看到的系统调用序列中，程序尚未附加到事件上。这一步必须发生；否则，程序将永远不会被触发。

公平警告：不同类型的 eBPF 程序以各种不同的方式附加到不同的事件上！本章后面我会向您展示在此示例中用于附加到 kprobe 事件的系统调用，而在这种情况下不涉及 `bpf()`。相比之下，在本章末尾的练习中，我将向您展示另一个示例，其中使用 `bpf()` 系统调用将程序附加到原始 tracepoint 事件。

在我们深入讨论这些细节之前，我想讨论一下当您停止运行程序时会发生什么。您会发现程序和映射会自动卸载，这是因为内核使用*引用计数*来跟踪它们。

# BPF 程序和映射引用

您知道使用 `bpf()` 系统调用将 BPF 程序加载到内核会返回一个文件描述符。在内核中，这个文件描述符是对程序的*引用*。进行系统调用的用户空间进程拥有这个文件描述符；当该进程退出时，文件描述符被释放，对程序的引用计数减少。当没有对 BPF 程序的引用时，内核会移除该程序。

将程序*固定*到文件系统时会创建额外的引用。

## 固定

您已经在第三章中看到了固定的实际操作，使用以下命令：

```cpp
bpftool prog load hello.bpf.o /sys/fs/bpf/hello
```

###### 注意

这些固定的对象并不是持久保存到磁盘的真实文件。它们是在*伪文件系统*上创建的，它的行为类似于常规的基于磁盘的文件系统，具有目录和文件。但它们保存在内存中，这意味着它们在系统重启后不会保留在原地。

如果`bpftool`允许您加载程序而不将其固定，那将是没有意义的，因为当`bpftool`退出时，文件描述符会被释放，如果引用计数为零，则程序将被删除，因此将无法实现任何有用的功能。但是将其固定到文件系统意味着程序有了额外的引用，因此在命令完成后程序仍然保持加载状态。

当将 BPF 程序附加到将触发它的钩子时，引用计数器也会递增。这些引用计数的行为取决于 BPF 程序类型。您将在第七章中了解更多关于这些程序类型的信息，但其中一些与跟踪相关（如 kprobes 和 tracepoints），并且始终与用户空间进程相关联；对于这些类型的 eBPF 程序，当该进程退出时，内核的引用计数会递减。在网络堆栈或 cgroups（“控制组”的缩写）中附加的程序不与任何用户空间进程相关联，因此即使加载它们的用户空间程序退出，它们也会保持在原地。当使用`ip link`命令加载 XDP 程序时，您已经看到了这种情况的一个示例：

```cpp
ip link set dev eth0 xdp obj hello.bpf.o sec xdp
```

`ip`命令已经完成，并且没有固定位置的定义，但是`bpftool`将向您显示 XDP 程序已加载到内核中：

```cpp
$ bpftool prog list
… 
1255: xdp  name hello  tag 9d0e949f89f1a82c  gpl
        loaded_at 2022-11-01T19:21:14+0000  uid 0
        xlated 48B  jited 108B  memlock 4096B  map_ids 612
```

该程序的引用计数为非零，因为它附加到`ip link`命令完成后仍然存在的 XDP 钩子上。

eBPF 映射也有引用计数器，当引用计数降至零时，它们将被清除。每个使用映射的 eBPF 程序都会增加计数器，用户空间程序可能持有的每个文件描述符也会增加计数器。

eBPF 程序的源代码可能定义一个映射，该程序实际上并不引用。假设您想要存储有关程序的一些元数据；您可以将其定义为全局变量，并且正如您在上一章中看到的那样，这些信息将存储在映射中。如果 eBPF 程序对该映射不执行任何操作，则程序到映射的引用计数不会自动增加。有一个`BPF(BPF_PROG_BIND_MAP)`系统调用，它将映射与程序关联起来，以便在用户空间加载程序退出并不再持有映射的文件描述符引用时，映射不会立即被清除。

映射也可以固定到文件系统，用户空间程序可以通过知道映射的路径来访问映射。

###### 注意

Alexei Starovoitov 在他的博客文章[“BPF 对象的生命周期”](https://oreil.ly/vofxH)中对 BPF 引用计数和文件描述符进行了很好的描述。

创建对 BPF 程序的引用的另一种方法是使用 BPF 链接。

## BPF 链接

BPF 链接为 eBPF 程序和其附加的事件之间提供了一层抽象。BPF 链接本身可以固定到文件系统，这会为程序创建一个额外的引用。这意味着将程序加载到内核中的用户空间进程可以终止，而程序仍然保持加载状态。用户空间加载程序的文件描述符被释放，减少了对程序的引用计数，但由于 BPF 链接的存在，引用计数仍将为非零。

如果您按照本章末尾的练习，您将有机会看到 BPF 链接的实际应用。现在，让我们回到*hello-buffer-config.py*使用的`bpf()`系统调用序列。

# eBPF 中涉及的其他系统调用

迄今为止，您已经看到了`bpf()`系统调用，该调用将 BTF 数据、程序和映射数据添加到内核中。`strace`输出显示的下一步是设置 perf 缓冲区。

###### 注意

本章的其余部分相对深入地介绍了使用性能缓冲区、环形缓冲区、kprobes 和映射迭代时涉及的系统调用序列。并非所有的 eBPF 程序都需要执行这些操作，因此如果您赶时间或者觉得太详细，可以随意跳到章节摘要。我不会生气的！

## 初始化性能缓冲区

您已经看到了将条目添加到“config”映射的“bpf（BPF_MAP_UPDATE_ELEM）”调用。接下来，输出显示了一些看起来像这样的调用：

```cpp
bpf(BPF_MAP_UPDATE_ELEM, {map_fd=4, key=0xffffa7842490, value=0xffffa7a2b410,
flags=BPF_ANY}, 128) = 0
```

这些看起来与定义“config”映射条目的调用非常相似，只是在这种情况下，映射的文件描述符是“4”，代表“output”性能缓冲区映射。

与以前一样，键和值都是指针，因此您无法从这个“strace”输出中知道键或值的数值。我看到这个系统调用重复了四次，所有参数的值都相同，尽管无法知道指针保存的值在每次调用之间是否发生了变化。查看这些“BPF_MAP_UPDATE_ELEM bpf（）”调用留下了一些关于如何设置和使用缓冲区的未解答问题：

+   为什么有四次调用“BPF_MAP_UPDATE_ELEM”？这是否与“output”映射创建了最多四个条目有关？

+   在这四个“BPF_MAP_UPDATE_ELEM”实例之后，“strace”输出中不再出现“bpf（）”系统调用。这可能看起来有点奇怪，因为映射是为了让 eBPF 程序在每次触发时写入数据，而您已经看到用户空间代码显示了数据。显然，这些数据并不是通过“bpf（）”系统调用从映射中获取的，那么它是如何获取的呢？

您还没有看到任何证据表明 eBPF 程序是如何附加到触发它的 kprobe 事件的。为了解释所有这些问题，我需要“strace”在运行此示例时显示更多的系统调用，就像这样：

```cpp
$ strace -e bpf,perf_event_open,ioctl,ppoll ./hello-buffer-config.py
```

为简洁起见，我将忽略与本示例的 eBPF 功能无关的“ioctl（）”调用。

## 附加到 Kprobe 事件

您已经看到文件描述符 6 被分配为表示 eBPF 程序* hello*一旦加载到内核中。要将 eBPF 程序附加到事件，您还需要一个表示特定事件的文件描述符。来自“strace”输出的以下行显示了为“execve（）”kprobe 创建文件描述符：

```cpp
perf_event_open({type=0x6 /* PERF_TYPE_??? */, ...},...) = 7
```

根据[“perf_event_open（）系统调用的 manpage”](https://oreil.ly/xpRJs)，它“创建一个允许测量性能信息的文件描述符。”您可以从输出中看到，“strace”不知道如何解释值为“6”的类型参数，但是如果您进一步查看该 manpage，它描述了 Linux 如何支持动态类型的性能测量单元：

> …在*/sys/bus/event_source/devices*下每个 PMU 实例都有一个子目录。在每个子目录中，都有一个类型文件，其内容是可以在类型字段中使用的整数。

如果您查看该目录，您会发现一个*kprobe/type*文件：

```cpp
$ cat /sys/bus/event_source/devices/kprobe/type
6
```

从中可以看出，“perf_event_open（）”的调用类型设置为值“6”，表示这是一种 kprobe 类型的性能事件。

不幸的是，“strace”没有输出细节，可以明确显示 kprobe 是否附加到“execve（）”系统调用，但我希望这里有足够的证据来说服你，这就是这里返回的文件描述符代表的内容。

“perf_event_open（）”的返回代码是“7”，这代表了 kprobe 的性能事件的文件描述符，您知道文件描述符“6”代表了*hello* eBPF 程序。 “perf_event_open（）”的 manpage 还解释了如何使用“ioctl（）”在两者之间创建附件：

> `PERF_EVENT_IOC_SET_BPF` [...] 允许将伯克利数据包过滤器（BPF）程序附加到现有的 kprobe 跟踪点事件上。参数是之前由`bpf(2)`系统调用创建的 BPF 程序文件描述符。

这解释了您在`strace`输出中看到的以下`ioctl()`系统调用，其参数指的是两个文件描述符：

```cpp
ioctl(7, PERF_EVENT_IOC_SET_BPF, 6)     = 0
```

还有另一个`ioctl()`调用，用于打开 kprobe 事件：

```cpp
ioctl(7, PERF_EVENT_IOC_ENABLE, 0)      = 0
```

有了这个，只要在这台机器上运行`execve()`，eBPF 程序就会被触发。

## 设置和读取性能事件

我已经提到我看到了四次与输出性能缓冲区相关的`bpf(BPF_MAP_UPDATE_ELEM)`调用。随着额外的系统调用被跟踪，`strace`输出显示了四个类似这样的序列：

```cpp
perf_event_open({type=PERF_TYPE_SOFTWARE, size=0 /* PERF_ATTR_SIZE_??? */, 
config=PERF_COUNT_SW_BPF_OUTPUT, ...}, -1, X, -1, PERF_FLAG_FD_CLOEXEC) = Y

ioctl(Y, PERF_EVENT_IOC_ENABLE, 0)      = 0

bpf(BPF_MAP_UPDATE_ELEM, {map_fd=4, key=0xffffa7842490, value=0xffffa7a2b410,
flags=BPF_ANY}, 128) = 0
```

我使用`X`表示输出中的四个实例中的值`0`，`1`，`2`和`3`。参考`perf_event_open()`系统调用的 manpage，您会发现这是`cpu`，在它之前的字段是`pid`或进程 ID。来自 manpage 的内容：

> pid == -1 and cpu >= 0

>

> 这会测量指定 CPU 上的所有进程/线程。

这个序列发生四次的事实对应于我的笔记本电脑上有四个 CPU 核心。这最终解释了“输出”性能缓冲区映射中有四个条目的原因：每个 CPU 核心都有一个。这也解释了映射类型名称`BPF_MAP_TYPE_PERF_EVENT_ARRAY`中的“数组”部分，因为该映射不仅代表一个性能环形缓冲区，还代表一个缓冲区数组，每个核心一个。

如果您编写 eBPF 程序，您不需要担心处理核心数量等细节，因为这将由第十章中讨论的任何 eBPF 库为您处理，但我认为这是您在使用`strace`时看到的系统调用的一个有趣方面。

每个`perf_event_open()`调用都会返回一个文件描述符，我用`Y`表示；这些值分别为`8`，`9`，`10`和`11`。`ioctl()`系统调用为这些文件描述符中的每一个启用了性能输出。`BPF_MAP_UPDATE_ELEM bpf()`系统调用设置了映射条目，指向每个 CPU 核心的性能环形缓冲区，以指示它可以提交数据的位置。

用户空间代码可以在所有四个输出流文件描述符上使用`ppoll()`，以便它可以获取数据输出，无论哪个核心运行 eBPF 程序*hello*来执行任何给定的`execve()` kprobe 事件。这是对`ppoll()`的系统调用：

```cpp
ppoll([{fd=8, events=POLLIN}, {fd=9, events=POLLIN}, {fd=10, events=POLLIN},
{fd=11, events=POLLIN}], 4, NULL, NULL, 0) = 1 ([{fd=8, revents=POLLIN}])
```

如果您尝试自己运行示例程序，您会发现这些`ppoll()`调用会阻塞，直到有数据从其中一个文件描述符中读取出来。直到触发`execve()`，才会看到返回代码写入屏幕，这会导致 eBPF 程序写入数据，用户空间使用`ppoll()`调用检索数据。

在第二章中，我提到如果您的内核版本为 5.8 或更高，现在更倾向于使用 BPF 环形缓冲区而不是性能缓冲区。⁴ 让我们看一下使用环形缓冲区的相同示例代码的修改版本。

# 环形缓冲区

如[内核文档](https://oreil.ly/RN_RA)中所讨论的，环形缓冲区比性能缓冲区更受欢迎，部分原因是性能，但也是为了确保数据的顺序被保留，即使数据由不同的 CPU 核心提交。只有一个缓冲区，跨所有核心共享。

将*hello-buffer-config.py*转换为使用环形缓冲区并不需要太多更改。在附带的 GitHub 存储库中，您会发现这个示例作为*chapter4/hello-ring-buffer-config.py*。表 4-2 显示了差异。

表 4-2。使用性能缓冲区和环形缓冲区的示例 BCC 代码之间的差异

| *hello-buffer-config.py* | *hello-ring-buffer-config.py* |
| --- | --- |
| `BPF_PERF_OUTPUT(output);` | `BPF_RINGBUF_OUTPUT(output, 1);` |
| `output.perf_submit(ctx, &data, sizeof(data));` | `output.ringbuf_output(&data, sizeof(data), 0);` |
| `b["output"]. open_perf_buffer(print_event)` | `b["output"]. open_ring_buffer(print_event)` |
| `b.perf_buffer_poll()` | `b.ring_buffer_poll()` |

正如您所期望的那样，由于这些更改仅涉及 `output` 缓冲区，因此与加载程序和 `config` 映射以及将程序附加到 kprobe 事件相关的系统调保持不变。

创建 `output` 环形缓冲映射的 `bpf()` 系统调看起来是这样的：

```cpp
bpf(BPF_MAP_CREATE, {map_type=BPF_MAP_TYPE_RINGBUF, key_size=0, value_size=0,
max_entries=4096, ... map_name="output", ...}, 128) = 4
```

`strace` 输出中的主要差异在于，在设置 perf 缓冲区时，没有出现您在一系列四个不同的 `perf_event_open()`、`ioctl()` 和 `bpf(BPF_MAP_UPDATE_ELEM)` 系统调期间观察到的情况。对于环形缓冲区，只有一个文件描述符在所有 CPU 核心之间共享。

在撰写本文时，BCC 使用了我之前展示的 perf 缓冲区的 `ppoll` 机制，但它使用了更新的 `epoll` 机制来等待环形缓冲区中的数据。让我们利用这个机会来了解 `ppoll` 和 `epoll` 之间的区别。

在 perf 缓冲区示例中，我展示了 *hello-buffer-config.py* 生成了一个 `ppoll()` 系统调，如下所示： 

```cpp
ppoll([{fd=8, events=POLLIN}, {fd=9, events=POLLIN}, {fd=10, events=POLLIN},
{fd=11, events=POLLIN}], 4, NULL, NULL, 0) = 1 ([{fd=8, revents=POLLIN}])
```

请注意，这会传入文件描述符集 `8`、`9`、`10` 和 `11`，用户空间进程希望从中检索数据。每次此轮询事件返回数据时，都必须再次调用 `ppoll()` 来设置相同的文件描述符集。使用 `epoll` 时，文件描述符集在内核对象中进行管理。

您可以在 *hello-ring-buffer-config.py* 设置对 `output` 环形缓冲区的访问时所做的一系列 `epoll` 相关系统调中看到这一点。

首先，用户空间程序要求在内核中创建一个新的 `epoll` 实例：

```cpp
epoll_create1(EPOLL_CLOEXEC) = 8
```

这返回文件描述符 `8`。然后调用 `epoll_ctl()`，告诉内核将文件描述符 `4`（`output` 缓冲区）添加到该 `epoll` 实例的文件描述符集中：

```cpp
epoll_ctl(8, EPOLL_CTL_ADD, 4, {events=EPOLLIN, data={u32=0, u64=0}}) = 0
```

用户空间程序使用 `epoll_pwait()` 等待环形缓冲区中有数据可用。只有在有数据可用时，此调用才会返回：

```cpp
epoll_pwait(8,  [{events=EPOLLIN, data={u32=0, u64=0}}], 1, -1, NULL, 8) = 1
```

当然，如果您正在使用像 BCC（或 *libbpf* 或我稍后在本书中描述的任何其他库）这样的框架编写代码，您实际上不需要了解有关用户空间应用程序如何通过 perf 或环形缓冲区从内核获取信息的这些基础细节。我希望您发现了解这些工作原理的底层细节很有趣。

但是，您可能会发现自己编写的代码访问了用户空间的映射，并且看到如何实现这一点可能会有所帮助。在本章的前面，我使用 `bpftool` 检查了 `config` 映射的内容。由于它是在用户空间运行的实用程序，让我们使用 `strace` 来查看它用于检索此信息的系统调用。

# 从映射中读取信息

以下命令显示了 `bpftool` 在读取 `config` 映射内容时所做的 `bpf()` 系统调的摘录：

```cpp
$ strace -e bpf bpftool map dump name config
```

正如您将看到的，该序列由两个主要步骤组成：

+   遍历所有映射，查找其中名称为 `config` 的映射。

+   如果找到匹配的映射，则遍历该映射中的所有元素。

## 查找映射

输出以重复的类似调用序列开始，因为 `bpftool` 遍历所有映射，查找其中名称为 `config` 的映射：

```cpp
bpf(BPF_MAP_GET_NEXT_ID, {start_id=0,...}, 12) = 0             // ①
bpf(BPF_MAP_GET_FD_BY_ID, {map_id=48...}, 12) = 3              // ②
bpf(BPF_OBJ_GET_INFO_BY_FD, {info={bpf_fd=3, ...}}, 16) = 0    // ③

bpf(BPF_MAP_GET_NEXT_ID, {start_id=48, ...}, 12) = 0           // ④
bpf(BPF_MAP_GET_FD_BY_ID, {map_id=116, ...}, 12) = 3
bpf(BPF_OBJ_GET_INFO_BY_FD, {info={bpf_fd=3...}}, 16) = 0
```

①

`BPF_MAP_GET_NEXT_ID` 获取指定 `start_id` 后的下一个映射的 ID。

②

`BPF_MAP_GET_FD_BY_ID` 返回指定映射 ID 的文件描述符。

③

`BPF_OBJ_GET_INFO_BY_FD` 通过文件描述符检索有关对象（在本例中为映射）的信息。此信息包括其名称，因此 `bpftool` 可以检查这是否是它正在寻找的映射。

④

该序列重复，获取步骤 1 中的映射之后的下一个映射的 ID。

对于加载到内核中的每个映射，都有一组这三个系统调用，您还应该看到`start_id`和`map_id`的值与这些映射的 ID 匹配。当没有更多的映射可供查看时，重复的模式结束，导致`BPF_MAP_GET_NEXT_ID`返回`ENOENT`的值，如下所示：

```cpp
bpf(BPF_MAP_GET_NEXT_ID, {start_id=133,...}, 12) = -1 ENOENT (No such file or
directory)
```

如果找到匹配的映射，`bpftool`将保持其文件描述符，以便可以从该映射中读取元素。

## 读取映射元素

此时，`bpftool`具有对要从中读取的映射的文件描述符引用。让我们看一下读取该信息的系统调用序列：

```cpp
bpf(BPF_MAP_GET_NEXT_KEY, {map_fd=3, key=NULL,                    // ①
next_key=0xaaaaf7a63960}, 24) = 0
bpf(BPF_MAP_LOOKUP_ELEM, {map_fd=3, key=0xaaaaf7a63960,           // ②
value=0xaaaaf7a63980, flags=BPF_ANY}, 32) = 0
[{                                                                // ③
        "key": 0,
        "value": {
            "message": "Hey root!"
        }
bpf(BPF_MAP_GET_NEXT_KEY, {map_fd=3, key=0xaaaaf7a63960,          // ④
next_key=0xaaaaf7a63960}, 24) = 0
bpf(BPF_MAP_LOOKUP_ELEM, {map_fd=3, key=0xaaaaf7a63960, 
value=0xaaaaf7a63980, flags=BPF_ANY}, 32) = 0
    },{                                                   
        "key": 501,
        "value": {
            "message": "Hi user 501!"
        }
bpf(BPF_MAP_GET_NEXT_KEY, {map_fd=3, key=0xaaaaf7a63960,          // ⑤
next_key=0xaaaaf7a63960}, 24) = -1 ENOENT (No such file or directory)
    }                                                             // ⑥
]
+++ exited with 0 +++
```

①

首先，应用程序需要找到映射中存在的有效键。它使用`bpf()`系统调用的`BPF_MAP_GET_NEXT_KEY`来实现这一点。`key`参数是一个指向键的指针，系统调用将返回此键之后的下一个有效键。通过传递一个空指针，应用程序请求映射中的第一个有效键。内核将键写入`next_key`指针指定的位置。

②

给定一个键，应用程序请求关联的值，该值被写入由`value`指定的内存位置。

③

此时，`bpftool`具有第一个键-值对的内容，并将此信息写入屏幕。

④

在这里，`bpftool`继续到映射中的下一个键，检索其值，并将此键-值对写入屏幕。

⑤

对`BPF_MAP_GET_NEXT_KEY`的下一次调用返回`ENOENT`，表示映射中没有更多的条目。

// ⑥

在这里，`bpftool`完成了写入屏幕的输出并退出。

请注意，这里`bpftool`已被分配文件描述符`3`，以对应`config`映射。这与*hello-buffer-config.py*中使用文件描述符`4`引用的相同映射。正如我已经提到的，文件描述符是进程特定的。

对`bpftool`行为的分析显示了用户空间程序如何遍历可用的映射和存储在映射中的键-值对。

# 总结

在本章中，您看到了用户空间代码如何使用`bpf()`系统调用来加载 eBPF 程序和映射。您看到了使用`BPF_PROG_LOAD`和`BPF_MAP_CREATE`命令创建程序和映射。

您了解到内核跟踪对 eBPF 程序和映射的引用计数，并在引用计数降至零时释放它们。您还介绍了将 BPF 对象固定到文件系统并使用 BPF 链接创建附加引用的概念。

您看到了`BPF_MAP_UPDATE_ELEM`的一个示例，用于从用户空间创建映射中的条目。还有类似的命令——`BPF_MAP_LOOKUP_ELEM`和`BPF_MAP_DELETE_ELEM`——用于从映射中检索和删除值。还有一个命令`BPF_MAP_GET_NEXT_KEY`，用于查找映射中存在的下一个键。您可以使用此命令来遍历所有有效的条目。

您看到了用户空间程序如何使用`perf_event_open()`和`ioctl()`来将 eBPF 程序附加到 kprobe 事件。对于其他类型的 eBPF 程序，附加方法可能会有很大不同，其中一些甚至使用`bpf()`系统调用。例如，有一个`bpf(BPF_PROG_ATTACH)`系统调用，可用于附加 cgroup 程序，以及`bpf(BPF_RAW_TRACEPOINT_OPEN)`用于原始跟踪点（请参见本章末尾的练习 5）。

我还展示了如何使用`BPF_MAP_GET_NEXT_ID`、`BPF_MAP_GET_FD_BY_ID`和`BPF_OBJ_GET_INFO_BY_FD`来定位内核中保存的映射（和其他）对象。

还有一些其他`bpf()`命令我在本章中没有涵盖，但是您在这里看到的已经足够了解一个好的概述。

您还看到一些 BTF 数据被加载到内核中，并且我提到`bpftool`使用此信息来了解数据结构的格式，以便可以将其打印出来。我还没有解释 BTF 数据的外观以及如何使用它使 eBPF 程序在内核版本之间可移植。这将在下一章中介绍。

# 练习

如果您想进一步探索`bpf()`系统调用，可以尝试以下几件事：

1.  确认`BPF_PROG_LOAD`系统调用的`insn_cnt`字段是否对应于使用`bpftool`转储该程序的翻译 eBPF 字节码的指令数量。（这在[“bpf()”系统调用的 manpage](https://oreil.ly/NJdIM)中有记录。）

1.  运行两个示例程序的实例，以便有两个名为`config`的映射。如果运行`bpftool map dump name config`，输出将包括有关两个不同映射以及其内容的信息。在`strace`下运行此命令，并通过系统调用输出跟踪不同文件描述符的使用。您能看到它在哪里检索有关映射的信息以及在哪里检索其中存储的键-值对吗？

1.  在运行示例程序之一时使用`bpftool map update`修改`config`映射。使用`sudo -u username`检查 eBPF 程序是否捕获到这些配置更改。

1.  在*hello-buffer-config.py*运行时，使用`bpftool`将程序固定到 BPF 文件系统，如下所示：

```cpp
    bpftool prog pin name hello /sys/fs/bpf/hi
    ```

退出运行的程序，并使用`bpftool prog list`检查内核中是否仍然加载了*hello*程序。您可以通过使用`rm /sys/fs/bpf/hi`删除引脚来清理链接。

1.  在系统调用级别，与附加到 kprobe 相比，附加到原始跟踪点要简单得多，因为它只涉及一个`bpf()`系统调用。尝试将*hello-buffer-config.py*转换为附加到`sys_enter`的原始跟踪点，使用 BCC 的`RAW_TRACEPOINT_PROBE`宏（如果您在第二章中做了练习，您已经有一个适合的程序可以使用）。您不需要在 Python 代码中显式附加程序，因为 BCC 会为您处理。在`strace`下运行此命令，您应该会看到类似于这样的系统调用：

```cpp
    bpf(BPF_RAW_TRACEPOINT_OPEN, {raw_tracepoint={name="sys_enter",
    prog_fd=6}}, 128) = 7
    ```

内核中的跟踪点的名称为`sys_enter`，并且具有文件描述符`6`的 eBPF 程序被附加到其中。从现在开始，每当内核中的执行到达该跟踪点时，它将触发 eBPF 程序。

1.  从[BCC 的*libbpf 工具*](https://oreil.ly/D31R4)中运行 opensnoop 应用程序。此工具设置了一些 BPF 链接，您可以使用`bpftool`查看，如下所示：

```cpp
    $ bpftool link list
    116: perf_event  prog 1849  
            bpf_cookie 0
            pids opensnoop(17711)
    117: perf_event  prog 1851  
            bpf_cookie 0
            pids opensnoop(17711)
    ```

确认程序 ID（在我的示例输出中为 1849 和 1851）是否与列出的已加载 eBPF 程序的输出匹配：

```cpp
    $ bpftool prog list
    ...
    1849: tracepoint  name tracepoint__syscalls__sys_enter_openat
            tag 8ee3432dcd98ffc3  gpl run_time_ns 95875 run_cnt 121
            loaded_at 2023-01-08T15:49:54+0000  uid 0
            xlated 240B  jited 264B  memlock 4096B  map_ids 571,568
            btf_id 710
            pids opensnoop(17711)
    1851: tracepoint  name tracepoint__syscalls__sys_exit_openat
            tag 387291c2fb839ac6  gpl run_time_ns 8515669 run_cnt 120
            loaded_at 2023-01-08T15:49:54+0000  uid 0
            xlated 696B  jited 744B  memlock 4096B  map_ids 568,571,569
            btf_id 710
            pids opensnoop(17711)
    ```

1.  在 opensnoop 运行时，尝试使用`bpftool link pin id 116 /sys/fs/bpf/mylink`固定其中一个链接（使用您从`bpftool link list`输出的链接 ID 之一）。您应该看到即使在终止 opensnoop 之后，链接和相应的程序仍然加载在内核中。

1.  如果您跳到第五章的示例代码，您将找到使用*libbpf*库编写的*hello-buffer-config.py*的版本。此库会自动设置 BPF 链接到加载到内核中的程序。使用`strace`检查它所做的`bpf()`系统调用，并查看`bpf(BPF_LINK_CREATE)`系统调用。

1 如果您想查看完整的 BPF 命令集，可以在*linux/bpf.h*头文件中找到文档。

2 BTF 在 5.1 内核中引入，但已经在一些 Linux 发行版上进行了回溯，您可以从[此讨论](https://oreil.ly/LjcPN)中看到。

³ 这些在*linux/bpf.h*中的`bpf_attach_type`枚举器中定义。

⁴ 提醒您，要了解更多关于差异的信息，请阅读 Andrii Nakryiko 的“BPF 环形缓冲区”博客文章。
