# 手机 Kali chroot 修复 `systemd` ENOTCONN 升级失败 — 完整流程记录

## 1. 背景与症状

- **场景**：Android 手机（Xiaomi SM8250 / 骁龙 865）上跑 Kali 真 chroot（有 root），rootfs 基于 `kali-desktop-xfce`（642 个可升级包）。
- **症状**：`apt full-upgrade` 配置 `systemd 261.1-2` 时 postinst 报错并退出 1：

  ```
  Setting up systemd (261.1-2) ...
  Cannot open '/etc/machine-id': Protocol driver not attached
  dpkg: error processing package systemd (--configure): ...
  ```

- 同系列错误还有 `Failed to enable units: Protocol driver not attached`。
- `Protocol driver not attached` = `ENOTCONN`。
- **后果**：systemd 半配置（half-configured），后面几十个包（binutils/gcc-15 栈、perl 5.40/5.42 栈、xserver-xorg 等）跟着"解包未配置"，触发器挂起。

## 2. 排查过程

### 2.1 machine-id 不是"文件缺失"

- `/etc/machine-id` 存在且有内容（如 `a9799c5ce8f04e6db68592fd8154c714`），`cat` 能正常读出。
- 问题在 **open() 被内核拒绝（ENOTCONN）**，而非文件不存在。Shell 读没问题，systemd 的写打开却拿 ENOTCONN。

### 2.2 方案 A：purge systemd/udev —— 不可行

想直接删掉 systemd 系，dry-run 证明被硬依赖链锁死：

```
kali-desktop-xfce → kali-desktop-core → dbus-user-session → libpam-systemd  (无替代)
init 的 PreDepends: systemd-sysv 或 sysvinit-core
bluez / udisks2 / upower / xserver-xorg-core 硬依赖 udev
```

### 2.3 方案 B：保留 systemd + 假 systemctl 中和 —— 仍报错

- 生成 machine-id、`/usr/local/bin/systemctl` 假 stub（no-op）、`policy-rc.d` 101、`/run/systemd/system` 标记。
- 升级时**依旧**死在同一个 machine-id 错误 → 说明不是"unit enable"那步，是 `systemd-machine-id-setup` 的 open 本身。

### 2.4 官方定位

检索确认这是**已知官方 bug**：
- **Kali bug tracker #0009722**（rootless-nethunter，同样报错）。
- **NetHunter GitLab issue #1558**（"Cannot open machine-id no data available"：文件存在、能读，但 open 返回 ENOTCONN，重新生成也失败）。
- NetHunter 维护者 **yesimxev** 给出官方临时方案：**整套 systemd 栈降到 259.1-1 并 `apt-mark hold`**（脚本见附录 A）。

## 3. 根因结论

- systemd **260/261** 的 postinst 在 Android/NetHunter chroot 内核上跑不通（machine-id-setup 打开 `/etc/machine-id` 返回 ENOTCONN），**是官方 bug，不是用户配置问题**。
- systemd **259** 无此问题（实测 259 的 postinst 顺利跑完、不再报错）。
- **深层根因（上游确认）**：systemd 260+ 把最低内核版本要求提到了 **≥ 5.10**，在 4.19 内核上执行新的 syscall/文件系统操作就返回 ENOTCONN（"Protocol driver not attached"）。见 upstream systemd issue #41250（"Updating hwdb database causes Protocol driver not attached"，确认最低内核 5.10）；已有人用 4.19.325 内核复现同一错误。
- 所以这不是"装坏了"或"配置错"，而是 **systemd 版本策略 vs 手机旧内核**的硬性不兼容。

## 4. 解决方案（降级到 259.1-1 + hold）

流程概要：

```
下载 12 个 259.1-1 arm64 deb（old.kali.org，https）
→ apt-mark hold 全部（必须在任何 apt 操作之前）
→ 删除 261 才拆出的子包（systemd-tpm 等，它们 Breaks systemd < 261）
→ dpkg 降级（全程只用 dpkg，绝不用 apt install -f）
→ systemd 本体最后装（避开 Pre-Depends 顺序问题）
→ 验证 dpkg --audit 为空
```

最终状态：12 个 systemd 系包停在 **259.1-1 已配置 + hold**，`dpkg --audit` 为空，`full-upgrade` 显示 `0 upgraded, 14 not upgraded`（12 个 hold + 连带被卡的 `network-manager-applet`/`nm-connection-editor`）。

## 5. 踩坑记录（重要）

1. **http 下载卡死** → 镜像 `old.kali.org` 用 **https** 就正常。
2. **deb 截断损坏**：`libsystemd-dev` 下载损坏（`dpkg-split: non-even byte size`）。→ 脚本每次先删旧 deb 重下，并校验"奇数大小 = 损坏"。
3. **最致命：`apt install -f -y` 在 hold 之前执行** → apt 认为"降级 = broken state"，把 10 个包**全部升回 261.1-2**，于是 systemd 261 的 postinst 又跑、又报错，白折腾一轮。**教训：hold 必须在一切 apt 操作之前，降级中途禁用 `apt install -f`，只用 dpkg。**
4. **systemd 的 Pre-Depends 顺序**：`systemd pre-depends on libsystemd-shared (= 259.1-1)`，而 dpkg 批量安装时 `libsystemd-shared` 只是"解包未配置"，pre-dep 检查不过。→ **systemd 本体单独放最后装**（配合 `--force-depends`）。
5. **261 拆出新子包 `systemd-tpm`**：它 `Breaks systemd (<< 261-2~)`，等于声明"systemd 低于 261 我就拆台"，直接挡住降级。259 池里没有这个包 → **只能 `dpkg -r` 删掉**（手机上用不到 TPM）。同类可能存在的：`systemd-networkd`、`systemd-udevd`、`systemd-tmpfiles`、`systemd-sysusers`（259 池均无，见 4/8 步骤脚本）。

## 6. 升级过程中的"无害噪音"（不必慌）

```
logger: socket /dev/log: No such file or directory   ← chroot 无 syslog 守护，PHP 等 postinst 写日志失败，无影响
Running in chroot, ignoring command '...'             ← 假 systemctl stub 正常吞调用
policy-rc.d returned 101, not running '...'           ← 设计内，阻止 postinst 启服务
```

## 7. 后续维护

### 7.1 保持现状

- 以后每次 `full-upgrade` 都会显示那 14 个 not upgraded，属预期，不是错误。
- 12 个 systemd 系包**一直锁在 259**，直到官方确认修复。

### 7.2 何时升级、怎么升

等 Kali 出修好 ENOTCONN 的新版本，**确认后再升**（看 bug #0009722 状态或 NetHunter 发布说明）：

```bash
apt-mark unhold systemd udev libsystemd0 libsystemd-shared libsystemd-dev libudev1 libudev-dev libpam-systemd libnss-systemd systemd-sysv systemd-timesyncd systemd-cryptsetup
apt-get update && apt-get full-upgrade -y
```

- 若新版仍有 ENOTCONN：会回到"systemd 半配置"状态，**重跑一遍脚本即可回 259**，脚本别删。
- 升级成功后，那 2 个连带被卡的 `nm-*` 包也会恢复可升级。

### 7.3 已知副作用：`libsystemd0` 锁 259 → 少数构建失败

- systemd 栈内部是**精确版本互锁**的（`systemd 259 Depends: libsystemd-shared (= 259.1-1)`、`libsystemd0 (= 259.1-1)`，libpam-systemd/libnss-systemd/systemd-sysv 也全部 `= 259.1-1`）→ **整套必须同版本，无法单独把 libsystemd0 升到 261**。
- **多数情况不受影响**：SONAME 为 `libsystemd.so.0`，259→261 未变，公开 API（sd-bus/sd-journal/sd-login）向后兼容，用 259 的头/库照常编译运行。
- **少数会挂**：Build-Depends 写死 `libsystemd-dev (= 261.1-2)` / `>= 260`，或使用 260/261 新 API 的程序。
- 该 bug 与"用户配置/自编内核的改动"无关，是 systemd 260+ 抬升最低内核版本（≥5.10）与 4.19 旧内核的硬性不兼容。此副作用**无解，只能保持锁 259**。

### 7.4 结论：本机 systemd 永久锁 259

- **根因**：systemd 260+ 最低内核要求 **≥ 5.10**，而手机（SM8250/K40）是 **4.19** 内核。
- **为什么升级无望**：LineageOS/Android 非官方内核锁死在厂商 BSP 版本。高通闭源驱动（显示/相机/ISP/基带/音频 DSP）绑定 4.19 的 KMI/驱动接口，主版本不升级。LineageOS 只跟进 **4.19 LTS 安全补丁**（4.19.325 → 4.19.3xx 不断涨次版本），主版本永远停 4.19。
- 结论：**这台设备 = systemd 永久锁 259，这是终态而非临时方案**。实际损失很小：
  - 除 12 个 systemd 系 + 2 个 `nm-*` 外，其余包全部正常升级。
  - 锁的是 `libsystemd.so.0`（SONAME/ABI 稳定），依赖它的绝大多数程序照常编译运行。
  - 只有显式要求 `libsystemd0 ≥ 261` 或 260/261 新 API 的构建会挂。
- **未来出路**：只有换 **GKI/社区 5.10+ 内核**（非 GKI 设备移植大工程，驱动难齐）或换一台出厂 Android 12+/GKI 的新设备，才有可能正常升级 systemd。
- **上游已明确抛弃 4.19**：这不是 Kali 的锅——Kali 只是打包上游 systemd。260+ 要求内核 ≥5.10 是上游统一策略（主流用户空间只支持仍在维护期的内核）；4.19 主线早已 EOL，只是 **CIP SLTS**（工业基础设施长期支持）在续命 4.19.3xx，所以内核"有更新"但永远不换主版本。**结论：没有任何"等修复"的希望，锁 259 就是永久状态。**

### 7.5 补充方案：APT 版本钉（pin），让官方安装命令原样跑通

- **问题**：NetHunter App 的官方安装命令（如蓝牙 Arsenal）里**显式点名 `libsystemd-dev`**。显式请求会**绕过 `apt-mark hold`**，apt 于是把它升到 261.1-2 → 需要 `libsystemd0 (= 261.1-2)` → 被 hold 卡死 → **整个安装事务失败**（连同 `girepository-tools`、`python3-gattlib` 等也被连坐报 "not going to be installed"）。
- **解法**：apt pin（优先级 1001，>1000 = "即使被显式点名也强制用指定版本"）。建 `/etc/apt/preferences.d/pin-systemd259`（内容见附录 B）后，官方命令里的 `libsystemd-dev` 解析到已装的 259.1-1，直接 "is already the newest version"，不再尝试升级。**之后 App 里任何菜单命令点名 systemd 系包都不会再撞墙。**
- 蓝牙栈本身**不依赖 `libsystemd-dev`**（那是给用 sd-bus API 编译程序的人用的头文件），已装的 259 版本足够。
- **将来升级**：删掉 pin 文件 + `apt-mark unhold` + `apt-get full-upgrade -y` 即可。

### 7.6 坑：遗留的 `/run/systemd/system` 标记会让 `service ssh start` 失效

- **现象**：`service ssh start` / NetHunter App 里启动 sshd 都"成功但没起来"；直接 `/usr/sbin/sshd -D -d -e`（绝对路径 + debug）却能起。
- **原因**：Plan B 时代手工建的 `/run/systemd/system` 目录仍留在 chroot 里。`service`（以及部分 init 脚本）看到该目录存在就认为 systemd 在跑，把 `service ssh start` 委派成 `systemctl start ssh.service` → 落到 `/usr/local/bin/systemctl` 的**假 stub**（no-op 返回 0）→ 静默无事发生。`/tmp/systemctl.log` 里能看到 `[fake] systemctl start ssh.service`。
- **修复**：`rm -rf /run/systemd/system`。chroot 的 `/run` 是 rootfs 真实目录，删除**重启后依然生效**（标记不会自动重建）。
- **注意**：新版 OpenSSH 还要求以绝对路径执行（`sshd` 裸敲报 "requires execution with an absolute path"），请用 `/usr/sbin/sshd` 或 `service ssh start`。
- **遗留物检查清单**：`ls -d /run/systemd`（应为空/不存在）、`/usr/local/bin/systemctl` stub（可留，仅在被显式调用时生效）、`/usr/sbin/policy-rc.d`（可留）。

## 8. 参考资料

- Kali bug tracker **0009722** — rootless-nethunter 同错误：`https://bugs.kali.org/view.php?id=9722`
- NetHunter GitLab **kalilinux/nethunter/build-scripts/-/issues/1558** — 维护者 yesimxev 给出 259 锁定方案：`https://gitlab.com/kalilinux/nethunter/build-scripts/kali-nethunter-project/-/issues/1558`
- NetHunter GitLab **issue #1560** — 重复报告（"Error /etc/machine-id driver not attached"）：`https://gitlab.com/kalilinux/nethunter/build-scripts/kali-nethunter-project/-/work_items/1560`
- 上游 systemd **issue #41250** — "Updating hwdb database causes Protocol driver not attached"，确认最低内核版本 **5.10**：`https://github.com/systemd/systemd/issues/41250`
- 镜像：`https://old.kali.org/kali/pool/main/s/systemd/`

## 附录 A — 修复脚本（已存于 `C:\Users\user\work\nethunter-fix-systemd.sh`）

```bash
#!/bin/bash
#
# Downgrade the whole systemd stack to 259.1-1 and hold it.
# NetHunter maintainer workaround for:
#   Setting up systemd ... Cannot open '/etc/machine-id': Protocol driver not attached
#   Failed to enable units: Protocol driver not attached
# References: Kali bug 0009722; gitlab.com/kalilinux/nethunter/build-scripts/-/issues/1558

set +e

if [ "$(id -u)" != "0" ]; then
    echo "ERROR: run this script as root inside the chroot"
    exit 1
fi

BASE="https://old.kali.org/kali/pool/main/s/systemd"
DIR="/tmp/sysd259"
mkdir -p "$DIR"
cd "$DIR" || exit 1

# packages that have a 259.1-1 build in the pool (maintainer list + 259-era extras)
HAS259="udev libudev-dev libsystemd-shared libudev1 libpam-systemd systemd-sysv libsystemd-dev libsystemd0 systemd-timesyncd libnss-systemd systemd systemd-cryptsetup systemd-resolved systemd-oomd systemd-container"
# 261-only split packages (no 259 build) that Breaks systemd < 261
BREAKS="systemd-tpm systemd-networkd systemd-udevd systemd-tmpfiles systemd-sysusers"

installed() { dpkg-query -W -f='${Status}' "$1" 2>/dev/null | grep -q 'install ok'; }

echo "[1/8] downloading systemd 259.1-1 (arm64) fresh"
rm -f *.deb
ok=1
for p in $HAS259; do
    f="${p}_259.1-1_arm64.deb"
    wget -q -t 3 -T 30 "$BASE/$f" -O "$f" || { echo "  FAILED: $f"; ok=0; }
done
ls -l *.deb
[ "$ok" = "1" ] || { echo "some downloads failed, aborting"; exit 1; }
for f in *.deb; do
    sz=$(stat -c %s "$f")
    if [ $(( sz % 2 )) -ne 0 ]; then echo "  CORRUPT (odd size): $f"; exit 1; fi
done
echo "  all debs OK"

echo "[2/8] holding installed 259-capable systemd packages BEFORE apt"
HOLD=""
for p in $HAS259; do
    if installed "$p"; then HOLD="$HOLD $p"; fi
done
apt-mark hold $HOLD
apt-mark showhold

echo "[3/8] removing 261-only subpackages that Breaks systemd < 261"
for p in $BREAKS; do
    if installed "$p"; then
        echo "  dpkg -r $p"
        DEBIAN_FRONTEND=noninteractive dpkg -r "$p"
    fi
done

echo "[4/8] downgrading installed stack packages to 259 (dpkg only)"
DOWN=""
for p in $HAS259; do
    [ "$p" = "systemd" ] && continue
    if installed "$p"; then DOWN="$DOWN ${p}_259.1-1_arm64.deb"; fi
done
DEBIAN_FRONTEND=noninteractive dpkg -i $DOWN
DEBIAN_FRONTEND=noninteractive dpkg --configure -a

echo "[5/8] installing systemd 259 LAST"
DEBIAN_FRONTEND=noninteractive dpkg --force-depends -i systemd_259.1-1_arm64.deb
DEBIAN_FRONTEND=noninteractive dpkg --configure systemd
DEBIAN_FRONTEND=noninteractive dpkg --configure -a

echo "[6/8] audit"
dpkg --audit
dpkg -l systemd | tail -n 1

echo "[7/8] remaining systemd-family packages above 259 (should be none)"
dpkg-query -W -f='${Package} ${Version}\n' 2>/dev/null \
  | grep -E '^(systemd|udev|libsystemd|libudev|libpam-systemd|libnss-systemd)' \
  | grep -v ' 259\.1-1' || echo "  none"

echo "[8/8] done. next: apt-get update && apt-get full-upgrade -y"
```

## 附录 B — apt pin 文件（已存于 `C:\Users\user\work\pin-systemd259`，放到 chroot 的 `/etc/apt/preferences.d/pin-systemd259`）

```
# Force the systemd stack to stay at 259.1-1.
# Reason: systemd 260+ requires kernel >= 5.10; this phone runs a 4.19 kernel,
# so systemd 260/261 postinst fails with "Cannot open '/etc/machine-id':
# Protocol driver not attached" (ENOTCONN). 259 is the last working version.
# NetHunter official install commands explicitly name libsystemd-dev etc.;
# an explicit request overrides `apt-mark hold`, so we pin here with
# Pin-Priority 1001 (>1000 = forced even when explicitly requested).
Package: systemd udev libsystemd0 libsystemd-shared libsystemd-dev libudev1 libudev-dev libpam-systemd libnss-systemd systemd-sysv systemd-timesyncd systemd-cryptsetup systemd-resolved systemd-oomd systemd-container
Pin: version 259.1-1
Pin-Priority: 1001
```
