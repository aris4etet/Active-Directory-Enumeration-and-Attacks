## Credentialed Enumeration from Windows – Quick Runbook

### 0️⃣ Setup

* You’re on a **domain-joined** (or domain-reachable) Windows host.
* You have **domain creds** (e.g. `forend / Klmcargo2`).
* Tools you might use:

  * Built-in **AD PowerShell** module
  * **PowerView** / **SharpView**
  * **Snaffler**
  * **SharpHound** + BloodHound GUI

---

## 1️⃣ ActiveDirectory PowerShell Module

### Load module

```powershell
Import-Module ActiveDirectory
```

### Domain info (SID, trusts, child domains)

```powershell
Get-ADDomain
```

### Kerberoastable users (SPN set)

```powershell
Get-ADUser -Filter {ServicePrincipalName -ne "$null"} -Properties ServicePrincipalName
```

### Domain trusts

```powershell
Get-ADTrust -Filter *
```

### Groups + members

```powershell
Get-ADGroup -Filter * | Select-Object Name
Get-ADGroup -Identity "Backup Operators"
Get-ADGroupMember -Identity "Backup Operators"
```

---

## 2️⃣ PowerView – AD Recon on Steroids

(Assume PowerView loaded in PS already.)

### User info (targeted)

```powershell
Get-DomainUser -Identity mmorgan -Domain inlanefreight.local |
  Select-Object name,samaccountname,memberof,pwdlastset,lastlogontimestamp,admincount
```

### Recursive Domain Admins membership

```powershell
Get-DomainGroupMember -Identity "Domain Admins" -Recurse
```

### Trust mapping

```powershell
Get-DomainTrustMapping
```

### Check local admin on a host

```powershell
Test-AdminAccess -ComputerName ACADEMY-EA-MS01
```

### Users with SPNs (Kerberoast targets)

```powershell
Get-DomainUser -SPN -Properties samaccountname,ServicePrincipalName
```

---

## 3️⃣ SharpView – .NET Port (when PS is hot)

```powershell
.\SharpView.exe Get-DomainUser -Identity forend
```

* Use similar functions/flags as PowerView (`Get-DomainUser`, `Get-DomainGroupMember`, etc.).
* Good when you want **no PowerShell script** footprint.

---

## 4️⃣ Snaffler – Auto Loot Shares for Secrets

From domain-joined context:

```powershell
.\Snaffler.exe -d INLANEFREIGHT.LOCAL -s -v data -o snaffler.log
```

* `-s` → print to screen
* `-v data` → only interesting hits
* Look for **Red/interesting** file types: `.key`, `.ppk`, `.kdb`, `.mdf`, `.sqldump`, `.keychain`, etc.
* Use log for reporting + follow-up manual triage.

---

## 5️⃣ SharpHound (BloodHound Collector) from Windows

Run from the Windows attack host:

```powershell
.\SharpHound.exe -c All --zipfilename ILFREIGHT
```

* Generates a **ZIP** with JSON data (users, groups, computers, ACLs, trusts, sessions, local admins, etc.).

Then:

1. Launch BloodHound GUI (`bloodhound`).
2. Log in to Neo4j (e.g. `neo4j / HTB_@cademy_stdnt!` if pre-set).
3. **Upload Data** → select `ILFREIGHT.zip`.

### Useful built-in queries (Analysis tab)

* **Find Shortest Paths to Domain Admins**
* **Find Computers with Unsupported Operating Systems**
* **Find Computers where Domain Users are Local Admin**

Use these to:

* Plan **lateral movement / escalation paths**.
* Identify legacy/unsupported hosts.
* Spot bad local admin practices.

---


