# wp3 (wifipumpkin3) 卡死修复记录 — 旧Android内核上scapy的IPv6路由读取

> 场景：Kali NetHunter chroot（Android），wp3 = wifipumpkin3 1.1.7，
> 入口 `/usr/bin/wp3`，scapy 2.7.0（apt `python3-scapy 2.7.0+dfsg1-1`），Python 3.14。
> 日期：2026-08-27。状态：已修复并验证。

## 一、现象

```bash
wp3 -i wlan0       # 在加载时卡死，无响应
```

Ctrl+C 后 traceback 显示卡在：

```
scapy/route6.py:333         conf.route6 = Route6()
  → scapy/route6.py:64      self.routes = read_routes6()
    → scapy/arch/linux/rtnetlink.py:945  results = _read_routes(socket.AF_INET6)
      → rtnetlink.py:869    _sr1_rtrequest(...)
        → rtnetlink.py:734  msgs = rtmsghdrs(sock.recv(65535))
          → packet.py       do_dissect_payload 无限递归（RecursionError/卡死）
```

## 二、根因

scapy 2.6+ 用 **rtnetlink（netlink套接字）** 读取内核路由表（重写 PR #4352）。
在这颗**旧 Android 内核**上，IPv6 路由 dump 在 scapy 的 `rtmsghdrs(sock.recv(65535))`
解析阶段陷入 `do_dissect_payload` 无限递归 → 导入卡死。

- **IPv4 读取正常**（同样的 `_read_routes(socket.AF_INET)` 不卡），只有 IPv6 路径崩。
- 上游旧内核修复 PR #4482（两条 setsockopt 加 try/except）与
  IPv6 解析 workaround #4560 已内置于 2.7.0，但**仍不能**覆盖本内核。
- 临时方案（早期尝试，用户拒绝）：`echo 1 > /proc/sys/net/ipv6/conf/all/disable_ipv6`
  —— 会关掉整个系统 IPv6，且 chroot 重启即失效。

## 三、最终方案（彻底修复）

**不关 IPv6、不动系统/内核配置**；把 scapy 的 `read_routes6()` 函数体替换为
解析 `/proc/net/ipv6_route` 的纯文本实现（复刻 scapy 2.5 官方代码）。
IPv4 路径零改动（`/proc/net/route` 系同样逻辑本就用得好）。

原理：scapy ≤2.5 一直是直接 split `/proc/net/ipv6_route` 文本，永不递归，多年稳定。
2.6+ 改用 netlink 后才引入此 bug。

### 效果（已验证）

```bash
cat /proc/sys/net/ipv6/conf/all/disable_ipv6       # 0（IPv6 开启）
python3 -c "from scapy.all import *; print(len(conf.route6.routes))"   # 31（真实路由！）
timeout 15 python3 -c "from scapy.all import *"; echo "exit=$?"        # 0（不卡）
wp3 -i wlan0                                        # 进入 wp3 > 界面，正常
```

`conf.route6.routes` 真实包含 `fe80::/64 → docker0` 等条目，scapy 的 IPv6
发包路由选择功能也恢复可用。

## 四、依赖检查（改前确认，已通过）

```bash
python3 -c "import scapy.arch.linux.rtnetlink as m; print(m.conf.loopback_name, m.struct)"
python3 -c "import scapy.utils6 as u; print(hasattr(u,'in6_ptop'), hasattr(u,'construct_source_candidate_set'))"
# lo <module 'struct' ...>
# True True
```

## 五、安装/重打步骤

> 注意：`apt` 重装/升级 `python3-scapy` 会覆盖 `rtnetlink.py`，需按本步骤重打。

**① 备份（只做一次）**
```bash
cp /usr/lib/python3/dist-packages/scapy/arch/linux/rtnetlink.py /root/rtnetlink.py.bak
```

**② 替换 `read_routes6()`（ast 精确替换单个函数，粘贴执行）**
```bash
python3 - <<'EOF'
import ast
p = "/usr/lib/python3/dist-packages/scapy/arch/linux/rtnetlink.py"
src = open(p, encoding="utf-8").read()
tree = ast.parse(src)
node = next(n for n in ast.walk(tree)
            if isinstance(n, ast.FunctionDef) and n.name == "read_routes6")
new_func = '''def read_routes6():
    # type: () -> List[Tuple[str, int, str, str, List[str], int]]
    """
    Read IPv6 routes for current process (from /proc/net/ipv6_route)
    """
    RTF_UP = 0x0001
    RTF_REJECT = 0x0200
    routes = []
    try:
        f = open("/proc/net/ipv6_route", "rb")
    except IOError:
        return routes
    lifaddr = []
    try:
        fdesc = open("/proc/net/if_inet6", "rb")
        for line in fdesc:
            tmp = line.decode().split()
            addr = scapy.utils6.in6_ptop(
                b":".join(struct.unpack("4s4s4s4s4s4s4s4s", tmp[0].encode())).decode()
            )
            lifaddr.append((addr, int(tmp[3], 16), tmp[5]))
        fdesc.close()
    except (IOError, ValueError):
        pass
    def proc2r(p):
        ret = struct.unpack("4s4s4s4s4s4s4s4s", p)
        return scapy.utils6.in6_ptop(b":".join(ret).decode())
    for line in f.readlines():
        try:
            d_b, dp_b, _, _, nh_b, metric_b, rc, us, fl_b, dev_b = line.split()
        except ValueError:
            continue
        metric = int(metric_b, 16)
        fl = int(fl_b, 16)
        dev = dev_b.decode()
        if fl & RTF_UP == 0:
            continue
        if fl & RTF_REJECT:
            continue
        d = proc2r(d_b)
        dp = int(dp_b, 16)
        nh = proc2r(nh_b)
        cset = []
        if dev == conf.loopback_name:
            if d == "::":
                continue
            cset = ["::1"]
        else:
            devaddrs = (x for x in lifaddr if x[2] == dev)
            cset = scapy.utils6.construct_source_candidate_set(d, dp, devaddrs)
        if len(cset) != 0:
            routes.append((d, dp, nh, dev, cset, metric))
    f.close()
    return routes
'''
lines = src.splitlines(keepends=True)
lines[node.lineno - 1:node.end_lineno] = [new_func]
open(p, "w", encoding="utf-8", newline="").write("".join(lines))
print("patched OK")
EOF
```

**③ 恢复 wp3 官方自动加载**（本机已处于该状态；若曾用过 hack 才需要）
```bash
cp /root/wp3.bak /usr/bin/wp3
```

**④ 验证**
```bash
cat /proc/sys/net/ipv6/conf/all/disable_ipv6          # 0
python3 -c "from scapy.all import *; print(len(conf.route6.routes))"   # 非0
timeout 15 python3 -c "from scapy.all import *"; echo "exit=$?"        # 0
wp3 -i wlan0                                          # 进入界面
```

**⑤ 回滚**
```bash
cp /root/rtnetlink.py.bak /usr/lib/python3/dist-packages/scapy/arch/linux/rtnetlink.py
cp /root/wp3.bak /usr/bin/wp3
```

## 六、人员/环境备注

- wp3 入口 `/usr/bin/wp3` 是 easy-install console_scripts 入口，本身不 import scapy，
  scapy 由 wifipumpkin3 包内部导入。此前尝试的 hack：
  在 wp3 顶部插入 `from scapy.config import conf; conf.route6_autoload = False`
  （官方开关 PR #4253，禁IPv6路由自动加载）——有效但"等效于对 scapy 关 IPv6"，
  `conf.route6.routes` 为空。后已弃用，被本彻底方案取代。
- 备份文件：`/root/rtnetlink.py.bak`、`/root/wp3.bak`
- 上游相关：PR #4253（route6_autoload 开关）、#4352（rtnetlink 重写）、
  #4482（旧内核 setsockopt 保护）、#4560（IPv6 解析 workaround，修 #4541）