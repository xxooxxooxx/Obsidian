
https://learn.microsoft.com/en-us/powershell/scripting/install/install-powershell-on-windows?view=powershell-7.6

Install the PowerShell 7 MSIX package:

PowerShell

```
winget install --id Microsoft.PowerShell --source winget
```

Use the following command to install the PowerShell 7 MSI package:

PowerShell
```
winget install --id Microsoft.PowerShell --source winget --installer-type wix
```


| 特性 | **第一条命令 (MSIX / 默认)** | **第二条命令 (`--installer-type wix` / MSI)** |
| :--- | :--- | :--- |
| **包格式** | **MSIX**（现代 Windows 应用格式，类似 Microsoft Store 应用） | **MSI**（传统的 Windows Installer 格式） |
| **安装路径** | 位于高度受限的沙盒目录：<br>`$Env:ProgramFiles\WindowsApps\` | 位于标准程序目录：<br>`$Env:ProgramFiles\PowerShell\7\` |
| **后台更新** | **全自动**。由 Windows Update 或应用商店在后台静默更新。 | **手动或通过 WinGet**。需要通过运行 `winget upgrade` 或 Microsoft Update 接收更新。 |
| **写入权限** | **受限**。无法直接修改 PowerShell 根目录（`$PSHOME`）下的文件。 | **完全访问**。管理员可以自由修改和管理该目录。 |
| **功能限制** | 无法运行某些需要写入核心目录的传统命令（如注册会话、全局更新帮助文档等）。 | **没有任何限制**。完美兼容所有管理和企业级方案。 |
| **适用人群** | **普通/临时用户**。只想简单体验、不希望手动管更新的人。 | **高级用户/系统管理员/开发者**。需要无限制环境或进行企业自动化部署的人。 |

解决方案：通过配置文件强制指定 `pwsh`

OpenCode 的高级设置存在于本地的 `opencode.json` 文件中，直接修改它即可： [[1](https://github.com/anomalyco/opencode/issues/30615)]

1. **找到配置文件：**  
    按下 `Win + R` 打开运行窗口，输入以下路径并回车：
    
    text
    
    ```
    %USERPROFILE%\.config\opencode\
    ```
    
    请谨慎使用此类代码。
    
    _(或者直接在你的用户文件夹，寻找 `.config/opencode/` 目录)_。 [[1](https://github.com/anomalyco/opencode/issues/30615)]
2. **编辑 `opencode.json`：**  
    在这个目录下找到 **`opencode.json`** 文件，用记事本（Notepad）或任何代码编辑器打开它。 [[1](https://github.com/anomalyco/opencode/issues/30615)]
3. **修改或添加 `shell` 字段：**  
    在 JSON 的主大括号 `{}` 内部，找到 `"shell"` 配置项（如果没有就手动加上一行）。将其值直接改为 **`"pwsh"`**：
    
    json
    
    ```
    {
      "shell": "pwsh"
    }
    ```
    
    请谨慎使用此类代码。
    
    _提示：如果里面本来就有其他代码，记得在各行之间用逗号 `,` 隔开。_ [[1](https://github.com/anomalyco/opencode/issues/30615)]
4. **重启 OpenCode：**  
    保存文件并关闭。彻底关闭 OpenCode 的后台进程再重新打开它，此时它的底层的 PowerShell 就会强制锁定为全新的 PowerShell 7。
