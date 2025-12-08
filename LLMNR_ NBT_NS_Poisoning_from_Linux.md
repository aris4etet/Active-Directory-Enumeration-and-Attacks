# 🧨 **LLMNR / NBT-NS Poisoning from Linux — Mega-Cheatsheet**

## 📝 **Why This Matters**

LLMNR & NBT-NS poisoning is one of the *fastest* and *easiest* ways to get **cleartext credentials or NTLMv2 hashes** on an internal network.
If a single user mistypes a hostname, you win.

This attack is used to:

✔ Capture NTLMv1/NTLMv2 hashes
✔ Crack them offline (Hashcat/John)
✔ Potentially relay them (SMB Relay)
✔ Obtain an initial foothold in an Active Directory domain

---

# 🌐 1. **LLMNR & NBT-NS — What They Are**

### 🔹 If DNS fails → Windows falls back to LLMNR (UDP 5355)

### 🔹 If LLMNR fails → Windows falls back to NetBIOS (UDP 137)

Both are **broadcast-based**, meaning:

👉 *Any* machine on the subnet can claim to be the host being requested.
👉 That lets us *pretend* to be the requested host and capture authentication attempts.

---

# 🎯 2. **The Attack Flow (High-Level)**

1. Victim mistypes a hostname:
   `\\printer01.inlanefreight.local` (instead of `print01`)

2. DNS says: ❌ Host not found

3. Victim broadcasts LLMNR/NBT-NS query ⇒
   *“Does anyone know where printer01 is?”*

4. **Responder answers:**
   *“Yep, I’m printer01 – talk to me.”*

5. Windows sends NTLMv2 authentication:

```
Username + Domain + NetNTLMv2 Hash
```

6. We now have the hash ⇒
   ✔ Crack it
   ✔ Or relay it (if SMB signing is disabled)

Instant foothold.

---

# ⚔️ 3. **Tools for Poisoning**

| Tool           | Use                                                        |
| -------------- | ---------------------------------------------------------- |
| **Responder**  | Fast, reliable LLMNR/NBT-NS/MDNS poisoning (Linux/Windows) |
| **Inveigh**    | C#/PowerShell version (great from a Windows foothold)      |
| **Metasploit** | Has poisoning modules but heavier                          |

Linux → **Responder is the default king.**

---

# 🚀 4. **Using Responder (Linux)**

### 🔥 Start poisoning on your active interface

```bash
sudo responder -I eth0
```

This runs the full suite:
✔ LLMNR
✔ NBT-NS
✔ MDNS
✔ WPAD
✔ SMB, HTTP, SQL rogue servers

Responder **listens + responds**, capturing NTLM hashes.

---

# 🧭 5. Key Responder Flags

| Flag      | Meaning                                         |
| --------- | ----------------------------------------------- |
| `-I eth0` | Select network interface                        |
| `-A`      | Analyze mode (passive — NO poisoning)           |
| `-w`      | Enable WPAD rogue proxy (captures browser auth) |
| `-f`      | Fingerprints hosts                              |
| `-v`      | Verbose output                                  |

Typical useful run:

```bash
sudo responder -I eth0 -w -f
```

---

# 📂 6. Where Hashes Are Stored

Hashes appear in:

```
/usr/share/responder/logs/
```

Common filenames:

```
SMB-NTLMv2-SSP-<IP>.txt
HTTP-NTLMv2-<IP>.txt
Proxy-Auth-NTLMv2-<IP>.txt
```

Example NetNTLMv2 hash:

```
FOREND::INLANEFREIGHT:4af70a7...:0f85ad...:010100...
```

---

# 🔐 7. **Cracking NetNTLMv2 With Hashcat**

Mode for NTLMv2:

```
-m 5600
```

Example:

```bash
hashcat -m 5600 captured_hash.txt /usr/share/wordlists/rockyou.txt
```

Example "cracked" output:

```
FOREND::INLANEFREIGHT:...: Klmcargo2
```

Boom — valid domain credentials.

---

# 🏆 8. When This Attack Works Best

✔ Users mistype UNC paths (`\\fileserver1`)
✔ Scripts or services request nonexistent hosts
✔ WPAD auto-detect enabled
✔ SMB signing is **disabled** (enables relay!)
✔ Flat networks (broadcast domains)

---

# 🚫 9. When It Fails

❌ Network segmentation (no broadcast visibility)
❌ LLMNR & NBT-NS disabled via GPO
❌ WPAD disabled
❌ Strong passwords (hashes don’t crack)
❌ SMB signing required (relay blocked)

---

# 🛠 10. Port Requirements

Make sure Responder can bind to:

```
UDP: 137, 138, 53, 389, 5355, 5353
TCP: 80, 135, 139, 445, 389, 1433, 3141, 21, 25, 110, 587, 3128
```

If ports are used by other services, disable modules in:

```
/usr/share/responder/Responder.conf
```

---

# 🧨 11. Typical Pentest Workflow

1. Plug into network / get VPN access
2. Start `responder` in a tmux pane
3. Let it run while doing other enumeration
4. When a hash comes in:
   ✔ identify it
   ✔ crack it
5. Use cracked creds:

   * WinRM
   * SMB
   * LDAP
   * Kerberos
6. Begin *credentialed enumeration in the domain*

---

# ✔️ 12. Example: Full Real-World Flow

Captured hash:

```
SMB-NTLMv2 for user FOREND
```

Cracked:

```
Password = Klmcargo2
```

Use it:

```bash
crackmapexec smb <DC-IP> -u FOREND -p Klmcargo2
```

If valid:

→ You now have your initial domain foothold.

---

# 🎯 13. What Comes Next?

After poisoning → options:

### ✔ **Password spraying**

Try FOREND’s password across multiple users.

### ✔ **LDAP enumeration**

See what the user can access.

### ✔ **Kerberoasting**

If user has SPN privileges.

### ✔ **SMB Relay**

If signing is off.

### ✔ **Move laterally**

FOOT → FOOTHOLD → PRIVESC.

---
