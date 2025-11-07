# 18 会话：减少工作量进行远程控制

在第十三章中，我们向您介绍了 PowerShell 的远程功能。在那个章节中，您使用了两个主要的命令——`Invoke-Command` 和 `Enter-PSSession`——来访问一对一和多对一的远程控制。这两个命令通过创建一个新的远程连接，执行您指定的任何工作，然后关闭该连接来工作。

这种方法没有问题，但不断地指定计算机名称、凭据、替代端口号等可能会让人感到疲倦。在本章中，你将了解一种更简单、更可重用的方法来处理远程控制。你还将了解使用远程控制的第三种方法，称为*隐式远程控制*，这将允许你通过将远程机器上的模块导入到你的远程会话中，添加代理命令。

无论何时你需要连接到远程计算机，无论是使用 `Invoke-Command` 还是 `Enter-PSSession`，你至少需要指定计算机的名称（或者名称列表，如果你在多台计算机上执行命令）。根据你的环境，你可能还需要指定替代凭据，这意味着需要输入密码。你可能还需要指定替代端口或身份验证机制，这取决于你的组织如何配置远程访问。

虽然指定这些内容并不困难，但重复这个过程可能会很繁琐。幸运的是，我们知道有一种更好的方法：可重用会话。

注意：本章中的示例只能在您有另一台计算机可以连接并且已启用 PS 远程访问的情况下完成。有关更多信息，请参考第十三章。

## 18.1 创建和使用可重用会话

*会话*是您的 PowerShell 副本与远程 PowerShell 副本之间的一种持久连接。当会话处于活动状态时，您的计算机和远程机器都会分配一小部分内存和处理器时间来维护连接。然而，在连接中涉及的网络流量很少。PowerShell 维护一个您已打开的所有会话的列表，您可以使用这些会话来调用命令或进入远程外壳。

要创建一个新的会话，请使用 `New-PSSession` 命令。指定计算机名称或主机名（或名称列表），如果需要，指定替代用户名、端口、身份验证机制等。我们也不应忘记，我们可以通过使用 `-hostname` 参数来使用 SSH 而不是 WinRM。无论如何，结果都将是一个会话对象，该对象存储在 PowerShell 的内存中：

```
PS C:\> new-pssession -computername srv02,dc17,print99

PS C:\> new-pssession -hostname LinuxWeb01,srv03
```

要检索这些会话，请运行 `Get-PSSession`：

```
PS C:\> get-pssession
```

提示：如第十三章所述，当使用 `-computername` 参数时，我们使用的是 WinRM（HTTP/HTTPS）协议。当我们使用 `-hostname` 参数时，我们指定使用 SSH 作为我们的通信协议。

虽然这可行，但我们更喜欢创建会话并立即将它们存储在变量中以供以后访问。例如，朱莉有多个网络服务器，她通常通过使用 `Invoke-Command` 来定期重新配置它们。为了使过程更容易，她将这些会话存储在特定的变量中：

```
PS C:\> $iis_servers = new-pssession -computername web01,web02,web03          
 ➥ -credential WebAdmin

PS C:\> $web_servers = new-pssession -hostname web04,web05,web06              
 ➥ -username WebAdmin
```

永远不要忘记那些会话会消耗资源。如果你关闭外壳，它们会自动关闭，但如果你没有积极使用它们，即使你打算继续使用外壳进行其他任务，手动关闭它们也是一个好主意，这样你就不必在你的机器或远程机器上占用资源。

要关闭会话，请使用 `Remove-PSSession` 命令。例如，要仅关闭 IIS 会话，请使用以下命令：

```
PS C:\> $iis_servers | remove-pssession
```

或者，如果你想关闭所有打开的会话，请使用以下命令：

```
PS C:\> get-pssession | remove-pssession
```

这很简单。

但是一旦你启动了一些会话，你会如何使用它们？在接下来的几节中，我们假设你已经创建了一个名为 `$sessions` 的变量，它包含至少两个会话。我们将使用 `localhost` 和 `SRV02`（你应该指定你自己的计算机名称）。使用 `localhost` 并不是作弊：PowerShell 会启动一个与自身另一个副本的真实远程会话。请记住，这只有在您已启用所有连接的计算机的远程访问时才会工作，所以如果您还没有启用远程访问，请回顾第十三章。

现在试试看 开始跟随并运行这些命令，并确保使用有效的计算机名称。如果你只有一台计算机，请使用其名称和 `localhost`。希望你也会有一台运行 macOS 或 Linux 的机器可以跟随。

超越

有一种酷炫的语法允许你使用一条命令创建多个会话，并且每个会话都分配给一个唯一的变量（而不是像我们之前那样将它们全部合并到一个变量中）：

```
$s_server1,$s_server2 = new-pssession -computer SRV02,dc01
```

这种语法将 `SRV02` 的会话放入 `$s_server1`，将 `DC01` 的会话放入 `$s_server2`，这可以使得独立使用这些会话变得更容易。

但要小心：我们见过会话并不是按照你指定的顺序创建的情况，所以 `$s_server1` 可能最终包含 `DC01` 的会话而不是 `SRV02`。你可以显示变量的内容来查看它连接到哪台计算机。

这是我们如何启动会话的方法：

```
PS C:\> $session01 = New-PSSession -computername SRV02,localhost

PS C:\> $session02 = New-PSSession -hostname linux01,linux02 -keyfilepath 
 ➥ {path to key file} 
```

记住我们已经在这些计算机上启用了远程访问，Windows 机器都在同一个域中。再次提醒，如果你想要复习如何启用远程访问，请回顾第十三章。

## 18.2 使用会话对象进入-PSSession

好的，现在你已经了解了使用会话的所有原因，让我们看看如何具体使用它们。正如我们希望你能从第十三章回忆起来，`Enter-PSSession` cmdlet 是你用来与单个远程计算机建立一对一远程交互式 shell 的命令。而不是使用该 cmdlet 指定计算机名或主机名，你可以指定一个会话对象。因为我们的`$session01`和`$session02`变量包含多个会话对象，我们必须使用索引（你是在第十六章首次学习如何使用索引）来指定其中一个：

```
PS C:\> enter-pssession -session $session010]
[SRV02]: PS C:\Users\Administrator\Documents>
```

你可以看到，我们的提示符已经改变，表明我们现在正在控制一台远程计算机。`Exit-PSSession`将我们返回到本地提示符，但会话仍然保持打开状态以供进一步使用：

```
[SRV02]: PS C:\Users\Administrator\Documents> exit-pssession
PS C:\>
```

如果你有多会话而忘记了特定会话的索引号怎么办？你可以将会话变量通过管道传递给`Get-Member`并检查会话对象的属性。例如，当我们通过管道将`$session02`传递给`Get-Member`时，我们得到以下输出：

```
PS C:\> $session01 | gm
   TypeName: System.Management.Automation.Runspaces.PSSession

Name                   MemberType     Definition
----                   ----------     ----------
Equals                 Method         bool Equals(System.Object obj)
GetHashCode            Method         int GetHashCode()
GetType                Method         type GetType()
ToString               Method         string ToString()
ApplicationPrivateData Property       psprimitivedictionary App...
Availability           Property       System.Management.Automat...
ComputerName           Property       string ComputerName {get;}
ComputerType           Property       System.Management.Automat...
ConfigurationName      Property       string ConfigurationName {get;}
ContainerId            Property       string ContainerId {get;}
Id                     Property       int Id {get;}
InstanceId             Property       guid InstanceId {get;}
Name                   Property       string Name {get;set;}
Runspace               Property       runspace Runspace {get;}
Transport              Property       string Transport {get;}
VMId                   Property       System.Nullable[guid] VMId {get;}
VMName                 Property       string VMName {get;}
DisconnectedOn         ScriptProperty System.Object DisconnectedOn...
ExpiresOn              ScriptProperty System.Object ExpiresOn {get...
IdleTimeout            ScriptProperty System.Object IdleTimeout {get=$t... 
State                  ScriptProperty System.Object State {get=$this...
```

在前面的输出中，你可以看到会话对象有一个`ComputerName`属性，这意味着你可以针对该会话进行筛选：

```
PS C:\> enter-pssession -session ($sessions | where { $_.computername -eq 
 ➥ 'SRV02' })
[SRV02]: PS C:\Users\Administrator\Documents>
```

虽然这种语法有些尴尬。如果你需要从一个变量中使用单个会话，而你又记不起哪个索引号对应哪个会话，可能更容易忘记使用变量。

即使你将会话对象存储在变量中，它们也仍然存储在 PowerShell 的开放会话主列表中。你可以通过使用`Get-PSSession`来访问它们：

```
PS C:\> enter-pssession -session (get-pssession -computer SRV02)
```

`Get-PSSession`检索名为`SRV02`的会话并将其传递给`Enter-PSSession`的`-session`参数。

当我们第一次弄懂那种技术时，我们感到印象深刻，但它也引导我们进一步深入挖掘。我们调出了`Enter-PSSession`的全局帮助，并更仔细地阅读了关于`-session`参数的内容。以下是我们的观察结果：

```
-Session <System.Management.Automation.Runspaces.PSSession>
        Specifies a PowerShell session ( PSSession ) to use for the    interactive session. This parameter takes a
        session object. You can also use the Name , InstanceID , or ID parameters to specify a PSSession .

        Enter a variable that contains a session object or a command that creates or gets a session object, such as a
        `New-PSSession` or `Get-PSSession` command. You can also pipe a session object to `Enter-PSSession`. You can
        submit only one PSSession by using this parameter. If you enter a variable that contains more than one
        PSSession , the command fails.

        When you use `Exit-PSSession` or the EXIT keyword, the interactive session ends, but the PSSession that you
        created remains open and available for use.
```

如果你回想起第九章，你会在帮助文档的末尾发现一些有趣的管道输入信息。它告诉我们`-session`参数可以接受来自管道的`PSSession`对象。我们知道`Get-PSSession`会产生`PSSession`对象，所以以下语法也应该有效：

```
PS C:\> Get-PSSession -ComputerName SRV02 | Enter-PSSession
[SRV02]: PS C:\Users\Administrator\Documents>
```

它确实有效。我们认为这是一种检索单个会话的更优雅的方式，即使你将它们全部存储在变量中。

小贴士：将会话存储在变量中作为便利性是可行的。但请记住，PowerShell 已经存储了所有开放会话的列表。将它们存储在变量中只有在你想一次性引用多个会话时才有用，正如你将在下一节中看到的。

## 18.3 使用会话对象的 Invoke-Command

会话通过`Invoke-Command`展示了它们的有用性，你可能记得，这是用来向多个远程计算机并行发送命令（或整个脚本）的。在我们的会话存储在`$session01`变量中时，我们可以轻松地使用以下命令来针对它们：

```
PS C:\> invoke-command -command { Get-Process } -session $session01
```

`Invoke-Command`的`-session`参数也可以接收一个括号内的命令，就像我们在前面的章节中处理计算机名时所做的那样。例如，以下命令会向列出的每个计算机的会话发送命令：

```
PS C:\> invoke-command -command { get-process bits } -session (get-pssession 
 ➥ –computername server1,server2,server3)
```

你可能期望`Invoke-Command`能够从管道接收会话对象，就像你知道`Enter-PSSession`可以那样。但查看`Invoke-Command`的完整帮助文档显示，它不能执行那个特定的管道技巧。真遗憾，但前面使用括号表达式提供的功能没有太复杂的语法。

## 18.4 隐式远程操作：导入会话

对于我们来说，隐式远程操作是我们认为最酷和最有用的——可能是*最酷和最有用的——功能，这是一个命令行界面在任何操作系统上都有过，而且几乎在 PowerShell 中几乎没有文档记录。当然，必要的命令有很好的文档记录，但它们如何组合形成这种令人难以置信的能力并没有提到。幸运的是，我们在这方面为你提供了覆盖。

让我们回顾一下场景：你已经知道微软正在将越来越多的模块与 Windows Server 和其他产品一起发货，但有时由于各种原因，你无法在本地计算机上安装这些模块。`ActiveDirectory`模块，首次随 Windows Server 2008 R2 一起发货，是一个完美的例子：它只存在于域控制器以及安装了远程服务器管理工具（RSAT）的服务器/客户端上。让我们通过一个单独的例子来看整个过程：

```
PS C:\> $session = new-pssession -comp SRV02                                   ❶
PS C:\> invoke-command -command { import-module activedirectory }              ❷
        session $session
PS C:\> import-pssession -session $session -module activedirectory -prefix rem ❸

ModuleType Name                      ExportedCommands                          ❹
---------- ----                      ----------------
Script     tmp_2b9451dc-b973-495d... {Set-ADOrganizationalUnit, Get-ADD...
```

❶ 建立连接

❷ 加载远程模块

❸ 导入远程命令

❹ 检查临时本地模块

下面是这个例子中发生的情况：

1.  我们首先与安装了 Active Directory 模块的远程计算机建立会话。

1.  我们告诉远程计算机导入其本地的 Active Directory 模块。这只是其中一个例子；我们本可以选择加载任何模块。因为会话仍然打开，模块会保留在远程计算机上。

1.  然后我们告诉我们的计算机从那个远程会话中导入命令。我们只想导入 Active Directory 模块中的命令，并且当它们被导入时，我们希望为每个命令的名词添加一个`rem`前缀。这使我们能够更容易地跟踪远程命令。这也意味着命令不会与已经加载到我们的 shell 中的任何同名的命令冲突。

1.  PowerShell 在我们计算机上创建了一个临时模块，代表远程命令。命令并没有被复制过来；相反，PowerShell 为它们创建了快捷方式，而这些快捷方式指向远程机器。

现在，我们可以运行 Active Directory 模块命令或甚至请求帮助。我们不是运行`New-ADUser`，而是运行`New-remADUser`，因为我们已经将`rem`前缀添加到命令的名词中。这些命令在我们关闭 shell 或关闭与远程计算机的会话之前都可用。当我们打开一个新的 shell 时，我们必须重复此过程才能重新获得对远程命令的访问权限。

当我们运行这些命令时，它们不会在我们的本地计算机上执行。相反，它们被隐式地远程到远程计算机上。它为我们执行它们并将结果发送到我们的计算机。

我们可以想象一个不再需要在我们的计算机上安装管理工具的世界。我们将会避免多少麻烦。今天，您需要能够在您的计算机操作系统上运行并与您试图管理的任何远程服务器通信的工具——让所有这些匹配起来可能是无法实现的。在未来，您将不会这样做。您将使用隐式远程。服务器将通过 Windows PowerShell 提供他们的管理功能作为另一个服务。

现在来说说坏消息：通过隐式远程传递到您计算机上的结果都是反序列化的，这意味着对象的属性被复制到一个 XML 文件中，以便在网络中传输。通过这种方式接收到的对象没有任何方法。在大多数情况下，这不会成为问题，但有些模块和插件生成的对象是您打算以更程序化的方式使用的，而这些对象不适合隐式远程。我们希望您很少（如果有的话）会遇到具有这种限制的对象，因为依赖于方法违反了一些 PowerShell 设计原则。如果您确实遇到了这样的对象，您将无法通过隐式远程使用它们。

## 18.5 使用断开连接的会话

PowerShell v3 对其远程控制能力引入了两个改进。首先，会话变得更加稳固，这意味着它们可以承受短暂的网络中断和其他短暂的干扰。即使您没有明确使用会话对象，您也能获得这种好处。即使您已经使用了`Enter-PSSession`及其`-ComputerName`参数，技术上您仍然在使用底层的会话，因此您获得了更稳健的连接性。

v3 版本中引入的另一个新特性是您必须显式使用的：断开连接的会话。假设您坐在`COMPUTER1`上，以`Admin1`（他是域管理员组的成员）的身份登录，并创建到`COMPUTER2`的新连接：

```
PS C:\> New-PSSession -ComputerName COMPUTER2
Id Name              ComputerName  State
-- ----------------- ------------- -----
 4 Session4          COMPUTER2     Opened
```

然后，您可以断开该会话。您仍然在您所在的`COMPUTER1`上这样做，它会断开两个计算机之间的连接，但会保留在`COMPUTER2`上运行的 PowerShell 副本。请注意，您通过指定会话的 ID 号来完成此操作，该 ID 号在您首次创建会话时显示：

```
PS C:\> Disconnect-PSSession -Id 4
Id Name              ComputerName  State
-- ----------------- ------------- -----
 4 Session4          COMPUTER2     Disconnected
```

这显然是你需要考虑的事情——你正在`COMPUTER2`上运行 PowerShell 的一个副本。分配有用的空闲超时时间等变得很重要。在 PowerShell 的早期版本中，你断开连接的会话会消失，所以你没有清理工作要做。从 v3 开始，你可以在环境中随意放置正在运行的会话，这意味着你必须承担更多的责任。

但这里有个酷的地方：我们将以同一域管理员`Admin1`的身份登录到另一台计算机`COMPUTER3`，并检索在`COMPUTER2`上运行的会话列表：

```
PS C:\> Get-PSSession -computerName COMPUTER2
Id Name              ComputerName  State
-- ----------------- ------------- -----
 4 Session4          COMPUTER2     Disconnected
```

真是整洁，对吧？如果你以不同的用户身份登录，即使是另一位管理员，你也看不到这个会话；你只能看到你在`COMPUTER2`上创建的会话。但现在，既然你已经看到了它，你可以重新连接它。这将允许你重新连接到一个你故意或无意中断的会话，并且你将能够从你离开的地方继续你的会话：

```
PS C:\> Get-PSSession -computerName COMPUTER2 | Connect-PSSession
Id Name              ComputerName  State
-- ----------------- ------------- -----
 4 Session4          COMPUTER2     Open
```

让我们花些时间来谈谈管理这些会话。在 PowerShell 的 WSMan 驱动器中，你可以找到可以帮助你控制断开连接会话的设置。你还可以通过组策略集中配置这些设置中的大多数。要查找的关键设置包括以下内容：

+   在 WSMan:\localhost\Shell:

    +   `IdleTimeout`—指定会话在自动关闭之前可以空闲的时间。默认值约为 2,000 小时（以秒为单位），或约 84 天。

    +   `MaxConcurrentUsers`—指定一次可以打开会话的用户数量。

    +   `MaxShellRunTime`—确定会话可以打开的最大时间。从所有实际目的来看，默认值是无限的。请注意，如果 shell 处于空闲状态，`IdleTimeout`可以覆盖此设置，而不是运行命令。

    +   `MaxShellsPerUser`—设置单个用户一次可以打开的会话数量上限。将此值乘以`MaxConcurrentUsers`，可以计算出所有用户在计算机上可能的最大会话数量。

+   在 WSMan:\localhost\Service:

    +   `MaxConnections`—设置整个远程基础设施的传入连接的上限。即使你允许每个用户或最大用户数有更多的 shell，`MaxConnections`也是传入连接的绝对限制。

作为管理员，你显然比标准用户有更高的责任。跟踪你的会话是你的责任，尤其是如果你会断开和重新连接。合理的时间超时设置可以帮助确保 shell 会话不会长时间空闲。

## 18.6 实验室

注意：对于这个实验室，你需要一台运行 PowerShell v7 或更高版本的 Windows Server 2016、macOS 或 Linux 机器。如果你只能访问客户端计算机（运行 Windows 10 或更高版本），你将无法完成这个实验室的第 6 项至第 9 项任务。

要完成这个实验，你应该有两台计算机：一台用于远程连接，另一台用于远程到。如果你只有一台计算机，请使用其计算机名来远程连接到它。这样你将获得类似的经验：

1.  关闭你在 shell 中打开的所有会话。

1.  建立到远程计算机的会话。将会话保存在名为 `$session` 的变量中。

1.  使用 `$session` 变量与远程计算机建立一对一的远程 shell 会话。显示进程列表然后退出。

1.  使用 `$session` 变量与 `Invoke-Command` 列出远程机器的时间区域。

1.  如果你是在 Windows 客户端，使用 `Get-PSSession` 和 `Invoke-Command` 从远程计算机获取最近的 20 条安全事件日志条目列表。

1.  如果你在 macOS 或 Linux 客户端，计算 `/var` 目录中的项目数量。*任务 7-10 只能在 Windows 机器上执行。*

1.  使用 `Invoke-Command` 和 `$session` 变量在远程计算机上加载 `ServerManager` 模块。

1.  从远程计算机导入 `ServerManager` 模块的命令到你的计算机。给导入命令的名词添加前缀 `rem`。

1.  运行导入的 `Get-WindowsFeature` 命令。

1.  关闭 `$session` 变量中的会话。

## 18.7 实验答案

1.  `get-pssession | Remove-PSSession`

1.  `$session=new-pssession –computername localhost`

1.  `enter-pssession $session`

    `Get-Process`

    `Exit`

1.  `invoke-command -ScriptBlock { get-timezone } -Session $session`

1.  `Invoke-Command -ScriptBlock {get-eventlog -LogName System`

    `-Newest 20} -Session (Get-PSSession)`

    `Get-ChildItem -Path /var | Measure-Object | select count`

1.  `Invoke-Command -ScriptBlock {Import-Module ServerManager}`

    `-Session $session`

1.  `Import-PSSession -Session $session -Prefix rem`

    `-Module ServerManager`

1.  `Get-RemWindowsFeature`

1.  `Remove-PSSession -Session $session`

## 18.8 进一步探索

快速盘点你的环境：你有哪些 PowerShell 启用的产品？Exchange 服务器？SharePoint 服务器？VMware vSphere？System Center 虚拟机管理器？这些和其他产品都包含 PowerShell 模块，其中许多可以通过 PowerShell 远程访问。
