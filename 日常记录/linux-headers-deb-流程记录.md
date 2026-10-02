# 手机 Kali chroot 生成 `linux-headers` deb 包 — 完整流程记录

## 1. 背景与目标

- **场景**：Android 手机（Xiaomi SM8250 / 骁龙 865）上跑 Kali chroot（真 chroot，有 root），运行内核 `4.19.325-cip128-st12-perf-g13b18ecca301`（LineageOS/NetHunter 分支编译刷入）。
- **问题**：Kali 是滚动发行，仓库里的 `linux-headers` 是 7.x，与手机 4.19 内核不匹配，DKMS/内核模块无法编译、无法加载。
- **目标**：用手头 4.19.325 内核源码，在 chroot 内**原生编译**生成匹配的 `linux-headers-<version>` deb 包并安装，走通 DKMS 编译流程。

**为什么选"手机内原生编译"**：若在 PC 上交叉编译，`scripts/mod/modpost` 等 host 工具会按 `HOSTCC`(x86) 编译，装到 arm64 手机上无法执行（Exec format error）。原生编译产物全是 arm64。

## 2. 环境

| 项      | 值                                                              |
| ------ | -------------------------------------------------------------- |
| 手机运行内核 | `4.19.325-cip128-st12-perf-g13b18ecca301`                      |
| 源码     | LineageOS 分支 `nethunter-dev`，4.19.325（CAF 回迁了大量 5.x kbuild 脚本） |
| chroot | 真 chroot（有 root），`/proc/config.gz` 可读                          |
| 磁盘     | `/` 226G，可用 124G                                               |
| 内存     | 7.6G 总量，可用 ~2.3G → 编译并发必须 `-j2`                                |
| CPU    | 6 核                                                            |
|        |                                                                |

## 3. 总体流程

```
打包源码 → 传到手机 → 解压
→ 导出运行内核配置 (.config)
→ 版本对齐 (localversion + .scmversion)
→ make prepare (只编 scripts 工具，不编内核)
→ 组装/打包 deb
→ dpkg 安装 + 符号链接
→ DKMS 验证
```

## 4. 详细步骤与踩坑记录

### 4.0 编译机预装依赖（Kali chroot 内）

```bash
apt update
apt install -y build-essential gcc-12 bc bison flex \
  libncurses-dev libssl-dev libelf-dev device-tree-compiler \
  rsync dkms
```

> `dpkg-deb` 随 dpkg 预装，无需额外安装。

### 4.1 打包并传输源码

PC 上（WSL）：
```bash
cd ~/android/lineage/kernel/xiaomi/sm8250
tar --exclude=.git --exclude=out --exclude=Documentation -czf kernel-sm8250.tar.gz .
adb push kernel-sm8250.tar.gz /sdcard/
```

> 注意：tar 的根目录是 `sm8250/`，解压时多了一层目录。

chroot 内解压：
```bash
mkdir -p /usr/src/kernel-sm8250
tar xzf /root/sm8250_kernel.tar.gz -C /usr/src/kernel-sm8250 --strip-components=1
```

**坑 1：多套一层目录**
直接 `tar xzf` 得到 `/usr/src/kernel-sm8250/sm8250/`。加 `--strip-components=1` 解决。

### 4.2 首次 `make kernelrelease` 失败 —— 缺 `.config`

```bash
cd /usr/src/kernel-sm8250
make -s kernelrelease
```
报错：
```
Makefile:638: include/config/auto.conf: No such file or directory
make: *** No rule to make target 'kernelrelease'.  Stop.
```

**原因**：干净源码树没有 `.config`，任何 Linux 内核（包括官方 4.19）首次跑 `kernelrelease` 都这样，与"是不是安卓内核"无关。
**解决**：顺序反了 —— 先有 `.config`，再看版本。

### 4.3 `olddefconfig` 失败 —— `scripts/min-tool-version.sh` 缺失

```bash
zcat /proc/config.gz > .config          # 导出运行内核的精确配置
make ARCH=arm64 olddefconfig
```
报错：
```
./scripts/as-version.sh: 59: ./scripts/min-tool-version.sh: not found
scripts/Kconfig.include:56: Sorry, this assembler is not supported.
make[1]: *** [scripts/kconfig/Makefile:69: olddefconfig] Error 1
make: *** [Makefile:582: olddefconfig] Error 2
```

**原因**（关键）：`as-version.sh` 是 CAF 从 5.x 回迁的 kbuild 脚本，第 59 行调用 `min-tool-version.sh` 做 binutils 最低版本检查，**但该文件从未被 git 跟踪**（残缺回迁）。"assembler not supported" 是误导性报错，真正原因是脚本缺失。排查：
```bash
grep -E '^(VERSION|PATCHLEVEL|SUBLEVEL|EXTRAVERSION)' Makefile   # 4.19.325
git ls-files scripts/ | grep -E 'min-tool|as-version'            # 只有 as-version.sh，没有 min-tool-version.sh
git status | head -30                                            # 干净树
```

**解决**：补回 `scripts/min-tool-version.sh`（标准内容，见附录 A），接口是"传 `binutils` 参数、输出最低版本号"。手机 binutils 2.4x ≥ 门槛 2.23，检查直接通过。

### 4.4 版本对齐 —— `localversion` 文件 + `.scmversion`

运行内核：`4.19.325-cip128-st12-perf-g13b18ecca301`，拆解：

| 部分 | 来源 |
|---|---|
| `4.19.325` | Makefile |
| `-cip128` | 树根 `localversion-cip` 文件 |
| `-st12` | 树根 `localversion-st` 文件 |
| `-perf` | `CONFIG_LOCALVERSION`（在 `.config` 里） |
| `-g13b18ecca301` | `CONFIG_LOCALVERSION_AUTO=y` + git 哈希 |

手机树里的 `.git` 是**悬空符号链接**（指向 `/.repo/projects/...`，实际不存在）→ git 后缀无法自动生成。
**解决**：树根创建 `.scmversion`，内容为一行 `-g13b18ecca301`。内核 `scripts/setlocalversion` 发现该文件就直接原样输出、不再调 git。

```bash
# 之后版本可正确计算
make ARCH=arm64 olddefconfig
make -s kernelrelease
# → 4.19.325-cip131-st15-perf-g13b18ecca301
```

**坑 4：与 `uname -r` 不一致（cip131/st15 vs cip128/st12）**
手机现跑的是较早编译的内核，之后源码被更新过（`localversion-cip`/`localversion-st` 已变成 `-cip131`/`-st15`）。用户确认更新后编译测试通过，**选择忽略此差异**。
含义：这版 headers 匹配的是"更新后的内核"，不匹配现跑内核 → 以后 `insmod` 会 vermagic 不符；刷入新内核后才可用。

### 4.5 `make prepare` 交互式提问 —— `.config` 缺新符号

```bash
make ARCH=arm64 HOSTCC=gcc-12 CC=gcc-12 prepare modules_prepare scripts -j2
```
报错（表现为卡住、逐个提问）：
```
scripts/kconfig/conf --syncconfig Kconfig
...
Hidden DRM configs needed for GKI (GKI_HIDDEN_DRM_CONFIGS) [N/y/?] n
...
Memfd ashmem ioctl compatibility support (MEMFD_ASHMEM_SHIM) [N/y/?] (NEW)
```

**原因**：`.config` 来自**旧运行内核**，缺少更新后内核新增的 Kconfig 符号（`GKI_HIDDEN_*`、`MEMFD_ASHMEM_SHIM` 等），`syncconfig` 发现就逐个问。

**解决**：`Ctrl+C` 中断 → 用 `olddefconfig` 静默补默认值 → 验证 → 重跑 prepare：
```bash
make ARCH=arm64 olddefconfig
grep -E 'GKI_HIDDEN|MEMFD_ASHMEM' .config    # 看到 CONFIG_GKI_HIDDEN_*=... 即成功
make ARCH=arm64 HOSTCC=gcc-12 CC=gcc-12 prepare modules_prepare scripts -j2
```

prepare 成功标志（生成了所有头文件与脚本工具，**未编译内核**）：
```
UPD  include/config/kernel.release
UPD  include/generated/utsrelease.h
UPD  include/generated/asm-offsets.h
UPD  include/generated/bounds.h
...
HOSTLD scripts/mod/modpost
```

### 4.6 补 `Module.symvers`

```bash
ls -l Module.symvers   # No such file or directory
```
**原因**：`make modules_prepare` 不生成它；它由 `make modules` 的 modpost 流程创建。headers-only 场景标准做法：
```bash
touch Module.symvers
```
外部模块编译不依赖它有内容（内核内建符号始终可用），Debian 官方 headers 包同理。

### 4.7 组装并打包 deb

```bash
KREL=4.19.325-cip131-st15-perf-g13b18ecca301
PKG=/tmp/headers-pkg
rm -rf $PKG
mkdir -p $PKG/usr/src/linux-headers-$KREL $PKG/DEBIAN
rsync -a --exclude='.git' ./ $PKG/usr/src/linux-headers-$KREL/

cat > $PKG/DEBIAN/control <<EOF
Package: linux-headers-$KREL
Version: $KREL-1
Architecture: arm64
Maintainer: user <user@localhost>
Section: kernel
Priority: optional
Depends: libc6-dev
Description: Linux kernel headers for version $KREL
EOF

dpkg-deb --build $PKG linux-headers-${KREL}_arm64.deb
ls -lh linux-headers-${KREL}_arm64.deb   # 130M
```

> 130M 正常：包内含**整个源码树**（未压缩 1-2G）+ 生成头文件 + scripts 二进制，gzip 压缩后 130M。Debian 官方 arm64 headers 包也在这个量级。

### 4.8 安装 + 符号链接

```bash
dpkg -i linux-headers-${KREL}_arm64.deb
mkdir -p /lib/modules/$KREL
ln -sf /usr/src/linux-headers-$KREL /lib/modules/$KREL/build
ln -sf /usr/src/linux-headers-$KREL /lib/modules/$KREL/source
ls -l /lib/modules/$KREL/
```
`/lib/modules/<ver>/build` 是 DKMS 查找头文件的入口，必须手动建（headers 包 postinst 不做这件事）。

### 4.9 DKMS 验证（走通全流程）

hello 测试模块（`/usr/src/hello-1.0/`，内容见附录 C）：
```bash
dkms add hello/1.0
dkms build hello/1.0
dkms install hello/1.0
modinfo hello        # vermagic 应等于 $KREL
```
**预期**：`dkms build` 成功、vermagic 对得上 headers = 流程走通。`insmod`/`modprobe` 会因**现跑内核版本 ≠ headers 版本**（cip128/st12 vs cip131/st15）而失败，属预期。

## 5. 关键经验 / 备忘

1. **Android 内核就是标准 Linux 内核**，报错排查方法完全相同。
2. `make kernelrelease` 必须**先有 `.config`**。
3. 版本串 = 基础版本 + `localversion*` 文件 + `CONFIG_LOCALVERSION` + `CONFIG_LOCALVERSION_AUTO`(git)。没有 git 时用树根 `.scmversion` 文件替代 git 后缀。
4. 配置必须与目标内核一致：`/proc/config.gz` 是**运行内核的精确配置**（需 `CONFIG_IKCONFIG_PROC=y`）。
5. 原生编译避免 scripts 工具架构不匹配（modpost/dtc/genksyms 均为 arm64）。
6. 更新内核源码后，旧配置缺新 Kconfig 符号 → `make olddefconfig` 用默认值补齐。
7. headers 包里的 `Module.symvers` 可以为空。
8. 4.19 源码 + 现代 gcc：建议 `gcc-12`，避免新 gcc 编译 4.19 报错。
9. 手机内存小：编译用 `-j2`，防止 OOM。
10. headers 版本必须与**将要运行模块的内核**一致，否则编译能过、加载失败。

## 6. 后续（真正可用的两个方向）

- **让 headers 匹配现跑内核**：把 `localversion-cip`/`localversion-st` 改回 `-cip128`/`-st12`，重新打包（`.config` 已来自现跑内核）。
- **配合新内核使用**：刷入更新后的内核（用户已确认编译通过），这版 deb 可直接配合；注意确认新内核 git 哈希与 `.scmversion` 一致，不一致就同步更新 `.scmversion`。

## 7. 追加调查：为什么 `apt install wifipumpkin3` 要 `linux-headers`

**结论**：wifipumpkin3 本身不依赖 `linux-headers`，真正元凶是安装命令里另外带的 **`xtables-addons-common`**。

### 7.1 依赖链

```
apt install wifipumpkin3 dnschef xtables-addons-common
  → xtables-addons-common
    → 依赖配套的 xtables-addons-dkms（包描述原话："only useful with a corresponding xtables-addons-dkms package"）
      → DKMS 编译内核模块需要 linux-headers-$(uname -r)
        → 手机是自定义 4.19.325-cip128-st12-perf-g13b18ecca301
          → Kali 仓库只有 7.x headers，不匹配 → 卡住
```

### 7.2 机制

- `xtables-addons` 给 iptables 提供**内核态扩展模块**：TARPIT / CHAOS / TEE / geoip / IPMARK / DELUDE / condition 等（`xt_TARPIT.ko`、`xt_geoip.ko`…），不是纯用户态工具。
- 这些 .ko 必须和运行内核 vermagic 一致 → 必须用对应 headers 经 DKMS 现编译。
- `wifipumpkin3` 的 Kali control 文件（v1.1.7-0kali3）Depends 只有：`hostapd, iptables, iw, net-tools, wireless-tools, python3-*`——**没有 linux-headers**。

### 7.3 两条路（按 7.4 顺序先试路线 1）

**路线 1：不需要 xtables 扩展（推荐先试）**
wifipumpkin3 核心功能（假 AP + captive portal + dnschef）不依赖这些内核模块。去掉 xtables-addons 即可：
```bash
apt install wifipumpkin3 dnschef
```
> 仅当用到 TARPIT/geoip 等高级规则的功能时才缺模块。

**路线 2：要完整工具链 → headers 匹配现跑内核**
当前 cip131/st15 那版 deb 满足不了 apt（apt 认字面包名 `linux-headers-4.19.325-cip128-st12-perf-g13b18ecca301`）。把版本改回现跑内核重打：
```bash
cd /usr/src/kernel-sm8250
printf -- '-cip128\n' > localversion-cip
printf -- '-st12\n'   > localversion-st
make -s kernelrelease                                # 必须 = 4.19.325-cip128-st12-perf-g13b18ecca301
make ARCH=arm64 HOSTCC=gcc-12 CC=gcc-12 prepare modules_prepare scripts -j2   # 重生成 utsrelease.h
# 再按 4.7-4.8 组装 deb → dpkg -i → 建 build/source 链接
```
装好后 apt 认名字，`xtables-addons-dkms` 会自己编译。
> ⚠️ 风险：① xtables-addons 3.x 对 4.19 内核的 API 兼容性一般，编译可能报错；② DKMS 默认用 Kali 新 gcc(13/14) 编 4.19 模块也可能报错，可试 `CC=gcc-12`。

### 7.4 验证/决策顺序

1. `apt install wifipumpkin3 dnschef`（路线 1）→ 跑基本功能。
2. 若报缺 `xt_TARPIT` / `xt_geoip` 之类 → 走路线 2 重打 headers deb。
3. 需要时 `apt-get -s install wifipumpkin3 dnschef xtables-addons-common | grep -i headers` 查看真实解析结果。

## 附录 A — `scripts/min-tool-version.sh`

```sh
#!/bin/sh
# SPDX-License-Identifier: GPL-2.0
# Usage: $0 <tool>  -- print the minimum supported version of the tool
case "$1" in
binutils)
	[ "$ARCH" = arm64 ] && echo 2.23 || echo 2.20
	;;
gcc)
	echo 4.9
	;;
clang)
	echo 10.0.1
	;;
make)
	echo 3.81
	;;
*)
	echo "Error: unknown tool '$1'" >&2
	exit 1
	;;
esac
```

## 附录 B — `.scmversion`（树根，一行）

```
-g13b18ecca301
```

## 附录 C — DKMS hello 测试模块

`/usr/src/hello-1.0/hello.c`：
```c
#include <linux/module.h>
#include <linux/kernel.h>

static int __init hello_init(void)
{
	printk(KERN_INFO "hello module loaded\n");
	return 0;
}

static void __exit hello_exit(void)
{
	printk(KERN_INFO "hello module unloaded\n");
}

module_init(hello_init);
module_exit(hello_exit);
MODULE_LICENSE("GPL");
```

`/usr/src/hello-1.0/dkms.conf`：
```
PACKAGE_NAME="hello"
PACKAGE_VERSION="1.0"
MAKE[0]="make -C ${kernel_source_dir} M=${dkms_tree}/${PACKAGE_NAME}/${PACKAGE_VERSION}/build"
CLEAN="make -C ${kernel_source_dir} M=${dkms_tree}/${PACKAGE_NAME}/${PACKAGE_VERSION}/build clean"
BUILT_MODULE_NAME[0]="hello"
DEST_MODULE_LOCATION[0]="/updates"
AUTOINSTALL="yes"
```



