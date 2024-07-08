# 第二章：eBPF 的“Hello World”

在前一章中，我讨论了为什么 eBPF 如此强大，但如果您还没有对运行 eBPF 程序的真正含义有一个具体的理解，这也是可以的。在本章中，我将使用一个简单的“Hello World”示例来让您更好地理解它。

正如您在阅读本书时将会了解的那样，有几种不同的库和框架可用于编写 eBPF 应用程序。作为一个热身，我将向您展示可能是从编程角度来看最容易的方法：[BCC Python 框架](https://github.com/iovisor/bcc)。这提供了一个非常简单的方式来编写基本的 eBPF 程序。出于我将在第五章中讨论的原因，这不一定是我建议今天为打算分发给其他用户的生产应用程序选择的方法，但对于初学者来说非常适合。

###### 注意

如果您想尝试这段代码，可以在[*https://github.com/lizrice/learning-ebpf*](https://github.com/lizrice/learning-ebpf)的*chapter2*目录中找到它。

您将在[*https://github.com/iovisor/bcc*](https://github.com/iovisor/bcc)找到 BCC 项目，并且安装 BCC 的说明在[*https://github.com/iovisor/bcc/blob/master/INSTALL.md*](https://github.com/iovisor/bcc/blob/master/INSTALL.md)。

# BCC 的“Hello World”

下面是使用 BCC 的 Python 库编写的 eBPF“Hello World”应用程序^(1)的完整源代码*hello.py*：

```
#!/usr/bin/python 
from bcc import BPF

program = r"""
int hello(void *ctx) {
 bpf_trace_printk("Hello World!");
 return 0;
}
"""

b = BPF(text=program)
syscall = b.get_syscall_fnname("execve")
b.attach_kprobe(event=syscall, fn_name="hello")

b.trace_print()
```

该代码包括两部分：将在内核中运行的 eBPF 程序本身，以及将 eBPF 程序加载到内核并读取其生成的跟踪的一些用户空间代码。正如您在图 2-1 中所看到的，*hello.py*是此应用程序的用户空间部分，`hello()`是运行在内核中的 eBPF 程序。

![“Hello World”的用户空间和内核组件](img/lebp_0201.png)

###### 图 2-1\. “Hello World”的用户空间和内核组件

让我们逐行分析源代码，以更好地理解它。

第一行告诉您这是 Python 代码，可以运行它的程序是 Python 解释器（*/usr/bin/python*）。

eBPF 程序本身是用 C 代码编写的，它就是这部分：

```
int hello(void *ctx) {
    bpf_trace_printk("Hello World!");
    return 0;
}
```

所有 eBPF 程序要做的就是使用一个辅助函数`bpf_trace_printk()`来写入消息。辅助函数是区分其“经典”前身的“扩展”BPF 的另一个特征。它们是 eBPF 程序可以调用以与系统交互的一组函数；我将在第五章进一步讨论它们。现在你可以简单地把它看作是打印一行文本。

整个 eBPF 程序在 Python 代码中定义为一个名为`program`的字符串。这个 C 程序在执行之前需要编译，但是 BCC 会为你处理这一切。（在下一章中，你将看到如何自己编译 eBPF 程序。）你只需要在创建 BPF 对象时将这个字符串作为参数传递即可，如下一行所示：

```
b = BPF(text=program)
```

eBPF 程序需要附加到一个事件上，例如我选择了附加到系统调用`execve`上，这是用于执行程序的系统调用。无论在这台机器上的任何事物或任何人启动新程序执行，都将调用`execve()`，这将触发 eBPF 程序。尽管`execve()`在 Linux 中是一个标准接口名称，但实现它的内核函数名称取决于芯片架构，但 BCC 为我们提供了一种便捷的方法来查找正在运行的机器的函数名称：

```
syscall = b.get_syscall_fnname("execve")
```

现在，`syscall`表示我要附加到的内核函数的名称，我将使用 kprobe（你在第一章中已经介绍过 kprobe 的概念）。^(2)你可以像这样将`hello`函数附加到该事件上：

```
b.attach_kprobe(event=syscall, fn_name="hello")
```

到此为止，eBPF 程序已加载到内核并附加到一个事件上，因此每当在机器上启动新的可执行程序时，程序就会被触发。在 Python 代码中，唯一剩下的就是读取内核输出的跟踪信息并将其显示在屏幕上：

```
b.trace_print()
```

此`trace_print()`函数将无限循环（直到你用 Ctrl+C 停止程序），显示任何跟踪信息。

图 2-2 展示了这段代码。Python 程序编译了 C 代码，将其加载到内核，并将其附加到`execve`系统调用的 kprobe 上。每当这台（虚拟）机器上的任何应用程序调用`execve()`时，它都会触发 eBPF 的`hello()`程序，后者将跟踪消息写入特定的伪文件中。（稍后在本章中我将介绍该伪文件的位置。）Python 程序从伪文件中读取跟踪消息并将其显示给用户。

![“Hello World”的操作](img/lebp_0202.png)

###### 图 2-2. “Hello World”的操作

# 运行“Hello World”

运行这个程序，根据你使用的（虚拟）机器上正在发生的情况，你可能会立即看到生成的跟踪，因为其他进程可能会使用`execve`系统调用执行程序^(3)。如果你没有看到任何内容，请打开第二个终端并执行任何你喜欢的命令^(4)，你将看到由“Hello World”生成的相应跟踪：

```
$ hello.py
b'     bash-5412    [001] .... 90432.904952: 0: bpf_trace_printk: Hello World'
```

###### 注意

由于 eBPF 非常强大，使用它需要特殊权限。特权会自动分配给 root 用户，所以以 root 用户身份运行 eBPF 程序是最简单的方式，可以使用 `sudo`。为了清晰起见，在本书的示例命令中我不会包括 `sudo`，但是如果你看到“操作不允许”错误，首先要检查的是你是否试图以非特权用户身份运行 eBPF 程序。

`CAP_BPF` 是在内核版本 5.8 中引入的，它允许执行一些 eBPF 操作，如创建特定类型的映射。但是，你可能需要额外的特权：

+   `CAP_PERFMON` 和 `CAP_BPF` 都是加载跟踪程序所需的权限。

+   `CAP_NET_ADMIN` 和 `CAP_BPF` 都是加载网络程序所需的权限。

在 Milan Landaverde 的博客文章 [“Introduction to CAP_BPF”](https://oreil.ly/G2zFO) 中有更详细的信息。

一旦 *hello* eBPF 程序被加载并附加到一个事件上，它就会被从已存在进程中生成的事件触发。这应该强化你在 第一章 中学到的一些要点：

+   eBPF 程序可以动态改变系统的行为。不需要重新启动机器或者重新启动现有进程。一旦 eBPF 代码附加到事件上，它就会立即开始生效。

+   不需要改变其他应用程序的任何内容，它们就能被 eBPF 看到。只要你在该机器上有终端访问权限，在其中运行可执行文件时，它将使用 `execve()` 系统调用；如果你将 *hello* 程序附加到该系统调用上，它将被触发以生成跟踪输出。同样，如果你有一个运行可执行文件的脚本，那也会触发 *hello* eBPF 程序。你不需要改变终端的 shell、脚本或者你运行的可执行文件。

跟踪输出不仅显示了 `"Hello World"` 字符串，还显示了触发 *hello* eBPF 程序运行的事件的一些额外上下文信息。在本节开头显示的示例输出中，执行 `execve` 系统调用的进程的进程 ID 是 5412，并且正在运行 `bash` 命令。对于跟踪消息，这些上下文信息是作为内核跟踪基础设施的一部分添加的（这并不特定于 eBPF），但是后面你将看到，在 eBPF 程序内部也可以检索到类似的上下文信息。

也许你想知道 Python 代码是如何知道从哪里读取跟踪输出的。答案并不复杂——内核中的 `bpf_trace_printk()` 辅助函数始终将输出发送到同一预定义的伪文件位置：*/sys/kernel/debug/tracing/trace_pipe*。你可以使用 `cat` 命令查看其内容；需要 root 权限来访问它。

对于简单的“Hello World”示例或基本的调试目的，单个跟踪管道位置就足够了，但其功能非常有限。输出格式的灵活性非常小，仅支持字符串输出，因此对于传递结构化信息并不是非常有用。也许最重要的是，在（虚拟）机器上只有这一个位置。如果同时运行多个 eBPF 程序，它们都会将跟踪输出写入同一个跟踪管道，这对于人类操作员来说可能会非常混乱。

从 eBPF 程序中获取信息的更好方法是使用 eBPF 映射。

# BPF 映射

*映射*是一种数据结构，可以从 eBPF 程序和用户空间访问。映射是区分扩展 BPF 和其经典前身的一项非常重要的特性之一。（你可能会认为这意味着它们通常被称为“eBPF 映射”，但你经常会看到“BPF 映射”的用法。通常情况下，这两个术语可以互换使用。）

映射可用于在多个 eBPF 程序之间共享数据，或在用户空间应用程序与运行在内核中的 eBPF 代码之间进行通信。典型用途包括以下内容：

+   用户空间编写配置信息以便由 eBPF 程序检索

+   一个 eBPF 程序存储状态，以便稍后由另一个 eBPF 程序检索（或同一个程序的未来运行）

+   一个 eBPF 程序将结果或指标写入一个映射，供用户空间应用程序检索并展示结果

在 Linux 的[*uapi/linux/bpf.h*文件](https://oreil.ly/1s1GM)中定义了各种类型的 BPF 映射，并且[内核文档](https://oreil.ly/5oUW7)中也提供了一些相关信息。总体而言，它们都是键-值存储，并且在本章中你将看到用于哈希表、perf 和环形缓冲区以及 eBPF 程序数组的映射示例。

一些映射类型被定义为数组，其键类型始终为 4 字节索引；而其他映射则是可以使用任意数据类型作为键的哈希表。

有一些映射类型经过优化，用于特定类型的操作，比如[先进先出队列](https://oreil.ly/VSoEp)，[后进先出栈](https://oreil.ly/VSoEp)，[最近最少使用数据存储](https://oreil.ly/vpsun)，[最长前缀匹配](https://oreil.ly/hZ5aM)，以及[Bloom 过滤器](https://oreil.ly/DzCTK)（一种概率数据结构，旨在快速确定元素是否存在）。

一些 eBPF 映射类型保存特定类型对象的信息。例如，[sockmaps](https://oreil.ly/UUTHO) 和 [devmaps](https://oreil.ly/jzKYh) 保存套接字和网络设备的信息，被网络相关的 eBPF 程序用来重定向流量。程序数组映射存储一组索引化的 eBPF 程序，（后面你会看到）用于实现尾调用，其中一个程序可以调用另一个。甚至还有[映射类型的映射](https://oreil.ly/038tN)支持存储关于映射的信息。

有些映射类型有每个 CPU 变体，也就是说内核为每个 CPU 核心的该映射版本使用不同的内存块。这可能会让你担心那些不是每个 CPU 的映射，多个 CPU 核心同时访问同一映射的并发问题。内核版本 5.1 添加了对（某些）映射的自旋锁支持，我们将在第五章中回到这个主题。

下一个示例（在[GitHub 仓库](https://github.com/lizrice/learning-ebpf)中的*chapter2/hello-map.py*）展示了使用散列表映射的一些基本操作。它还展示了一些 BCC 提供的便捷抽象，使得使用映射变得非常容易。

## 散列表映射

与本章中前面的示例类似，这个 eBPF 程序将附加到 `execve` 系统调用的入口处。它将使用键-值对填充一个散列表，其中键是用户 ID，值是由该用户 ID 下的进程调用 `execve` 的次数计数器。在实际操作中，这个示例将展示每个不同用户运行程序的次数。

首先，让我们看看 eBPF 程序本身的 C 代码：

```
BPF_HASH(counter_table);                                     ![1](img/1.png)

int hello(void *ctx) {
  u64 uid;                                                  
  u64 counter = 0;
  u64 *p;

  uid = bpf_get_current_uid_gid() & 0xFFFFFFFF;              ![2](img/2.png)
  p = counter_table.lookup(&uid);                            ![3](img/3.png)
  if (p != 0) {                                              ![4](img/4.png)
     counter = *p;
  }
  counter++;                                                 ![5](img/5.png)
  counter_table.update(&uid, &counter);                      ![6](img/6.png)
  return 0;
}
```

![1](img/#code_id_2_1)

`BPF_HASH()` 是一个 BCC 宏，用于定义散列表映射。

![2](img/#code_id_2_2)

`bpf_get_current_uid_gid()` 是一个帮助函数，用于获取触发此 kprobe 事件的进程的用户 ID。用户 ID 存储在返回的 64 位值的最低 32 位中。（最高的 32 位存储组 ID，但这部分被掩码掉了。）

![3](img/#code_id_2_3)

查找散列表中键与用户 ID 匹配的条目。它返回指向散列表中相应值的指针。

![4](img/#code_id_2_4)

如果此用户 ID 的散列表中有条目，将 `counter` 变量设置为散列表中当前值（由 `p` 指向）。如果散列表中没有此用户 ID 的条目，指针将为 `0`，计数器值将保持为 `0`。

![5](img/#code_id_2_5)

无论当前计数器值是多少，它都会增加一。

![6](img/#code_id_2_6)

更新散列表，使用新的计数器值更新此用户 ID。

详细查看访问散列表的代码行：

```
  p = counter_table.lookup(&uid);
```

稍后：

```
  counter_table.update(&uid, &counter);
```

如果您认为“这不是正确的 C 代码！”您是完全正确的。C 语言不支持在结构体上定义方法的方式。^(5) 这是一个很好的例子，BCC 的 C 版本非常宽松地类似于 C 语言，BCC 在将代码发送到编译器之前对其进行了重写。BCC 提供了一些方便的快捷方式和宏，它们会转换为“正确”的 C 代码。

就像前面的示例一样，C 代码被定义为一个名为`program`的字符串。该程序被编译，加载到内核中，并附加到`execve` kprobe，与前面的“Hello World”示例完全相同：

```
b = BPF(text=program)
syscall = b.get_syscall_fnname("execve")
b.attach_kprobe(event=syscall, fn_name="hello")
```

这次在 Python 侧需要更多的工作来从哈希表中读取信息：

```
while True:                                       ![1](img/1.png)
  sleep(2)                                         
  s = ""
  for k,v in b["counter_table"].items():          ![2](img/2.png)
    s += f"ID {k.value}: {v.value}\t"
  print(s)
```

![1](img/#code_id_2_7)

此代码部分无限循环，每两秒查找一次输出。

![2](img/#code_id_2_8)

BCC 自动创建一个 Python 对象来表示哈希表。此代码循环遍历任何值并将其打印到屏幕上。

当您运行此示例时，您会希望有第二个终端窗口，您可以在其中运行一些命令。右侧带有我在另一个终端中运行的命令的示例输出如下：

```
Terminal 1                          Terminal 2
$ ./hello-map.py 
                                    [blank line(s) until I run something]
ID 501: 1                           ls 
ID 501: 1
ID 501: 2                           ls
ID 501: 3       ID 0: 1             sudo ls
ID 501: 4       ID 0: 1             ls
ID 501: 4       ID 0: 1
ID 501: 5       ID 0: 2             sudo ls
```

此示例每两秒生成一行输出，无论是否发生任何事件。在此输出结束时，哈希表包含两个条目：

+   `key=501, value=5`

+   `key=0, value=2`

在第二个终端中，我有用户 ID 为 501。使用此用户 ID 运行`ls`命令会增加`execve`计数器。当我运行`sudo ls`时，会发生两次`execve`调用：一次是以用户 ID 501 执行`sudo`，另一次是以根用户 ID 0 执行`ls`。

在此示例中，我使用哈希表将数据从 eBPF 程序传输到用户空间。（我也可以在此处使用数组类型的映射，因为键是整数；哈希表允许您使用任意类型作为键。）当数据自然处于键值对时，哈希表非常方便，但用户空间代码必须定期轮询表格。Linux 内核已经支持从内核向用户空间发送数据的[perf 子系统](https://oreil.ly/nTvvH)，而 eBPF 包括使用 perf 缓冲区及其后续 BPF 环形缓冲区的支持。让我们来看看。

## Perf 和 Ring Buffer Maps

在本节中，我将描述一个稍微复杂的“Hello World”版本，它使用了 BCC 的`BPF_PERF_OUTPUT`功能，允许您将数据按您选择的结构写入到 perf 环形缓冲区映射中。

###### 注意

现在有一种称为“BPF 环形缓冲区”的新构造，如果您的内核版本为 5.8 或以上，通常优先于 BPF perf 缓冲区。Andrii Nakryiko 在他的[BPF 环形缓冲区](https://oreil.ly/ARRyV)博客文章中讨论了其中的区别。您将在第四章中看到 BCC 的`BPF_RINGBUF_OUTPUT`的示例。

你可以在 *Learning eBPF* [GitHub 仓库](http://github.com/lizrice/learning-ebpf) 的 *chapter2/hello-buffer.py* 中找到此示例的源代码。就像在本章早期看到的第一个`"Hello World"`示例一样，此版本每次使用 `execve()` 系统调用时都会向屏幕输出字符串 `"Hello World"`。它还会查找每次调用 `execve()` 的进程 ID 和命令名称，以便你得到类似于第一个示例的输出。这使我有机会展示一些更多的 BPF 辅助函数示例。

下面是将加载到内核中的 eBPF 程序：

```
BPF_PERF_OUTPUT(output);                                                ![1](img/1.png)

struct data_t {                                                         ![2](img/2.png)
   int pid;
   int uid;
   char command[16];
   char message[12];
};

int hello(void *ctx) {
   struct data_t data = {};                                             ![3](img/3.png)
   char message[12] = "Hello World";

   data.pid = bpf_get_current_pid_tgid() >> 32;                         ![4](img/4.png)
   data.uid = bpf_get_current_uid_gid() & 0xFFFFFFFF;                   ![5](img/5.png)

   bpf_get_current_comm(&data.command, sizeof(data.command));           ![6](img/6.png) 
   bpf_probe_read_kernel(&data.message, sizeof(data.message), message); ![7](img/7.png)

   output.perf_submit(ctx, &data, sizeof(data));                        ![8](img/8.png)

   return 0;
}
```

![1](img/#code_id_2_9)

BCC 定义了宏 `BPF_PERF_OUTPUT`，用于创建一个映射，将从内核传递消息到用户空间。我将这个映射称为 `output`。

![2](img/#code_id_2_10)

每次运行 `hello()` 时，该代码将写入一个数据结构的数据。以下是该结构的定义，其中包含进程 ID、当前运行命令的名称和文本消息。

![3](img/#code_id_2_11)

`data` 是一个保存要提交的数据结构的本地变量，`message` 包含字符串 `"Hello World"`。

![4](img/#code_id_2_12)

`bpf_get_current_pid_tgid()` 是一个辅助函数，用于获取触发此 eBPF 程序运行的进程 ID。它返回一个 64 位值，其中进程 ID 位于前 32 位中。^(6)

![5](img/#code_id_2_13)

`bpf_get_current_uid_gid()` 是你在前面示例中看到的获取用户 ID 的辅助函数。

![6](img/#code_id_2_14)

类似地，`bpf_get_current_comm()` 是一个辅助函数，用于获取进行 `execve` 系统调用的进程的可执行文件（或“命令”）名称。这是一个字符串，不像进程和用户 ID 那样是数值。在 C 语言中，你不能简单地使用 `=` 分配一个字符串。你必须将字符串应该写入的字段的地址 `&data.command` 作为辅助函数的参数传递。

![7](img/#code_id_2_15)

对于这个例子，每次的消息都是 `"Hello World"`。`bpf_probe_read_kernel()` 将其复制到数据结构的正确位置。

![8](img/#code_id_2_16)

此时数据结构已填充有进程 ID、命令名称和消息。调用 `output.perf_submit()` 将这些数据放入映射中。

正如在第一个“Hello World”示例中一样，此 C 程序被分配给 Python 代码中的一个名为 `program` 的字符串。接下来是 Python 代码的其余部分：

```
b = BPF(text=program)                                ![1](img/1.png)
syscall = b.get_syscall_fnname("execve")
b.attach_kprobe(event=syscall, fn_name="hello")

def print_event(cpu, data, size):                    ![2](img/2.png)
   data = b["output"].event(data)
   print(f"{data.pid} {data.uid} {data.command.decode()} " + \
         f"{data.message.decode()}")

b["output"].open_perf_buffer(print_event)            ![3](img/3.png)
while True:                                          ![4](img/4.png)
   b.perf_buffer_poll()
```

![1](img/#code_id_2_17)

编译 C 代码、将其加载到内核并将其附加到系统调用事件的代码行与之前的“Hello World”版本相同。

![2](img/#code_id_2_18)

`print_event` 是一个回调函数，将向屏幕输出一行数据。BCC 通过一些重活让我可以简单地将映射称为 `b["output"]` 并使用 `b["output"].event()` 从中获取数据。

![3](img/#code_id_2_19)

`b["output"].open_perf_buffer()` 打开性能环形缓冲区。该函数将 `print_event` 作为参数，用于定义每当从缓冲区读取数据时要使用的回调函数。

![4](img/#code_id_2_20)

程序现在将无限循环，^(7) 轮询性能环形缓冲区。如果有任何数据可用，`print_event` 将被调用。

运行此代码会给我们提供一个与原始“Hello World”相似的输出：

```
$ sudo ./hello-buffer.py
11654 node Hello World
11655 sh Hello World
...
```

与之前一样，您可能需要打开第二个终端到相同（虚拟）机器，并运行一些命令以触发一些输出。

这与原始的“Hello World”示例之间的主要区别在于，现在不再使用单一的中央跟踪管道，而是通过由此程序为自己使用创建的一个名为 `output` 的环形缓冲区映射来传递数据，如图 2-4 所示。

![使用性能环形缓冲区将数据从内核传递到用户空间](img/lebp_0204.png)

###### 图 2-4\. 使用性能环形缓冲区将数据从内核传递到用户空间

您可以通过使用 `cat /sys/kernel/debug/tracing/trace_pipe` 来验证信息是否不会发送到跟踪管道。

除了演示环形缓冲区映射的使用外，此示例还展示了一些用于检索触发 eBPF 程序运行事件相关信息的 eBPF 辅助函数。在这里，您已经看到了一些辅助函数，用于获取用户 ID、进程 ID 和当前命令的名称。正如您将在第七章中看到的那样，可用的上下文信息集和可用于检索它们的有效辅助函数集取决于程序类型及其触发事件。

eBPF 代码可以访问此类上下文信息的事实使其在可观察性方面非常有价值。每当事件发生时，eBPF 程序不仅可以报告事件发生的事实，还可以报告触发事件的相关信息。由于所有这些信息都可以在内核中收集，而无需进行任何同步的上下文切换到用户空间，因此它也具有高性能。

在本书的后续示例中，您将看到更多使用 eBPF 辅助函数来收集其他上下文数据的示例，以及使用 eBPF 程序更改上下文数据甚至阻止事件发生的示例。

## 函数调用

你已经看到 eBPF 程序可以调用内核提供的帮助函数，但如果你想将正在编写的代码拆分为函数，该怎么办？一般来说，在软件开发中，将常见代码提取为函数以供多处调用被视为良好实践^(8)，而不是一遍又一遍地复制相同的代码行。但在早期，除了帮助函数外，eBPF 程序不允许调用其他函数。为了解决这个问题，程序员们已经指示编译器“始终内联”它们的函数，就像这样：

```
static __always_inline void my_function(void *ctx, int val)
```

通常，源代码中的函数导致编译器生成跳转指令，该指令导致执行跳转到组成被调用函数的一组指令（并在该函数完成后再次跳回）。你可以在 图 2-5 的左侧看到这一点。右侧显示了内联函数时发生的情况：没有跳转指令；相反，函数的指令副本直接嵌入到调用函数中。

![非内联和内联函数指令的布局](img/lebp_0205.png)

###### 图 2-5\. 非内联和内联函数指令的布局

如果该函数被多处调用，那么编译后的可执行文件中将包含多个该函数的指令副本。（有时编译器可能会选择内联函数以进行优化，这也是你可能无法附加 kprobe 到某些内核函数的原因之一。我会在 第七章 中再次谈到这点。）

从 Linux 内核 4.16 和 LLVM 6.0 开始，解除了函数必须内联的限制，以便 eBPF 程序员可以更自然地编写函数调用。然而，这一特性称为“BPF 到 BPF 函数调用”或“BPF 子程序”，目前不受 BCC 框架支持，我们将在下一章中再回到这一点。（当然，如果函数被内联，你仍然可以继续在 BCC 中使用函数。）

eBPF 中还有另一种将复杂功能分解为较小部分的机制：尾调用。

## 尾调用

正如在 [ebpf.io](https://oreil.ly/Loyuz) 中描述的，“尾调用可以调用并执行另一个 eBPF 程序，并替换执行上下文，类似于 `execve()` 系统调用在常规进程中的操作。” 换句话说，执行在尾调用完成后不会返回给调用者。

###### 注意

[尾调用](https://oreil.ly/cOA1r)并不是 eBPF 编程专有的。尾调用的一般动机是避免在函数递归调用时一遍又一遍地向栈中添加帧，这最终可能导致栈溢出错误。如果能够安排代码在调用递归函数后作为最后一件事做尾调用，那么与调用函数相关联的栈帧实际上并没有做任何有用的事情。尾调用允许调用一系列函数而不会增长栈。在 eBPF 中特别有用，因为[栈限制为 512 字节](https://oreil.ly/SZmkd)。

使用`bpf_tail_call()`辅助函数进行尾调用，其签名如下：

```
long bpf_tail_call(void **`ctx`*, struct bpf_map **`prog_array_map`*, u32 *`index`*)
```

此函数的三个参数具有以下含义：

+   `ctx`允许从调用 eBPF 程序传递上下文到被调用程序。

+   `prog_array_map`是一个类型为`BPF_MAP_TYPE_PROG_ARRAY`的 eBPF 映射，其中包含用于标识 eBPF 程序的一组文件描述符。

+   `index`指示应调用那一组 eBPF 程序中的哪一个。

这个辅助程序有些不同寻常，如果成功，它永远不会返回。当前运行的 eBPF 程序在栈上被调用的程序替换。如果指定的程序在映射中不存在，例如，辅助程序可能会失败，在这种情况下，调用程序继续执行。

用户空间代码必须将所有 eBPF 程序加载到内核中（像往常一样），并设置程序数组映射。

让我们看一个简单的示例，使用 BCC 编写的 Python 代码；你可以在[GitHub 存储库](http://github.com/lizrice/learning-ebpf)中找到这段代码，位于*chapter2/hello-tail.py*。主要的 eBPF 程序附加到了一个跟踪点，该跟踪点是所有系统调用的通用入口点。此程序使用尾调用来跟踪特定系统调用操作码的特定消息。如果对于给定的操作码没有尾调用，程序将跟踪一个通用消息。

如果你正在使用 BCC 框架，为了进行[尾调用](https://oreil.ly/rT9e1)，可以使用略微简化的形式的代码行：

```
prog_array_map.call(ctx, index)
```

在将代码传递给编译步骤之前，BCC 将重写上述行为：

```
bpf_tail_call(ctx, prog_array_map, index)
```

下面是 eBPF 程序及其尾调用的源代码：

```
BPF_PROG_ARRAY(syscall, 300);                                   ![1](img/1.png)

int hello(struct bpf_raw_tracepoint_args *ctx) {                ![2](img/2.png)
   int opcode = ctx->args[1];                                   ![3](img/3.png)
   syscall.call(ctx, opcode);                                   ![4](img/4.png)
   bpf_trace_printk("Another syscall: %d", opcode);             ![5](img/5.png)
   return 0;
}

int hello_execve(void *ctx) {                                   ![6](img/6.png)
   bpf_trace_printk("Executing a program");
   return 0;
}

int hello_timer(struct bpf_raw_tracepoint_args *ctx) {          ![7](img/7.png)
   if (ctx->args[1] == 222) {
       bpf_trace_printk("Creating a timer");
   } else if (ctx->args[1] == 226) {
       bpf_trace_printk("Deleting a timer");
   } else {
       bpf_trace_printk("Some other timer operation");
   }
   return 0;
}

int ignore_opcode(void *ctx) {                                  ![8](img/8.png)
   return 0;
}
```

![1](img/#code_id_2_21)

BCC 提供了一个`BPF_PROG_ARRAY`宏，用于轻松定义类型为`BPF_MAP_TYPE_PROG_ARRAY`的映射。我称这个映射为`syscall`，并允许 300 个条目，^(9) 这对于这个示例来说已经足够了。

![2](img/#code_id_2_22)

在即将看到的用户空间代码中，我将把这个 eBPF 程序附加到`sys_enter`原始跟踪点上，每当进行任何系统调用时都会触发。传递给附加到原始跟踪点的 eBPF 程序的上下文采用`bpf_raw_tracepoint_args`结构的形式。

![3](img/#code_id_2_23)

对于`sys_enter`，原始跟踪点参数包括标识正在进行的系统调用的操作码。

![4](img/#code_id_2_24)

在这里，我们对与操作码匹配的程序数组条目进行了一个尾调用。在将源代码传递给编译器之前，BCC 将此行代码重写为对`bpf_tail_call()`辅助函数的调用。

![5](img/#code_id_2_25)

如果尾调用成功，将不会执行此行跟踪操作码值的代码行。我已经用它来为映射中没有程序入口的操作码提供一个默认的追踪行。

![6](img/#code_id_2_26)

`hello_exec()`是一个将加载到系统调用程序数组映射中的程序，在操作码指示为`execve()`系统调用时作为尾调用执行。它只会生成一行追踪，告诉用户正在执行一个新程序。

![7](img/#code_id_2_27)

`hello_timer()`是另一个将加载到系统调用程序数组中的程序。在这种情况下，它将被多个程序数组条目引用。

![8](img/#code_id_2_28)

`ignore_opcode()`是一个什么都不做的尾调用程序。我会将其用于那些我完全不想生成任何追踪的系统调用。

现在让我们看看加载和管理这组 eBPF 程序的用户空间代码：

```
b = BPF(text=program)                                              
b.attach_raw_tracepoint(tp="sys_enter", fn_name="hello")           ![1](img/1.png)

ignore_fn = b.load_func("ignore_opcode", BPF.RAW_TRACEPOINT)       ![2](img/2.png)
exec_fn = b.load_func("hello_exec", BPF.RAW_TRACEPOINT)
timer_fn = b.load_func("hello_timer", BPF.RAW_TRACEPOINT)

prog_array = b.get_table("syscall")                                ![3](img/3.png)
prog_array[ct.c_int(59)] = ct.c_int(exec_fn.fd)
prog_array[ct.c_int(222)] = ct.c_int(timer_fn.fd)
prog_array[ct.c_int(223)] = ct.c_int(timer_fn.fd)
prog_array[ct.c_int(224)] = ct.c_int(timer_fn.fd)
prog_array[ct.c_int(225)] = ct.c_int(timer_fn.fd)
prog_array[ct.c_int(226)] = ct.c_int(timer_fn.fd)

# Ignore some syscalls that come up a lot ![4](img/4.png)
prog_array[ct.c_int(21)] = ct.c_int(ignore_fn.fd)
prog_array[ct.c_int(22)] = ct.c_int(ignore_fn.fd)
prog_array[ct.c_int(25)] = ct.c_int(ignore_fn.fd)
...

b.trace_print()                                                    ![5](img/5.png)
```

![1](img/#code_id_2_29)

与之前看到的连接到 kprobe 不同，这次用户空间代码将主 eBPF 程序连接到了`sys_enter`跟踪点。

![2](img/#code_id_2_30)

这些对`b.load_func()`的调用每次都返回一个尾调用程序的文件描述符。请注意，尾调用需要与其父程序具有相同的程序类型——在本例中是`BPF.RAW_TRACEPOINT`。此外，需要指出的是，每个尾调用程序本身就是一个独立的 eBPF 程序。

![3](img/#code_id_2_31)

用户空间代码会在`syscall`映射中创建条目。映射不必为每个可能的操作码完全填充；如果某个特定操作码没有条目，那就意味着不会执行任何尾调用。此外，有多个条目指向同一个 eBPF 程序也是完全合理的。在这种情况下，我希望`hello_timer()`尾调用能够对一组与定时器相关的系统调用中的任何一个执行。

![4](img/#code_id_2_32)

某些系统调用由系统频繁运行，每次运行都生成一行追踪输出，使得输出难以阅读。我已经为几个系统调用使用了`ignore_opcode()`尾调用。

![5](img/#code_id_2_33)

将追踪输出打印到屏幕上，直到用户终止程序运行。

运行此程序会为在（虚拟）机器上运行的每个系统调用生成追踪输出，除非操作码已与`ignore_opcode()`尾调用链接。以下是在另一个终端中运行`ls`时生成的部分追踪输出示例（为了可读性已省略部分细节）：

```
./hello-tail.py 
b'   hello-tail.py-2767    ... Another syscall: 62'
b'   hello-tail.py-2767    ... Another syscall: 62'
...
b'            bash-2626    ... Executing a program'
b'            bash-2626    ... Another syscall: 220'
...
b'           <...>-2774    ... Creating a timer'
b'           <...>-2774    ... Another syscall: 48'
b'           <...>-2774    ... Deleting a timer'
...
b'              ls-2774    ... Another syscall: 61'
b'              ls-2774    ... Another syscall: 61'
...
```

正在执行的特定系统调用并不重要，但您可以看到不同的尾调用被调用并生成跟踪消息。您还可以看到对于没有在尾调用程序映射中具有条目的操作码，默认消息为`Another syscall`。

###### 注意

查看保罗·夏尼翁在不同内核版本上关于[BPF 尾调用成本](https://oreil.ly/jTxcb)的博客文章。

自内核版本 4.2 起，eBPF 支持尾调用，但长期以来它们与进行 BPF 到 BPF 函数调用不兼容。这一限制在内核 5.10 中被解除。^(10)

您可以将最多 33 个尾调用串联在一起，加上每个 eBPF 程序的指令复杂性限制为 100 万条指令，这意味着今天的 eBPF 程序员可以在内核中编写非常复杂的代码。

# 摘要

希望通过展示一些具体的 eBPF 程序示例，本章帮助您巩固对运行在内核中的 eBPF 代码的精神模型，这些代码由事件触发。您还看到了使用 BPF 映射将数据从内核传递到用户空间的示例。

使用 BCC 框架隐藏了构建程序、加载到内核并附加到事件的许多细节。在下一章中，我将向您展示编写“Hello World”的不同方法，并深入探讨这些隐藏的细节。

# 练习

如果您想进一步探索“Hello World”，可以尝试（或思考）一些可选的活动：

1.  将*hello-buffer.py* eBPF 程序调整为对奇数和偶数进程 ID 输出不同的跟踪消息。

1.  修改*hello-map.py*，以便 eBPF 代码可以被多个系统调用触发。例如，`openat()`通常用于打开文件，而`write()`用于向文件写入数据。您可以从将*hello* eBPF 程序附加到多个系统调用 kprobes 开始。然后尝试为不同的系统调用准备修改后的*hello* eBPF 程序版本，展示您可以从多个不同的程序访问同一映射。

1.  *hello-tail.py* eBPF 程序是一个示例程序，它附加到`sys_enter`原始跟踪点，每次调用系统调用时都会触发。修改*hello-map.py*，以显示每个用户 ID 执行的系统调用总数，方法是将其附加到相同的`sys_enter`原始跟踪点。

    在我进行了这些更改后，这是我得到的一些示例输出：

    ```
    $ ./hello-map.py 
    ID 104: 6     ID 0: 225
    ID 104: 6     ID 101: 34    ID 100: 45    ID 0: 332     ID 501: 19
    ID 104: 6     ID 101: 34    ID 100: 45    ID 0: 368     ID 501: 38
    ID 104: 6     ID 101: 34    ID 100: 45    ID 0: 533     ID 501: 57
    ```

1.  由 BCC 提供的[`RAW_TRACEPOINT_PROBE`宏](https://oreil.ly/kh-j4)简化了附加到原始跟踪点的过程，告诉用户空间的 BCC 代码自动将其附加到指定的跟踪点。尝试在*hello-tail.py*中使用它，就像这样：

    +   将`hello()`函数的定义替换为`RAW_TRACEPOINT_PROBE(sys_enter)`。

    +   从 Python 代码中删除显式附加调用`b.attach_raw_tracepoint()`。

    您应该看到 BCC 自动创建附件，程序的运行完全相同。这是 BCC 提供的许多便利宏的一个示例。

1.  您可以进一步调整*hello_map.py*，使哈希表中的键标识特定系统调用（而不是特定用户）。输出将显示该系统调用在整个系统中被调用的次数。

^(1) 最初我为一个名为“eBPF 编程入门指南”的演讲编写了这篇文章。您可以在[*https://github.com/lizrice/ebpf-beginners*](https://github.com/lizrice/ebpf-beginners)找到原始代码以及幻灯片和视频的链接。

^(2) 有一种更高效的方法可以将 eBPF 程序附加到函数上，从内核版本 5.5 开始可用，该方法使用 fentry（以及相应的 fexit，而不是 kretprobe 用于函数的退出）。我稍后会在书中讨论这个，但现在我使用 kprobe 使本章的示例尽可能简单。

^(3) 我经常使用 VScode 远程连接到云中的虚拟机。在虚拟机上运行许多节点脚本，这些节点脚本来自这个“Hello World”应用程序的跟踪。

^(4) 一些命令（例如`echo`）可能是作为 shell 内置运行的，而不是执行新程序。这些不会触发`execve()`事件，因此不会生成跟踪。

^(5) C++可以，但 C 语言不行。

^(6) 低 32 位是*线程组 ID*。对于单线程进程，这与进程 ID 相同，但进程的其他线程将获得不同的 ID。GNU C 库的文档对[进程和线程组 ID](https://oreil.ly/Wo9k3)的差异有很好的描述。

^(7) 这只是示例代码，所以我不会担心在键盘中断或其他方面的清理！

^(8) 这个原则通常被称为“DRY”（“不要重复自己”），由[《务实程序员》](https://oreil.ly/QFich)所推广。

^(9) Linux 中有大约 300 个系统调用，由于我在这个示例中没有使用最近添加的任何系统调用，这已经足够了。

^(10) 从 BPF 子程序中进行尾调用需要 JIT 编译器的支持，您将在下一章中了解到。在我编写本书示例所用的内核版本中，只有 x86 上的 JIT 编译器支持此功能，尽管[在内核 6.0 中已为 ARM 添加了支持](https://oreil.ly/KYUYS)。
