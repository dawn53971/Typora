### 跨windows和WSL文件系统

建议不要跨操作系统使用文件，除非有这么做的特定原因。 若想获得最快的性能速度，请将文件存储在 WSL 文件系统中，前提是在 Linux 命令行（Ubuntu、OpenSUSE 等）中工作。 如果使用 Windows 命令行（PowerShell、命令提示符）工作，请将文件存储在 Windows 文件系统中。

可使用以下命令从命令行打开 Windows 文件资源管理器，以查看存储文件的目录：

```bash
explorer.exe .
```

若要在 Windows 文件资源管理器中查看所有可用的 Linux 发行版及其根文件系统，请在地址栏中输入：`\\wsl$`

##### 下面是几个使用 PowerShell 混合 Linux 和 Windows 命令的示例。

若要使用 Linux 命令 `ls -la` 列出文件，并使用 PowerShell 命令 `findstr` 来筛选包含“git”的单词的结果，请组合这些命令：

```powershell
wsl ls -la | findstr "git"
```

若要使用 PowerShell 命令 `dir` 列出文件，并使用 Linux 命令 `grep` 来筛选包含“git”的单词的结果，请组合这些命令：

```powershell
C:\temp> dir | wsl grep git
```

WSL 可以使用 `[tool-name].exe` 直接从 WSL 命令行运行 Windows 工具。 例如，`notepad.exe`。