# 20 提高你的参数化脚本

在上一章中，我们给你留下了一个相当酷的脚本，它已经被参数化了。参数化脚本的想法是，其他人可以运行脚本而无需担心或修改其内容。脚本用户通过指定的接口——参数——提供输入，而且他们只能更改这些。在这一章中，我们将更进一步。

注意：仅就示例而言，本章非常侧重于 Windows。

## 20.1 起点

为了确保我们处于同一页面上，让我们同意以列表 20.1 作为起点。这个脚本具有基于注释的帮助、两个输入参数以及使用这些输入参数的命令。自从上一章以来，我们做了一处小的改动：我们将输出改为选择对象，而不是我们在第 19.4 节中使用的格式化表格。

列表 20.1 起点：Get-DiskInventory.ps1

```
<#
.SYNOPSIS
Get-DiskInventory retrieves logical disk information from one or
more computers.
.DESCRIPTION
Get-DiskInventory uses CIM to retrieve the Win32_LogicalDisk
instances from one or more computers. It displays each disk's
drive letter, free space, total size, and percentage of free
space.
.PARAMETER computername
The computer name, or names, to query. Default: Localhost.
.PARAMETER drivetype
The drive type to query. See Win32_LogicalDisk documentation
for values. 3 is a fixed disk, and is the default.
.EXAMPLE
Get-DiskInventory -ComputerName SRV02 -drivetype 3
#>
param (
  $computername = 'localhost',
  $drivetype = 3
)
Get-CimInstance -class Win32_LogicalDisk -ComputerName $computername `
 -filter "drivetype=$drivetype" |
 Sort-Object -property DeviceID |
 Select-Object -property DeviceID,                                       ❶
     @{label='FreeSpace(MB)';expression={$_.FreeSpace / 1MB -as [int]}},
     @{label='Size(GB)';expression={$_.Size / 1GB -as [int]}},
     @{label='%Free';expression={$_.FreeSpace / $_.Size * 100 -as [int]}}
```

❶ 注意到我们使用了 Select-Object 而不是在第十九章中使用的 Format-Table。

为什么我们切换到 Select-Object 而不是 Format-Table？我们通常认为编写生成预格式化输出的脚本是一个坏主意。毕竟，如果有人需要将数据放在 CSV 文件中，而脚本输出的是格式化表格，那么这个人就会很不幸。通过这次修订，我们可以这样运行我们的脚本来获取格式化表格：

```
PS C:\> .\Get-DiskInventory | Format-Table
```

或者我们可以这样运行它来获取那个 CSV 文件：

```
PS C:\> .\Get-DiskInventory | Export-CSV disks.csv
```

重点是输出对象（Select-Object 所做的），而不是格式化显示，从长远来看，使我们的脚本更加灵活。

## 20.2 让 PowerShell 做艰苦的工作

我们将通过在脚本中添加一行来开启一些 PowerShell 魔法。从技术上讲，这使我们的脚本变成了一个 *高级脚本*，从而启用了一系列有用的 PowerShell 功能。以下列表显示了修订版。

列表 20.2 将 Get-DiskInventory.ps1 转换为高级脚本

```
<#
.SYNOPSIS
Get-DiskInventory retrieves logical disk information from one or
more computers.
.DESCRIPTION
Get-DiskInventory uses WMI to retrieve the Win32_LogicalDisk
instances from one or more computers. It displays each disk's
drive letter, free space, total size, and percentage of free
space.
.PARAMETER computername
The computer name, or names, to query. Default: Localhost.
.PARAMETER drivetype
The drive type to query. See Win32_LogicalDisk documentation
for values. 3 is a fixed disk, and is the default.
.EXAMPLE
Get-DiskInventory -ComputerName SRV02 -drivetype 3
#>
[CmdletBinding()]        ❶
param (
  $computername = 'localhost',
  $drivetype = 3
)
Get-CimInstance -class Win32_LogicalDisk -ComputerName $computername `
 -filter "drivetype=$drivetype" |
 Sort-Object -property DeviceID |
 Select-Object -property DeviceID,
     @{name='FreeSpace(MB)';expression={$_.FreeSpace / 1MB -as [int]}},
     @{name='Size(GB)';expression={$_.Size / 1GB -as [int]}},
     @{name='%Free';expression={$_.FreeSpace / $_.Size * 100 -as [int]}}
```

❶ [CmdletBinding()] 必须是注释帮助之后的第一个行；PowerShell 知道在这里寻找它。

正如所提到的，确保 `[CmdletBinding()]` 指令是脚本中注释帮助之后的第一个行是非常重要的。PowerShell 只知道在那里寻找它。通过这个改动，脚本将继续正常运行，但我们已经启用了一些很酷的功能，我们将在下一部分进行探索。

## 20.3 使参数成为强制性的

从这里，我们可以说我们已经完成了，但这现在不会很有趣，对吧？我们的脚本之所以以现有的形式存在，是因为它为 `-ComputerName` 参数提供了一个默认值——我们不确定是否真的需要它。我们更愿意提示输入该值，而不是依赖于硬编码的默认值。幸运的是，PowerShell 使这变得很容易——再次，只需添加一行即可，如下一列表所示。

列表 20.3 给 Get-DiskInventory.ps1 添加强制参数

```
<#
.SYNOPSIS
Get-DiskInventory retrieves logical disk information from one or
more computers.
.DESCRIPTION
Get-DiskInventory uses WMI to retrieve the Win32_LogicalDisk
instances from one or more computers. It displays each disk's
drive letter, free space, total size, and percentage of free
space.
.PARAMETER computername
The computer name, or names, to query. Default: Localhost.
.PARAMETER drivetype
The drive type to query. See Win32_LogicalDisk documentation
for values. 3 is a fixed disk, and is the default.
.EXAMPLE
Get-DiskInventory -ComputerName SRV02 -drivetype 3
#>
[CmdletBinding()]
param (
  [Parameter(Mandatory=$True)]   
  [string]$computername,        ❶
  [int]$drivetype = 3
)
Get-CimInstance -class Win32_LogicalDisk -ComputerName $computername `
 -filter "drivetype=$drivetype" |
 Sort-Object -property DeviceID |
 Select-Object -property DeviceID,
     @{name='FreeSpace(MB)';expression={$_.FreeSpace / 1MB -as [int]}},
     @{name='Size(GB)';expression={$_.Size / 1GB -as [int]}},
     @{name='%Free';expression={$_.FreeSpace / $_.Size * 100 -as [int]}}
```

❶ `[Parameter(Mandatory=$True)]`装饰器将使 PowerShell 在运行此脚本的人忘记提供计算机名称时提示输入。

除此之外

当某人运行你的脚本但没有提供必需的参数时，PowerShell 将提示他们提供该参数。有两种方法可以使 PowerShell 的提示对用户更有意义。

首先，使用一个好的参数名称。提示某人填写*comp*并不像提示他们提供`computerName`那样有帮助，因此尽量使用描述性且与其他 PowerShell 命令一致的参数名称。

您还可以添加一条帮助信息：

```
[Parameter(Mandatory=$True,HelpMessage="Enter a computer name to query")
```

一些 PowerShell 宿主将帮助信息作为提示的一部分显示，这使得对用户来说更加清晰，但并非每个宿主应用程序都会使用此属性，所以如果你在测试时并不总是看到它，请不要沮丧。我们仍然喜欢在编写供其他人使用的内容时包含它。这永远不会有害。但为了简洁，我们将在本章的运行示例中省略`HelpMessage`。

就这样一个*装饰器*，`[Parameter(Mandatory=$True)]`，将使 PowerShell 在运行此脚本的人忘记提供计算机名称时提示输入。为了进一步帮助 PowerShell，我们给我们的两个参数都指定了数据类型：`-ComputerName`为`[string]`，`-drivetype`为`[int]`（表示整数）。

将这些类型的属性添加到参数中可能会变得令人困惑，因此让我们更仔细地检查`Param()`块的语法——参见图 20.1。

![图片](img/CH20_F01_Plunk.png)

图 20.1 解构`Param()`块语法

这里有一些需要注意的重要事项：

+   所有参数都包含在`Param()`块的括号内。

+   单个参数可以包含多个装饰器，这些装饰器可以放在一行上，也可以像我们在图 20.1 中那样放在不同的行上。我们认为多行更易于阅读——但重要的是它们都是一起的。在这里，`Mandatory`属性仅修改`-ComputerName`；它对`-drivetype`没有任何影响。

+   除了最后一个参数名之外，每个参数名后面都跟着一个逗号。

+   为了提高可读性，我们喜欢在参数之间添加一个空行。我们认为这有助于更好地在视觉上分离它们，使`Param()`块不那么令人困惑。

+   我们将每个参数定义为一个变量`$computername`和`$drivetype`，但运行此脚本的人将把它们当作正常的 PowerShell 命令行参数，如`-ComputerName`和`-drivetype`。

现在试试看 Try 将脚本保存到列表 20.3 中，并在 shell 中运行它。不要指定`-ComputerName`参数，看看 PowerShell 如何提示你提供该信息。

## 20.4 添加参数别名

当你想到计算机名时，*computername* 是第一个出现在你脑海中的吗？可能不是。我们使用 `-ComputerName` 作为参数名，因为它与其他 PowerShell 命令的书写方式保持一致。看看 `Get-Service`、`Get-CimInstance`、`Get-Process` 等，你会在它们上面看到 `-ComputerName` 参数。所以我们选择了这个。

但是，如果你更倾向于使用 `-host` 这样的名称，你可以将其添加为参数的另一个名称或别名。它只是另一个装饰器，如以下列表所示。然而，不要使用 `-hostname`，因为在使用 PowerShell 远程时，它会指示 SSH 连接。

列表 20.4 向 Get-DiskInventory.ps1 添加参数别名

```
<#
.SYNOPSIS
Get-DiskInventory retrieves logical disk information from one or
more computers.
.DESCRIPTION
Get-DiskInventory uses WMI to retrieve the Win32_LogicalDisk
instances from one or more computers. It displays each disk's
drive letter, free space, total size, and percentage of free
space.
.PARAMETER computername
The computer name, or names, to query. Default: Localhost.
.PARAMETER drivetype
The drive type to query. See Win32_LogicalDisk documentation
for values. 3 is a fixed disk, and is the default.
.EXAMPLE
Get-DiskInventory -ComputerName SRV02 -drivetype 3
#>
[CmdletBinding()]
param (
  [Parameter(Mandatory=$True)]
  [Alias('host')]               ❶
  [string]$computername,
  [int]$drivetype = 3
)
Get-CimInstance -class Win32_LogicalDisk -ComputerName $computername `
 -filter "drivetype=$drivetype" |
 Sort-Object -property DeviceID |
 Select-Object -property DeviceID,
     @{name='FreeSpace(MB)';expression={$_.FreeSpace / 1MB -as [int]}},
     @{name='Size(GB)';expression={$_.Size / 1GB -as [int]}},
     @{name='%Free';expression={$_.FreeSpace / $_.Size * 100 -as [int]}}
```

❶ 这个添加是 `-ComputerName` 参数的一部分；它对 `-drivetype` 没有影响。

通过这个小的改动，我们现在可以运行以下命令：

```
PS C:\> .\Get-DiskInventory -host SRV02
```

注意 记住，你只需要输入足够多的参数名，以便 PowerShell 能够理解你指的是哪个参数。在这个例子中，`-host` 已经足够 PowerShell 识别 `-hostname`。我们也可以输入完整的名称。

再次强调，这个新的添加是 `-ComputerName` 参数的一部分；它对 `-drivetype` 没有影响。现在 `-ComputerName` 参数的定义占据了三行文本，尽管我们也可以将所有内容放在一行中：

```
[Parameter(Mandatory=$True)][Alias('hostname')][string]$computername,
```

我们只是觉得这样读起来更困难。

## 20.5 验证参数输入

让我们稍微玩一下 `-drivetype` 参数。根据 `Win32_LogicalDisk` WMI 类的 MSDN 文档（搜索类名，其中一个顶部结果将是文档），驱动器类型 3 是本地硬盘。类型 2 是可移动磁盘，它应该也有大小和可用空间测量。驱动器类型 1、4、5 和 6 没有那么有趣（还有谁现在还在使用 RAM 驱动器，类型 6？），在某些情况下，它们可能没有可用空间（类型 5，对于光盘）。因此，我们希望阻止任何人在运行我们的脚本时使用这些类型。这个列表显示了我们需要做的微小改动。

列表 20.5 向 Get-DiskInventory.ps1 添加参数验证

```
<#
.SYNOPSIS
Get-DiskInventory retrieves logical disk information from one or
more computers.
.DESCRIPTION
Get-DiskInventory uses WMI to retrieve the Win32_LogicalDisk
instances from one or more computers. It displays each disk's
drive letter, free space, total size, and percentage of free
space.
.PARAMETER computername
The computer name, or names, to query. Default: Localhost.
.PARAMETER drivetype
The drive type to query. See Win32_LogicalDisk documentation
for values. 3 is a fixed disk, and is the default.
.EXAMPLE
Get-DiskInventory -ComputerName SRV02 -drivetype 3
#>
[CmdletBinding()]
param (
  [Parameter(Mandatory=$True)]
  [Alias('hostname')]   
  [string]$computername,
  [ValidateSet(2,3)]       ❶
  [int]$drivetype = 3
)
Get-CimInstance -class Win32_LogicalDisk -ComputerName $computername `
 -filter "drivetype=$drivetype" |
 Sort-Object -property DeviceID |
 Select-Object -property DeviceID,
     @{name='FreeSpace(MB)';expression={$_.FreeSpace / 1MB -as [int]}},
     @{name='Size(GB)';expression={$_.Size / 1GB -as [int]}},
     @{name='%Free';expression={$_.FreeSpace / $_.Size * 100 -as [int]}}
```

❶ 我们在脚本中添加了 `[ValidateSet(2,3)]` 来告诉 PowerShell，只有两个值，2 和 3，是被我们的 `-drivetype` 参数接受的，并且 3 是默认值。

你可以添加许多其他的验证技术到一个参数中，并且当这样做有意义时，你可以向同一个参数添加多个。运行 `help about_functions_advanced_parameters` 获取完整的列表。我们现在继续使用 `ValidateSet()`。

现在尝试一下 保存这个脚本并再次运行。尝试指定 `-drivetype 5` 并看看 PowerShell 会做什么。

## 20.6 通过冗长的输出添加温暖和舒适感

在第十七章中，我们提到了我们更喜欢使用`Write-Verbose`而不是`Write-Host`来生成一些喜欢看到脚本产生的逐步进度信息的用户的信息。现在是一个真正的例子的时候了。我们在下面的列表中添加了一些详细输出的消息。

列表 20.6 向 Get-DiskInventory.ps1 添加详细输出

```
<#
.SYNOPSIS
Get-DiskInventory retrieves logical disk information from one or
more computers.
.DESCRIPTION
Get-DiskInventory uses WMI to retrieve the Win32_LogicalDisk
instances from one or more computers. It displays each disk's
drive letter, free space, total size, and percentage of free
space.
.PARAMETER computername
The computer name, or names, to query. Default: Localhost.
.PARAMETER drivetype
The drive type to query. See Win32_LogicalDisk documentation
for values. 3 is a fixed disk, and is the default.
.EXAMPLE
Get-DiskInventory -ComputerName SRV02 -drivetype 3
#>
[CmdletBinding()]
param (
  [Parameter(Mandatory=$True)]
  [Alias('hostname')]   
  [string]$computername,
  [ValidateSet(2,3)]
  [int]$drivetype = 3
)
Write-Verbose "Connecting to $computername"              ❶
Write-Verbose "Looking for drive type $drivetype"        ❶
Get-CimInstance -class Win32_LogicalDisk -ComputerName $computername `
 -filter "drivetype=$drivetype" |
 Sort-Object -property DeviceID |
 Select-Object -property DeviceID,
     @{name='FreeSpace(MB)';expression={$_.FreeSpace / 1MB -as [int]}},
     @{name='Size(GB)';expression={$_.Size / 1GB -as [int]}},
     @{name='%Free';expression={$_.FreeSpace / $_.Size * 100 -as [int]}}
Write-Verbose "Finished running command"                 ❶
```

❶ 添加了三个详细输出消息

现在以两种方式尝试运行此脚本。第一次尝试不应该显示任何详细输出：

```
PS C:\> .\Get-DiskInventory -ComputerName localhost
```

现在是第二次尝试，我们希望显示详细输出：

```
PS C:\> .\Get-DiskInventory -ComputerName localhost -verbose
```

现在试试看 这当你亲自看到的时候会更酷。按照我们在这里展示的方式运行脚本，亲自看看差异。

这有多酷？当你想要详细的输出（如代码列表 20.6 所示），你可以得到它——而且你根本不需要编写`-Verbose`参数！当你添加`[Cmdlet-Binding()]`时，它会免费提供。而且一个真正酷的地方是，它还会激活脚本中每个命令的详细输出！所以任何设计用于产生详细输出的命令都将“自动”这样做。这种技术使得开启和关闭详细输出变得很容易，比`Write-Host`更加灵活。而且你不需要与`$VerbosePreference`变量纠缠，以使输出显示在屏幕上。

注意，在详细输出中，我们如何利用 PowerShell 的双引号技巧：通过在双引号内包含一个变量（`$computername`），输出能够包含变量的内容，这样我们就可以看到 PowerShell 在做什么。

## 20.7 实验室

这个实验室要求你回忆一下第十九章中学到的一些内容，因为你将使用以下命令，对其进行参数化，并将其转换为脚本——就像你在第十九章的实验室中所做的那样。但这次我们还想让你将`-ComputerName`参数设置为必填项，并给它一个`host`别名。让脚本在运行此命令前后显示详细输出。记住，你必须参数化计算机名——但在这个情况下，你只需要参数化这一项。

在开始修改之前，确保按原样运行命令，以确保它在你的系统上工作：

```
Get-CimInstance win32_networkadapter -ComputerName localhost |
 where { $_.PhysicalAdapter } |
 select MACAddress,AdapterType,DeviceID,Name,Speed
```

重申一下，这是你的完整任务列表：

+   在修改之前确保命令按原样运行。

+   参数化计算机名。

+   使计算机名参数成为必填项。

+   给计算机名参数一个别名，`hostname`。

+   添加基于注释的帮助，至少包含一个如何使用脚本的示例。

+   在修改命令前后添加详细输出。

+   将脚本保存为 Get-PhysicalAdapters.ps1。

## 20.8 实验室答案

```
<#
.Synopsis
Get physical network adapters
.Description
Display all physical adapters from the Win32_NetworkAdapter class.
.Parameter Computername
The name of the computer to check.
.Example
PS C:\> c:\scripts\Get-PhysicalAdapters -computer SERVER01
#>
[cmdletbinding()]
Param (
[Parameter(Mandatory=$True,HelpMessage="Enter a computername to query")]
[alias('host')]
[string]$Computername
)
Write-Verbose "Getting physical network adapters from $computername"
Get-CimInstance -class win32_networkadapter –computername $computername |
 where { $_.PhysicalAdapter } |
 select MACAddress,AdapterType,DeviceID,Name,Speed
Write-Verbose "Script finished."
```

除此之外

将你到目前为止学到的知识应用到我们在第十九章中制作的脚本 Get-FilePath.ps1（列表 19.2）中，

+   使其成为一个高级函数。

+   添加必填参数。

+   添加详细输出。

+   调整格式以使导出到 CSV 文件更容易。
