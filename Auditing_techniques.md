# **AD Auditing – Quick Cutsheet**

## **1️⃣ AD Explorer (Sysinternals) – Snapshot & Compare**

**Use cases:** Offline AD review, before/after change analysis, auditing permissions & attributes.

**Workflow:**

* Launch AD Explorer → authenticate with any domain user.
* Browse OU/object structure, view attributes, permissions, schema info.
* **Create Snapshot:**
  *File → Create Snapshot* → save `.dat` file → review offline.
* **Compare snapshots** to detect:

  * New/deleted objects
  * Permission changes
  * Attribute modifications

**Why use it:** Great for reports, visual inspections, and documenting risky permissions.

---

## **2️⃣ PingCastle – Fast AD Risk Assessment**

**Purpose:** High-level domain security scoring + maps + vulnerability insights.

**Common startup:**

```cmd
PingCastle.exe --interactive
```

Key modes:

* **1 – healthcheck:** Full AD security audit (recommended)
* **3 – carto:** Domain trust map
* **5 – export:** Dump users/computers for additional analysis
* **Scanner menu:** ACL checks, null sessions, LAPS, Zerologon, SMB, spooler, etc.

**Outputs include:**

* Domain maturity score (CMMI-based)
* Vulnerability list (password policy, delegation, SPNs, ACL issues)
* Trust diagrams
* User/computer summaries

**Use when writing reports** to clearly visualize weak configurations.

---

## **3️⃣ Group3r – Group Policy Vulnerability Scanner**

**Goal:** Identify insecure or misconfigured GPO settings.

**Basic usage:**

```cmd
group3r.exe -s        # output to console
group3r.exe -f results.log
```

**Findings highlight:**

* Insecure registry/NTLM/LANMAN settings
* Lax user rights assignments
* Privilege escalation vectors baked into GPO
* GPOs applying dangerous configurations to wide scopes

**Indentation levels:**

* 0 indent = GPO
* 1 indent = policy section
* 2 indent = specific finding

**Why run it:** Frequently uncovers obscure GPO-based privilege escalation paths other tools miss.

---

## **4️⃣ ADRecon – Bulk AD Data Extraction**

**Use case:** Full AD dump for reporting, Excel-based summaries, and cross-checking other tools.

**Run with:**

```powershell
.\ADRecon.ps1
```

**Data collected:**

* Forest/domain info
* Trusts
* Sites/subnets
* Password policies
* Domain Controllers
* OUs
* GPOs + links
* Users, SPNs, groups, memberships
* DNS zones
* Printers
* Computers
* LAPS data, Bitlocker keys (if privileged)

**Output:**

* HTML report
* CSV files
* Optional Excel workbook (`-GenExcel`)

**Best use:** Post-engagement analysis, documenting everything AD-related in one deliverable.

---

# **When to Use What**

| Tool            | Purpose                                 | Strength                                         |
| --------------- | --------------------------------------- | ------------------------------------------------ |
| **AD Explorer** | Offline snapshot + manual AD inspection | Change tracking, detailed attribute review       |
| **PingCastle**  | Domain-wide risk score + graphs         | Exec-friendly reporting & quick hygiene overview |
| **Group3r**     | GPO security issues                     | Finds policy-based escalation paths              |
| **ADRecon**     | Full AD inventory dump                  | Comprehensive dataset for deep reporting         |

---

# **Why These Matter in Reporting**

Running these tools helps you:

* Back up findings with concrete evidence
* Highlight misconfigurations visually
* Give clients actionable, organized remediation data
* Strengthen your recommendations with measurable metrics

More data = stronger justification for budget, fixes, and long-term AD hardening.

---
