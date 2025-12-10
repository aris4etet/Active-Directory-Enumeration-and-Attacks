**Miscellaneous Misconfigurations – Cheat Sheet**

High-level idea: these are “random but deadly” AD issues that often slip past people. Know what they are, how to find them, and what they give you.

---

### 1. Exchange-Related Misconfigurations

**Key Groups**

* **Exchange Windows Permissions**

  * Not a protected group, but:
  * Members can **write DACL on the domain object** → can give themselves **DCSync** / domain admin-level rights.
  * Often contains **power users, support staff, computers**.

* **Organization Management**

  * “Domain Admins of Exchange.”
  * Can access **all mailboxes**.
  * Has full control of **Microsoft Exchange Security Groups OU**, which contains **Exchange Windows Permissions**.

**Takeaway:**
Compromising Exchange (server or powerful Exchange group) often → **Domain Admin**, plus tons of creds in memory (OWA logins cached).

---

### 2. PrivExchange

* **Bug:** Exchange PushSubscription lets any domain user with a mailbox force Exchange to **authenticate to an arbitrary host** over HTTP.
* Exchange service runs as **SYSTEM** and is over-privileged (WriteDacl on domain pre-2019 CU).
* **Attack:**

  * Relay that auth to **LDAP** → write DACL → DCSync → **Domain Admin**.
  * If not LDAP, can still relay to other internal services.
* **Requirement:** Any **authenticated domain user** with a mailbox.
* **Impact:** One user → **domain compromise**.

---

### 3. Printer Bug (MS-RPRN)

* Abuse **Print System Remote Protocol** (MS-RPRN).
* Any domain user can:

  * Connect to **spooler** over named pipe.
  * Use `RpcRemoteFindFirstPrinterChangeNotificationEx` to **force server to auth to attacker SMB**.
* Spooler runs as **SYSTEM** → we relay that auth:

  * To **LDAP** → grant DCSync to our account.
  * Or grant **RBCD** (Resource-Based Constrained Delegation) to our controlled computer → impersonate any user on victim host.
* Used for:

  * **Cross-forest attacks** if remote DC has unconstrained delegation.
* **Tooling:**

  * `Get-SpoolStatus` (PowerShell modules) to find spooler-enabled machines.

---

### 4. MS14-068 (Old but Important)

* Kerberos PAC signing flaw.
* Allows forging a PAC so a normal user appears as **Domain Admin**.
* Exploit with:

  * **PyKEK**, **Impacket**, etc.
* **Mitigation:** Patch (MS14-068). Only defense.

---

### 5. Sniffing LDAP Credentials

* Many apps/printers store LDAP bind creds in their web UIs.
* Weak/default web admin creds → you can:

  * See LDAP creds in cleartext **or**
  * Change LDAP server IP to your host and listen on **389/tcp** (netcat / fake LDAP).
  * “Test connection” sends bind creds to you.
* These LDAP accounts are often **privileged** or at least good footholds.

---

### 6. Enumerating DNS Records (adidnsdump)

* AD DNS zones: by default, any authenticated user can list **child objects**.
* **adidnsdump**:

  * Uses LDAP to dump **all DNS records** into `records.csv`.
  * `-r` flag tries to resolve unknown hostnames into IPs.
* Use when hostnames are random (e.g. `SRV01934`) to find more meaningful names like `JENKINS.INLANEFREIGHT.LOCAL`.

---

### 7. Other Misconfigs

#### 7.1 Passwords in Description / Notes

* Admins sometimes put **passwords in `description`** fields.
* Use PowerView:

  ```powershell
  Get-DomainUser * | Select samaccountname,description | Where-Object {$_.Description -ne $null}
  ```
* Export to CSV and search offline.

#### 7.2 `PASSWD_NOTREQD` Flag

* `userAccountControl` flag where **password doesn’t need to meet policy**, and may be:

  * Very short, or even **blank** (if domain allows null passwords).
* Enumerate:

  ```powershell
  Get-DomainUser -UACFilter PASSWD_NOTREQD | Select samaccountname,useraccountcontrol
  ```
* Then test for empty/weak passwords (spray).

---

### 8. Credentials in SYSVOL & GPP

#### 8.1 Plaintext in SYSVOL Scripts

* `\\DC\SYSVOL\DOMAIN\scripts` is readable by **all domain users**.
* Look for `.vbs`, `.bat`, `.ps1` with hard-coded passwords (e.g. reset local admin password script).

#### 8.2 Group Policy Preferences (GPP) Passwords

* Old GPP XML files (`Groups.xml`, `Services.xml`, `ScheduledTasks.xml`, etc.) can have:

  * `cpassword` attribute (AES-256 encrypted).
* Microsoft published the **AES key** → anyone can decrypt:

  ```bash
  gpp-decrypt <cpassword>
  ```
* Tools:

  * `Get-GPPPassword.ps1`, Metasploit modules, various Python/Ruby scripts.
  * CrackMapExec modules:

    * `gpp_password`
    * `gpp_autologin` (for `Registry.xml` autologon creds).

**Key point:**
Even though MS “patched” GPP password setting, **old XML files remain** and are still exploitable.

---

### 9. ASREPRoasting

* Targets accounts with **“Do not require Kerberos pre-authentication”** (UAC flag `DONT_REQ_PREAUTH`).
* No SPN required, no domain join needed:

  * Request AS-REP from DC.
  * Get encrypted blob → **offline crack** (Hashcat mode 18200).

**Enumeration (Windows):**

```powershell
Get-DomainUser -PreauthNotRequired | select samaccountname,userprincipalname,useraccountcontrol
```

**Collect hash (Rubeus):**

```powershell
Rubeus.exe asreproast /user:<user> /nowrap /format:hashcat
```

**Linux tools:**

* `kerbrute userenum` will auto-dump AS-REP for users without pre-auth.
* `GetNPUsers.py` (Impacket) with `-no-pass` and a user list will:

  * Identify accounts without pre-auth.
  * Print AS-REP in Hashcat format.

**Mitigation:**
Don’t use “pre-auth disabled” unless truly necessary; monitor & lock down these accounts.

---

### 10. Group Policy Object (GPO) Abuse

**Idea:** If you have **write-level rights to a GPO**, you can push arbitrary config to all computers/users that GPO targets.

**What you can do with a compromised GPO:**

* Add a user you control to **local Administrators** on targeted hosts.
* Grant extra privileges (e.g. `SeDebugPrivilege`, `SeTakeOwnershipPrivilege`).
* Create an **immediate scheduled task** that runs a payload.
* Configure a **startup/logon script** that runs your reverse shell.

**Enumeration:**

* List GPOs (PowerView):

  ```powershell
  Get-DomainGPO | select displayname
  ```

* Same via built-in cmdlet:

  ```powershell
  Get-GPO -All | Select DisplayName
  ```

* Check if **Domain Users** or a specific user/group has dangerous rights on a GPO:

  ```powershell
  $sid = Convert-NameToSid "Domain Users"
  Get-DomainGPO | Get-ObjectAcl | ? { $_.SecurityIdentifier -eq $sid }
  ```

* Map GUID → Name:

  ```powershell
  Get-GPO -Guid <GUID>
  ```

* In BloodHound:

  * Look for edges like **GenericWrite / WriteDacl / WriteOwner** from your principal → GPO.
  * Check **Affected Objects** to see which OUs/computers are impacted.

**Exploitation:**

* Tools like **SharpGPOAbuse** can:

  * Inject a scheduled task.
  * Add local admins.
  * Add startup scripts, etc.
* Be careful: a GPO may target **hundreds/thousands** of machines. Aim at a specific host/OU where possible.

---

### 11. Big Picture / What to Research Next

Misconfigs to know deeply:

* **AD CS attacks** (especially with NTLM relay & templates).
* **Kerberos Constrained Delegation**.
* **Unconstrained Delegation**.
* **Resource-Based Constrained Delegation (RBCD)**.
* More on **GPO abuse** and **cross-forest trust abuse**.


# Here is the **fastest and cleanest way** to find users with the **PASSWD_NOTREQD** flag set, especially when you know the username starts with **"y"**.

---

# ✅ **PowerView Command (Best Method)**

Assuming you have PowerView imported:

```powershell
Get-DomainUser -LDAPFilter "(samAccountName=y*)" | ? { $_.userAccountControl -band 0x20 } | select samAccountName
```

Explanation:

* `samAccountName=y*` → only users starting with **y**
* `0x20` → PASSWD_NOTREQD flag
* Filters down to the correct account

---

# ✅ **Alternative (Full Search, Then Filter by Hand)**

```powershell
Get-DomainUser | ? { $_.userAccountControl -band 0x20 -and $_.samAccountName -like "y*" } | select samAccountName
```

