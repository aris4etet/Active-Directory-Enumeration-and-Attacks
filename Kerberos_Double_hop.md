## 🧠 What Is the Kerberos Double-Hop Problem?

**TL;DR:**
When you WinRM/PSRemoting into a box with Kerberos, your session gets a ticket to *that* machine, but not a reusable password/hash/TGT. So when you try to go **from that box to another resource** (DC, file share, etc.), your auth *can’t* be forwarded → AD queries / network access fail even though creds are valid.

Think:
**Attack Host → DEV01 → DC01**
You’re authenticated on DEV01, but DEV01 can’t prove *you* are legit to DC01.

---

## 🔍 Symptoms

Common signs you’ve hit the double-hop problem:

* You connect via **WinRM / evil-winrm** using a domain user (e.g., `backupadm`)
* Local commands work (hostname, whoami, local FS, etc.)
* **Anything that talks to the DC fails**, e.g.:

```powershell
Get-DomainUser -SPN
# DirectoryServicesCOMException / "An operations error occurred."
```

* `klist` on the remote box only shows a ticket for that host/service, e.g.:

```text
Server: HTTP/ACADEMY-AEN-DEV01.INLANEFREIGHT.LOCAL
```

* `mimikatz sekurlsa::logonpasswords` → **no clear creds / NT hash** for your user in that WinRM session.

---

## 🤔 Why It Happens (Short Mental Model)

* With **WinRM/PSRemoting using Kerberos**, your **TGS** for the *remote host* is used; your **TGT** is *not* forwarded.
* No TGT + no cached password/hash = that box **cannot** request new tickets (TGS) on your behalf.
* When the remote host tries to access DC/file shares/etc. as you, the DC goes:
  “Cool story, but where’s your TGT?” → access denied / ops error.

Compare:

* **PSExec / SMB / LDAP (password login)** → NTLM hash cached, TGT created, can do more hops.
* **WinRM Kerberos** → only enough to talk to that WinRM HTTP service, *not* to others.

---

## ✅ When You *Don’t* Really Suffer From It

* **RDP into host** with username/password → password/TGT cached locally; `klist` shows `krbtgt/DOMAIN` + other services; PowerView works normally.
* **Unconstrained delegation** box → user’s TGT *is* forwarded & cached there; that server can request tickets on the user’s behalf (and you basically already “won”).

---

## 🛠 Workaround #1 – PSCredential + `-Credential` Flag

Best when you’re in an **evil-winrm session** or basic WinRM shell and want tools like PowerView to work.

### 1. Build PSCredential object

```powershell
$SecPassword = ConvertTo-SecureString '!qazXSW@' -AsPlainText -Force
$Cred = New-Object System.Management.Automation.PSCredential('INLANEFREIGHT\backupadm', $SecPassword)
```

### 2. Call tools with `-Credential` explicitly

```powershell
Get-DomainUser -SPN -Credential $Cred | Select samaccountname
```

Now it works because **PowerView** is explicitly supplied creds and can bind to the DC.

If you forget `-Credential`:

```powershell
Get-DomainUser -SPN | Select samaccountname
# -> DirectoryServicesCOMException / "An operations error occurred."
```

**Use this pattern:**
Any AD query / remote access that fails due to double hop → rerun it with `-Credential $Cred`.

---

## 🛠 Workaround #2 – Register PSSession Configuration (Attack From Windows Host)

This is for when:

* You’re on a **Windows attack/jump host** (or domain-joined box with GUI),
* Using `Enter-PSSession` or `Invoke-Command` from that host to another,
* And you want **full Kerberos tickets (TGT) on the remote** without adding `-Credential` every time.

> ⚠️ Needs: elevated PowerShell + GUI pop-up / local cred entry. Does **not** work from `evil-winrm`.

### 1. On your Windows attack/jump host, register a RunAs endpoint:

```powershell
Register-PSSessionConfiguration -Name backupadmsess -RunAsCredential INLANEFREIGHT\backupadm
```

* You’ll get a creds popup → enter `backupadm`’s password.
* This creates a WinRM endpoint that **always runs as `backupadm`**.

Then restart WinRM:

```powershell
Restart-Service WinRM
```

(You’ll be kicked from current PSSessions.)

### 2. Connect using that configuration

```powershell
Enter-PSSession -ComputerName DEV01 -Credential INLANEFREIGHT\backupadm -ConfigurationName backupadmsess
```

Now inside that session:

```powershell
klist
# You should see krbtgt/INLANEFREIGHT.LOCAL etc. → you’ve got a TGT
```

### 3. Use tools normally, **no more -Credential spam**

```powershell
Import-Module .\PowerView.ps1
Get-DomainUser -SPN | Select samaccountname
# Works now
```

**Why this works:**
Your local machine now impersonates `backupadm` on DEV01 with a proper TGT, so DEV01 can get new TGS tickets to DC01/file shares/etc.

---

## 🧪 Quick “What Do I Do?” Flow

You’re in a WinRM shell and AD queries fail:

1. **Check tickets:**

   ```powershell
   klist
   ```

   * Only see HTTP/hostname → double hop problem.

2. **If you’re in evil-winrm / remote-only:**

   * Build `$Cred` → use `-Credential $Cred` for AD queries.

3. **If you’re on a Windows jump host with GUI + admin PS:**

   * Use `Register-PSSessionConfiguration -RunAsCredential ...`
   * `Restart-Service WinRM`
   * `Enter-PSSession -ConfigurationName <name>`
   * Enjoy full TGT + normal PowerView usage.

---

## 🧷 One-Liner Mental Model

> **WinRM + Kerberos gives you a ticket to the first box, not a portable “password”. No TGT = no second hop. Fix it by either resupplying creds each time (`-Credential`) or by creating a RunAs PSSession that holds a real TGT.**

