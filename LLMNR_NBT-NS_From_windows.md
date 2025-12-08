

# 🧨 LLMNR / NBT-NS Poisoning — From Windows (Inveigh Edition)

## 1️⃣ Why Use Inveigh?

When your **attack box is Windows** (or you’ve popped a Windows host with local admin) and you still want Responder-style magic, you use **Inveigh**.

Inveigh can:

* Listen for & spoof:

  * **LLMNR, NBNS, DNS, mDNS**
  * **SMB, HTTP, WebDAV, LDAP**
  * Proxy auth, Kerberos-related stuff (depending on config)
* Capture:

  * **NetNTLMv1 / NetNTLMv2 hashes**
  * Sometimes **cleartext creds**

Goal = same as Linux/Responder:

> Trick hosts into talking to you → capture hashes → crack offline → use creds.

---

## 2️⃣ Two Flavors of Inveigh

### 🅰 PowerShell Inveigh

* Original version (`Inveigh.ps1`)
* Runs directly in PowerShell
* Good for:

  * Quick use on a Windows attack box
  * Environments where you **can’t easily drop EXEs**

### 🅱 C# Inveigh (Inveigh.exe, aka InveighZero)

* Actively maintained version
* Compiled .exe
* Faster, cleaner, more stable for long runs
* Better choice when:

  * You control the host
  * AV/EDR situation allows it

---

## 3️⃣ PowerShell Inveigh — Mental Model

**Workflow:**

1. Load the module
2. Start Inveigh with LLMNR/NBNS spoofing enabled
3. Let it sit and watch the console scroll
4. Use the built-in console to list hashes / usernames
5. Export hashes → crack with Hashcat/John (offline)

### Key Concepts

* **Import-Module** to load Inveigh.
* `Invoke-Inveigh` is the main function.
* Common things you toggle:

  * LLMNR spoofing
  * NBNS spoofing
  * Console output
  * File logging

You’ll see:

* Status of:

  * LLMNR / NBNS / DNS / mDNS spoofers
  * SMB / HTTP / HTTPS / WPAD capture
* Incoming name resolution requests
* SMB auth attempts + NTLM challenges
* Warnings if ports are in use (e.g., port 80 already bound)

---

## 4️⃣ Inveigh’s Interactive Console (PowerShell & C#)

Once Inveigh is running, you can **hit `ESC`** to open its interactive console.

From there you can:

### 🔍 Useful Commands

* `GET NTLMV2UNIQUE`
  → One **unique NetNTLMv2 hash per user**

* `GET NTLMV2USERNAMES`
  → Shows `IP | Hostname | DOMAIN\User | Challenge`

* `GET CLEARTEXT` / `GET CLEARTEXTUNIQUE`
  → If any cleartext creds were captured

* `GET LOG`
  → General log lines (optionally filtered)

* `STOP`
  → Cleanly stops Inveigh

This console is **perfect** for:

* Quickly pulling a target user list
* Copy-pasting hashes into your cracking box
* Checking which hosts/users are talking to you

---

## 5️⃣ C# Inveigh (Inveigh.exe) — What to Expect

When you run the EXE with defaults, you’ll see a header showing:

* Enabled/disabled modules, e.g.:

  * `[+] LLMNR Packet Sniffer`
  * `[+] SMB Packet Sniffer`
  * `[+] HTTP Listener …`
  * `[ ] MDNS`
  * `[ ] NBNS`
  * etc.

* Listener and spoofing addresses (IPv4 & IPv6)

* Output directory (e.g., `C:\Tools`)

It will then:

* Log LLMNR name requests
* Print when it responds as spoofed host
* Show SMB connections and negotiation attempts
* Save hashes and logs to disk

Like PowerShell Inveigh, C# Inveigh also supports the **interactive console**:

* Hit `ESC` → type `HELP` to see available commands
* Same style `GET NTLMV2UNIQUE`, `GET NTLMV2USERNAMES`, `STOP`, etc.

---

## 6️⃣ What You Do With the Captured Hashes

Once you’ve got NetNTLMv2 hashes:

1. Copy them into a text file on your cracking rig.
2. Use a NetNTLMv2 mode (e.g. Hashcat’s appropriate mode).
3. Crack using a solid wordlist and/or rules.
4. If cracked → you now have:

   * `DOMAIN\user`
   * `cleartext password`

Then you can:

* Use SMB, WinRM, RDP, LDAP, Kerberos, etc.
* Treat these creds as your new foothold.

*(Exact cracking commands you already know from your Linux/Responder side.)*

---

## 7️⃣ Defensive Side — Remediation (Blue Team View)

LLMNR/NBT-NS poisoning is tracked under:

> **MITRE ATT&CK T1557.001 — Adversary-in-the-Middle: LLMNR/NBT-NS Poisoning & SMB Relay**

### 🔒 Main Hardening Steps

1. **Disable LLMNR via GPO**

   * `Computer Configuration → Administrative Templates → Network → DNS Client → Turn OFF Multicast Name Resolution`

2. **Disable NetBIOS over TCP/IP per interface**

   * Network Adapter → IPv4 → Advanced → WINS → “Disable NetBIOS over TCP/IP”
   * For domain-wide rollout: GPO startup PowerShell script that sets `NetbiosOptions = 2` under:

     * `HKLM\SYSTEM\CurrentControlSet\services\NetBT\Parameters\Interfaces\*`

3. **Enable SMB Signing**

   * Prevents many SMB relay attacks even if hashes are captured.

4. **Network segmentation & filtering**

   * Limit LLMNR/NBNS traffic between segments.
   * IDS/IPS to watch for spoofing patterns.

---

## 8️⃣ Detection Ideas

When you *can’t* fully disable LLMNR/NBNS, you try to **catch** abuse:

* **Inject fake LLMNR/NBT-NS queries** for non-existent hosts across subnets and:

  * Alert if any system responds ⇒ likely an attacker tool (Responder/Inveigh).

* Monitor:

  * UDP **5355** (LLMNR), UDP **137** (NetBIOS)
  * Windows event IDs such as:

    * **4697, 7045** (new services / persistence indicators)
  * Registry key:

    * `HKLM\Software\Policies\Microsoft\Windows NT\DNSClient`

      * `EnableMulticast` value:

        * `0` = LLMNR disabled
        * Unexpected changes may indicate tampering

---

## 9️⃣ How This Fits Into the Bigger Kill Chain

1. **No creds yet** → start Inveigh on a Windows foothold/box.
2. Wait for:

   * Mistyped hostnames
   * Auto-discovery / WPAD traffic
3. Capture NetNTLMv2 hashes.
4. Crack offline.
5. Use creds for:

   * Domain logon
   * Lateral movement
   * Privilege escalation
6. Repeat as needed, prioritizing high-value accounts (service users, backup agents, monitoring agents, etc.).


