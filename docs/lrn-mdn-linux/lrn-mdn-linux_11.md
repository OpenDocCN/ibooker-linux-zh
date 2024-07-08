# 附录 B. 现代 Linux 工具

本附录聚焦于现代 Linux 工具和命令。其中一些命令是现有命令的即插即用替代品，而另一些是全新的。这些工具大多数都改善了用户体验（UX），包括简化用法和使用彩色输出，从而提高了效率流程。

我已经整理了一个相关工具列表，详见表 B-1，展示了功能和潜在的替代方案。

表 B-1\. 现代 Linux 工具和命令

| 命令 | 许可证 | 特性 | 可替换或增强： |
| --- | --- | --- | --- |
| [`bat`](https://oreil.ly/zg9xE) | MIT 许可证 和 Apache 许可证 2.0 | 显示、分页、语法高亮 | `cat` |
| [`envsubst`](https://oreil.ly/4i1gz) | MIT 许可证 | 基于模板的环境变量 | 不适用 |
| [`exa`](https://oreil.ly/F3dRV) | MIT 许可证 | 有意义的彩色输出，理智的默认设置 | `ls` |
| [`dog`](https://oreil.ly/tHgYT) | 欧洲联盟公共许可证 v1.2 | 简单强大的 DNS 查询 | `dig` |
| [`fx`](https://oreil.ly/oCQ20) | MIT 许可证 | JSON 处理工具 | `jq` |
| [`fzf`](https://oreil.ly/0I0Va) | MIT 许可证 | 命令行模糊查找器 | `ls` + `find` + `grep` |
| [`gping`](https://oreil.ly/psKX3) | MIT 许可证 | 多目标，图形化 | `ping` |
| [`httpie`](https://oreil.ly/pu9f2) | BSD 3-Clause “New” 或 “Revised” 许可证 | 简单的用户体验 | `curl`（还有 `curlie`） |
| [`jo`](https://oreil.ly/VhLXG) | GPL 许可证 | 生成 JSON | 不适用 |
| [`jq`](https://oreil.ly/tL5fR) | MIT 许可证 | 本地 JSON 处理器 | `sed`，`awk` |
| [`rg`](https://oreil.ly/n9Jmj) | MIT 许可证 | 快速，理智的默认设置 | `find`，`grep` |
| [`sysz`](https://oreil.ly/aYGlL) | The Unlicense | `fzf` 的 `systemctl` 用户界面 | `systemctl` |
| [`tldr`](https://oreil.ly/wDQwB) | CC-BY（内容）和 MIT 许可证（脚本） | 重点放在命令的使用示例上 | `man` |
| [`zoxide`](https://oreil.ly/Fx2kI) | MIT 许可证 | 快速更改目录 | `cd` |

要了解本附录中列出的许多工具的背景和用法，请利用以下资源：

+   欢迎收听来自 *The Changelog: Software Development, Open Source* 的 [现代 UNIX 工具](https://oreil.ly/9sfmW)的播客集。

+   现在可以通过 GitHub 仓库[Modern UNIX](https://oreil.ly/LdtI2)查看现代工具的活跃列表。
