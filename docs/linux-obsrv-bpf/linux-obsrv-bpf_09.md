# 第八章：Linux 内核安全性、权限和 Seccomp

BPF 是一种在不牺牲稳定性、安全性和速度的情况下扩展内核的强大方式。因此，内核开发人员认为，利用其多功能性来通过实现由 BPF 程序支持的 Seccomp 过滤器来改进 Seccomp 中的进程隔离是一个好主意，也被称为 Seccomp BPF。在本章中，我们将探讨 Seccomp 是什么以及如何使用它。然后，您将学习如何使用 BPF 程序编写 Seccomp 过滤器。之后，您将探索内核为 Linux 安全模块提供的内置 BPF 钩子。

Linux 安全模块（LSM）是一个提供一套函数的框架，可以用标准化的方式实现不同的安全模型。LSM 可以直接在内核源代码树中使用，比如 Apparmor、SELinux 和 Tomoyo。

我们从讨论 Linux 权限开始。

# 权限

Linux 权限的处理方式是，你需要为你的非特权进程提供执行特定任务的权限，但又不希望给二进制文件赋予`SUID`权限或以其他方式使进程具备特权，因此通过仅给予进程完成特定任务所需的特定能力来减少攻击面。例如，如果你的应用程序需要打开一个特权端口，比如 80 端口，你可以只赋予它`CAP_NET_BIND_SERVICE`能力，而不是以 root 身份启动进程。

考虑以下名为*main.go*的 Go 程序：

```
package main

import (
	"net/http"
	"log"
)

func main() {
    log.Fatalf("%v", http.ListenAndServe(":80", nil))
}
```

该程序在端口 80 上提供 HTTP 服务器，这是一个特权端口。

我们通常在编译后直接运行该程序如下：

```
$ go build -o capabilities main.go
$ ./capabilities
```

然而，由于我们没有给予 root 权限，当绑定端口时，该代码将输出错误：

```
2019/04/25 23:17:06 listen tcp :80: bind: permission denied
exit status 1
```

###### 提示

`capsh`（能力外壳包装器）是一个工具，它将以特定的一组能力启动一个 shell。

在这种情况下，正如所述，我们可以通过允许`cap_net_bind_service`权限及程序已有的其他权限，而不是给予完整的 root 权限，来允许绑定特权端口。为此，我们可以用`capsh`来包装我们的程序运行：

```
# capsh --caps='cap_net_bind_service+eip cap_setpcap,cap_setuid,cap_setgid+ep' \
    --keep=1 --user="nobody" \
    --addamb=cap_net_bind_service -- -c "./capabilities"
```

让我们稍微解析一下那个命令：

`capsh`

我们使用`capsh`作为包装器。

`--caps='cap_net_bind_service+eip cap_setpcap,cap_setuid,cap_setgid+ep'`

因为我们需要更改用户（我们不希望以 root 身份运行），所以我们需要指定`cap_net_bind_service`以及实际执行用户 ID 从`root`变为`nobody`所需的能力，即`cap_setuid`和`cap_setgid`：

`--keep=1`

我们希望在从 root 切换完成后保持已设置的能力。

`--user="nobody"`

运行我们程序的最终用户将是`nobody`。

`--addamb=cap_net_bind_service`

我们设置周围能力，因为这些能力在从 root 切换后会被清除。

`-- -c "./capabilities"`

最后，我们只需运行我们的程序。

###### 注意

环境权限是当前程序使用`execve()`执行它们时由子程序继承的一种特定类型的权限。只有在环境中允许且可继承的权限才能成为环境权限。

此时，您可能会问自己`--caps`选项中的`+eip`是什么。这些标志用于确定：

+   需要激活该功能（p）。

+   该功能可用（e）。

+   可以通过子进程继承该功能（i）。

因为我们想要使用我们的`cap_net_bind_service`，我们需要使其为`e`；然后在我们的命令中，我们启动了一个 shell。然后启动了`capabilities`二进制文件，我们需要使其为`i`。最后，我们希望激活该能力（因为我们更改了 UID，它没有被激活），使用`p`。这最终变成了`cap_net_bind_service+eip`。

您可以使用`ss`来验证；我们将切断输出以使其适合本页，但它将显示绑定端口和用户 ID 与`0`不同，在本例中为`65534`：

```
# ss -tulpn -e -H | cut -d' ' -f17-
128    *:80    *:*
users:(("capabilities",pid=30040,fd=3)) uid:65534 ino:11311579 sk:2c v6only:0
```

我们在此示例中使用了`capsh`，但您可以通过使用*libcap*编写包装器来获取更多信息，请参阅`man 3 libcap`。

在编写程序时，开发人员通常不能预先知道程序在运行时所需的所有功能；而且，随着新版本的发布，这些功能可能会发生变化。

要更好地了解程序使用的能力，我们可以使用 BCC 中的`capable`工具在内核函数`cap_capable`上设置一个 kprobe：

```
 /usr/share/bcc/tools/capable
TIME      UID    PID    TID    COMM             CAP  NAME                 AUDIT
10:12:53  0      424    424    systemd-udevd    12   CAP_NET_ADMIN        1
10:12:57  0      1103   1101   timesync         25   CAP_SYS_TIME         1
10:12:57  0      19545  19545  capabilities     10   CAP_NET_BIND_SERVICE 1
```

我们可以使用`bpftrace`来实现相同的功能，通过一行代码在`cap_capable`内核函数上设置一个 kprobe：

```
bpftrace -e \
    'kprobe:cap_capable {
 time("%H:%M:%S  ");
 printf("%-6d %-6d %-16s %-4d %d\n", uid, pid, comm, arg2, arg3);
 }' \
    | grep -i capabilities
```

如果我们的程序`capabilities`在 kprobe 之后启动，那么将输出类似以下内容：

```
12:01:56  1000   13524  capabilities     21   0
12:01:56  1000   13524  capabilities     21   0
12:01:56  1000   13524  capabilities     21   0
12:01:56  1000   13524  capabilities     12   0
12:01:56  1000   13524  capabilities     12   0
12:01:56  1000   13524  capabilities     12   0
12:01:56  1000   13524  capabilities     12   0
12:01:56  1000   13524  capabilities     10   1
```

第五列是进程所需的功能，因为此输出还包括非审计事件，我们可以看到所有的非审计检查，并最终看到具有审计标志（前一个输出的最后一个）设置为 1 的所需功能。我们感兴趣的功能是`CAP_NET_BIND_SERVICE`，在内核源代码的`include/uapi/linux/capability.h`中定义为常量，其 ID 为`10`：

```
/* Allows binding to TCP/UDP sockets below 1024 */
/* Allows binding to ATM VCIs below 32 */

#define CAP_NET_BIND_SERVICE 10
```

容器运行时（如 runC 或 Docker）经常使用权限来使容器无特权，并仅允许运行大多数应用程序所需的权限。当应用程序需要特定的权限时，在 Docker 中可以通过`--cap-add`完成：

```
docker run -it --rm --cap-add=NET_ADMIN ubuntu ip link add dummy0 type dummy
```

此命令将为该容器赋予`CAP_NET_ADMIN`能力，允许它设置一个 netlink 以添加`dummy0`接口。

下一节将展示如何使用另一种技术实现能力，例如过滤，这将使我们能够以编程方式实现自己的过滤器。

# Seccomp

Seccomp 代表安全计算，是 Linux 内核中实现的安全层，允许开发人员过滤特定的系统调用。虽然 Seccomp 与 capabilities（能力）类似，但其能够控制特定系统调用的能力使其比 capabilities 更加灵活。

Seccomp 和 capabilities 并不互斥；它们通常一起使用，从两个世界中带来好处。例如，你可能想要给一个进程赋予`CAP_NET_ADMIN`权限，但通过阻止`accept`和`accept4`系统调用来防止其在套接字上接受连接。

Seccomp 过滤器的方式基于使用`SECCOMP_MODE_FILTER`模式的 BPF 过滤器，系统调用的过滤与包过滤一样进行。

Seccomp 过滤器使用`prctl`加载，通过`PR_SET_SECCOMP`操作表达为 BPF 程序形式，在每个使用`seccomp_data`结构表达的 Seccomp *packet*上执行。该结构包含参考架构，系统调用时 CPU 指令指针，以及最多六个系统调用参数作为`uint64`表达。

下面是来自内核源码`linux/seccomp.h`中`seccomp_data`结构的展示：

```
struct seccomp_data {
	int nr;
	__u32 arch;
	__u64 instruction_pointer;
	__u64 args[6];
};
```

通过阅读结构可以看出，我们可以基于系统调用本身、其参数或它们的组合来进行过滤。

在接收每个 Seccomp packet 后，过滤器负责进行处理以做出最终决策，告诉内核接下来应该做什么。最终决策通过其中一个返回值（状态码）表达，如以下所述：

`SECCOMP_RET_KILL_PROCESS`

在过滤系统调用后会立即终止整个进程，因此系统调用不会被执行。

`SECCOMP_RET_KILL_THREAD`

在过滤系统调用后会立即终止当前线程，因此系统调用不会被执行。

`SECCOMP_RET_KILL`

这是`SECCOMP_RET_KILL_THREAD`的别名，保留以保证兼容性。

`SECCOMP_RET_TRAP`

系统调用被禁止，并且会向调用该系统调用的任务发送`SIGSYS`（Bad System Call）信号。

`SECCOMP_RET_ERRNO`

系统调用不被执行，并且过滤器返回值的`SECCOMP_RET_DATA`部分作为`errno`值传递给用户空间。根据错误的原因不同，会返回不同的`errno`值。你可以在下面的部分找到错误号列表。

`SECCOMP_RET_TRACE`

这用于通过`PTRACE_O_TRACESECCOMP`通知`ptrace`跟踪器截获系统调用的调用，以便观察和控制系统调用的执行。如果没有附加跟踪器，则返回错误，设置`errno`为`-ENOSYS`，并且不执行系统调用。

`SECCOMP_RET_LOG`

允许系统调用并记录。

`SECCOMP_RET_ALLOW`

系统调用仅仅被允许。

###### 注意

`ptrace` 是一个系统调用，用于在进程（称为 *tracee*）上实现跟踪机制，其效果是能够观察和控制进程的执行。追踪程序可以有效地影响执行并更改 tracee 的内存寄存器。在 Seccomp 的上下文中，当由 `SECCOMP_RET_TRACE` 状态码触发时，会使用 `ptrace`，因此追踪程序可以防止系统调用执行并实施自己的逻辑。

## Seccomp 错误

在使用 Seccomp 时，您会不时遇到由 `SECCOMP_RET_ERRNO` 类型的返回值给出的不同错误。为了通知发生了错误，`seccomp` 系统调用将返回 `-1` 而不是 `0`。

可能的错误如下：

`EACCESS`

调用者不允许执行系统调用 —— 通常是因为它没有 `CAP_SYS_ADMIN` 特权或者没有使用 `prctl` 设置 `no_new_privs`，我们将在本章后面详细解释。

`EFAULT`

传递的参数（`seccomp_data` 结构中的 `args`）没有有效的地址。

`EINVAL`

它可以有四种含义：

+   请求的操作在当前配置的内核中不被知道或支持。

+   指定的标志对于请求的操作无效。

+   操作包括 `BPF_ABS`，但是指定的偏移量可能超出了 `seccomp_data` 结构的大小。

+   传递给过滤器的指令数超过了最大指令数。

`ENOMEM`

没有足够的内存来执行程序。

`EOPNOTSUPP`

操作指定了 `SECCOMP_GET_ACTION_AVAIL`，该操作是可用的，但实际上内核在参数返回中不支持该返回操作。

`ESRCH`

在另一个线程同步期间出现问题。

`ENOSYS`

`SECCOMP_RET_TRACE` 操作中没有追踪程序附加。

###### 注意

`prctl` 是一个系统调用，允许用户空间程序控制（设置和获取）进程的特定方面，如端序、线程名称、安全计算（Seccomp）模式、权限、性能事件等。

对你来说，Seccomp 可能听起来像是一个沙盒机制，但这并不正确。Seccomp 是一个实用程序，允许其用户开发沙盒机制。现在这里是如何编写程序以直接调用 Seccomp 系统调用的过滤器来编写自定义交互。

## Seccomp BPF 过滤器示例

在此示例中，我们展示如何组合前述的两个操作：

+   编写 Seccomp BPF 程序，用作根据其做出的决定返回不同返回码的过滤器。

+   使用 `prctl` 加载过滤器。

首先，示例需要一些来自标准库和 Linux 内核的头文件：

```
#include <errno.h>
#include <linux/audit.h>
#include <linux/bpf.h>
#include <linux/filter.h>
#include <linux/seccomp.h>
#include <linux/unistd.h>
#include <stddef.h>
#include <stdio.h>
#include <stdlib.h>
#include <sys/prctl.h>
#include <unistd.h>
```

在尝试执行此示例之前，我们需要确保我们的内核已编译并设置为 `CONFIG_SECCOMP` 和 `CONFIG_SECCOMP_FILTER` 为 `y`。在实时机器上，可以通过以下方式检查：

```
cat /proc/config.gz| zcat  | grep -i CONFIG_SECCOMP
```

代码的其余部分是`install_filter`函数，由两部分组成。第一部分包含我们的 BPF 过滤指令列表：

```
static int install_filter(int nr, int arch, int error) {
  struct sock_filter filter[] = {
    BPF_STMT(BPF_LD + BPF_W + BPF_ABS, (offsetof(struct seccomp_data, arch))),
    BPF_JUMP(BPF_JMP + BPF_JEQ + BPF_K, arch, 0, 3),
    BPF_STMT(BPF_LD + BPF_W + BPF_ABS, (offsetof(struct seccomp_data, nr))),
    BPF_JUMP(BPF_JMP + BPF_JEQ + BPF_K, nr, 0, 1),
    BPF_STMT(BPF_RET + BPF_K, SECCOMP_RET_ERRNO | (error & SECCOMP_RET_DATA)),
    BPF_STMT(BPF_RET + BPF_K, SECCOMP_RET_ALLOW),
  };
```

这些指令使用`linux/filter.h`中定义的`BPF_STMT`和`BPF_JUMP`宏设置。

让我们逐步执行这些指令：

`BPF_STMT(BPF_LD + BPF_W + BPF_ABS (offsetof(struct seccomp_data, arch)))`

这将加载并累积`BPF_LD`形式的字`BPF_W`，数据包数据包含在固定的`BPF_ABS`偏移量处。

`BPF_JUMP(BPF_JMP + BPF_JEQ + BPF_K, arch, 0, 3)`

这会使用`BPF_JEQ`检查累加器中的体系结构值是否等于`arch`。如果是，则跳转到下一条指令（偏移量为零）；否则，跳转到偏移量为三的错误处理指令，因为体系结构不匹配。

`BPF_STMT(BPF_LD + BPF_W + BPF_ABS (offsetof(struct seccomp_data, nr)))`

这将加载并累积`BPF_LD`形式的字`BPF_W`，它是包含在固定`BPF_ABS`偏移量处的系统调用号数据。

`BPF_JUMP(BPF_JMP + BPF_JEQ + BPF_K, nr, 0, 1)`

这将比较系统调用号中的值与`nr`变量中的值。如果它们相等，将转到下一条指令并禁止系统调用；否则，将允许系统调用使用`SECCOMP_RET_ALLOW`。

`BPF_STMT(BPF_RET + BPF_K, SECCOMP_RET_ERRNO | (error &` `SECCOMP_RET_DATA))`

这会使用`BPF_RET`终止程序，并返回一个带有从`err`变量中指定的错误号的错误，`SECCOMP_RET_ERRNO`。

`BPF_STMT(BPF_RET + BPF_K, SECCOMP_RET_ALLOW)`

这将使用`BPF_RET`终止程序，并允许使用`SECCOMP_RET_ALLOW`进行系统调用执行。

如果您需要进一步理解该汇编内容，可能会发现一些做同样事情的伪代码很有用：

```
if (arch != AUDIT_ARCH_X86_64) {
    return SECCOMP_RET_ALLOW;
}
if (nr == __NR_write) {
    return SECCOMP_RET_ERRNO;
}
return SECCOMP_RET_ALLOW;
```

在`socket_filter`结构中定义过滤器代码后，我们需要定义一个`sock_fprog`，其中包含过滤器代码的长度计算结果。此数据结构需要作为声明后续过程操作的参数：

```
  struct sock_fprog prog = {
    .len = (unsigned short)(sizeof(filter) / sizeof(filter[0])),
    .filter = filter,
  };
```

现在，在`install_filter`函数中只剩下一件事要做：加载程序本身！为此，我们使用`prctl`并以`PR_SET_SECCOMP`为选项，因为我们想进入安全计算模式。然后，我们指示模式加载一个带有`sock_fprog`类型的`prog`变量中包含的`SECCOMP_MODE_FILTER`过滤器：

```
  if (prctl(PR_SET_SECCOMP, SECCOMP_MODE_FILTER, &prog)) {
    perror("prctl(PR_SET_SECCOMP)");
    return 1;
  }
  return 0;
}
```

最后，我们可以利用我们的`install_filter`函数，但在使用之前，我们需要使用`prctl`在当前执行中设置`PR_SET_NO_NEW_PRIVS`，以避免子进程具有比父进程更广泛的权限。这使得我们可以在没有根权限的情况下在`install_filter`函数中进行以下`prctl`调用。

现在我们可以调用 `install_filter` 函数。我们将阻止所有与 `X86-64` 架构相关的 `write` 系统调用，并只给予所有尝试的权限被拒绝。在过滤器安装后，我们通过使用第一个参数继续执行：

```
int main(int argc, char const *argv[]) {
  if (prctl(PR_SET_NO_NEW_PRIVS, 1, 0, 0, 0)) {
    perror("prctl(NO_NEW_PRIVS)");
    return 1;
  }
  install_filter(__NR_write, AUDIT_ARCH_X86_64, EPERM);
  return system(argv[1]);
}
```

现在让我们来试试吧！

要编译我们的程序，我们可以使用 `clang` 或 `gcc`；无论哪种方式，只需编译 `main.c` 文件，没有特殊的选项：

```
clang main.c -o filter-write
```

我们说我们在我们的程序中阻止了所有的写入。为了测试它，我们需要一个进行写入操作的程序；`ls` 程序似乎是一个很好的选择，这是它正常行为的样子：

```
ls -la
total 36
drwxr-xr-x 2 fntlnz users  4096 Apr 28 21:09 .
drwxr-xr-x 4 fntlnz users  4096 Apr 26 13:01 ..
-rwxr-xr-x 1 fntlnz users 16800 Apr 28 21:09 filter-write
-rw-r--r-- 1 fntlnz users    19 Apr 28 21:09 .gitignore
-rw-r--r-- 1 fntlnz users  1282 Apr 28 21:08 main.c
```

太棒了！这是我们的包装程序用法的样子；我们只需将要测试的程序作为第一个参数传递：

```
./filter-write "ls -la"
```

执行后，该程序将完全空白输出，没有任何输出。然而，我们可以使用 `strace` 查看发生了什么：

```
strace -f ./filter-write "ls -la"
```

结果已经去除了许多噪音，相关部分显示写入操作因 `EPERM` 错误被阻止，这与我们设置的相同。这意味着程序是静默的，因为现在它无法访问该系统调用：

```
[pid 25099] write(2, "ls: ", 4)         = -1 EPERM (Operation not permitted)
[pid 25099] write(2, "write error", 11) = -1 EPERM (Operation not permitted)
[pid 25099] write(2, "\n", 1)           = -1 EPERM (Operation not permitted)
```

现在您已经了解了 Seccomp BPF 的运作方式，并且对您可以使用它做什么有了一个良好的认识。但如果有一种方法可以使用 eBPF 来实现相同的功能，而不是使用 cBPF 来利用它的强大能力，那不是更好吗？

在考虑 eBPF 程序时，大多数人认为您只需编写它们并使用 root 权限加载它们。虽然这种说法通常是正确的，但内核实现了一组机制来在各个层次上保护 eBPF 对象；这些机制称为 BPF LSM *钩子*。

# BPF LSM 钩子

为了在系统事件上提供与架构无关的控制，LSM 实现了钩子的概念。从技术上讲，钩子调用类似于系统调用；然而，由于钩子是系统独立的并且与 LSM 框架集成在一起，使得钩子变得非常有趣，因为提供的抽象层可以很方便地避免在不同架构上处理系统调用时可能出现的问题。

在撰写本文时，内核有七个与 BPF 程序相关的钩子，并且 SELinux 是唯一实现它们的内核 LSM。

您可以在此文件中查看内核源码树中的内容：`include/linux/security.h`：

```
extern int security_bpf(int cmd, union bpf_attr *attr, unsigned int size);
extern int security_bpf_map(struct bpf_map *map, fmode_t fmode);
extern int security_bpf_prog(struct bpf_prog *prog);
extern int security_bpf_map_alloc(struct bpf_map *map);
extern void security_bpf_map_free(struct bpf_map *map);
extern int security_bpf_prog_alloc(struct bpf_prog_aux *aux);
extern void security_bpf_prog_free(struct bpf_prog_aux *aux);
```

这些钩子中的每一个都会在执行的不同阶段被调用：

`security_bpf`

对执行的 BPF 系统调用进行初始检查

`security_bpf_map`

当内核为映射返回文件描述符时进行检查

`security_bpf_prog`

当内核为 eBPF 程序返回文件描述符时进行检查

`security_bpf_map_alloc`

初始化 BPF 映射内部的安全字段

`security_bpf_map_free`

清理 BPF 映射内部的安全字段

`security_bpf_prog_alloc`

初始化 BPF 程序内部的安全字段

`security_bpf_prog_free`

清理 BPF 程序内部的安全字段

现在我们已经看到它们，可以明显看出 LSM BPF 钩子背后的理念是，它们可以为 eBPF 对象提供逐个对象的保护，以确保只有具有适当权限的用户才能对映射和程序执行操作。

# 结论

安全并不是一种可以普遍应用于所有需要保护的东西的东西。能够在不同层面和不同方式上确保系统安全非常重要，不管你相不相信，确保系统安全的最佳方法是通过堆叠具有不同视角的不同层次，以便于被攻破的层次不会导致能够访问整个系统。内核开发人员在提供了一组不同层次和交互点的同时也做得非常出色；我们希望能够让您充分了解这些层次是什么以及如何使用 BPF 程序与它们交互。
