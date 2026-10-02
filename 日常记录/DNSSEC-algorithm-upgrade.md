# DNSSEC 算法升级全程记录

## 1. 背景与目标

- ICANN 新规：注册局停止接受使用不安全算法的 DNSSEC 记录（废除算法 1/5/7/3/6/12 与摘要 1/3）
- 域名 `zzzxxx.xyz`（Spaceship 注册）原 DS 使用 **algorithm 7（RSASHA1-NSEC3-SHA1）** → 不满足新规
- 目标：将 KSK 升级为 **algorithm 8（RSASHA256）**，摘要改用 **SHA-256（digest 2）**
- 环境：自建 BIND 9.10.3-P4（Debian），zone 由 cron `signer.sh` 每 3 天自动重签（NSEC3 + 随机 salt）

## 2. 环境信息

| 项 | 值 |
|----|----|
| 域名 | zzzxxx.xyz（.xyz TLD） |
| 注册商 | Spaceship |
| DNS 服务器 | 自建 BIND 9.10.3-P4-Debian |
| 签名方式 | cron → `/root/signer.sh zzzxxx.xyz db.zzzxxx.xyz`（每 3 天） |
| 原密钥 | `Kzzzxxx.xyz.+007+00156`（KSK）/ `+007+62759`（ZSK） |

## 3. 操作步骤（已完成 ✓）

### 3.1 生成算法 8 密钥对

```bash
cd /etc/bind && cp -r /etc/bind /etc/bind.bak   # 备份
dnssec-keygen -a RSASHA256 -b 2048 -n ZONE zzzxxx.xyz                 # ZSK
dnssec-keygen -a RSASHA256 -b 2048 -n ZONE -f KSK zzzxxx.xyz          # KSK
```

产出：
- `Kzzzxxx.xyz.+008+20940.key`（ZSK，flag 256）
- `Kzzzxxx.xyz.+008+60124.key`（KSK，flag 257）

### 3.1b 如何确认哪个是 KSK、哪个是 ZSK

光看 keyid（文件名中的编号）无法区分，需查看 key 文件内容：

- **首行注释**：含 `This is a key-signing key` → KSK；含 `This is a zone-signing key` → ZSK
- **DNSKEY 记录的 flag**：`257` = KSK（SEP 位置位）；`256` = ZSK
- **生成命令**：带 `-f KSK` 参数生成的那个就是 KSK

批量查看命令（实际输出即本任务中的两个 key）：

```bash
for k in Kzzzxxx.xyz.+008+*.key; do
  echo "== $k =="
  head -1 "$k"
  grep DNSKEY "$k"
done
```

```
== Kzzzxxx.xyz.+008+20940.key ==
; This is a zone-signing key, keyid 20940, for zzzxxx.xyz.
zzzxxx.xyz. IN DNSKEY 256 3 8 AwEA...          ← 256 = ZSK
== Kzzzxxx.xyz.+008+60124.key ==
; This is a key-signing key, keyid 60124, for zzzxxx.xyz.
zzzxxx.xyz. IN DNSKEY 257 3 8 AwEA...          ← 257 = KSK
```

### 3.2 zone 加入新密钥（与旧密钥共存）

`db.zzzxxx.xyz` 末尾追加（**不删除**旧 `+007` 两行）：

```
$INCLUDE Kzzzxxx.xyz.+008+20940.key
$INCLUDE Kzzzxxx.xyz.+008+60124.key
```

### 3.3 重新签名 + reload

```bash
cd /etc/bind
SERIAL=$(named-checkzone zzzxxx.xyz db.zzzxxx.xyz | egrep -ho '[0-9]{10}')
sed -i "s/$SERIAL/$((SERIAL+1))/" db.zzzxxx.xyz
dnssec-signzone -A -3 $(head -c 1000 /dev/random | sha1sum | cut -b 1-16) -N increment -o zzzxxx.xyz -t db.zzzxxx.xyz
/etc/init.d/bind9 reload
```

### 3.4 验证新 DNSKEY 已公开可查

```bash
dig @8.8.8.8 zzzxxx.xyz DNSKEY +short   # 应见 256/257 3 8 的新 key
dnssec-verify -o zzzxxx.xyz /etc/bind/db.zzzxxx.xyz.signed
```

结果：`NSEC3RSASHA1 + RSASHA256` 双算法，各 1 KSK + 1 ZSK 全部 active。

### 3.5 生成新 DS 记录

```bash
dnssec-dsfromkey -2 Kzzzxxx.xyz.+008+60124.key
```

输出：`zzzxxx.xyz. IN DS 60124 8 2 983F11ECB73BE2400DEEB58F9A4E4EA669A86E873CCD0AAC410B130C33D16833`

### 3.6 提交 DS 到 Spaceship（遇阻与解决）

- **首次尝试**：在「旧 DS 存在」前提下「替换/新增」→ 报错 `DnsSec record was not saved on registry due to business error`
- **排查**：zone/DNSKEY/DS 值经多解析器交叉验证全部正确 → 问题在 registry EPP 层
- **解决方案**：在 Spaceship **先删除旧 DS**（`156 7 2`）→ 立即**新增新 DS**（`60124 8 2`）→ 成功
- **关键结论**：该注册局拒绝「旧 DS 存在时」的 DS 集变更事务，必须分两步（先删后加）

### 3.7 确认 registry 生效

```bash
dig zzzxxx.xyz DS +trace @1.1.1.1
```

权威应答 `z.nic.xyz`：`zzzxxx.xyz. 3600 IN DS 60124 8 2 983F11...`，RRSIG 签名时间 = 当天。
注意：传播有延迟（约 1 小时）；期间多解析器仍返回旧 DS 属正常缓存现象，须以 `+trace` 权威结果为准。

### 3.8 全链路验证 ✓

```bash
delv zzzxxx.xyz A                  # "fully validated" + RRSIG 双签名
dig @8.8.8.8 zzzxxx.xyz A +dnssec  # flags 含 "ad"
```

## 4. 遗留清理任务（静默期后执行）

静默期：24–48h，确保全球解析器旧 DS 缓存（`156 7 2`）失效

```bash
sed -i '/+007/d' db.zzzxxx.xyz        # 删除旧 $INCLUDE 两行
signer.sh zzzxxx.xyz db.zzzxxx.xyz    # 重签（自动只留 +008）
delv zzzxxx.xyz A                     # 必须仍 "fully validated"
rm Kzzzxxx.xyz.+007*                  # 删旧密钥文件（备份保留在 /etc/bind.bak）
```

终验：

```bash
dig @8.8.8.8 zzzxxx.xyz DS    # 只应出现 60124 8 2
delv zzzxxx.xyz A             # fully validated
```

## 5. 经验教训

1. **「business error」不代表配置错**：先在 `+trace`/多解析器证明数据正确，再排注册局事务问题
2. **注册局侧新旧 DS 不可共存**时，用**先删后加**两阶段切换（需接受短暂「无 DNSSEC」窗口）
3. **延迟判断信号**：RRSIG 签名时间戳（8月23旧 vs 8月27新），比解析器值更可靠
4. **DNSKEY 先发布、DS 后提交**的顺序不可逆，否则 registry 实时预检会拒绝
5. **双密钥共存的过渡期设计**天然免疫旧缓存——旧 DS 还有效期间，zone 里旧 KSK 仍在，零故障切换

## 6. 长期运维注意

- cron `signer.sh` 无需改动，自动发现 `/etc/bind/` 目录内 `+008` 密钥
- 可选优化：BIND 9.10.3 已 EOL（2015），建议未来升级至 9.18+，但非本次必需