---
title: "一步一步带你安装 OpenCode，并使用 ollama 本地模型"
source: "https://zhuanlan.zhihu.com/p/2007823258364424221"
author:
  - "[[人工智能研究所​]]"
published:
created: 2026-10-04
description: "简单的说， OpenCode 就是你终端里的“全能程序员分身”——它不仅能动口帮你出主意，还能直接动手帮你读文件、改代码并跑通命令。 为什么你会需要这个“终端老司机”？如果你也厌倦了在 ChatGPT 网页和 IDE 之间…"
tags:
  - "clippings"
---
[收录于 · AI 人工智能](https://www.zhihu.com/column/c_1561766820297846784)

18 人赞同了该文章

简单的说， **OpenCode 就是你终端里的“全能程序员分身”——它不仅能动口帮你出主意，还能直接动手帮你读文件、改代码并跑通命令。**

![](https://pica.zhimg.com/v2-4722674e610fe8e6ce78bf07a4032092_1440w.jpg)

## 为什么你会需要这个“终端老司机”？

如果你也厌倦了在 ChatGPT 网页和 IDE 之间来回切窗口、不停地复制粘贴代码，那么 OpenCode 就是为你准备的。它最大的魅力在于“本地执行力”：

- **它有“眼睛”：** 能直接扫描你的整个工程目录，不需要你手动把几百行代码贴给它，它自己就能看懂复杂的项目结构。
- **它有“手”：** 它可以直接在你的终端里运行 npm install、git commit 或者执行单元测试。
- **它有“脑子”：** 你给它一个模糊的目标（比如“帮我把这个项目的背景色改成深色模式，并确保所有按钮依然清晰”），它会自己拆解步骤：查找 CSS、修改文件、运行预览、检查报错，直到任务完成。
![](https://pic3.zhimg.com/v2-0c6208e1f680599a82eb9b94ecbd5d4c_1440w.jpg)

## 一个典型的使用场景

想象一下，你接手了一个乱糟糟的旧项目，正愁无从下手：

> **你：** “OpenCode，帮我分析下这个项目的 API 调用逻辑，写个文档存到 docs 文件夹，顺便把过时的 axios 调用改成最新的 fetch 语法。”  
> **OpenCode：** “没问题。”（然后你就能看到它在终端里自动跳动，翻找文件、重写逻辑、创建文档，最后深藏功名地告诉你：搞定了。）

它把 AI 从一个“只会聊天的专家”变成了“能帮你干活的学徒”。

![](https://pic1.zhimg.com/v2-b6346ce6e6a71498dce4ea3716e9d6fe_1440w.jpg)

## 从 0 安装 opencode

opencode 的安装，Windows 版本有对应的 exe 文件，当然 Mac 与 Linux 也有对应的安装包，可以一键安装，但是是 beta 版本，我第一次安装 Mac 版本的安装包，打开直接提示有问题，我们从源码安装。

```
第一步：curl -fsSL https://opencode.ai/install | bash
```

只需在 cmd 终端里面输入以上命令，脚本会自动下载，并安装

![](https://pic3.zhimg.com/v2-80ed0461b576d2c17cb49f1ed1204322_1440w.jpg)

这里只需要等待安装完成即可

```
Installing opencode version: 1.1.53
■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■ 100%
Manually add the directory to /Users/Macname/.zshrc (or similar):
  export PATH=/Users/Macname/.opencode/bin:$PATH
                                 ▄     
█▀▀█ █▀▀█ █▀▀█ █▀▀▄ █▀▀▀ █▀▀█ █▀▀█ █▀▀█
█░░█ █░░█ █▀▀▀ █░░█ █░░░ █░░█ █░░█ █▀▀▀
▀▀▀▀ █▀▀▀ ▀▀▀▀ ▀  ▀ ▀▀▀▀ ▀▀▀▀ ▀▀▀▀ ▀▀▀▀

OpenCode includes free models, to start:
cd <project>  # Open directory
opencode      # Run command
For more information visit https://opencode.ai/docs
```

这里小编电脑安装完成后，提示没有添加环境变量，需要自行添加

直接在 cmd 终端输入

```
echo 'export PATH="/Users/Macname/.opencode/bin:$PATH"' >> ~/.zshrc
```

直接提示

```
zsh: permission denied: /Users/Macname/.zshrc
```

说明我没有文件夹权限，无法设置本文件夹的环境变量

```
ls -l ~/.zshrc
提示
-rw-------  root  staff  ...
说明文件夹权限不属于用户
```

修改文件夹权限

```
sudo chown Macname:staff ~/.zshrc
chmod u+w ~/.zshrc
```

获取文件夹权限后

```
source ~/.zshrc
或者
重启 cmd 终端
输入opencode --version就能看到版本号了
第二步：与 opencode 的第一次接触
直接在 cmd 终端里面输入 opencode 即可打开
```
![](https://picx.zhimg.com/v2-2d48e792d0dada8e887fddfaf5eed053_1440w.jpg)

打开 opencode 后，输入

```
/models
```
![](https://pic2.zhimg.com/v2-01d5c60c50d2f310f925227efdf47539_1440w.jpg)

这里选择使用的 free 模型即可，当然，你若有Claude 或者 ChatGPT 的 API ，也可以使用收费模型，选择完成后，即可跟 opencode 沟通了。

![](https://pic4.zhimg.com/v2-2526d2f9a45f7cb3df7c86a3f69850a7_1440w.jpg)

随便输入一些信息，opencode 回复你，说明整个 opencode 完成了。

## 使用 ollama 下载本地模型

opencode 自带 kimi-k2.5以及 minimax 与GLM-4.7 的免费模型，针对大家平时使用完全够用了，但是很多情况是，以上模型是在线的模型，出于文件，信息安全的考虑，我们可以考虑使用本地模型。

ollama使用开源的，并下载本地模型及其简单，我们前期的文章介绍了很多 ollama 的教程，这里不再赘述。

下载完成本地模型，并打开 ollama，加载本地模型

```
通用对话（轻量）：llama3.2:3b（约 2.0GB 下载）
通用对话（更强）：llama3.1:8b（约 4.9GB 下载）
中文/通用能力：qwen2.5:7b（约 4.7GB 下载）、qwen2.5:14b（约 9.0GB 下载）
编程优先：qwen2.5-coder:7b（约 4.7GB 下载）、qwen2.5-coder:14b（约 9.0GB 下载）、qwen2.5-coder:32b（约 20GB 下载）
其他常用：gemma2:9b（约 5.4GB 下载）、mistral:7b（约 4.4GB 下载）
MacBook（Apple Silicon）16GB 统一内存（例如 M4 16GB）    3B~8B    llama3.2:3b / llama3.1:8b / qwen2.5-coder:7b
独显 16GB 显存（例如 16GB VRAM 的 NVIDIA/AMD）    7B~14B    qwen2.5-coder:14b / qwen2.5:14b / gemma2:9b
独显 24GB 显存    14B~32B    qwen2.5-coder:32b / qwen2.5:32b / gemma2:27b
独显 32GB 显存（以 RTX 5090 32GB 为例）    32B    qwen2.5-coder:32b / qwen2.5:32b
独显 48GB+ 显存    70B/72B    llama3.1:70b / qwen2.5:72b
```
![](https://pica.zhimg.com/v2-178df8dd9313a9daac95c740ece0fac0_1440w.jpg)

当然，本地模型需要根据自己的配置，下载合适的模型。

```
配置模型
全局：~/.config/opencode/opencode.json（所有项目通用）
项目级：把 opencode.json 放到当前项目根目录（只对这个项目生效）
```

opencode.json是一个配置文件，opencode 打开时会读取其文件

```
{
  "$schema": "https://opencode.ai/config.json",
  "provider": {
    "ollama": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "Ollama (local)",
      "options": {
        "baseURL": "http://localhost:11434/v1"
      },
      "models": {
        "qwen2.5-coder:7b": { "name": "Qwen2.5 Coder 7B (local)" },
        "llama3.1:8b": { "name": "Llama 3.1 8B (local)" },
        "llama3.2:3b": { "name": "Llama 3.2 3B (local)" }
      }
    }
  }
}
```

当然，前提是需要使用 ollama 下载本地模型到你电脑里面，重启 opencode 即可看到本地模型。若你对文件信息安全问题不考虑，建议直接使用在线的免费模型，不需要下载几个甚至几十 G 的模型文件。

## VScode 与 opencode 联合使用

opencode 是一个终端界面，你在里面聊天或者修改代码，查看文件等，都不是很方便，我们可以直接在 vs code 里面打开 opencode，让 vs code 作为 GUI 编程界面，而 opencode 作为一个 agent。

我们只需要在 vscode终端打开opencode 即可

![](https://picx.zhimg.com/v2-b54979d2456724b5122a57e2873319df_1440w.jpg)

小编第一次在 vscode 里面打开提示

```
command not found: opencode
VS Code 没读到你的 shell PATH
```

这里依然是环境变量问题，在 cmd 终端里面，输入

```
echo 'export PATH="/Users/liruitao/.opencode/bin:$PATH"' >> ~/.zprofile
```

重启 opencode与 VScode 即可



