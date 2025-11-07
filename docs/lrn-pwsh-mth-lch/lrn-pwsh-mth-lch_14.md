# 14 使用后台作业进行多任务处理

每个人总是告诉你要“多任务处理”，对吧？为什么 PowerShell 不能通过一次做几件事来帮助你呢？事实证明，PowerShell 确实可以这样做，特别是对于可能涉及多个目标计算机的长时间运行的任务。在深入本章之前，请确保你已经阅读了第十三章，因为我们将进一步探讨那些远程概念。

注意：在本章中，我们将使用大量的 Az 命令，这需要有效的 Azure 订阅。这些只是我们选择突出显示的示例。

## 14.1 让 PowerShell 同时执行多项任务

你应该将 PowerShell 视为一个单线程应用程序，这意味着它一次只能做一件事。你输入一个命令，按下 Enter，外壳等待该命令执行。你无法在第一个命令完成之前运行第二个命令。

但凭借其后台作业功能，PowerShell 有能力将命令移动到单独的后台线程或单独的后台 PowerShell 进程。这使命令能够在后台运行，同时你继续使用外壳执行其他任务。在运行命令之前，你必须做出这个决定；按下 Enter 后，你不能决定将长时间运行的命令移入后台。

命令进入后台后，PowerShell 提供了检查其状态、检索任何结果等机制。

## 14.2 同步与异步

让我们先澄清一些术语。PowerShell 以同步方式运行正常命令，这意味着你按下 Enter 然后等待命令完成。将作业移动到后台允许它异步运行，这意味着你可以在命令完成时继续使用外壳执行其他任务。让我们看看以这两种方式运行命令的一些重要区别：

+   当你同步运行一个命令时，你可以响应输入请求。当你后台运行命令时，没有机会看到输入请求——实际上，它们会阻止命令运行。

+   同步命令在出错时会产生错误消息。后台命令会产生错误，但你不会立即看到它们。如果需要，你必须做出安排来捕获它们。（第二十四章讨论了如何做到这一点。）

+   如果你省略了同步命令上的必需参数，PowerShell 可以提示你输入缺失的信息。在后台命令上，它不能这样做，因此命令将失败。

+   同步命令的结果一旦可用就开始显示。对于后台命令，你等待命令运行完成，然后检索缓存的成果。

我们通常同步运行命令以测试它们并确保它们正常工作，只有在我们知道它们已经完全调试并且按预期工作后，才在后台运行它们。我们采取这些措施以确保命令可以无问题运行，并且有最大的机会在后台完成。PowerShell 将后台命令称为 *作业*。你可以通过多种方式创建作业，并且可以使用多个命令来管理它们。

超越

从技术上来说，本章所讨论的工作岗位只是你可能会遇到的一小部分工作。工作（Job）是 PowerShell 的一个扩展点，这意味着任何人（无论是微软内部人员还是第三方）都可以创建其他被称为工作（Job）的东西，这些工作看起来和运作方式与本章所描述的略有不同。当你为了各种目的扩展 shell 时，可能会遇到其他类型的工作。我们希望你能理解这个细节，并知道你在本章所学的内容仅适用于 PowerShell 自带的、常规的工作岗位。

## 14.3 创建进程工作（Process Job）

我们首先介绍的工作类型可能是最简单的：进程工作（Process Job）。这是一个在机器上的另一个 PowerShell 进程中后台运行的命令。

要启动这些作业之一，你使用 `Start-Job` 命令。一个 `-ScriptBlock` 参数允许你指定要运行的命令（或命令序列）。PowerShell 会自动生成一个默认作业名称（`Job1`、`Job2` 等），或者你可以使用 `-Name` 参数指定一个自定义作业名称。你不必指定脚本块，也可以指定 `-FilePath` 参数，让作业执行一个包含多个命令的整个脚本文件。以下是一个简单的示例：

```
PS /Users/travisp/> start-job -scriptblock { gci }
Id   Name  PSJobTypeName  State    HasMoreData     Location    Command
--   ----  -------------  -----    -----------     --------    -------
1    Job1  BackgroundJob  Running  True            localhost    gci
```

该命令创建作业对象，正如前面的示例所示，作业立即开始运行。作业还分配了一个顺序作业 ID 号，如表中所示。

该命令还有一个 `-WorkingDirectory` 参数，允许你更改工作在文件系统中的起始位置。默认情况下，它始终从主目录开始。绝不要在后台作业中假设文件路径：使用绝对路径以确保你可以引用作业命令可能需要的任何文件，或者使用 `-WorkingDirectory` 参数。以下是一个示例：

```
PS /Users/travisp/> start-job -scriptblock { gci } -WorkingDirectory /tmp
Id   Name  PSJobTypeName  State    HasMoreData     Location    Command
--   ----  -------------  -----    -----------     --------    -------
3    Job3  BackgroundJob  Running  True            localhost    gci
```

留意细节的读者会注意到，我们创建的第一个作业被命名为 `Job1` 并分配了 ID `1`，但第二个作业是 `Job3`，ID 为 `3`。实际上，每个作业至少有一个 *子作业*，第一个子作业（`Job1` 的子作业）被命名为 `Job2` 并分配了 ID `2`。我们将在本章后面讨论子作业。

这里有一个需要注意的事项：尽管进程工作在本地运行，但它们确实需要启用 PowerShell 远程，这在第十三章中已经介绍过。

## 14.4 创建线程工作（Thread Job）

我们还想讨论 PowerShell 中作为其一部分提供的第二种作业类型。它被称为 *线程作业*。线程作业不会在完全不同的 PowerShell 进程中运行，而是在 *同一* 进程中启动另一个线程。以下是一个示例：

```
PS /Users/travisp/> start-threadjob -scriptblock { gci }
Id   Name  PSJobTypeName  State    HasMoreData     Location    Command
--   ----  -------------  -----    -----------     --------    -------
1    Job1  ThreadJob      Running  False           PowerShell   gci
```

看起来与上一个作业输出非常相似，对吧？只有两个区别——`PSJobTypeName`，它是 `ThreadJob`，以及 `Location`，它是 `PowerShell`。这告诉我们这个作业是在我们目前正在使用的进程中运行的，但在不同的线程中。

由于启动新线程的开销比启动新进程快得多，因此线程作业非常适合短期脚本和您希望快速启动并在后台运行的命令。相反，您可以使用进程作业在您的机器上运行长时间运行的脚本。

注意：尽管线程作业启动得更快，但请记住，在进程开始变慢之前，一个进程只能同时运行这么多线程。PowerShell 内置了一个“节流限制”为 10，以帮助您防止 PowerShell 过度负载。这意味着一次只能运行 10 个线程作业。如果您想提高限制，可以这样做。只需指定 `-ThrottleLimit` 参数并传递您想要使用的新限制。如果您一次启动 50、100 或 200 个线程作业，您最终会看到收益递减。请记住这一点。

## 14.5 作为作业的远程

让我们回顾一下您可以使用来创建新作业的最终技术：PowerShell 的远程功能，您在第十三章中已经学过。有一个重要的区别：您在 `-scriptblock`（或 `-command`，它是同一参数的别名）中指定的任何命令都将并行传输到您指定的每台计算机。一次最多可以联系 32 台计算机（除非您修改 `-throttleLimit` 参数以允许更多或更少的计算机），因此如果您指定了超过 32 台计算机的名称，只有前 32 台将启动。其余的将在第一组开始完成后启动，并且顶级作业将在所有计算机完成后显示完成状态。

与启动作业的其他两种方式不同，这种技术要求您在每个目标计算机上安装 PowerShell v6 或更高版本，并在每个目标计算机上的 PowerShell 中启用 SSH 远程。因为命令在每台远程计算机上物理执行，所以您正在分配计算工作负载，这可以帮助提高复杂或长时间运行的命令的性能。结果将返回到您的计算机，并在您准备好查看之前与作业一起存储。

在以下示例中，您还将看到 `-JobName` 参数，它允许您指定除无聊的默认值以外的作业名称：

```
PS C:\> invoke-command -command { get-process } 
-hostname (get-content .\allservers.txt ) 
-asjob -jobname MyRemoteJob
WARNING: column "Command" does not fit into the display and was removed.
Id              Name            State      HasMoreData     Location
--              ----            -----      -----------     --------
8               MyRemoteJob     Running    True            server-r2,lo...
```

## 14.6 野外的作业

我们想用这个部分来展示一个 PowerShell 模块，该模块公开了自己的 PSJobs，这样您可以在 PowerShell 的旅程中寻找这种模式。以命令 `New-AzVm` 为例：

```
PS /Users/travisp/> gcm New-AzVM -Syntax

New-AzVM -Name <string> -Credential <pscredential> [-ResourceGroupName <string>]
 [-Location <string>] [-Zone <string[]>] [-VirtualNetworkName <string>] [-AddressPrefix <string>]
 [-SubnetName <string>] [-SubnetAddressPrefix <string>] [-PublicIpAddressName <string>]
 [-DomainNameLabel <string>] [-AllocationMethod <string>] [-SecurityGroupName <string>]
 [-OpenPorts <int[]>] [-Image <string>] [-Size <string>] [-AvailabilitySetName <string>]
 [-SystemAssignedIdentity] [-UserAssignedIdentity <string>] [-AsJob] [-DataDiskSizeInGb <int[]>]
 [-EnableUltraSSD] [-ProximityPlacementGroup <string>] [-HostId <string>]
 [-DefaultProfile <IAzureContextContainer>] [-WhatIf] [-Confirm] [<CommonParameters>]

New-AzVM [-ResourceGroupName] <string> [-Location] <string> [-VM] <PSVirtualMachine>
 [[-Zone] <string[]>] [-DisableBginfoExtension] [-Tag <hashtable>] [-LicenseType <string>]
 [-AsJob] [-DefaultProfile <IAzureContextContainer>] [-WhatIf] [-Confirm] [<CommonParameters>]

New-AzVM -Name <string> -DiskFile <string> [-ResourceGroupName <string>]
 [-Location <string>] [-VirtualNetworkName <string>] [-AddressPrefix <string>]
 [-SubnetName <string>] [-SubnetAddressPrefix <string>] [-PublicIpAddressName <string>]
 [-DomainNameLabel <string>] [-AllocationMethod <string>] [-SecurityGroupName <string>]
 [-OpenPorts <int[]>] [-Linux] [-Size <string>] [-AvailabilitySetName <string>]
 [-SystemAssignedIdentity] [-UserAssignedIdentity <string>] [-AsJob]
 [-DataDiskSizeInGb <int[]>] [-EnableUltraSSD] [-ProximityPlacementGroup <string>]
 [-HostId <string>] [-DefaultProfile <IAzureContextContainer>] [-WhatIf] [-Confirm] [<CommonParameters>]
```

注意到一个熟悉的参数吗？`-AsJob`！让我们看看它在命令中做了什么：

```
PS /Users/travisp/> Get-Help New-AzVM -Parameter AsJob

-AsJob <System.Management.Automation.SwitchParameter>
    Run cmdlet in the background and return a Job to track progress.

    Required?                    false
    Position?                    named
    Default value                False
    Accept pipeline input?       False
    Accept wildcard characters?  false
```

此参数告诉`New-AzVM`返回一个`Job`。如果我们执行此命令，并在为虚拟机输入用户名和密码后，我们会看到我们得到了一个`Job`。

```
PS /Users/travisp/> New-AzVm -Name myawesomevm -Image UbuntuLTS  -AsJob

cmdlet New-AzVM at command pipeline position 1
Supply values for the following parameters:
Credential
User: azureuser
Password for user azureuser: ***********

Id Name            PSJobTypeName   State   HasMoreData  Location  Command
-- ----            -------------   -----   -----------  --------  -------
8  Long Running O... AzureLongRunni... Running True     localhost New-AzVM 
```

这之所以如此出色，是因为您可以像管理从`Start-Job`或`Start-ThreadJob`返回的作业一样管理这些作业。您将在稍后看到我们如何管理作业，但这是一个自定义作业可能出现的示例。寻找`-AsJob`参数！

## 14.7 获取作业结果

在启动作业后，您可能首先想要做的是检查作业是否已完成。`Get-Job`命令检索系统当前定义的每个作业，并显示每个作业的状态：

```
PS /Users/travisp/> get-job
Id Name            PSJobTypeName   State     HasMoreData  Location  Command
-- ----            -------------   -----     -----------  --------  -------
1  Job1            BackgroundJob   Completed True         localhost  gci
3  Job3            BackgroundJob   Completed True         localhost  gci
5  Job5            ThreadJob       Completed True         PowerShell gci
8  Job8            BackgroundJob   Completed True         server-r2, lo...
11 MyRemoteJob     BackgroundJob   Completed True         server-r2, lo...
13 Long Running O... AzureLongRunni... Running   True     localhost  New-AzVM
```

您也可以通过其 ID 或其名称检索特定作业。我们建议您这样做，并将结果管道到`Format-List *`，因为您已经收集了一些有价值的信息：

```
PS /Users/travisp/> get-job -id 1 | format-list *
State         : Completed
HasMoreData   : True
StatusMessage :
Location      : localhost
Command       :  gci
JobStateInfo  : Completed
Finished      : System.Threading.ManualResetEvent
InstanceId    : e1ddde9e-81e7-4b18-93c4-4c1d2a5c372c
Id            : 1
Name          : Job1
ChildJobs     : {Job2}
PSBeginTime   : 12/12/2019 7:18:58 PM
PSEndTime     : 12/12/2019 7:18:58 PM
PSJobTypeName : BackgroundJob
Output        : {}
Error         : {}
Progress      : {}
Verbose       : {}
Debug         : {}
Warning       : {}
Information   : {}
```

现在尝试一下 如果您正在跟随，请记住，您的作业 ID 和名称可能与我们的不同。关注`Get-Job`的输出以检索您的作业 ID 和名称，并在示例中替换您的信息。此外，请记住，Microsoft 在过去的几个 PowerShell 版本中扩展了作业对象，因此您查看所有属性时的输出可能不同。

`ChildJobs`属性是最重要的信息之一，我们将在稍后讨论。要从作业中检索结果，请使用`Receive-Job`。但在运行此命令之前，您需要了解一些事情：

+   您必须指定您想要接收结果的作业。您可以通过作业 ID 或作业名称来做到这一点，或者通过使用`Get-Job`并使用管道将它们传递到`Receive-Job`来获取作业。

+   如果您接收父作业的结果，这些结果将包括所有子作业的所有输出。或者，您可以选择从一个或多个子作业中获取结果。

+   通常，从作业中接收结果会将其从作业输出缓存中清除，因此您无法再次获取它们。指定`-keep`以在内存中保留结果的副本。或者，如果您想保留一个副本以供工作使用，可以将结果输出到 CLIXML 文件。

+   作业结果可能是反序列化对象，您在第十三章中了解过。这些是从它们生成时的快照，它们可能没有任何您可以执行的方法。但根据需要，您可以将作业结果直接管道到`Sort-Object`、`-Format-List`、`Export-CSV`、`ConvertTo-HTML`、`Out-File`等命令。

这里有一个例子：

```
PS /Users/travisp/> receive-job -id 1

    Directory: /Users/travisp

Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
d----          11/24/2019 10:53 PM                Code
d----          11/18/2019 11:23 PM                Desktop
d----           9/15/2019  9:12 AM                Documents
d----           12/8/2019 11:04 AM                Downloads
d----           9/15/2019  7:07 PM                Movies
d----           9/15/2019  9:12 AM                Music
d----           9/15/2019  6:51 PM                Pictures
d----           9/15/2019  9:12 AM                Public
```

上述输出显示了一组有趣的结果。这是启动此作业的命令的快速提醒：

```
PS /Users/travisp/> start-job -scriptblock { gci }
```

当我们从`Job1`接收结果时，我们没有指定`-keep`。如果我们尝试再次获取相同的结果，我们将一无所获，因为结果不再与作业一起缓存：

```
PS /Users/travisp/> receive-job -id 1
```

这是强制结果保留在内存中的方法：

```
PS /Users/travisp/> receive-job -id 3 –keep

    Directory: /Users/travisp

Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
d----          11/24/2019 10:53 PM                Code
d----          11/18/2019 11:23 PM                Desktop
d----           9/15/2019  9:12 AM                Documents
d----           12/8/2019 11:04 AM                Downloads
d----           9/15/2019  7:07 PM                Movies
d----           9/15/2019  9:12 AM                Music
d----           9/15/2019  6:51 PM                Pictures
d----           9/15/2019  9:12 AM                Public  
```

你最终会想要释放用于缓存作业结果的内存，我们稍后会讨论这一点。但首先，让我们看看一个将作业结果直接管道传输到另一个 cmdlet 的快速示例：

```
PS /Users/travisp> receive-job -name myremotejob | sort-object PSComputerName 
 ➥ | Format-Table -groupby PSComputerName
   PSComputerName: localhost
NPM(K)    PM(M)      WS(M) CPU(s)     Id ProcessName PSComputerName
------    -----      ----- ------     -- ----------- --------------
     0        0      56.92   0.70    484 pwsh        localhost
     0        0     369.20  70.17   1244 Code        localhost
     0        0      71.92   0.20   3492 pwsh        localhost
     0        0     288.96  15.31    476 iTerm2      localhost
```

这是使用 `Invoke-Command` 启动的作业。该 cmdlet 添加了 `PSComputerName` 属性，这样我们就可以跟踪哪个对象来自哪个计算机。因为我们从顶级作业中检索了结果，这包括我们指定的所有计算机，这使得这个命令可以根据计算机名称对它们进行排序，并为每个计算机创建一个单独的表格组。`Get-Job` 也可以让你了解哪些作业还有剩余结果：

```
PS /Users/travisp> get-job
Id Name            PSJobTypeName   State     HasMoreData  Location  Command
-- ----            -------------   -----     -----------  --------  -------
1  Job1            BackgroundJob   Completed False        localhost  gci
3  Job3            BackgroundJob   Completed True         localhost  gci
5  Job5            ThreadJob       Completed True         PowerShell gci
8  Job8            BackgroundJob   Completed True         server-r2, lo...
11 MyRemoteJob     BackgroundJob   Completed False        server-r2, lo...
13 Long Running O... AzureLongRunni... Running   True     localhost  New-AzVM
```

当没有输出与该作业一起缓存时，`HasMoreData` 列将显示为 `False`。在 `Job1` 和 `MyRemoteJob` 的情况下，我们已经收到了那些结果，当时没有指定 `-keep`。

## 14.8 与子作业一起工作

我们之前提到，大多数作业由一个顶级父作业和至少一个子作业组成。让我们再次看看一个作业：

```
PS /Users/travisp> get-job -id 1 | format-list *
State         : Completed
HasMoreData   : True
StatusMessage :
Location      : localhost
Command       :  dir
JobStateInfo  : Completed
Finished      : System.Threading.ManualResetEvent
InstanceId    : e1ddde9e-81e7-4b18-93c4-4c1d2a5c372c
Id            : 1
Name          : Job1
ChildJobs     : {Job2}
PSBeginTime   : 12/27/2019 2:34:25 PM
PSEndTime     : 12/27/2019 2:34:29 PM
PSJobTypeName : BackgroundJob
Output        : {}
Error         : {}
Progress      : {}
Verbose       : {}
Debug         : {}
Warning       : {}
Information   : {}
```

现在尝试一下 不要跟随这一部分，因为如果你到现在一直跟着做，你已经收到了 `Job1` 的结果。如果你想尝试这个，通过运行 `Start-Job` `-script` `{` `dir` `}` 开始一个新的作业，并使用那个新作业的 ID 而不是我们示例中使用的 ID 号 1。

你可以看到 `Job1` 有一个子作业，`Job2`。现在你知道它的名字，你可以直接获取它：

```
PS /Users/travisp> get-job -name job2 | format-list *
State         : Completed
StatusMessage :
HasMoreData   : True
Location      : localhost
Runspace      : System.Management.Automation.RemoteRunspace
Debugger      : System.Management.Automation.RemotingJobDebugger
IsAsync       : True
Command       :  dir
JobStateInfo  : Completed
Finished      : System.Threading.ManualResetEvent
InstanceId    : a21a91e7-549b-4be6-979d-2a896683313c
Id            : 2
Name          : Job2
ChildJobs     : {}
PSBeginTime   : 12/27/2019 2:34:25 PM
PSEndTime     : 12/27/2019 2:34:29 PM
PSJobTypeName :
Output        : {Applications, Code, Desktop, Documents, Downloads, Movies,            
            ➥ Music...}
Error         : {}
Progress      : {}
Verbose       : {}
Debug         : {}
Warning       : {}
Information   : {}
```

有时候一个作业有太多的子作业，无法以那种形式列出，因此你可能想以不同的方式列出它们，如下所示：

```
PS /Users/travisp> get-job -id 1 | select-object -expand childjobs
Id Name            PSJobTypeName   State     HasMoreData  Location  Command
-- ----            -------------   -----     -----------  --------  -------
2  Job2                            Completed True         localhost  gci
```

这个技术为作业 ID 1 创建了一个子作业表，表可以扩展到任何长度，以列出所有子作业。你可以通过指定其名称或 ID 来接收任何单个子作业的结果，使用 `Receive-Job`。

## 14.9 管理作业的命令

作业还使用三个额外的命令。对于这些命令中的每一个，你可以通过提供其 ID、提供其名称或获取作业并将其管道传输到这些 cmdlet 之一来指定一个作业：

+   `Remove-Job`——这个命令会从内存中删除一个作业及其任何仍然缓存的输出。

+   `Stop-Job`——如果一个作业看起来卡住了，这个命令会终止它。你仍然可以接收到目前为止生成的任何结果。

+   `Wait-Job`——如果脚本将要启动一个作业或多个作业，而你又想让脚本仅在作业完成后继续执行，这个命令非常有用。这个命令会强制 shell 停止并等待作业（或多个作业）完成，然后允许 shell 继续执行。

例如，为了删除我们已经收到输出的作业，我们会使用以下命令：

```
PS /Users/travisp> get-job | where { -not $_.HasMoreData } | remove-job
PS /Users/travisp> get-job
Id Name            PSJobTypeName   State     HasMoreData  Location  Command
-- ----            -------------   -----     -----------  --------  -------
3  Job3            BackgroundJob   Completed True         localhost  gci
5  Job5            ThreadJob       Completed True         PowerShell gci
8  Job8            BackgroundJob   Completed True         server-r2, lo...
13 Long Running O... AzureLongRunni... Completed True     localhost New-AzVM
```

作业也可能失败，这意味着它们的执行过程中出现了问题。考虑以下示例：

```
PS /Users/travisp> invoke-command -command { nothing } -hostname notonline
    -asjob -jobname ThisWillFail
Id Name            PSJobTypeName   State     HasMoreData  Location  Command
-- ----            -------------   -----     -----------  --------  -------
11 ThisWillFail    BackgroundJob   Failed    False        notonline  nothing
```

在这里，我们使用一个无效的命令并针对一个不存在的计算机启动了一个作业。作业立即失败，如其状态所示。此时我们不需要使用 `Stop-Job`；作业没有在运行。但我们可以获取其子作业的列表：

```
PS /Users/travisp> get-job -id 11 | format-list *
State         : Failed
HasMoreData   : False
StatusMessage :
Location      : notonline
Command       :  nothing
JobStateInfo  : Failed
Finished      : System.Threading.ManualResetEvent
InstanceId    : d5f47bf7-53db-458d-8a08-07969305820e
Id            : 11
Name          : ThisWillFail
ChildJobs     : {Job12}
PSBeginTime   : 12/27/2019 2:45:12 PM
PSEndTime     : 12/27/2019 2:45:14 PM
PSJobTypeName : BackgroundJob
Output        : {}
Error         : {}
Progress      : {}
Verbose       : {}
Debug         : {}
Warning       : {}
Information   : {}
```

然后，我们可以获取那个子作业：

```
PS /Users/travisp> get-job -name job12
Id Name  PSJobTypeName   State     HasMoreData  Location  Command
-- ----  -------------   -----     -----------  --------  -------
12 Job12                Failed     False        notonline  nothing
```

正如你所见，这个任务没有创建任何输出，所以你不会有任何结果可以检索。但任务中的错误存储在结果中，你可以通过使用 `Receive-Job` 来获取它们：

```
PS /Users/travisp> receive-job -name job12
OpenError: [notonline] The background process reported an error with the 
➥ following message: The SSH client session has ended with error message: 
➥ ssh: Could not resolve hostname notonline: nodename nor servname provided, 
➥ or not known.
```

完整的错误信息要长得多；我们在这里截断以节省空间。你会注意到错误中包含了错误来源的主机名 `[notonline]`。如果只有一台计算机无法连接会发生什么？让我们试试：

```
PS /Users/travisp> invoke-command -command { nothing } 
-computer notonline,server-r2 -asjob -jobname ThisWilLFail
Id Name         PSJobTypeName   State    HasMoreData  Location        Command
-- ----         -------------   -----    ---------    --------        -------
13 ThisWillFail BackgroundJob   Running  True         notonline,lo... nothing
```

稍微等待一下，我们运行以下命令：

```
PS /Users/travisp> get-job 13
Id Name         PSJobTypeName   State    HasMoreData  Location        Command
-- ----         -------------   -----    -----------  --------        -------
13 ThisWillFail BackgroundJob   Failed   False        notonline,lo... nothing
```

任务仍然失败，但让我们看看单个子任务：

```
PS /Users/travisp> get-job -id 13 | select -expand childjobs
Id Name  PSJobTypeName   State     HasMoreData  Location        Command
-- ----  -------------   -----     -----------  --------        -------
14 Job14                 Failed    False        notonline       nothing
15 Job15                 Failed    False        localhost       nothing
```

好吧，它们两个都失败了。我们感觉我们知道为什么 `Job14` 不工作，但 `Job15` 有什么问题？

```
PS /Users/travisp> receive-job -name job15
Receive-Job : The term 'nothing' is not recognized as the name of a cmdlet
, function, script file, or operable program. Check the spelling of the na
me, or if a path was included, verify that the path is correct and try aga
in.
```

哎，没错，我们告诉它运行一个虚假命令。正如你所见，每个子任务可能会因为不同的原因而失败，而 PowerShell 会单独跟踪每一个。

## 14.10 常见混淆点

任务通常很简单，但我们见过有人做了一件事导致混淆。不要这样做：

```
PS /Users/travisp> invoke-command -command { Start-Job -scriptblock { dir } } 
-hostname Server-R2
```

这样做会启动到 `Server-R2` 的临时连接并启动一个本地任务。不幸的是，这个连接立即终止，所以你无法重新连接并检索该任务。总的来说，不要混合使用启动任务的三种方式。以下也是一个坏主意：

```
PS /Users/travisp> start-job -scriptblock { invoke-command -command { dir }
-hostname SERVER-R2 }
```

这完全是多余的；保留 `Invoke-Command` 部分，并使用 `-AsJob` 参数使其在后台运行。

更少混淆但同样有趣的是，新用户经常询问关于任务的问题。其中最重要的问题可能是，“我们能看到别人启动的任务吗？”答案是不了。任务和线程任务完全包含在 PowerShell 进程中，尽管你可以看到另一个用户正在运行 PowerShell，但你无法看到那个进程内部。就像任何其他应用程序一样：你可以看到另一个用户正在运行 Microsoft Word，例如，但你无法看到该用户正在编辑哪些文档，因为那些文档完全存在于 Word 的进程内部。

任务只在你打开 PowerShell 会话期间存在。在你关闭它之后，其中定义的任何任务都会消失。任务不在 PowerShell 之外定义，因此它们依赖于其进程继续运行以维持自身。

## 14.11 实验室

以下练习应该有助于你了解如何在 PowerShell 中处理各种类型的任务和作业。在完成这些练习时，不要觉得你必须写出一个单行解决方案。有时将事情分解成单独的步骤更容易。

1.  创建一个一次性线程任务来查找文件系统中的所有文本文件（`*.txt`）。任何可能需要很长时间才能完成的任务都是使用任务的绝佳候选。

1.  你会意识到识别你服务器上的一些文本文件将是有帮助的。你将如何从任务 1 在一组远程计算机上运行相同的命令？

1.  你会用哪个 cmdlet 来获取任务的结果，以及你将如何将结果保存在任务队列中？

## 14.12 实验答案

1.  `Start-ThreadJob {gci / -recurse –filter '*.txt'}`

1.  `Invoke-Command –scriptblock {gci / -recurse –filter *.txt}`

    `–computername (get-content computers.txt) -asjob`

1.  `Receive-Job –id 1 –keep`

    当然，你会使用适用的作业 ID 或作业名称。
