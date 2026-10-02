# Android 内核跑 Docker：macvlan / ipvlan 排查与「B2 双网卡」最终方案

## 背景与目标

- 设备：Android 手机（已按 gist 教程改过内核，原生跑 Docker，非 VM/chroot 方案）。
- 起初现象：`bridge`、`host` 网络模式正常，`macvlan`、`ipvlan`"不能正常工作"。
- 最终目标（全部达成）：容器获得 **LAN 独立 IP + 外网 + 能 ping 通手机本体 IP**。

## 测试环境

- 手机：WiFi 客户端（STA）上网，自身 IP `192.168.0.126`，出口 `wlan0`。
- 网关真相：`192.168.0.172` **不是路由器，而是透明代理设备**（只旁路代理 TCP/HTTP；对外 ICMP 常被其状态表/上游处理掉）。真正路由器在其上游（宿主 `ping 8.8.8.8` 时会收到 `.172` 发出的 `ICMP Redirect → nexthop 192.168.0.1`，宿主内核吃下 redirect 后实际走 `.1`）。
- 内核：`CONFIG_NET_NS=y`、`CONFIG_MACVLAN=y`、`CONFIG_IPVLAN=y`、`CONFIG_VETH=y`（`zcat /proc/config.gz` 实测）。

## 关键结论（先行）

1. **不是内核配置问题，也不是 Docker 的问题。**
   - 四个相关选项全部 `=y` 内置；`ip link add link wlan0 name mv0 type macvlan mode bridge` 能成功建口；`docker network create -d macvlan/ipvlan` 都能建成功。
2. **macvlan 在 WiFi 客户端(STA)上数据面物理不可行**（所谓"驱动限制"的本质）：
   - AP 会丢弃"源 MAC ≠ 关联 STA MAC"的帧；macvlan 虚拟口带 Docker 生成的假 MAC（如 `02:42:c0:a8:32:02`,对应虚构 IP `192.168.50.2`），上空气发帧 → AP 过滤。
   - 实测容器内 ping 网关、外网全丢（仅自 ping 通）。任何 WiFi 客户端都一样；PC 上"没问题"是因为走有线。
3. **ipvlan L2 是 WiFi 上唯一可行的"独立 LAN IP"路线**（虚拟口共享手机真实 MAC，源 MAC 合法，AP 不丢帧），已在真机验证：独立 IP + 外网可用。
4. **死穴**：ipvlan L2 单网卡下容器 **ping 不通手机本体 IP**（源目同 MAC 自环，AP 同口丢帧）。
5. **最终解 = B2 双网卡**：`ipvlan(L2) + bridge` 双网卡，ipvlan 走 LAN/外网，bridge 走宿主栈回手机。**已实测定稿，五项验收全绿。**

## 排查证据链

### 1. 内核配置确认

```bash
zcat /proc/config.gz | grep -E 'CONFIG_(MACVLAN|IPVLAN|VETH|NET_NS)'
# CONFIG_NET_NS=y / CONFIG_MACVLAN=y / CONFIG_IPVLAN=y / CONFIG_VETH=y
```

### 2. "operation not supported / 驱动限制"的来历

- gist 教程只按 `check-config.sh` 勾选项（VETH、BRIDGE、iptables/netfilter、overlay…），该脚本**不检测 MACVLAN/IPVLAN**，厂方 defconfig 默认关闭，所以"驱动限制"。手机无 modprobe 场景，必须 `=y` 内置。
- `bridge`/`host` 正常无关 macvlan/ipvlan，它们依赖完全不同的内核组件。

### 3. macvlan 网络在宿主 `ip a` 看不到接口（正常）

- macvlan/ipvlan 驱动在 `network create` 时**不创建宿主可见接口**；虚拟口只在容器 attach 后放进**容器的网络命名空间**（容器内见 `eth0@if25`，`@if25` 即宿主 `wlan0` 的 index）。宿主侧永远看不到。

### 4. macvlan 判死实测

- 容器 MAC `02:42:c0:a8:32:02` 是从 Docker 虚构子网算出来的，与手机真实 MAC 无关 → 假 MAC 上空气出帧 → AP 丢。
- 容器内 `ping 网关/.1/.126` 全丢，仅自 ping 通 → **macvlan 在 WiFi STA 数据面失效定论**。

完整复现命令（当时实测记录）：

```bash
# 1) 建 macvlan 网络（能做成功 → 不是"建不了"）
sudo docker network create -d macvlan --subnet=192.168.0.0/24 \
  --gateway=192.168.0.172 --ip-range=192.168.0.200/29 \
  -o parent=wlan0 mvlan

# 2) 容器挂到 mvlan（能 attach → 口在容器 netns 内，宿主 ip a 看不到属正常）
sudo docker run --rm -it --network mvlan --ip 192.168.0.202 alpine sh

# 3) 容器内分级 ping（划分"建得成"与"连不通"两阶段）
#    容器内：ping 网关(192.168.0.172) / ping 外网(8.8.8.8)
#    通 => macvlan 全通；不通 => AP 按源 MAC 丢帧，macvlan 判死
```

实测：容器内 ping 网关、公网、手机 `.126` 全丢，仅自 ping `.202` 通 → 数据面失效。

### 5. ipvlan 首次运行 `device or resource busy`

- 根因：内核 `netdev_rx_handler` 一个物理口只能挂一个；之前手动建的 `mv0`(macvlan) 没删，占着 wlan0。修复：`sudo ip link del mv0` 后重试成功。

### 6. 为什么不能照搬 blog 的 macvlan2 trick（重要）

- macOS/Linux 常见套路：宿主再建一个 `macvlan2`，容器指 endpoint macvlan、宿主 `ip route add <容器IP> dev macvlan2`，实现容器↔宿主互通。
- 它只对 **macvlan 的独立 MAC** 生效（内核在 host/macvlan 对之间按 MAC 内切包）；ipvlan 共享手机 MAC 无法移植该套路；且本机 macvlan 在 WiFi 已判死，无从借用。

## ipvlan L2 单网卡方案（可行，但有死穴）

### 前置：获取真实网段（Termux 特殊点）

安卓由 netd 管理网络，主表没有 `default via`：

```bash
ip -4 addr show wlan0                    # 手机 IP+掩码 → 子网（192.168.0.0/24）
ip route show table 1025                 # netd 表里的 default via（表号以 ip rule 为准）
```

### 网络创建（生产模板）

```bash
sudo docker network create -d ipvlan \
  --subnet=192.168.0.0/24 \
  --gateway=192.168.0.172 \
  --ip-range=192.168.0.200/29 \
  --aux-address="phone=192.168.0.126" \
  -o parent=wlan0 \
  -o ipvlan_mode=l2 \
  l2net
```

要点（踩过的坑）：

- `--aux-address` 必须 `名字=IP`，值填**手机自身 IP**，把它排除出分配池。
- `--gateway` 必须真实网关（本环境 = 透明代理 `.172`）；**绝不能配成手机 IP**。
- `--ip-range` 从高位预留段分配，避开局域网 DHCP 池。
- 本方案不需要 gist 里 bridge 用的 `ip rule`/ndc 路由注册——ipvlan L2 帧在 `rx_handler` 层直接上下 `wlan0`，不经过宿主 IP 栈。

### 容器级测试

```bash
sudo docker run --rm -it --network l2net --ip 192.168.0.211 alpine sh
# 容器内依次执行：
ip addr; ip route          # 期望：eth0@if25，ip 正确，default via 网关
ping -c3 192.168.0.172     # ① 网关（LAN 层通不通）
ping -c3 192.168.0.126     # ② 手机本体（预期丢包，见下文）
ping -c3 8.8.8.8           # ③ 外网
wget -T 5 -qO- http://connectivitycheck.gstatic.com/generate_204 && echo OK
```

### 单网卡实测

| 目标 | 结果 | 说明 |
|---|---|---|
| 网关 `.172` | 通 | 共享 MAC 骗过 AP |
| 手机本体 `.126` | 100% 丢 | 源目同 MAC 自环（死穴） |
| 公网 `8.8.8.8` / HTTP | 通 | 方案可用性成立的依据 |

→ 单网卡通 LAN/外网，唯独够不到手机本体 → 引出 B2。

## B2 双网卡最终方案（定稿）

### 原理与拓扑

```
  svc 容器
   ├─ eth0  ipvlan L2 (192.168.0.210, 共享手机MAC)
   │        default via 192.168.0.172 dev eth0   ← LAN + TCP 外网 + ICMP 公网
   └─ eth1  veth→docker0 (172.17.0.x)
            192.168.0.126 via 172.17.0.1 dev eth1 ← 容器 ↔ 手机本体（经宿主栈）
```

### 完整命令

```bash
# 1) 网络（若重建；生产务必带 --ip-range + --aux-address）
sudo docker network create -d ipvlan --subnet=192.168.0.0/24 \
  --gateway=192.168.0.172 --ip-range=192.168.0.200/29 \
  --aux-address="phone=192.168.0.126" \
  -o parent=wlan0 -o ipvlan_mode=l2 l2net

# 2) 服务容器（--cap-add NET_ADMIN 必须；用真实镜像替换占位）
sudo docker run -d --name svc --network l2net --ip 192.168.0.210 \
  --cap-add NET_ADMIN --restart unless-stopped alpine sleep infinity

# 3) 挂第二网卡（实测本版 Docker 会在此把默认路由切到 eth1！）
sudo docker network connect bridge svc

# 4) 一次性路由修复（重启后由 entrypoint 兜底，勿靠 exec）
sudo docker exec svc ip route replace default via 192.168.0.172 dev eth0
sudo docker exec svc ip route replace 192.168.0.126 via 172.17.0.1 dev eth1
```

### 入口脚本（进镜像，解决持久化）

容器每次重启 Docker 都会重新 attach 两个网络并重抢默认路由到 eth1，
因此两条路由必须由 entrypoint 兜底：

```sh
#!/bin/sh
sleep 1                                                    # 等 eth0/eth1 就绪
ip route replace default via 192.168.0.172 dev eth0        # connect 抢走 → 归位
ip route replace 192.168.0.126 via 172.17.0.1 dev eth1     # 宿主通路 → 手机
exec "$@"
```

### 验收判据（`.210` 实测全绿）

| 项 | 结果 |
|---|---|
| `ping -c3 192.168.0.172`（透明代理） | 3/3 |
| `ping -c3 172.17.0.1`（宿主 docker0） | 3/3 |
| `ping -c3 192.168.0.126`（手机本体） | 3/3 |
| `ping -c3 8.8.8.8`（公网 ICMP） | 3/3 |
| `wget -T5 -qO- http://connectivitycheck.gstatic.com/generate_204` | 返回 0 |

完整命令：

```bash
sudo docker exec svc sh -c \
  'ip route; ping -c3 8.8.8.8; wget -T5 -qO- http://connectivitycheck.gstatic.com/generate_204; echo ===$?; \
   ping -c3 192.168.0.172; ping -c3 172.17.0.1; ping -c3 192.168.0.126'
```

## 已知风险与注意事项

- **IP 残留状态污染**：曾出现 `.201` 双网卡容器"三路 LAN 全通、HTTP 通、唯独 8.8.8.8 ICMP 0/3"，而同一时刻新 IP 单网卡 `.209` 3/3 全通；同 IP 重建为 `.210` 后全绿。系透明代理/网关对旧 IP 的 NAT/跟踪残留，`docker rm -f svc` 重建并不能清代理侧状态 → **外网异常时先换新 IP**。
- **透明代理接管 TCP**：`.172` 只代理 TCP，ICMP 公网是否可达取决于其状态表；服务对外连通性验收以 HTTP/TCP 为准（ping 当参考）。
- **`docker network connect` 抢默认路由**：本版 Docker attach 第二网络时把 default 指到 eth1，不归位则外网走 docker0（无 MASQUERADE 时必挂）。归位必须做进 entrypoint。
- **当前 l2net 无 `--ip-range`**：`.209/.210` 都在预留段外仍被 docker 放行。若局域网有 DHCP 服务，生产网络重建务必带 `--ip-range` + `--aux-address`。
- **WiFi 不稳定**：ipvlan 在 STA 上偏脆（漫游/省电/固件 offload 可能打断会话）。
- **路由器白名单**：若家用路由开"防蹭网/客户端隔离/DHCP 白名单"，需把容器 IP 绑静态租约并指向手机 MAC。
- **有线是最稳底牌**：OTG USB-Ethernet 做父口时 macvlan/ipvlan 都能满速稳定。

## 参考链接

- Docker on Android（内核 patch、docker 套件编译）：https://gist.github.com/FreddieOliveira/efe850df7ff3951cb62d74bd770dce27
- Droidspaces-OSS（LXC 式运行时，网络为 host/NAT/none，不含 macvlan/ipvlan）：https://github.com/ravindu644/Droidspaces-OSS
- Droidspaces 内核配置文档：https://github.com/ravindu644/Droidspaces-OSS/blob/main/Documentation/Kernel-Configuration.md