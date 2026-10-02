# VoiceStudio (OmniVoice) 后端端口绑定失败修复备忘 — Errno 13 / port 3900

> 场景：Windows 上安装/启动 VoiceStudio（原 OmniVoice-Studio），后端硬编码监听
> `127.0.0.1:3900`，启动报错退出。
> 日期：2026-09-01。状态：已修复并验证。

## 一、现象

安装/启动 VoiceStudio 时，后端日志报错后退出：

```
INFO:     Started server process [4xxx]
INFO:     Application startup complete.
ERROR:    [Errno 13] error while attempting to bind on address ('127.0.0.1', 3900): 以一种访问权限不允许的方式做了一个访问套接字的尝试。
INFO:     Application shutdown complete.
```

前端（`msedgewebview2.exe`）反复向 3900 发连接（netstat 显示大量 `SYN_SENT`），
因为没人监听而连不上。

## 二、根因

这台机器**启用了 Hyper-V（因为要用 WSL）**。Hyper-V/WSL/WinNAT 会**动态预留**
一段 TCP 端口排除范围（Excluded Port Range），位于该范围内的端口**无法被普通应用
bind**。用下述命令查看：

```
netsh int ipv4 show excludedportrange protocol=tcp
```

故障时输出含：

```
开始端口    结束端口
----------    --------
      3874        3973      ← 3900 正好落在这个 Hyper-V 动态排除段里
      3974        4073
      4074        4173
```

因为拦截发生在**系统网络栈层面**，所以以下手段全部无效、绕不过：
- 以管理员身份运行（Failed）
- 临时关闭 Windows 防火墙（Failed）
- 在安装界面点"清理并重试"（不是管理员运行）

## 三、修复（最终生效的关键）

`excludedportrange` 后来显示 3900 已变为"**管理的端口排除**"（带 `*`，手动固定
预留），之前的大动态段 `3874-3973` 消失——即 3900 已被成功固定预留，Hyper-V 不再
抢占它，VoiceStudio 得以成功绑定并启动。

让 3900 进入固定预留的原因/操作（Windows 手动固定预留端口的通用命令）：

```
netsh int ipv4 add excludedportrange protocol=tcp startport=3900 numberofports=1
```

配套思路（若需先释放动态段再固定，供将来复发时参考）：
```
wsl --shutdown                 # 停 WSL 释放动态段（非系统重启）
net stop winnat                # 停 WinNAT，触发动态段重建
net stop WslService
netsh int ipv4 add excludedportrange protocol=tcp startport=3900 numberofports=1
net start winnat
net start WslService
wsl
```

## 四、验证方法

- **排除段查看**（判断3900归属）：
  ```
  netsh int ipv4 show excludedportrange protocol=tcp
  ```
- **注意**：`Test-NetConnection 127.0.0.1 -Port 3900` 返回 False 是**正常**的，
  它只表示"当前没有进程在监听3900"（后端没启动时本来就没有），**测不出能否绑定**；
  真正的判定=启动后端是否还报 Errno 13。

## 五、遗留注意点（将来可能复发）

1. **VoiceStudio 端口 3900 是硬编码的**（前后端都基于 `http://127.0.0.1:3900`，
   无官方"改端口"配置开关，`OMNIVOICE_API_BASE` 仅用于转发/LAN 场景）。
2. 3900 现在是**手动固定预留**状态。若**重启系统后 WSL/Hyper-V 动态段重建**、
   3900 又落回动态段，VoiceStudio 可能再次报 Errno 13——届时按"三"的固定预留操作
   重新处理。
3. 若想移除固定预留让该端口归还系统管理：
   ```
   netsh int ipv4 delete excludedportrange protocol=tcp startport=3900 numberofports=1
   ```
   注意可能在 Hyper-V 动态分配后无法删除（会报错被占用）。

## 六、附：ffmpeg（非本次问题，仅记录）

ffmpeg/ffprobe 由 VoiceStudio 自解析可在 **Settings → Audio tools** 里处理
（Restore bundled / Use system copy），或 `winget install ffmpeg`。日志中的
"ffmpeg 下载失败"非致命，可重试。

## 七、参考

- VoiceStudio 官方排障文档：仓库 `docs/install/troubleshooting.md`（含各错误分类）
  确认 3900 为硬编码端口，且已知存在"fails on port 3900"一类问题。