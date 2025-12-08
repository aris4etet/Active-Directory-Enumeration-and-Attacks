# 🔐 Enumerating & Interpreting AD Password Policies

## 1️⃣ Why Password Policy Matters

Knowing the domain password policy lets you:

* Design **safe password spraying** (avoid lockouts).
* Choose **realistic candidate passwords** (length + complexity).
* Prioritize **which attack paths are practical** (spray vs brute vs targeted).
* Understand how likely **weak / reused passwords** are.

Key knobs you care about:

* **Minimum password length**
* **Complexity** (enabled or not)
* **History length**
* **Max/min password age** (rotation habits)
* **Lockout threshold / duration / reset window**
* **Auto vs manual unlock**

---

## 2️⃣ From Linux (Credentialed) – SMB

### 🔹 CrackMapExec

With valid domain creds:

```bash
crackmapexec smb <DC_IP> -u <user> -p <pass> --pass-pol
```

What CME gives you:

* Min / max password length & age
* Password history length
* Complexity flag (enabled/disabled)
* Lockout threshold, duration, reset window
* Whether passwords are stored / refuse change / cleartext options

Use this when:

* You already have **one set of valid creds**
* You want a **quick, high-level policy snapshot** from Linux

---

## 3️⃣ From Linux (No Creds) – SMB NULL Session

If the DC allows **SMB NULL sessions**, you can pull policy anonymously.

### 🔹 rpcclient (NULL session)

```bash
rpcclient -U "" -N <DC_IP>
```

Inside rpcclient:

```text
querydominfo       # general domain info
getdompwinfo       # password policy (min length, complexity, etc.)
```

If this works, you know:

* NULL sessions are allowed (legacy / misconfig)
* You can **pull policy & often enumerate users/groups** without credentials

---

## 4️⃣ From Linux – enum4linux & enum4linux-ng

These wrap smbclient/rpcclient/etc. to auto-enumerate:

### 🔹 Classic enum4linux

```bash
enum4linux -P <DC_IP>
```

Useful for:

* Password policy
* Domain info
* NULL session feasibility

### 🔹 enum4linux-ng (Python rewrite)

```bash
enum4linux-ng -P <DC_IP> -oA <basename>
enum4linux-ng -P 172.16.5.5 -oA ilfreight     
```

You get:

* Cleaner, structured output
* JSON/YAML exports (e.g. `ilfreight.json`) with:

  * Min length, history, complexity
  * Lockout threshold, duration, reset
  * SMB dialects & signing
  * Whether **null_session_possible: true**

Great when you want to feed data into other tools or scripts.

---

## 5️⃣ From Windows – Null Sessions via `net use`

You can also test null/guest access from a Windows box:

```cmd
net use \\<DC_NAME>\ipc$ "" /u:""
```

* **Success** → NULL session is allowed.
* Then you can try further enumeration with other tools.

Common error messages (good for OSINT from “broken” attempts):

* **Account disabled**
  `System error 1331 has occurred. This user can't sign in because this account is currently disabled.`
* **Bad password**
  `System error 1326 has occurred. The user name or password is incorrect.`
* **Account locked out (policy)**
  `System error 1909 has occurred. The referenced account is currently locked out...`

---

## 6️⃣ From Linux – LDAP Anonymous Bind

If the DC allows **LDAP anonymous binds**, you can also pull policy via LDAP.

### 🔹 ldapsearch example

```bash
ldapsearch -h <DC_IP> -x \
  -b "DC=INLANEFREIGHT,DC=LOCAL" -s sub "*" \
  | grep -m 1 -B 10 pwdHistoryLength
```

Look for fields like:

* `minPwdLength`
* `pwdHistoryLength`
* `lockoutThreshold`
* `minPwdAge` / `maxPwdAge`
* `pwdProperties` (bitmask – e.g. 1 = complexity enabled)

Note: newer ldapsearch uses `-H ldap://<DC_IP>` instead of `-h`.

---

## 7️⃣ From Windows (Credentialed) – Built-in + PowerView

### 🔹 Using `net.exe` (no extra tools)

```cmd
net accounts
```

Gives:

* Minimum / maximum password age
* Minimum password length
* Password history length
* Lockout threshold
* Lockout duration & observation window

Perfect when:

* You popped a box
* Can’t (or don’t want to) drop extra binaries

### 🔹 Using PowerView

```powershell
Import-Module .\PowerView.ps1
Get-DomainPolicy
```

You’ll see:

* `SystemAccess` block:

  * `MinimumPasswordLength`
  * `MaximumPasswordAge` / `MinimumPasswordAge`
  * `PasswordHistorySize`
  * `PasswordComplexity`
  * `LockoutBadCount`, `LockoutDuration`, `ResetLockoutCount`
* GPO path (`Default Domain Policy`) for extra context

---

## 8️⃣ Interpreting a Typical Policy (INLANEFREIGHT Example)

From all sources (CME, enum4linux, net, PowerView), the example domain policy is:

* **Minimum password length**: `8`
* **Password history**: `24`
* **Maximum password age**: `Not set / Unlimited`
* **Minimum password age**: `1 day`
* **Lockout threshold**: `5` bad attempts
* **Lockout duration**: `30 minutes`
* **Reset lockout counter**: `30 minutes`
* **Password complexity**: `Enabled` (`PasswordComplexity=1`)

### 🔎 What this tells you operationally

* 8-char minimum + complexity = users often choose:

  * `Welcome1`, `Password1`, `SeasonYYYY!`, `Company123!` etc.
* Lockout threshold `5`:

  * You should keep **sprays below that** (e.g., 2–3 passwords per user per window).
* Lockout duration `30 minutes`:

  * Even if a lockout happens, it **auto-unlocks**, but:

    * You still want to avoid drawing attention
    * Especially in orgs where helpdesk has to manually unlock

### Default AD policy (if never changed)

Rough defaults for a fresh AD domain:

| Setting                                     | Default  |
| ------------------------------------------- | -------- |
| Enforce password history                    | 24       |
| Maximum password age                        | 42 days  |
| Minimum password age                        | 1 day    |
| Minimum password length                     | 7        |
| Complexity                                  | Enabled  |
| Store passwords using reversible encryption | Disabled |
| Account lockout duration                    | Not set  |
| Account lockout threshold                   | 0        |
| Reset lockout counter after                 | Not set  |

If you see something close to this, the org **probably never hardened** their password settings.
Nice, this plugs in perfectly after the password policy section. Here’s a continuation in the same “field notes” style.

---

## 9️⃣ Building a Target User List for Password Spraying

Before you spray, you need **two things**:

1. A **list of valid usernames**
2. The **password policy** (min length, complexity, lockout threshold, timers)

You already handled #2 in the previous section — now this is about #1.

High-level ways to get a user list:

* SMB **NULL session** → pull users directly from the DC
* **LDAP anonymous bind** → dump users via LDAP
* **Kerbrute** + username wordlists → validate users with Kerberos pre-auth behavior
* **Existing credentials** (from Responder/Inveigh, previous spray, client-supplied) → query AD normally
* Fallback: **external OSINT** (email scraping, LinkedIn → username patterns)

Always log your activity when spraying, including:

* Usernames targeted
* DC used
* Date / time of spray
* Password(s) attempted

So if lockouts or alerts occur, you can correlate with your actions.

---

## 🔹 SMB NULL Session → User List (Linux)

If the DC allows **SMB NULL sessions**, you can enumerate users without creds.

### a) enum4linux

Quick user dump:

```bash
enum4linux -U 172.16.5.5 | grep "user:" | cut -f2 -d"[" | cut -f1 -d"]"
```

Gives a nice **one-username-per-line** output like:

```text
administrator
guest
krbtgt
lab_adm
htb-student
avazquez
...
```

Good for:

* Fast, dirty user list you can pipe directly into spraying tools.

---

### b) rpcclient (NULL session)

```bash
rpcclient -U "" -N 172.16.5.5
```

Inside:

```text
rpcclient $> enumdomusers
user:[administrator] rid:[0x1f4]
user:[guest]        rid:[0x1f5]
user:[krbtgt]       rid:[0x1f6]
user:[lab_adm]      rid:[0x3e9]
...
```

You can then post-process to strip out just the usernames.

---

### c) CrackMapExec `--users`

```bash
crackmapexec smb 172.16.5.5 --users
```

Output example:

```text
INLANEFREIGHT.LOCAL\administrator  badpwdcount: 0  baddpwdtime: 2022-01-10 ...
INLANEFREIGHT.LOCAL\guest          badpwdcount: 0  baddpwdtime: 1600-12-31 ...
INLANEFREIGHT.LOCAL\avazquez       badpwdcount: 20 baddpwdtime: 2022-02-17 ...
```

Why this is extra useful:

* You see **`badpwdcount`** → how many bad attempts already
* You see **`baddpwdtime`** → when the last bad attempt occurred

👉 That lets you **remove risky accounts** (e.g., `badpwdcount` already high) from your spray list so you don’t accidentally trigger lockout.

Note: in multi-DC environments, `badpwdcount` is per-DC; the most accurate value comes from the **PDC Emulator**.

---

## 🔹 LDAP Anonymous Bind → User List (Linux)

If the DC allows **anonymous LDAP binds**, you can pull users via LDAP.

### a) ldapsearch

```bash
ldapsearch -h 172.16.5.5 -x \
  -b "DC=INLANEFREIGHT,DC=LOCAL" -s sub \
  "(&(objectclass=user))" \
  | grep sAMAccountName: | cut -f2 -d" "
```

Example output:

```text
guest
ACADEMY-EA-DC01$
htb-student
avazquez
pfalcon
...
```

You’ll get both **user accounts and computer accounts** (`$` at the end). Easy to filter out machines later.

---

### b) windapsearch

More ergonomic way to do the same thing:

```bash
./windapsearch.py --dc-ip 172.16.5.5 -u "" -U
```

* `-u ""` → anonymous
* `-U` → users only

Example snippet:

```text
cn: Guest

cn: Htb Student
userPrincipalName: htb-student@inlanefreight.local

cn: Annie Vazquez
userPrincipalName: avazquez@inlanefreight.local
...
```

You can then extract:

* `userPrincipalName` (full email-style username)
* or derive the `sAMAccountName` depending on format

---

## 🔹 Kerbrute → Username Validation (No Creds)

When you **can’t** do SMB NULL or LDAP anon, Kerbrute is your friend.

### How it works (username enumeration mode)

* Sends **TGT requests without pre-auth**.
* If KDC returns **`PRINCIPAL UNKNOWN`** → username invalid.
* If KDC asks for **Kerberos pre-auth** → username exists.

This:

* **Does not** cause logon failure (4625)
* **Does** generate Kerberos ticket request events (4768) if logging is enabled

### Example

```bash
kerbrute userenum -d inlanefreight.local --dc 172.16.5.5 /opt/jsmith.txt
```

Using a wordlist like `jsmith.txt` from `statistically-likely-usernames`:

```text
[+] VALID USERNAME:  jjones@inlanefreight.local
[+] VALID USERNAME:  sbrown@inlanefreight.local
[+] VALID USERNAME:  njohnson@inlanefreight.local
...
```

Result: a clean list of **valid UPNs** that you can then feed into password spraying.

⚠️ Once you switch Kerbrute into **password spraying mode**, failed pre-auth attempts **do count** toward bad password attempts and can lock accounts, so the **password policy rules still apply.**

---

## 🔹 Credentialed Enumeration → User List (Linux/Windows)

If you already have a valid domain account, just use it.

### CrackMapExec with creds

```bash
sudo crackmapexec smb 172.16.5.5 \
  -u htb-student -p 'Academy_student_AD!' --users
```

You get:

* Verified that your creds work
* Full list of users
* `badpwdcount` + `baddpwdtime` for each

You can then:

* Export usernames to a file
* Drop risky accounts (e.g., admin with `badpwdcount` already at 3/5)
* Use the final cleaned list for spraying

---

## 🔹 OSINT / External Sources (Fallback)

If **everything internal is locked down** (no NULL, no LDAP anon, no Kerbrute allowed):

* Scrape email patterns from:

  * company site
  * public documents
  * breach data (if allowed, e.g. under a red-team scope)
* Use tools like **linkedin2username** to generate likely usernames:

  * `first.last`
  * `flast`
  * `first_last`
* Spray carefully, with:

  * **very low volume**
  * long delays
  * strong justification in your engagement notes

---


