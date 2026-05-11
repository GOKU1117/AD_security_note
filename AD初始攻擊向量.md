## 1. LLMNR Poisoning

**什麼是 LLMNR？**
- DNS 解析失敗時的備援協定，會對區網廣播詢問主機名稱
- 類似協定：NBT-NS、mDNS

**攻擊流程：**
1. 受害者 DNS 解析失敗 → 發出 LLMNR 廣播
2. 攻擊者（Responder）偽裝回應：「我就是！」
3. 受害者嘗試連線 → 送出 NTLM 認證
4. 攻擊者捕捉 **Net-NTLMv2 Hash**

**工具：Responder**
```bash
sudo responder -I eth0 -dwv
```

**破解 Hash：**
```bash
hashcat -m 5600 hashes.txt rockyou.txt
```

**防禦：**
- 停用 LLMNR / NBT-NS（Group Policy）
- 啟用網路存取控制（NAC）
- 強制 SMB 簽章

---

## 2. SMB Relay Attacks

**概念：**
不破解 hash，直接將捕捉到的 NTLM hash **轉發**到其他機器進行認證。

**前提條件：**
- 目標未啟用 SMB 簽章
- 轉發的帳號在目標機器上有管理員權限

**防禦：**
- 強制啟用 SMB 簽章
- 停用 NTLM 認證
- 啟用 Local Admin Restriction

---

## 3. Gaining Shell Access

取得憑證後，可用以下工具取得 Shell：

| 工具 | 說明 |
|------|------|
| `psexec.py` | 最常用，返回 SYSTEM shell |
| `wmiexec.py` | 透過 WMI 執行指令 |
| `smbexec.py` | 透過 SMB 執行指令 |

```bash
psexec.py CORP/Administrator:Password123@192.168.1.100
```

---

## 4. IPv6 Attacks（mitm6）

**原理：**
Windows 預設優先使用 IPv6，攻擊者偽裝成 IPv6 DNS 伺服器，進行中間人攻擊，可接管 AD 認證（LDAP / LDAPS）。

**工具：** `mitm6`

**防禦：**
- 若不使用 IPv6，封鎖 DHCPv6 流量
- 啟用 LDAP 簽章與 LDAPS

---

## 5. Passback Attacks

**目標：** 針對印表機、掃描器等網路設備

**原理：**
這類設備常設定 LDAP / SMTP 認證，可修改設備設定，將認證請求導向攻擊者控制的伺服器，捕捉明文憑證。

---

## 6. 初始內部攻擊策略

```
進入內網後優先步驟：
1. 執行 Responder（被動監聽）
2. 執行 mitm6（IPv6 毒化）
3. 掃描網路（nmap）
4. 尋找明顯漏洞（舊系統、預設密碼）
5. 若取得憑證 → 橫向移動
```

---

## Hashcat 速查

| 模式 | 類型 | 使用場景 |
|------|------|---------|
| `-m 5600` | Net-NTLMv2 | Responder、LNK 攻擊 |
| `-m 1000` | NTLM | SAM dump |