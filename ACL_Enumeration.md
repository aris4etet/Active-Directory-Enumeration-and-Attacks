# 🔥 ACL Enumeration — Ultra-Condensed Cheatsheet (PowerView & BloodHound)

## 🎯 Goal

Find **Access Control Entries (ACEs)** showing where our compromised user can:

* Reset passwords
* Write to groups
* Modify users
* Gain transitive privileges
* Eventually reach **DCSync** level access.

---

# 1️⃣ PowerView Basics

### Import & get SID

```powershell
Import-Module .\PowerView.ps1
$sid = Convert-NameToSid wley
```

---

# 2️⃣ Targeted ACL Search (fast method)

### Find all objects where the user has rights

```powershell
Get-DomainObjectACL -ResolveGUIDs -Identity * |
    ? { $_.SecurityIdentifier -eq $sid }
```

**ResolveGUIDs = HUMAN-READABLE rights**
(e.g., `User-Force-Change-Password` instead of GUIDs)

---

# 3️⃣ Example Attack Chain (from the text)

### Step 1 — wley → ForceChangePassword on damundsen

Right found:

```
User-Force-Change-Password
```

Reset password without knowing the old one.

---

### Step 2 — damundsen → GenericWrite on "Help Desk Level 1"

Means:

* Can **add users to this group**
* Gains whatever rights this group inherits.

```powershell
$sid2 = Convert-NameToSid damundsen
Get-DomainObjectACL -ResolveGUIDs -Identity * |
    ? {$_.SecurityIdentifier -eq $sid2}
```

---

### Step 3 — Help Desk Level 1 → nested into Information Technology

Check membership:

```powershell
Get-DomainGroup "Help Desk Level 1" | select memberof
```

---

### Step 4 — Information Technology → GenericAll over user adunn

Meaning:

* Full control over adunn
  (password reset, group membership, Kerberoast, etc.)

```powershell
$it = Convert-NameToSid "Information Technology"
Get-DomainObjectACL -ResolveGUIDs -Identity * |
    ? { $_.SecurityIdentifier -eq $it }
```

---

### Step 5 — adunn → Has DCSync rights

Look for:

* `DS-Replication-Get-Changes`
* `DS-Replication-Get-Changes-In-Filtered-Set`

```powershell
$adunn = Convert-NameToSid adunn
Get-DomainObjectACL -ResolveGUIDs -Identity * |
    ? { $_.SecurityIdentifier -eq $adunn }
```

**This = DCSync capability → dump entire domain password database.**

---

# 4️⃣ If ResolveGUIDs is blocked: Reverse-lookup GUIDs

```powershell
$guid = "00299570-246d-11d0-a768-00aa006e0529"
Get-ADObject -SearchBase "CN=Extended-Rights,$((Get-ADRootDSE).ConfigurationNamingContext)" `
    -Filter {ObjectClass -like 'ControlAccessRight'} -Properties * |
    ? {$_.rightsGuid -eq $guid} |
    select Name,DisplayName
```

---

# 5️⃣ No PowerView? Native tools fallback

### Dump all AD users to a file

```powershell
Get-ADUser -Filter * | select -Expand SamAccountName > users.txt
```

### Enumerate ACL per-object

```powershell
foreach($u in Get-Content users.txt) {
    Get-Acl "AD:\$(Get-ADUser $u)" |
        select Path -Expand Access |
        ? { $_.IdentityReference -match "INLANEFREIGHT\\wley" }
}
```

Slow but works even in locked-down environments.

---

# 6️⃣ BloodHound — Instant Attack Path Visualization

### What BloodHound instantly shows:

* **Outbound Control Rights** for user → (like ForceChangePassword)
* **Transitive Object Control** (complete privilege chain)
* Built-in queries:

  * *“Users with DCSync Rights”*
  * *“Shortest Path to Domain Admin”*

### In the UI:

* Select wley → Node Info → Outbound Control Rights
* Path:
  `wley → damundsen → Help Desk L1 → Information Tech → adunn → DCSync`

This automatically identifies the ENTIRE attack chain we manually enumerated.

---

# 🚀 FINAL SUMMARY (the entire section in 8 lines)

1. Use PowerView + ResolveGUIDs to find ACEs your user has.
2. wley has **ForceChangePassword** over damundsen.
3. damundsen has **GenericWrite** over Help Desk L1 group.
4. Help Desk L1 is nested into **Information Technology**.
5. IT group has **GenericAll** over user adunn.
6. adunn has **DCSync rights** → full domain compromise.
7. BloodHound visualizes all this instantly.
8. If no tools allowed → use Get-Acl + Get-ADUser fallback.

---

