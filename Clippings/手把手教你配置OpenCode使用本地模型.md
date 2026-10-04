---
title: "手把手教你配置OpenCode使用本地模型"
source: "https://zhuanlan.zhihu.com/p/2004217236290835124"
author:
  - "[[AGI杂货铺]]"
published:
created: 2026-10-04
description: "今天我将详细介绍如何配置 OpenCode 使用本地 Ollama 模型，实现隐私保护且无使用限制的 AI 辅助编程体验。 安装 Ollama方式一：使用官方安装脚本（Linux/macOS）curl -fsSL https://ollama.com/install.sh | sh方…"
tags:
  - "clippings"
---
14 人赞同了该文章

今天我将详细介绍如何配置 OpenCode 使用本地 [Ollama](https://zhida.zhihu.com/search?content_id=270173542&content_type=Article&match_order=1&q=Ollama&zhida_source=entity) 模型，实现隐私保护且无使用限制的 AI 辅助编程体验。

## 安装 Ollama

### 方式一：使用官方安装脚本（Linux/macOS）

```
curl -fsSL https://ollama.com/install.sh | sh
```

### 方式二：应用下载

直接点击下载应用，地址 [ollama.com/download](https://ollama.com/download)

![](https://pic3.zhimg.com/v2-48d7d7d25abc02ed657962a4ee82ec34_1440w.jpg)

### 验证安装

安装完成后，启动 Ollama 服务：

```
ollama serve
```

在另一个终端窗口验证：

```
ollama --version
```
![](https://pic4.zhimg.com/v2-13dde52abd5c0bf99a11946088760a9b_1440w.jpg)

## 安装 OpenCode

OpenCode 可以通过 npm 安装。确保您已安装 Node.js 20+。

### 使用 npm 全局安装

```
npm install -g opencode
```

### 验证安装

```
opencode --version
```

如果显示版本号，说明安装成功。

## 下载并配置模型

### 推荐模型

根据 Ollama 官方文档，以下模型适合用于代码编写：

- `qwen3-coder` - 专门用于代码生成的模型
- `gpt-oss:20b` - 20B 参数的开源模型
- `gpt-oss:120b` - 120B 参数的大模型（需要更多资源）

### 下载模型

```
# 下载 Qwen 3 8b 模型进行测试，参数量小
ollama pull qwen3:8b 

# 或者下载 GPT-OSS 20B 模型
ollama pull gpt-oss:20b
```

应用版ollama可以直接点击下载

![](https://pic1.zhimg.com/v2-8bc04d09d9e3c4f367c97ddd6d619c1a_1440w.jpg)

### 修改模型参数

运行起模型后，可以修改模型的参数，比如上下文长度

```
# 运行模型
ollama run qwen3:8b

# 在 Ollama 提示符中设置上下文
>>> /set parameter num_ctx 16384

# 保存为新模型变体
>>> /save qwen3:8b-16k

# 退出
>>> /bye
```

执行 `ollama list` 后可以看到保存的新模型

![](https://pic3.zhimg.com/v2-28d43c23fda67a654cc98fd0a27de16a_1440w.jpg)

## 配置 OpenCode 连接 Ollama

### 快速配置

快速启动

```
ollama launch opencode
```

也可以在不启动的情况下配置

```
ollama launch opencode --config
```
![](https://picx.zhimg.com/v2-1ff7710d9f19d991e4608b72b1935def_1440w.jpg)

### 手动配置

OpenCode 的配置文件位于： `~/.config/opencode/opencode.json`

如果文件不存在，创建它：

```
mkdir -p ~/.config/opencode
nano ~/.config/opencode/opencode.json
```

### 基本配置

将以下内容添加到配置文件：

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
        "qwen3:8b": {
          "_launch": true,
          "name": "qwen3:8b"
        },
        "qwen3:8b-16k": {
          "_launch": true,
          "name": "Qwen3 8B(16k)"
        }
      }
    }
  }
}
```

### 配置说明

- **`ollama`**: 自定义提供商 ID（可自定义名称）
- **`npm`**: 指定 AI SDK 包，Ollama 使用 OpenAI 兼容 API，所以使用 `@ai-sdk/openai-compatible`
- **`baseURL`**: 本地 Ollama 服务地址，默认为 `http://localhost:11434/v1`
- **`models`**: 模型映射，键为 Ollama 模型名称，值为显示名称
![](https://pica.zhimg.com/v2-0c7a50476b5b17f8808787eb90b960f4_1440w.jpg)

## 参考资料

Ollama 官方文档 [docs.ollama.com/integra](https://docs.ollama.com/integrations/opencode)
