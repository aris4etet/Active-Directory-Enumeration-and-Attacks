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

