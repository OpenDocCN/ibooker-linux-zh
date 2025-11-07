# 9 实践插曲

是时候将你的新知识付诸实践了。在本章中，我们不会教你任何新东西。相反，我们将通过一个详细的例子来引导你，使用你所学到的知识。这是一个绝对的现实世界例子：我们将给自己设定一个任务，然后让你跟随我们的思维过程，了解我们如何完成它。这一章是本书主题的精髓，因为本书不是仅仅给你一个如何做某事的答案，而是帮助你意识到*你可以自学*。

## 9.1 定义任务

首先，我们假设你正在使用运行 PowerShell 7.1 或更高版本的任何操作系统。我们将要处理的例子可能非常适用于早期版本的 Windows PowerShell，但我们没有测试这一点。

在 DevOps 的世界里，除了 PowerShell 之外，几乎总是会出现一种特定的语言——当然，还有 YAML。这在 IT 专业人士和 DevOps 工程师中是一个相当有争议的语言——人们要么爱它，要么恨它。有什么猜测吗？如果你猜到了 YAML，你就对了！YAML 代表“YAML ain’t markup language”（它是一个递归缩写，当缩写包含缩写时），尽管它说它不是，但在很多方面它类似于简单的标记语言——换句话说，它只是一个具有特定结构的文件，就像 CSV 和 JSON 也有特定的结构一样。由于我们在 DevOps 世界中看到了很多 YAML，因此我们拥有与之交互的工具是很重要的。

## 9.2 查找命令

解决任何任务的第一个步骤是找出哪些命令可以为你完成它。你的结果可能与我们不同，这取决于你安装了什么，但重要的是我们正在经历的过程。因为我们知道我们想要管理一些虚拟机，所以我们将以 YAML 作为关键词开始：

```
PS C:\Scripts\ > Get-Help *YAML*
PS C:\Scripts\ >
```

嗯。这没有帮助。没有显示任何内容。好吧，让我们尝试另一种方法——这次，我们关注命令而不是帮助文件：

```
PS C:\Scripts\ > get-command -noun *YAML*
PS C:\Scripts\ >
```

好吧，没有命令的名称包含*YAML*。令人失望！所以现在我们必须看看在线 PowerShell Gallery 中可能有什么：

```
PS C:\Scripts\ > find-module *YAML* | format-table -auto
Version Name            Repository Description
------- ----            ---------- -----------
0.4.0   powershell-yaml PSGallery  Powershell module for serializing...
1.0.3   FXPSYaml        PSGallery  PowerShell module used to...
0.2.0   Gainz-Yaml      PSGallery  Gainz: Yaml...
0.1.0   Gz-Yaml         PSGallery  # Gz-Yaml...
```

这看起来更有希望！所以让我们安装第一个模块：

```
PS C:\Scripts\ > install-module powershell-yaml 
You are installing the module(s) from an untrusted repository. If you
trust this repository, change its InstallationPolicy value by
running the Set-PSRepository cmdlet.
Are you sure you want to install software from
'https://go.microsoft.com/fwlink/?LinkID=397631&clcid=0x409'?
[Y] Yes  [A] Yes to All  [N] No  [L] No to All  [S] Suspend
[?] Help(default is "N"): y
```

现在，在这个时候，你必须小心。虽然 Microsoft 运行 PowerShell Gallery，但它不会验证其他人发布的任何代码。因此，我们在我们的过程中暂停了一下，审查了我们刚刚安装的代码，并在继续之前确保我们对此感到舒适。这也得益于这个模块的作者是一个共同的价值专家（MVP），Boe Prox，我们信任他。现在让我们看看我们刚刚获得的命令：

```
PS C:\Scripts\ > get-command -module powershell-yaml | format-table -auto
CommandType Name              Version Source
----------- ----              ------- ------
Function    ConvertFrom-Yaml 0.4.0   powershell-yaml
Function    ConvertTo-Yaml   0.4.0   powershell-yaml 
```

好吧，这些看起来很简单。`ConvertTo`和`ConvertFrom`——听起来我们需要的就这些。

## 9.3 学习使用命令

希望作者在他们的模块中包含了帮助信息。如果你有一天编写了一个模块，请始终记住，如果其他人将要使用它，你很可能——不，你 *应该*——包含帮助信息，这样模块的消费者就知道如何使用它。如果你没有这样做，那就好比在没有说明书的情况下运送宜家家具——不要这样做！还有另一本书叫做 *Learn PowerShell Toolmaking in a Month of Lunches*（Manning，2012），其中两位作者 Don Jones 和 Jeffery Hicks 讲述了如何编写模块以及如何在其中添加帮助信息——添加帮助信息是正确的事情。让我们看看作者是否做了正确的事情：

```
PS C:\Scripts\ > help ConvertFrom-Yaml
NAME
    ConvertFrom-Yaml

SYNTAX
    ConvertFrom-Yaml [[-Yaml] <string>] [-AllDocuments] [-Ordered] 
  ➥ [-UseMergingParser] [<CommonParameters>]

PARAMETERS
    -AllDocuments     Add-Privilege
```

Drat! 没有帮助。嗯，在这个例子中，情况并不太糟糕，因为即使作者没有编写任何帮助信息，PowerShell 仍然提供了帮助。PowerShell 仍然会给你语法、参数、输出和别名——对于这个简单的命令来说，这些信息已经足够了。所以我们需要一个样本 YAML 文件……嗯，使用 PowerShell GitHub 仓库中的实时示例是值得的：

[`raw.githubusercontent.com/PowerShell/PowerShell/master/.vsts-ci/templates/credscan.yml`](https://raw.githubusercontent.com/PowerShell/PowerShell/master/.vsts-ci/templates/credscan.yml)

这是 PowerShell 团队使用的 Azure Pipelines YAML 文件，用于运行 CredScan 工具——一个用于扫描代码中是否意外添加了机密或凭证的工具。PowerShell 团队将其设置为每次有人向 PowerShell GitHub 仓库发送拉取请求（即代码更改）时运行，以便可以立即捕获。在拉取请求期间运行任务是一种常见的做法，称为 *持续集成*（CI）。

好吧，先下载这个文件，然后让我们从 PowerShell 中读取这个文件：

```
PS C:\Scripts\ > Get-Content -Raw /Users/travis/Downloads/credscan.yml
parameters:
  pool: 'Hosted VS2017'
  jobName: 'credscan'
  displayName: Secret Scan

jobs:
- job: ${{ parameters.jobName }}
  pool:
    name: ${{ parameters.pool }}

  displayName: ${{ parameters.displayName }}

  steps:
  - task: securedevelopmentteam.vss-secure-development-tools.build-task
  ➥ -credscan.CredScan@2
    displayName: 'Scan for Secrets'
    inputs:
      suppressionsFile: tools/credScan/suppress.json
      debugMode: false

  - task: securedevelopmentteam.vss-secure-development-tools.build-task
  ➥ -publishsecurityanalysislogs.PublishSecurityAnalysisLogs@2
    displayName: 'Publish Secret Scan Logs to Build Artifacts'
    continueOnError: true

  - task: securedevelopmentteam.vss-secure-development-tools.build-task
  ➥ -postanalysis.PostAnalysis@1
    displayName: 'Check for Failures'
    inputs:
      CredScan: true
      ToolLogsNotFoundAction: Error
```

好极了，所以我们已经弄清楚如何读取 YAML 文件。接下来，让我们将其转换为更容易处理的东西：

```
PS C:\Scripts\ > Get-Content -Raw /Users/travis/Downloads/credscan.yml | 
 ➥ ConvertFrom-Yaml

Name                           Value
----                           -----
parameters                     {pool, jobName, displayName}
jobs                           {${{ parameters.displayName }}} 
```

嗯，那就简单了。让我们看看我们有什么类型的对象：

```
PS C:\Scripts\ > Get-Content -Raw /Users/travis/Downloads/credscan.yml | 
 ➥ ConvertFrom-Yaml | gm

   TypeName: System.Collections.Hashtable
Name              MemberType            Definition
----              ----------            ----------
Add               Method                void Add(System.Object key...
Clear             Method                void Clear(), void IDictionary.Clear()
Clone             Method                System.Object Clone(), ...
Contains          Method                bool Contains(System.Object key)...
ContainsKey       Method                bool ContainsKey(System.Object key)
ContainsValue     Method                bool ContainsValue(System.Object...
CopyTo            Method                void CopyTo(array array, int...
Equals            Method                bool Equals(System.Object obj)
GetEnumerator     Method                System.Collections.IDictionary...
GetHashCode       Method                int GetHashCode()
GetObjectData     Method                void GetObjectData(System.Runtim... GetType
       Method                type GetType()
OnDeserialization Method                void OnDeserialization(System.Object...Remove
       Method                void Remove(System.Object key), voi... ToString
       Method                string ToString()
Item              ParameterizedProperty System.Object Item(System.Object 
                ➥ key...
Count             Property              int Count {get;}
IsFixedSize       Property              bool IsFixedSize {get;}
IsReadOnly        Property              bool IsReadOnly {get;}
IsSynchronized    Property              bool IsSynchronized {get;}
Keys              Property              System.Collections.ICollection K...
SyncRoot          Property              System.Object SyncRoot {get;}
Values            Property              System.Collections.ICollection Value...
```

好吧，所以这是一个散列表。散列表只是各种东西的集合。你会在 PowerShell 的旅程中看到很多这样的散列表。它们很棒，因为它们可以轻松地转换为其他格式。让我们尝试将我们在 YAML 中得到的内容转换为 DevOps 中另一个非常重要的数据结构——JSON。让我们看看我们有什么可以工作的：

```
PS C:\Scripts\ > Get-Help *json*

Name             Category Module                       Synopsis
----             -------- ------                       --------
ConvertFrom-Json Cmdlet   Microsoft.PowerShell.Utility...
ConvertTo-Json   Cmdlet   Microsoft.PowerShell.Utility...
Test-Json        Cmdlet   Microsoft.PowerShell.Utility...
```

Bingo，一个 `ConvertTo-Json` 命令。让我们使用管道将 YAML 转换为 JSON。我们需要使用 `ConvertTo-Json` 的 `Depth` 参数（你可以在帮助中阅读有关此信息），这允许我们指定命令应该深入到什么程度来尝试创建 JSON 结构。对于我们正在做的事情，100 是一个安全的选择。好吧，让我们把它放在一起：

```
PS C:\Scripts\ > Get-Content -Raw /Users/travis/Downloads/credscan.yml | 
 ➥ ConvertFrom-Yaml | ConvertTo-Json -Depth 100

{
  "parameters": {
    "pool": "Hosted VS2017",
    "jobName": "credscan",
    "displayName": "Secret Scan"
  },
  "jobs": [
    {
      "job": "${{ parameters.jobName }}",
      "pool": {
        "name": "${{ parameters.pool }}"
      },
      "steps": [
        {
          "task": "securedevelopmentteam.vss-secure-development-tools.build
        ➥ -task-credscan.CredSca          "inputs": {
            "debugMode": false,
            "suppressionsFile": "tools/credScan/suppress.json"
          },
          "displayName": "Scan for Secrets"
        },
        {
nalysislogs.PublishSecurityAnalysisLogs@2",
          "continueOnError": true,
          "displayName": "Publish Secret Scan Logs to Build Artifacts"
        },
        {
          "task": "securedevelopmentteam.vss-secure-development-tools.build
         ➥ -task-postanalysis.PostAnalysis@1",
          "inputs": {
            "CredScan": true,
            "ToolLogsNotFoundAction": "Error"
          },
          "displayName": "Check for Failures"
        }
      ],
      "displayName": "${{ parameters.displayName }}"
    }
  ]
}
```

它成功了！我们现在根据 YAML 文件生成了一些 JSON。这是一个有用的练习，因为野外有许多不同的 DevOps 工具接受 YAML 或 JSON（例如 AutoRest、Kubernetes），因此你可能更喜欢 YAML，但你的同事可能更喜欢 JSON。现在你有一个简单的方法可以通过这种方式相互分享。

现在，我们坦白承认，这并不是一个复杂的任务。但任务本身并不是本章的重点。重点是*我们是如何找到答案的*。我们做了什么？

1.  我们首先在本地帮助文件中搜索包含特定关键词的任何文件。当我们的搜索词与命令名称不匹配时，PowerShell 会对所有帮助文件的内容进行全文搜索。这很有用，因为如果某个文件甚至*提到了* YAML，我们就能找到它。

1.  我们继续搜索特定的命令名称。这将帮助我们找到*没有安装帮助文件的命令*。理想情况下，命令应该总是有帮助文件，但我们并不生活在一个理想的世界，所以我们总是采取这一额外步骤。

1.  在本地找不到任何东西后，我们在 PowerShell Gallery 中搜索，并找到了一个看起来很有希望的模块。我们安装了该模块并审查了其命令。

1.  即使模块作者没有提供帮助，PowerShell 也帮助我们找到了如何运行命令以将数据转换为 YAML 的方法。这有助于我们了解命令数据的结构以及命令期望的值类型。

1.  使用我们到那时收集到的信息，我们能够实现我们想要的更改。

## 9.4 自学小贴士

再次强调，这本书的真正目的是教你如何自学——这正是本章所展示的。这里有一些小贴士：

+   不要害怕寻求帮助，并且一定要阅读示例。我们反复这么说，但好像没有人相信我们。我们仍然看到新来的新手，就在我们眼前，偷偷地去谷歌寻找示例。帮助文件有什么可怕的呢？如果你愿意阅读某人的博客，为什么不先尝试一下帮助文件中的示例呢？

+   注意。屏幕上的每一信息都可能很重要——不要在心理上跳过你立即寻找的东西。这很容易做到，但不要这么做。相反，看看每一件事，并试图弄清楚它的用途以及你可以从中得到的信息。

+   不要害怕失败。希望你有可以玩耍的虚拟机——那么就使用它。新手们不断地向我们提出问题，比如，“嘿，如果我这样做，会发生什么？”我们开始回答，“不知道——试试看。”实验是好的。在虚拟机中，最坏的情况是你不得不回滚到快照，对吧？所以试试看，无论你在做什么。

+   如果一件事不起作用，不要头撞南墙——尝试其他方法。

随着时间的推移、耐心和练习，一切都会变得容易——但一定要确保你在*思考*。

## 9.5 实验室

注意：对于这个实验，你可以使用你喜欢的任何操作系统（如果你想的话，可以在多个操作系统上尝试），并确保你正在使用 PowerShell 7.1 或更高版本。

现在轮到你了。我们假设你在一个虚拟机或其他机器上工作，这个机器在学习的过程中可以稍微出点问题。请不要在生产环境中的关键计算机上这样做！

这个练习将涉及秘密管理。DevOps 工程师应该非常熟悉这个概念。想法很简单：我们有一堆敏感信息（密码、连接字符串等），这些信息需要在我们的命令中使用，但我们需要将这些秘密保存在一个安全的地方。我们可能还想与其他团队成员分享这些秘密——电子邮件并不足够安全，朋友们！

PowerShell 团队最近一直在开发一个名为“秘密管理”的模块，专门用于执行这项任务。这是一个通用的模块，可以与支持它的任何秘密存储进行交互。一些将是本地秘密存储，如 macOS Keychain，而其他将是云服务，如 Azure Key Vault 和 HashiCorp Vault。你的目标是获取这个模块，在你的选择秘密存储中存储一个秘密，然后检索它。如果你使用基于云的秘密存储，尝试从不同的机器检索秘密，作为最终测试。

## 9.6 实验答案

我们确信你们期待我们给出一个完整的命令列表来完成这个实验练习。然而，这一章的全部内容都是关于自己找出这些事情。秘密管理模块有很好的文档记录。如果你一直跟着我们学习，那么你在这个实验中不会遇到任何问题。
