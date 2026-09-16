# 🔐 Active Directory Hash Dumping & Offline Password Cracking Lab

> **Educational purposes only.** This lab was conducted in a controlled environment to demonstrate common Active Directory attack techniques and the security vulnerabilities that enable them.

---

## 📋 Table of Contents

- [Overview](#overview)
- [Environment](#environment)
- [Tools Used](#tools-used)
- [Lab Walkthrough](#lab-walkthrough)
  - [Phase 1 — Hash Extraction](#phase-1--hash-extraction)
  - [Phase 2 — Offline Password Cracking](#phase-2--offline-password-cracking)
- [Vulnerabilities Identified](#vulnerabilities-identified)
- [Mitigations & Recommendations](#mitigations--recommendations)
- [Conclusion](#Conclusion)

---

## Overview

This lab simulates a post-exploitation scenario within an Active Directory (AD) environment. After gaining access to a domain controller, an attacker can extract password hashes for all domain accounts and crack them offline — entirely without triggering network-based detection systems.

The goal of this lab is to:
- Understand how AD credential hashes are stored and exposed
- Demonstrate how legacy authentication protocols weaken domain security
- Practice offline cracking techniques using dictionary attacks
- Identify and document findings as a security professional would

---

## Environment

| Component | Details |
|---|---|
| **Attacker Machine** | Kali Linux (`ACIKALI`) |
| **Domain** | `acilab.com` |
| **Domain Controller** | `ACIDC01$` |
| **Target User** | `acilab.com\testuser` |
| **Lab Date** | September 13, 2025 |

---

## Tools Used

| Tool | Purpose |
|---|---|
| **Impacket / secretsdump** | Extract Kerberos and NTLM hashes from AD |
| **John the Ripper** | Offline dictionary-based password cracking |
| **rockyou.txt** | Common wordlist used for the dictionary attack |
| **nano** | Text editor used to isolate target hash |

---

## Lab Walkthrough

### Phase 1 — Hash Extraction

After obtaining domain admin access, hashes were dumped from the Active Directory environment. The output revealed credentials for multiple accounts in several formats:

**Accounts Exposed:**

| Account | Type |
|---|---|
| `krbtgt` | Kerberos Ticket Granting Ticket account |
| `acilab.com\testuser` | Standard domain user |
| `ACIDC01$` | Domain Controller machine account |
| `ACIDM01$` | Domain member machine account |
| `ACIWIN11$` | Windows 11 workstation account |

**Hash Formats Present:**

| Format | Notes |
|---|---|
| `aes256-cts-hmac-sha1-96` | Modern Kerberos encryption |
| `aes128-cts-hmac-sha1-96` | Older Kerberos encryption |
| `des-cbc-md5` | ⚠️ Legacy DES — should be disabled |

The presence of `des-cbc-md5` indicates that legacy Kerberos encryption types are still enabled on this domain, which expands the attack surface considerably.

---

### Phase 2 — Offline Password Cracking

The hash for `acilab.com\testuser` was isolated and saved to a local file, then fed into **John the Ripper** with the `rockyou.txt` wordlist:

```bash
# Save the target hash
nano testuserhash

# Run dictionary attack
john --wordlist=/usr/share/wordlists/rockyou.txt testuserhash > crackedhash

# View the cracked result
cat crackedhash
```

<img width="814" height="612" alt="lab_screenshot_redacted" src="https://github.com/user-attachments/assets/15268614-5e5f-40aa-af53-c5f805d1ad42" />


**Results:**

```
Loaded 1 password hash (LM [DES 512/512 AVX512F])
ABC123!   (acilab.com\testuser)
```

John the Ripper identified the hash as **LM (LAN Manager)** format and cracked it at approximately **450,000 passwords/second**, completing almost instantly.

---

## Vulnerabilities Identified

### 1. 🔴 LM Hashes Enabled (Critical)
**What it is:** LAN Manager (LM) is a legacy Windows hashing scheme that splits passwords into two 7-character chunks before hashing, making brute-force attacks trivial.

**Impact:** An attacker with hash access can crack most passwords in seconds to minutes.

---

### 2. 🔴 Weak User Password (Critical)
**What it is:** The cracked password `ABC123!` appears in common wordlists and does not meet modern complexity standards.

**Impact:** Immediately cracked via dictionary attack with no computational difficulty.

---

### 3. 🟠 Legacy Kerberos DES Encryption Enabled (High)
**What it is:** `des-cbc-md5` hashes were present for all accounts, indicating DES Kerberos support is still active in the domain.

**Impact:** DES is cryptographically broken and allows for faster offline attacks and potential Kerberoasting escalation paths.

---

### 4. 🟡 Insufficient Hash Exposure Controls (Medium)
**What it is:** All domain account hashes — including machine accounts and `krbtgt` — were accessible post-compromise.

**Impact:** Exposure of the `krbtgt` hash enables **Golden Ticket** attacks, granting persistent, stealthy domain-level access.

---

## Mitigations & Recommendations

| Finding | Recommended Fix |
|---|---|
| LM hashes enabled | Disable via GPO: `Computer Configuration → Windows Settings → Security Settings → Security Options → Network security: Do not store LAN Manager hash value` |
| Weak passwords | Enforce a password policy: minimum 12 characters, uppercase, lowercase, numbers, and symbols. Consider a blocklist of common passwords. |
| DES Kerberos encryption | Disable legacy encryption types via GPO and ensure all accounts have `msDS-SupportedEncryptionTypes` set to exclude DES |
| Hash exposure post-compromise | Implement **Protected Users** security group, enable **Credential Guard**, and restrict administrative access using the principle of least privilege |
| `krbtgt` exposure | Regularly rotate the `krbtgt` account password (twice in succession) and monitor for Golden Ticket activity |

---

## ✅ Conclusion

This lab demonstrated how a combination of legacy configurations, weak credentials, and insufficient hardening can lead to a full credential compromise within an Active Directory environment — with minimal effort from an attacker.

The most critical takeaway is how quickly offline cracking works once hashes are obtained. With LM hashing enabled and a weak password in place, the credentials for `acilab.com\testuser` were recovered in under a second, at a rate of 450,000 guesses per second, requiring no network connection and generating no authentication logs. This highlights a fundamental problem: **perimeter defenses alone are not enough**. If an attacker reaches the point of hash extraction, the speed of the subsequent crack depends entirely on how well the organization hardened its password policies and legacy protocol configurations beforehand.

The four vulnerabilities identified in this lab — LM hash storage, weak passwords, DES Kerberos encryption, and uncontrolled hash exposure — are all well-documented, well-understood, and entirely preventable. None of them require advanced tooling or zero-days to exploit; they are default or misconfigured settings that have existed in Windows environments for decades and continue to appear in real-world assessments today.

Key lessons from this lab:

- **Offense informs defense.** Understanding how attackers extract and crack hashes is essential to building configurations that resist it.
- **Legacy protocol support is a liability.** LM and DES exist for backwards compatibility but introduce serious risk. If your environment doesn't need them, disable them.
- **Passwords are the last line of defense against offline attacks.** When hashes are stolen, the strength and uniqueness of the password is all that stands between an attacker and full access.
- **Assume breach.** Detection and response strategies should account for the possibility that hashes have already been obtained and are being cracked offline — outside the visibility of any SIEM or IDS.

Applying the mitigations outlined in this lab — disabling LM hashes, enforcing strong password policies, removing legacy Kerberos encryption types, and implementing Credential Guard — would have prevented or significantly slowed each phase of this attack.



---

> ⚠️ **Disclaimer:** All techniques demonstrated in this lab were performed in an isolated, controlled lab environment for educational purposes. Unauthorized use of these techniques against real systems is illegal and unethical.
