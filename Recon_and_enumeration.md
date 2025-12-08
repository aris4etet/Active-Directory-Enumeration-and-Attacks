# External Reconnaissance — What, Where & How

## What Are We Looking For?

When conducting external reconnaissance, there are several key items to search for. This information may not always be publicly accessible, but it’s prudent to see what’s out there. If we get stuck during a penetration test, reviewing what could be obtained through passive recon can provide the nudge needed to move forward — for example, password breach data that could be used to access a VPN or other externally-facing service.

The table below highlights the **what** we search for during this phase of engagement.

| Data Point             | Description                                                                                                                                                                                                                                                                                                                           |
| ---------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **IP Space**           | Valid ASN for our target, netblocks used for the organization's public-facing infrastructure, cloud presence and hosting providers, DNS record entries, etc.                                                                                                                                                                          |
| **Domain Information** | Based on IP data, DNS, and site registrations. Who administers the domain? Are there any subdomains tied to the target? Are there any publicly accessible domain services present (mail servers, DNS, websites, VPN portals, etc.)? Can we determine what kind of defenses are in place (SIEM, AV, IPS/IDS, etc.)?                    |
| **Schema Format**      | Can we discover the organization's email accounts, AD usernames, and even password policies? Anything that helps build a valid username list to test external-facing services for password spraying, credential stuffing, brute forcing, etc.                                                                                         |
| **Data Disclosures**   | Publicly accessible files (`.pdf`, `.ppt`, `.docx`, `.xlsx`, etc.) that reveal information about the target. Examples: published files containing intranet site listings, user metadata, shares, or other critical software/hardware (credentials pushed to a public GitHub repo, internal AD username format in PDF metadata, etc.). |
| **Breach Data**        | Any publicly released usernames, passwords, or other critical information that can help an attacker gain a foothold.                                                                                                                                                                                                                  |

We have covered the **why** and **what** of external reconnaissance; next — the **where** and **how**.

---

## Where Are We Looking?

The data points above can be gathered in many ways. Numerous websites and tools provide some or all of the information needed for an assessment. Below are example resources and typical uses.

| Resource                           | Examples / Notes                                                                                                                                                                                                                                                                                         |
| ---------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **ASN / IP registrars**            | IANA, ARIN (for the Americas), RIPE (for Europe), BGP Toolkit                                                                                                                                                                                                                                            |
| **Domain Registrars & DNS**        | DomainTools, PTRArchive, ICANN, manual DNS record queries against the domain or public DNS servers (e.g., `8.8.8.8`)                                                                                                                                                                                     |
| **Social Media**                   | LinkedIn, Twitter, Facebook, regional social platforms, news articles — search for employee names, job titles, tech stack clues, org changes, announcements                                                                                                                                              |
| **Public-Facing Company Websites** | Corporate websites often contain embedded documents, news, and “About Us” / “Contact Us” pages that can expose useful details                                                                                                                                                                            |
| **Cloud & Dev Storage**            | GitHub, public AWS S3 / Azure Blob containers, Google dork searches to find exposed storage or code containing secrets                                                                                                                                                                                   |
| **Breach Data Sources**            | HaveIBeenPwned (check corporate email exposure), Dehashed (search corporate emails for plaintext passwords or hashes). Reused or leaked credentials can be tested against exposed login portals (Citrix, RDS, OWA, Office 365, VPN, VMware Horizon, custom apps, etc.) that may authenticate against AD. |


---

# Enumerating Security Controls (Quick Overview)

After gaining a foothold, one of the first steps is understanding **what defensive controls are active** in the environment. This helps us adjust our tooling, avoid detection, and decide when to "live off the land" using built-in Windows utilities. Different hosts may have different policies applied, so checking from multiple machines is useful.

---

## 🔹 Windows Defender

Modern Windows Defender can block many common enumeration tools. We can quickly check its status with:

```powershell
Get-MpComputerStatus
```

If **RealTimeProtectionEnabled = True**, Defender is actively monitoring the system.

---

## 🔹 AppLocker

AppLocker restricts what executables, scripts, and installers users can run. Organizations often block `powershell.exe` or `cmd.exe`, but other PowerShell paths (e.g., SysWOW64) may still work.

Check rules with:

```powershell
Get-AppLockerPolicy -Effective | select -ExpandProperty RuleCollections
```

This tells us what is allowed or denied and helps identify bypass opportunities.

---

## 🔹 PowerShell Constrained Language Mode

Some environments restrict PowerShell features, blocking many offensive functions.

Check your mode:

```powershell
$ExecutionContext.SessionState.LanguageMode
```

* **FullLanguage** = normal
* **ConstrainedLanguage** = heavily restricted

---

## 🔹 LAPS (Local Administrator Password Solution)

LAPS rotates local admin passwords to prevent lateral movement. We can enumerate:

* Who can read LAPS passwords
* Which machines have LAPS enabled
* Passwords themselves (if our user has permission)

Examples:

```powershell
Find-LAPSDelegatedGroups
Find-AdmPwdExtendedRights
Get-LAPSComputers
```

This helps identify targets where LAPS is misconfigured or overly permissive.

---
