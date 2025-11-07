# 22 使用他人的脚本

尽管我们希望你能从头开始构建自己的 PowerShell 命令和脚本，但我们也意识到你将严重依赖互联网上的示例。无论你是从某人的博客中重新利用示例，还是调整你在在线脚本存储库中找到的脚本，能够重用他人的 PowerShell 脚本是一项重要的核心技能。在本章中，我们将带你了解我们理解他人脚本并将其变为己用的过程。

感谢感谢归功于 Brett Miller，他为我们提供了本章中使用的脚本。我们故意要求他提供一个不太完美的脚本，这个脚本不一定反映我们通常喜欢看到的最佳实践。在某些情况下，我们甚至 *恶化* 了此脚本，以便使本章更好地反映现实世界。我们真正感谢他对这个学习练习的贡献！

注意，我们之所以特别选择这些脚本，是因为它们使用了我们尚未教授你的高级 PowerShell 功能。再次强调，我们认为这是现实的：你可能会遇到看起来不熟悉的东西，而这个练习的一部分就是如何快速弄清楚脚本在做什么，即使你对脚本使用的每个技术并不完全训练有素。

## 22.1 脚本

这是一个真实的世界场景，我们的大多数学生都经历过。他们遇到问题，上网，找到一个能完成他们所需工作的脚本。理解正在发生的事情非常重要。以下列表显示了完整的脚本，其标题为 Get-AdExistence.ps1。此脚本旨在与 Microsoft 的 AD cmdlets 一起工作。这只能在基于 Windows 的计算机上工作。如果你没有访问安装了 Active Directory 的 Windows 机器，你仍然可以跟随我们，因为我们将逐个分析这个脚本。

列表 22.1 Get-AdExistence.Ps1

```
<#
.Synopsis
   Checks if computer account exists for computer names provided
.DESCRIPTION
   Checks if computer account exists for computer names provided
.EXAMPLE
   Get-ADExistence $computers
.EXAMPLE
   Get-ADExistence "computer1","computer2"
#>
function Get-ADExistence{
    [CmdletBinding()]
    Param(
        # single or array of machine names
        [Parameter(Mandatory=$true,
                   ValueFromPipeline=$true,
                   ValueFromPipelineByPropertyName=$true,
                   HelpMessage="Enter one or multiple computer names")]
        [String[]]$Computers
     )
    Begin{}
    Process {
        foreach ($computer in $computers) {
            try {
                $comp = get-adcomputer $computer -ErrorAction stop
                $properties = @{computername = $computer
                                Enabled = $comp.enabled
                                InAD = 'Yes'}
            } 
            catch {
                $properties = @{computername = $computer
                                Enabled = 'Fat Chance'
                                InAD = 'No'}
            } 
            finally {
                $obj = New-Object -TypeName psobject -Property $properties
                Write-Output $obj
            }
        } #End foreach

    } #End Process
    End{}
} #End Function
```

### 22.1.1 参数块

首先是参数块，你已经在第十九章中学到了如何创建它：

```
Param(
        # single or array of machine names
        [Parameter(Mandatory=$true,
                   ValueFromPipeline=$true,
        [String[]]$Computers
     )
```

这个参数块看起来有些不同，但它似乎是在定义一个可以接受数组并强制性的 `-Computers` 参数。这是合理的。当你运行它时，你需要提供这些信息。接下来的几行更加神秘：

```
Begin{}
Process
```

我们还没有介绍过程块，但现在只需知道这是脚本的主要内容所在。我们将在 *Learn Scripting in a Month of* *Lunches*（Manning，2017）中更详细地介绍这一点。

### 22.1.2 过程块

我们还没有介绍 `Try Catch`，但很快就会介绍，不用担心。现在，只需知道你会尝试做某事，如果那不起作用，你将 `CATCH` 到它抛出的错误。接下来我们看到两个变量，`$comp` 和 `$properties`。

```
foreach ($computer in $computers) {
            try {
                $comp = get-adcomputer $computer
                $properties = @{computername = $computer
                                Enabled = $comp.enabled
                                InAD = 'Yes'}
            } 
            catch {
                $properties = @{computername = $computer
                                Enabled = 'Fat Chance'
                                InAD = 'No'}
            }
```

`$Comp`正在运行一个 Active Directory 命令来查看计算机是否存在，如果存在，它将 AD 信息存储在`$comp`变量中。`$Properties`是我们创建的一个哈希表，它存储了一些我们需要的信息，包括`ComputerName`、`Enabled`以及它是否在`AD`中。

我们脚本的其余部分将我们创建的哈希表转换成 PS 自定义对象，然后使用`Write-Output`将其写入屏幕。

```
finally {
           $obj = New-Object -TypeName psobject -Property $properties
           Write-Output $obj
         }
```

除此之外

我们需要更改什么才能将此写入文本文件或 CSV 文件？

## 22.2 这是对每一行的检查

上一节的过程是对脚本的逐行分析，这是我们建议你遵循的过程。随着你逐行前进，请执行以下操作：

+   识别变量，尝试弄清楚它们将包含什么内容，并将这些内容写在一张纸上。因为变量通常会被传递给命令参数，所以拥有一个关于你认为每个变量包含什么的便捷参考可以帮助你预测每个命令将做什么。

+   当你遇到新的命令时，阅读它们的帮助并尝试理解它们在做什么。对于`Get-`命令，尝试运行它们——将脚本传递给变量的任何值插入参数中——以查看产生的输出。

+   当你遇到不熟悉的内容，如`if`或`[environment]`时，考虑在虚拟机中运行简短的代码片段以查看这些片段做什么（使用虚拟机有助于保护你的生产环境）。在帮助中搜索这些关键字（使用通配符）以了解更多信息。

最重要的是，不要跳过任何一行。不要想，“嗯，我不知道那是什么，所以我继续。”停下来弄清楚每一行做什么，或者你认为它做什么。这有助于你确定你需要调整脚本以适应你的特定需求。

## 22.3 实验

列表 22.2 显示了一个完整的脚本。看看你是否能弄清楚它做什么以及如何使用它。你能预测出任何可能导致的错误吗？为了在你的环境中使用它，你可能需要做什么？

注意，这个脚本应该直接运行（你可能需要以管理员身份运行以访问安全日志），但如果它在你的系统上无法运行，你能追踪到问题的原因吗？记住，你已经看到了大多数这些命令，而对于你没有看到的命令，有 PowerShell 的帮助文件。这些文件中的示例包括脚本中展示的每个技术。

列表 22.2 Get-LastOn.ps1

```
function get-LastOn {
    <#
    .DESCRIPTION
    Tell me the most recent event log entries for logon or logoff.
    .BUGS
    Blank 'computer' column
    .EXAMPLE
    get-LastOn -computername server1 | Sort-Object time -Descending | 
    Sort-Object id -unique | format-table -AutoSize -Wrap
    ID              Domain       Computer Time                
    --              ------       -------- ----                
    LOCAL SERVICE   NT AUTHORITY          4/3/2020 11:16:39 AM
    NETWORK SERVICE NT AUTHORITY          4/3/2020 11:16:39 AM
    SYSTEM          NT AUTHORITY          4/3/2020 11:16:02 AM
    Sorting -unique will ensure only one line per user ID, the most recent.
    Needs more testing
    .EXAMPLE
    PS C:\Users\administrator> get-LastOn -computername server1 -newest 10000
     -maxIDs 10000 | Sort-Object time -Descending |
     Sort-Object id -unique | format-table -AutoSize -Wrap
    ID              Domain       Computer Time
    --              ------       -------- ----
    Administrator   USS                   4/11/2020 10:44:57 PM
    ANONYMOUS LOGON NT AUTHORITY          4/3/2020 8:19:07 AM
    LOCAL SERVICE   NT AUTHORITY          10/19/2019 10:17:22 AM
    NETWORK SERVICE NT AUTHORITY          4/4/2020 8:24:09 AM
    student         WIN7                  4/11/2020 4:16:55 PM
    SYSTEM          NT AUTHORITY          10/18/2019 7:53:56 PM
    USSDC$          USS                   4/11/2020 9:38:05 AM
    WIN7$           USS                   10/19/2019 3:25:30 AM
    PS C:\Users\administrator>
    .EXAMPLE
    get-LastOn -newest 1000 -maxIDs 20 
    Only examines the last 1000 lines of the event log
    .EXAMPLE
    get-LastOn -computername server1| Sort-Object time -Descending | 
    Sort-Object id -unique | format-table -AutoSize -Wrap
    #>
    param (
            [string]$ComputerName = 'localhost',
            [int]$MaxEvents = 5000,
            [int]$maxIDs = 5,
            [int]$logonEventNum = 4624,
            [int]$logoffEventNum = 4647
        )
        $eventsAndIDs = Get-WinEvent -LogName security -MaxEvents $MaxEvents 
      ➥ -ComputerName $ComputerName | 
        Where-Object {$_.id -eq $logonEventNum -or `
        $_.instanceid -eq  $logoffEventNum} | 
        Select-Object -Last $maxIDs -Property TimeCreated,MachineName,Message
        foreach ($event in $eventsAndIDs) {
            $id = ($event | 
            parseEventLogMessage | 
            where-Object {$_.fieldName -eq "Account Name"}  | 
            Select-Object -last 1).fieldValue
            $domain = ($event | 
            parseEventLogMessage | 
            where-Object {$_.fieldName -eq "Account Domain"}  | 
            Select-Object -last 1).fieldValue
            $props = @{'Time'=$event.TimeCreated;
                'Computer'=$ComputerName;
                'ID'=$id
                'Domain'=$domain}
            $output_obj = New-Object -TypeName PSObject -Property $props
            write-output $output_obj
        }  
    }
    function parseEventLogMessage()
    {
        [CmdletBinding()]
        param (
            [parameter(ValueFromPipeline=$True,Mandatory=$True)]
            [string]$Message 
        )    
        $eachLineArray = $Message -split "`n"
        foreach ($oneLine in $eachLineArray) {
            write-verbose "line:_$oneLine_"
            $fieldName,$fieldValue = $oneLine -split ":", 2
                try {
                    $fieldName = $fieldName.trim() 
                    $fieldValue = $fieldValue.trim() 
                }
                catch {
                    $fieldName = ""
                }
                if ($fieldName -ne "" -and $fieldValue -ne "" ) 
                {
                $props = @{'fieldName'="$fieldName";
                        'fieldValue'=$fieldValue}
                $output_obj = New-Object -TypeName PSObject -Property $props
                Write-Output $output_obj
                }
        }
    }
Get-LastOn
```

## 22.4 实验答案

脚本文件似乎定义了两个函数，这些函数在调用之前不会做任何事情。在脚本末尾有一个命令`Get-LastOn`，它与其中一个函数的名称相同，所以我们可以假设这就是执行的内容。查看该函数，你可以看到它有多个参数默认值，这解释了为什么不需要调用其他内容。基于注释的帮助也解释了该函数的功能。这个函数的第一部分使用`Get-WinEvent`：

```
$eventsAndIDs = Get-WinEvent -LogName security -MaxEvents $MaxEvents | 
  Where-Object { $_.id -eq $logonEventNum -or $_.id -eq $logoffEventNum } | 
  Select-Object -Last $maxIDs -Property TimeCreated, MachineName, Message
```

如果这是一个新的 cmdlet，我们会查看帮助和示例。表达式似乎返回一个用户定义的事件最大值。在查看`Get-WinEvent`的帮助后，我们看到参数`-MaxEvents`将返回从最新到最旧的排序的最大事件数。因此，我们的变量`$MaxEvents`来自一个参数，默认值为`5000`。这些事件日志随后通过`Where-Object`进行过滤，寻找两个事件日志值（事件 ID 为`4627`和`4647`），也来自参数。

接下来，看起来在`foreach`循环中对每个事件日志都进行了某种处理。这里有一个潜在的陷阱：在`foreach`循环中，看起来其他变量正在被设置。第一个是将事件对象传递到名为`parseEventmessage`的某个东西。这看起来不像是一个 cmdlet 名称，但我们确实看到它被用作一个函数。跳转到它，我们可以看到它接受一个消息作为参数，并将每个消息分割成一个数组。我们可能需要研究`-Split`运算符。

数组中的每一行都通过另一个`foreach`循环进行处理。看起来行再次被分割，并且有一个`try/catch`块来处理错误。同样，我们可能需要阅读有关它的内容以了解它是如何工作的。最后，有一个`if`语句，看起来如果分割后的字符串不为空，则创建一个名为`$props`的变量作为散列表或关联数组。如果作者包含一些注释，这个函数将更容易理解。无论如何，解析函数通过调用`New-Object`（另一个需要了解的 cmdlet）结束。

这个函数的输出随后传递给调用函数。看起来重复了同样的过程以获取`$domain`。

哦，看，另一个散列表和`New-Object`，但到现在我们应该理解这个函数在做什么。这是函数的最终输出，因此是脚本。
