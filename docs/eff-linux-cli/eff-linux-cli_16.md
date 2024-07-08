# 附录 B. 如果你使用不同的 Shell

本书假设您的登录 Shell 是`bash`，但如果不是，表 B-1 可能会帮助您调整本书中其他 Shell 的示例。勾号 ✓ 表示兼容性——给定特性与`bash`相似到足以正确运行本书示例。但是，该特性的行为可能在其他方面与`bash`不同，请仔细阅读任何脚注。

###### 注

无论您的登录 Shell 是哪个，以 `#!/bin/bash` 开头的脚本都将由 `bash` 处理。

要在系统上安装的另一个 Shell 上进行实验，只需按其名称运行它（例如，`ksh`），完成后按 Ctrl-D。要更改登录 Shell，请阅读`man chsh`。

表 B-1\. 其他 Shell 支持的`bash`特性，按字母顺序排列

| `bash`特性 | dash | fish | ksh | tcsh | zsh |
| --- | --- | --- | --- | --- | --- |
| `alias`内建命令 | ✓ | ✓，但`alias` *`name`* 不打印别名 | ✓ | 没有等号：`alias g grep` | ✓ |
| 使用`&`进行后台化 | ✓ | ✓ | ✓ | ✓ | ✓ |
| `bash -c` | `dash -c` | `fish -c` | `ksh -c` | `tcsh -c` | `zsh -c` |
| `bash`命令 | `dash` | `fish` | `ksh` | `tcsh` | `zsh` |
| `bash`在 */bin/bash* 的位置 | */bin/dash* | */bin/fish* | */bin/ksh* | */bin/tcsh* | */bin/zsh* |
| `BASH_SUBSHELL`变量 |  |  |  |  |  |
| 使用 `{}` 进行大括号展开 | 使用 `seq` | 仅支持 `{a,b,c}`，不支持 `{a..c}` | ✓ | 使用 `seq` | ✓ |
| `cd -`（切换目录） | ✓ | ✓ | ✓ | ✓ | ✓ |
| `cd`内建命令 | ✓ | ✓ | ✓ | ✓ | ✓ |
| `CDPATH`变量 | ✓ | `set CDPATH` *`value`* | ✓ | `set cdpath =` (*`dir1 dir2 …`*`)` | ✓ |
| 使用 `$()` 进行命令替换 | ✓ | 使用 `()` | ✓ | 使用反引号 | ✓ |
| 使用反引号进行命令替换 | ✓ | 使用 `()` | ✓ | ✓ | ✓ |
| 使用箭头键进行命令行编辑 |  | ✓ | ✓^(a) | ✓ | ✓ |
| 使用 Emacs 键进行命令行编辑 |  | ✓ | ✓^(a) | ✓ | ✓ |
| 使用`set -o vi`启用 Vim 键的命令行编辑 |  |  | ✓ | 运行`bindkey -v` | ✓ |
| `complete`内建命令 |  | 不同的语法^(b) | 不同的语法^(b) | 不同的语法^(b) | `compdef`^(b) |
| 使用`&&`和`&#124;&#124;`进行条件列表 | ✓ | ✓ | ✓ | ✓ | ✓ |
| `$HOME`中的配置文件（详细信息请参阅 manpage） | *.profile* | *.config/fish/config.fish* | *.profile*, *.kshrc* | *.cshrc* | *.zshenv*, *.zprofile*, *.zshrc*, *.zlogin*, *.zlogout* |
| 控制结构：`for`循环，`if`语句等 | ✓ | 不同的语法 | ✓ | 不同的语法 | ✓ |
| `dirs`内建命令 |  | ✓ |  | ✓ | ✓ |
| `echo`内建命令 | ✓ | ✓ | ✓ | ✓ | ✓ |
| 使用 `\` 转义别名 | ✓ |  | ✓ | ✓ | ✓ |
| 使用`\`进行转义 | ✓ | ✓ | ✓ | ✓ | ✓ |
| `exec`内建命令 | ✓ | ✓ | ✓ | ✓ | ✓ |
| 使用`$?`获取退出代码 | ✓ | `$status` | ✓ | ✓ | ✓ |
| `export`内建命令 | ✓ | `set -x` *`name value`* | ✓ | `setenv` *`name value`* | ✓ |
| 函数 | ✓^(c) | 不同的语法 | ✓ |  | ✓ |
| `HISTCONTROL`变量 |  |  |  |  | 在 man 页上查看以`HIST_`开头的变量名 |
| `HISTFILE`变量 |  | `set` `fish_history` *`path`* | ✓ | `set` `histfile` `=` *`path`* | ✓ |
| `HISTFILESIZE`变量 |  |  |  | `set` `savehist` `=` *`value`* | +SAVEHIST |
| `history`内置命令 |  | 有`history`，但命令没有编号 | `history`是`hist` `-l`的别名 | ✓ | ✓ |
| `history -c` |  | `history clear` | 删除*~/.sh_history*并重新启动`ksh` | ✓ | `history` `-p` |
| 使用`!`和`^`进行历史扩展 |  |  |  | ✓ | ✓ |
| 使用 Ctrl-R 进行历史增量搜索 |  | 输入命令的开头，然后按向上箭头搜索，右箭头选择 | ✓^(a) ^(d) | ✓^(e) | ✓^(f) |
| `history` *`number`* |  | `history` `-`*`number`* | `history -N` *`number`* | ✓ | `history` `-*number*` |
| 使用箭头键的历史记录 |  | ✓ | ✓^(a) | ✓ | ✓ |
| 使用 Emacs 键进行历史记录 |  | ✓ | ✓^(a) | ✓ | ✓ |
| 使用`set -o vi`的 Vim 键进行历史记录 |  |  | ✓ | 运行`bindkey -v` | ✓ |
| `HISTSIZE`变量 |  |  | ✓ |  | ✓ |
| 使用`fg`、`bg`、Ctrl-Z、`jobs`进行作业控制 | ✓ | ✓ | ✓ | ✓^(g) | ✓ |
| 使用`*`、`?`、`[]`进行模式匹配 | ✓ | ✓ | ✓ | ✓ | ✓ |
| 管道 | ✓ | ✓ | ✓ | ✓ | ✓ |
| `popd`内置命令 |  | ✓ |  | ✓ | ✓ |
| 使用`<()`进行进程替换 |  |  | ✓ |  | ✓ |
| `PS1`变量 | ✓ | `set PS1` *`value`* | ✓ | `set prompt =` *`value`* | ✓ |
| `pushd`内置命令 |  | ✓ |  | 有`pushd`，但没有负参数 | ✓ |
| 双引号 | ✓ | ✓ | ✓ | ✓ | ✓ |
| 单引号 | ✓ | ✓ | ✓ | ✓ | ✓ |
| 将 stderr 重定向（`2>`） | ✓ | ✓ | ✓ |  | ✓ |
| 将 stdin（`<`）、stdout（`>`、`>>`）重定向 | ✓ | ✓ | ✓ | ✓ | ✓ |
| 将 stdout 和 stderr 重定向（`&>`） | 追加 `2>&1`^(h) | ✓ | 追加 `2>&1`^(h) | `>&` | ✓ |
| 使用`source`或`.`（点号）来源化文件 | 只有点号^(i) | ✓ | ✓^(i) | 只有`source` | ✓^(i) |
| 使用`()`创建子 shell | ✓ |  | ✓ | ✓ | ✓ |
| 文件名的制表完成 |  | ✓ | ✓^(a) | ✓ | ✓ |
| `type`内置命令 | ✓ | ✓ | `type`是`whence -v`的别名 | 不是，但`which`是内置命令 | ✓ |
| `unalias`内置命令 | ✓ | `functions` `--erase` | ✓ | ✓ | ✓ |
| 使用`name=value`定义变量 | ✓ | `set` *`name value`* | ✓ | `set` *`name`* `=` *`value`* | ✓ |
| 使用`$name`进行变量评估 | ✓ | ✓ | ✓ | ✓ | ✓ |
| ^(a) 此功能默认禁用。运行 `set -o emacs` 可以启用它。旧版本的 `ksh` 可能有不同的行为。^(b) 自定义命令补全，使用 `complete` 命令或类似命令，在不同的 shell 中可能有显著不同；请参阅相应 shell 的 man 手册。^(c) 函数：此 shell 不支持以 `function` 关键字开头的新式定义。^(d) 在 `ksh` 中，历史记录的增量搜索方式有所不同。按 Ctrl-R 键，输入字符串，然后按 Enter 键可以调出包含该字符串的最近命令。再次按 Ctrl-R 键和 Enter 键可以向后搜索下一个匹配的命令，依此类推。按 Enter 键执行命令。^(e) 要在 `tcsh` 中启用 Ctrl-R 的增量历史搜索功能，请运行命令 `bindkey ^R i-search-back`（并将其添加到 shell 配置文件）。搜索行为与 `bash` 有所不同；参阅 `man tcsh`。^(f) 在 `vi` 模式下，输入 `/` 后跟搜索字符串，然后按 Enter 键。按 `n` 跳转到下一个搜索结果。^(g) 作业控制：`tcsh` 不像其他 shell 那样智能地跟踪默认作业号，因此您可能经常需要将作业号（例如 `%1`）作为 `fg` 和 `bg` 的参数提供。^(h) stdout 和 stderr 的重定向：此 shell 的语法是：*`command`* `>` *`file`* `2>&1`。最后的术语 `2>&1` 意味着“重定向 stderr，即文件描述符 2，到 stdout，即文件描述符 1”。^(i) 在此 shell 中进行源文件时，需要明确指定源文件的路径，例如对于当前目录中的文件使用 `./myfile`，否则 shell 将无法找到文件。或者，将文件放入 shell 搜索路径中的一个目录中。 |
