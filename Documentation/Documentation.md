---
title: "Operation Endgame — Active Directory Security Assessment"
description: "Professional TryHackMe Active Directory assessment documenting Guest exposure, LDAP enumeration, Kerberoasting, password reuse, ACL abuse, lateral movement, hardcoded credentials, and Domain Administrator compromise."
platform: "TryHackMe"
room: "Operation Endgame"
difficulty: "Hard"
environment: "Microsoft Active Directory / Windows"
domain: "thm.local"
author: "Anurag Ravankar"
---

# Operation Endgame — Active Directory Security Assessment

> **Platform:** TryHackMe  
> **Room:** Operation Endgame  
> **Difficulty:** Hard  
> **Environment:** Microsoft Active Directory / Windows  
> **Domain:** `thm.local`  
> **Target Role:** Domain Controller  
> **Assessment Type:** Authorized lab-based security assessment  
> **Publication Status:** Public portfolio edition — challenge secrets redacted

---

## Documentation Map

<div class="ctf-toc">
<div class="ctf-toc-title">Documentation Map</div>
<ul>
<li><a href="#mission-and-executive-summary">Mission and Executive Summary</a></li>
<li><a href="#assessment-overview">Assessment Overview</a></li>
<li><a href="#scope-and-objectives">Scope and Objectives</a></li>
<li><a href="#methodology-and-tooling">Methodology and Tooling</a></li>
<li><a href="#reconnaissance-and-service-enumeration">Reconnaissance and Service Enumeration</a></li>
<li><a href="#dns-and-web-service-assessment">DNS and Web Service Assessment</a></li>
<li><a href="#kerberos-user-enumeration">Kerberos User Enumeration</a></li>
<li><a href="#rpc-null-authentication">RPC Null Authentication</a></li>
<li><a href="#smb-and-guest-access">SMB and Guest Access</a></li>
<li><a href="#ldap-information-disclosure">LDAP Information Disclosure</a></li>
<li><a href="#as-rep-roasting-assessment">AS-REP Roasting Assessment</a></li>
<li><a href="#guest-accessible-kerberoasting">Guest-Accessible Kerberoasting</a></li>
<li><a href="#service-ticket-cracking">Service-Ticket Cracking</a></li>
<li><a href="#bloodhound-collection-and-analysis">BloodHound Collection and Analysis</a></li>
<li><a href="#password-reuse-and-credential-validation">Password Reuse and Credential Validation</a></li>
<li><a href="#genericwrite-attack-path">GenericWrite Attack Path</a></li>
<li><a href="#targeted-kerberoasting">Targeted Kerberoasting</a></li>
<li><a href="#interactive-access-through-rdp">Interactive Access Through RDP</a></li>
<li><a href="#local-enumeration-and-credential-discovery">Local Enumeration and Credential Discovery</a></li>
<li><a href="#domain-administrator-validation">Domain Administrator Validation</a></li>
<li><a href="#end-to-end-attack-chain">End-to-End Attack Chain</a></li>
<li><a href="#security-findings">Security Findings</a></li>
<li><a href="#mitre-attck-mapping">MITRE ATT&amp;CK Mapping</a></li>
<li><a href="#defensive-recommendations">Defensive Recommendations</a></li>
<li><a href="#lessons-learned">Lessons Learned</a></li>
<li><a href="#evidence-and-artifact-management">Evidence and Artifact Management</a></li>
<li><a href="#challenge-flag-protection">Challenge Flag Protection</a></li>
<li><a href="#conclusion">Conclusion</a></li>
<li><a href="#references">References</a></li>
</ul>
</div>

---

## 1. Mission and Executive Summary

Operation Endgame is a hard-difficulty Active Directory lab that demonstrates how weaknesses in identity exposure, Kerberos service authentication, password management, directory permissions, and credential storage can combine into a domain-wide compromise.

The assessment begins with reconnaissance of a Windows Domain Controller and identifies a broad set of exposed infrastructure services, including DNS, Kerberos, LDAP, SMB, RPC, Global Catalog, and Remote Desktop Protocol (RDP).

Anonymous and Guest access provide opportunities for user and directory enumeration. The resulting account information supports the assessment of Kerberos attack surfaces. Although the AS-REP roasting branch identifies accounts with weak pre-authentication configurations, the supplied wordlist does not recover a usable password from that branch.

A more productive path emerges through Guest-accessible Kerberoasting. A service ticket associated with `CODY_ROY` is recovered and successfully cracked. The resulting password is also valid for `ZACHARY_HUNT`, demonstrating the impact of password reuse across distinct domain identities.

BloodHound analysis then identifies a `GenericWrite` relationship from `ZACHARY_HUNT` to `JERRI_LANCASTER`. This excessive directory permission enables a targeted Kerberoasting workflow. The recovered credential provides RDP access to the Windows host, where a readable PowerShell synchronization script exposes credentials for `SANFORD_DAUGHERTY`.

The supplied evidence subsequently identifies `SANFORD_DAUGHERTY` as a Domain Administrator, completing the documented compromise path.

### Executive assessment

The principal security issues demonstrated by the lab are:

- Excessive information disclosure through anonymous and Guest access.
- Exposure of domain user and service-account information.
- Kerberos service-ticket material susceptible to offline password cracking.
- Password reuse between domain identities.
- Excessive Active Directory object permissions.
- Credentials embedded in readable automation scripts.
- A credential-discovery path leading to Domain Administrator access.

**Overall conclusion:** Active Directory compromise does not always depend on one isolated critical vulnerability. A sequence of identity, authentication, authorization, and secret-management weaknesses can create a viable path from low-privilege access to complete domain compromise.

> **Portfolio publication note:** This public edition intentionally redacts challenge flags, recovered passwords, and complete crackable hashes. It documents the technical reasoning and security impact without publishing answer-only secrets.

---

## 2. Assessment Overview

| Attribute | Details |
|---|---|
| Platform | TryHackMe |
| Room | Operation Endgame |
| Difficulty | Hard |
| Environment | Windows / Microsoft Active Directory |
| Domain | `thm.local` |
| Target role | Domain Controller |
| Primary assessment focus | Active Directory enumeration and privilege escalation |
| Initial access vector | Anonymous and Guest information exposure |
| Credential-access techniques | Kerberoasting, offline cracking, password reuse |
| Privilege-escalation vector | Excessive Active Directory object permissions |
| Lateral movement | Remote Desktop Protocol |
| Final privilege level | Domain Administrator |
| Publication edition | Secrets redacted |

### 2.1 Environment characteristics

The target exposes the services commonly associated with an Active Directory Domain Controller. The assessment therefore focuses on the relationships among domain identities, authentication protocols, directory permissions, service accounts, and privileged credentials.

The lab's security posture is evaluated as a connected attack surface rather than as a collection of unrelated open ports.

### 2.2 Assessment outcome

The supplied evidence supports the following outcome:

- Domain users and directory information were enumerated.
- A service-account Kerberos ticket was recovered and cracked.
- Password reuse enabled authentication as a second domain identity.
- BloodHound identified an exploitable `GenericWrite` relationship.
- Targeted Kerberoasting produced another recoverable credential.
- RDP access enabled local investigation of automation files.
- A script disclosed credentials for a Domain Administrator.
- Administrative access to the domain controller was confirmed in the supplied room material.

---

## 3. Scope and Objectives

### 3.1 Scope

The assessment was conducted against the designated TryHackMe lab target.

The supplied reconnaissance command identifies the target as `10.49.128.29`. Lab addresses can change between sessions, so this address should be treated as an instance-specific value rather than a permanent target address.

All techniques described here are intended for authorized lab environments and systems for which explicit testing permission has been granted.

### 3.2 Objectives

The assessment aimed to:

1. Identify exposed network services and establish the target's likely role.
2. Evaluate anonymous and Guest access.
3. Enumerate domain users and directory information.
4. Assess Kerberos pre-authentication and service-ticket attack surfaces.
5. Determine whether recovered credential material could be cracked offline.
6. Validate whether credentials were reused across domain identities.
7. Map Active Directory relationships and permissions using BloodHound.
8. Investigate the security impact of excessive object-level permissions.
9. Obtain an interactive session using a recovered domain credential.
10. Identify additional credential material exposed through local files.
11. Validate the final privilege level using the supplied evidence.
12. Derive practical defensive recommendations from the attack chain.

### 3.3 Out of scope

This document does not claim to assess production infrastructure, external systems, or other environments beyond the supplied lab evidence. It also does not claim that every identified weakness was independently validated outside the room.

---

## 4. Methodology and Tooling

The assessment follows a progressive methodology: identify the attack surface, enumerate identities, investigate credential-access opportunities, map authorization relationships, validate access, and document the resulting privilege path.

| Phase | Tools | Purpose |
|---|---|---|
| Reconnaissance | RustScan, Nmap | Identify open ports and services |
| Domain enumeration | Kerbrute, LDAP utilities | Discover domain users and naming information |
| RPC and SMB testing | `rpcclient`, `smbclient`, NetExec | Assess anonymous and Guest exposure |
| Kerberos assessment | Impacket, NetExec | Test AS-REP roasting and Kerberoasting paths |
| Offline password recovery | Hashcat | Assess recovered ticket material |
| Relationship analysis | BloodHound, bloodhound-python | Map directory permissions and privilege paths |
| Targeted credential access | `targetedKerberoast.py` | Request a service ticket through the identified permission path |
| Interactive access | FreeRDP | Validate remote desktop access |
| Windows enumeration | PowerShell, Windows command shell | Inspect local scripts and identify credential exposure |

Tool output and command results should always be interpreted in context. A command completing successfully does not, by itself, establish the impact of a finding; the resulting access or security consequence must also be validated.

---

## 5. Reconnaissance and Service Enumeration

### 5.1 Full TCP scan

RustScan is used to identify open TCP ports and invoke Nmap service detection and default scripts.

```bash
rustscan -a 10.49.128.29 --range 1-65535 -- -A -sC -Pn
```

The supplied results identify the following services:

| Port | Service | Assessment relevance |
|---|---|---|
| 53/tcp | DNS | Domain name resolution |
| 80/tcp | HTTP / IIS | Web-service identification |
| 88/tcp | Kerberos | Domain authentication |
| 135/tcp | MSRPC | Windows remote procedure calls |
| 139/tcp | NetBIOS | Legacy Windows networking |
| 389/tcp | LDAP | Directory enumeration |
| 443/tcp | HTTPS | Encrypted web service |
| 445/tcp | SMB | File sharing and Windows authentication |
| 464/tcp | Kerberos password change | Kerberos-related password operations |
| 593/tcp | RPC over HTTP | Windows RPC transport |
| 636/tcp | LDAPS | LDAP over TLS |
| 3268/tcp | Global Catalog LDAP | Forest-wide directory queries |
| 3269/tcp | Global Catalog LDAPS | Encrypted Global Catalog queries |
| 3389/tcp | RDP | Interactive Windows access |
| 9389/tcp | .NET Message Framing | Commonly associated with Active Directory Web Services |
| 47001/tcp | Microsoft HTTPAPI | Windows HTTP service endpoint |
| 49xxx/tcp | Windows RPC | Dynamic RPC endpoints |

The exact dynamic RPC ports depend on the scan output and environment; the source material does not specify every individual port.

### 5.2 Service interpretation

The combination of Kerberos, LDAP, Global Catalog, SMB, and Windows RPC strongly indicates that the target is an Active Directory Domain Controller.

This observation shapes the rest of the assessment. Rather than treating the target as a conventional web server, the investigation prioritizes:

- Domain identity exposure.
- Authentication and credential weaknesses.
- Service-account security.
- Directory object permissions.
- Trust and privilege relationships.
- Potential paths to privileged domain access.

**Assessment result:** Reconnaissance establishes a broad Windows domain attack surface and identifies multiple services relevant to identity enumeration.

---

## 6. DNS and Web Service Assessment

### 6.1 DNS

Port 53 is identified as running Simple DNS Plus. The supplied enumeration does not reveal additional useful subdomains.

DNS provides environmental context, but no successful DNS-based compromise path is established by the supplied evidence.

### 6.2 HTTP and HTTPS

Port 80 exposes Microsoft IIS 10.0 and a default Windows landing page. The supplied material does not identify an application-specific vulnerability on the HTTPS service.

These services help establish the target's operating environment but do not form the principal credential-compromise path described in this assessment.

**Assessment result:** DNS and web-service enumeration provide context; the more consequential findings emerge from Active Directory identity and credential analysis.

---

## 7. Kerberos User Enumeration

Kerbrute is used to identify valid domain principals.

```bash
kerbrute userenum -d thm.local \
  /usr/share/seclists/Usernames/xato-net-10-million-usernames.txt \
  --dc 10.49.128.29 -t 100
```

The supplied output confirms valid principals, including:

```text
guest@thm.local
administrator@thm.local
```

Subsequent LDAP enumeration expands the candidate set to hundreds of user principals.

### 7.1 Security significance

Validated usernames provide inputs for multiple assessment activities:

- AS-REP roasting checks.
- Kerberos service-account investigation.
- Controlled password-reuse validation.
- Authentication testing.
- BloodHound relationship analysis.

User enumeration does not automatically provide authentication or privileged access. Its significance is that it reduces uncertainty and makes subsequent identity-focused testing more targeted.

**Assessment result:** Domain account enumeration establishes a candidate identity set for further authentication and authorization analysis.

---

## 8. RPC Null Authentication

Unauthenticated RPC access is tested using `rpcclient`.

```bash
rpcclient -U "" -N 10.49.128.29
```

The supplied SID and share enumeration attempts return:

```text
NT_STATUS_ACCESS_DENIED
```

### 8.1 Interpretation

The result indicates that the tested RPC operations are denied under the supplied unauthenticated context. It does not establish unrestricted anonymous RPC enumeration.

The distinction matters: successful session establishment and successful access to privileged RPC operations are separate conditions.

**Assessment result:** The tested RPC enumeration operations are restricted, although anonymous authentication boundaries remain relevant to the broader assessment.

---

## 9. SMB and Guest Access

SMB enumeration indicates that the server accepts sessions using empty credentials. Guest access is then tested.

```bash
smbclient //TARGET/ -U 'guest'
```

RID enumeration is also performed:

```bash
nxc smb TARGET -u 'guest' -p '' --rid-brute
```

### 9.1 Security significance

Guest access provides an additional source of domain identity and group information before a normal domain credential is obtained.

When combined with directory information disclosure, this exposure can significantly reduce the effort required to identify service accounts and candidate identities for later authentication testing.

**Assessment result:** Guest access contributes to the initial information-gathering phase and supports the later Kerberos attack path.

---

## 10. LDAP Information Disclosure

Unauthenticated LDAP querying is tested against the domain naming context.

```bash
ldapsearch -x -H ldap://TARGET -b "DC=thm,DC=local"
```

The response confirms the `thm.local` naming context and standard Active Directory directory structures.

User principals are extracted using:

```bash
ldapsearch -x -H ldap://TARGET \
  -b "DC=thm,DC=local" | grep userPrincipalName
```

The supplied results contain hundreds of user entries.

### 10.1 Security significance

Bulk directory information disclosure creates a larger candidate set for authentication testing and exposes organizational identity information.

In this assessment, LDAP-derived user information supports subsequent Kerberos investigation and credential-reuse testing.

**Finding:** Directory information is accessible without the level of authentication restriction expected for a hardened environment.

**Defensive priority:** Review anonymous LDAP behavior, restrict unnecessary directory visibility, and monitor unusual bulk directory queries.

---

## 11. AS-REP Roasting Assessment

AS-REP roasting targets accounts configured not to require Kerberos pre-authentication. Depending on the account's password strength, the returned authentication material may permit offline password guessing.

The supplied username list is tested using Impacket:

```bash
impacket-GetNPUsers thm.local/ \
  -dc-ip TARGET \
  -usersfile users.txt \
  -format hashcat \
  -outputfile hashes.txt
```

The supplied evidence reports that multiple roastable accounts are identified and ticket material is recovered.

However, the selected wordlist does not recover a usable credential from this branch.

### 11.1 Assessment result

AS-REP roasting identifies a configuration weakness, but it is **not the successful credential-acquisition branch** used to complete the documented compromise.

This is an important distinction in professional reporting: a technique may confirm exposure without producing an immediately usable credential.

### 11.2 Defensive recommendations

- Audit accounts that do not require Kerberos pre-authentication.
- Enable pre-authentication wherever operationally appropriate.
- Review exceptions and document any justified business requirements.
- Enforce strong, unique passwords for domain accounts.
- Monitor unusual Kerberos authentication activity.

---

## 12. Guest-Accessible Kerberoasting

Kerberoasting targets service accounts associated with Service Principal Names (SPNs). A user who can request a service ticket may be able to obtain ticket material for offline password analysis.

In this lab, Guest-accessible LDAP information exposes a service principal associated with `CODY_ROY`.

The supplied workflow uses NetExec:

```bash
nxc ldap TARGET \
  -u guest -p '' \
  --kerberoasting output_hash.txt \
  --kdcHost TARGET
```

The resulting material identifies:

```text
CODY_ROY
```

The supplied output contains a Kerberos service-ticket hash using the `$krb5tgs$23$` format.

### 12.1 Security significance

This stage converts directory information exposure into credential-attack material.

The key issue is not simply that a service ticket can be requested. The risk increases when a service account has a weak password that can be recovered through offline guessing.

**Assessment result:** The Guest-accessible Kerberoasting branch produces the ticket material used in the successful credential-recovery path.

---

## 13. Service-Ticket Cracking

Hashcat is used to assess the recovered service-ticket material against a wordlist.

```bash
hashcat -m 13100 cody.hash /usr/share/wordlist/rockyou.txt
```

The supplied source reports successful credential recovery.

The recovered password is intentionally omitted from this public edition.

### 13.1 Impact

A successfully recovered service-account password can enable authentication as that account. The impact depends on the account's permissions, group membership, service relationships, and whether the password has been reused elsewhere.

In this case, the recovered credential becomes the starting point for a broader identity and permission investigation.

**Assessment result:** Offline cracking produces a usable domain credential and advances the compromise beyond unauthenticated enumeration.

---

## 14. BloodHound Collection and Analysis

BloodHound is used to investigate Active Directory relationships and identify potential privilege paths.

The supplied collection workflow is:

```bash
bloodhound-python -c All \
  -u CODY_ROY \
  -p 'REDACTED' \
  -d thm.local
```

The initial graph does not reveal a direct privileged path from the recovered account.

Rather than treating this as the end of the investigation, the assessment proceeds to validate whether the recovered password is reused by other identities.

### 14.1 Why graph analysis matters

Active Directory privilege is determined not only by group membership but also by object-level permissions, delegated control, and relationships between identities and resources.

BloodHound helps visualize these relationships and prioritize paths that warrant validation.

**Assessment result:** The initial collection provides useful relationship data, while subsequent credential-reuse testing reveals a more valuable identity for further analysis.

---

## 15. Password Reuse and Credential Validation

The recovered password is tested against the enumerated user list using Kerbrute's password-spraying functionality.

```bash
kerbrute passwordspray \
  --dc TARGET \
  -d thm.local \
  users.txt \
  "REDACTED"
```

The supplied evidence reports successful authentication for:

```text
CODY_ROY
ZACHARY_HUNT
```

The exact password is omitted.

### 15.1 Security significance

The second authentication result establishes that a password recovered from one identity is also valid for another domain user.

Password reuse allows the impact of one compromised account to extend beyond that account's intended privileges. It also makes privilege analysis more valuable: a reused credential may unlock a different set of directory permissions and access paths.

**Assessment result:** `ZACHARY_HUNT` becomes the identity used to investigate the next privilege-escalation opportunity.

**Defensive priority:** Enforce unique credentials, use managed service identities where appropriate, and investigate cross-account credential reuse after any password compromise.

---

## 16. GenericWrite Attack Path

BloodHound identifies the following relationship:

```text
ZACHARY_HUNT
     |
     | GenericWrite
     v
JERRI_LANCASTER
```

**Figure 01 — BloodHound identifies a `GenericWrite` relationship from `ZACHARY_HUNT` to `JERRI_LANCASTER`.**

![BloodHound showing GenericWrite from Zachary Hunt to Jerri Lancaster](docs/assets/figure-1-bloodhound-genericwrite.png)

### 16.1 Understanding GenericWrite

`GenericWrite` is an Active Directory permission that can allow an identity to modify properties of a target directory object, subject to the effective permissions and object configuration.

Such permissions can become security-critical when they allow an attacker to manipulate an account or service-related property in a way that enables additional credential access.

In this lab, the identified relationship supports the targeted Kerberoasting workflow against `JERRI_LANCASTER`.

### 16.2 Security impact

The permission creates a path that is not necessarily obvious from ordinary group-membership review.

An account that appears to be a standard domain user may still have a consequential relationship with another identity because of delegated object permissions.

**Finding:** Excessive object-level permissions create an exploitable credential-access path.

**Defensive priority:** Review directory ACLs, remove unnecessary write permissions, and validate effective permissions rather than relying solely on group membership.

---

## 17. Targeted Kerberoasting

The identified `GenericWrite` relationship is used in a targeted Kerberoasting workflow.

```bash
targetedKerberoast.py \
  -d thm.local \
  -u ZACHARY_HUNT \
  -p 'REDACTED' \
  --dc-ip TARGET \
  --request-user JERRI_LANCASTER
```

The supplied evidence reports that a Kerberos service-ticket hash is obtained for `JERRI_LANCASTER`.

The complete hash is omitted from the public portfolio edition.

### 17.1 Assessment result

The permission relationship identified in BloodHound provides the basis for targeted credential access against a different domain identity.

This illustrates why directory ACL review is essential: an authorization weakness can create an attack path even when the initiating account is not directly privileged.

---

## 18. Targeted Ticket Cracking

Hashcat is used to assess the captured ticket material.

```bash
hashcat -m 13100 \
  jerri.hash \
  /usr/share/wordlists/rockyou.txt
```

The supplied source reports successful password recovery.

The recovered password is intentionally redacted.

### 18.1 Impact

The recovered credential provides access as `JERRI_LANCASTER`, allowing the investigation to progress from directory-level permission abuse to an interactive Windows session.

**Assessment result:** The targeted Kerberoasting branch produces the credential required for the next stage of the attack chain.

---

## 19. Interactive Access Through RDP

The recovered `JERRI_LANCASTER` credential is used to connect to the Windows target through Remote Desktop Protocol.

The supplied command is:

```bash
xfreerdp3 \
  /u:JERRI_LANCASTER \
  /p:'REDACTED' \
  /v:TARGET \
  +clipboard \
  +drives \
  /drive:share,/root/ \
  /dynamic-resolution
```

The environment restricts access to several Windows interfaces. According to the supplied walkthrough, a command shell can still be opened using:

```text
Windows + R → cmd
```

### 19.1 Assessment result

The credential enables an interactive Windows desktop session.

This is a meaningful transition in the attack chain: the investigation can now examine resources available to the authenticated user, including scripts and files that may contain additional credentials.

### 19.2 Security significance

Remote desktop access can expose a wider range of local resources than a network authentication check alone. Its risk depends on the privileges of the connecting identity, the host's configuration, and the files accessible to that identity.

---

## 20. Local Enumeration and Credential Discovery

During local enumeration, a PowerShell synchronization script is discovered:

```text
C:\Scripts\syncer.ps1
```

The directory is listed using:

```powershell
ls C:\Scripts
```

The script is then inspected:

```powershell
type .\syncer.ps1
```

The supplied source reports that the script contains a domain credential pair associated with an Active Directory synchronization task.

The password is not reproduced in this document.

### 20.1 Security finding

The script functions as a credential store despite being intended for automation.

When a user can read a script containing reusable credentials, file access can become credential access. If the embedded credential belongs to a privileged identity, the impact can extend far beyond the permissions of the account that reads the file.

### 20.2 Assessment result

The script exposes credentials associated with `SANFORD_DAUGHERTY`, providing the next identity for validation.

### 20.3 Defensive recommendations

- Remove plaintext credentials from scripts and configuration files.
- Use Group Managed Service Accounts (gMSAs) where applicable.
- Use an approved secrets-management solution for credentials that must be retrieved by automation.
- Restrict access to scripts containing sensitive operational logic.
- Rotate any credential that has been exposed.
- Audit repositories, scheduled tasks, deployment files, and synchronization scripts for embedded secrets.

---

## 21. Domain Administrator Validation

The recovered `SANFORD_DAUGHERTY` credential is validated over SMB.

```bash
nxc smb TARGET \
  -u 'SANFORD_DAUGHERTY' \
  -p 'REDACTED' \
  --shares
```

The supplied evidence reports successful authentication and access to administrative or standard domain shares, including:

```text
ADMIN$
C$
IPC$
NETLOGON
SYSVOL
```

The presence of these shares alone does not independently prove Domain Administrator membership. The supplied BloodHound evidence provides the additional privilege relationship used in the final validation.

### 21.1 BloodHound evidence

**Figure 02 — BloodHound identifies the Domain Administrator relationship associated with `SANFORD_DAUGHERTY` and the domain controller.**

![BloodHound showing Sanford Daugherty associated with Domain Admins and the domain controller](docs/assets/figure-2-bloodhound-domain-admins.png)

The supplied source states that BloodHound confirms the recovered identity's association with the Domain Admins group and the domain controller.

### 21.2 Final administrative access

The credentials are used for RDP access, and the supplied room material reports administrative access to the Windows host.

The final challenge answer remains redacted.

**Assessment result:** The documented chain reaches Domain Administrator privileges, establishing a domain-compromise outcome within the lab.

---

## 22. End-to-End Attack Chain

The following sequence summarizes the principal compromise path established by the supplied evidence.

```text
Domain Controller Discovery
          |
          v
Anonymous / Guest Exposure
          |
          v
LDAP User Enumeration
          |
          v
AS-REP Roasting Assessment
          |
          | No usable password recovered
          v
Guest-Accessible Kerberoasting
          |
          v
CODY_ROY Ticket Recovery
          |
          v
Offline Password Cracking
          |
          v
Password Reuse
          |
          v
ZACHARY_HUNT
          |
          v
BloodHound Relationship Analysis
          |
          v
GenericWrite Permission
          |
          v
JERRI_LANCASTER
          |
          v
Targeted Kerberoasting
          |
          v
Offline Password Cracking
          |
          v
RDP Access
          |
          v
C:\Scripts\syncer.ps1
          |
          v
SANFORD_DAUGHERTY Credentials
          |
          v
Domain Administrator Validation
          |
          v
Final Administrative Access
```

### 22.1 Attack-chain interpretation

The compromise depends on several weaknesses reinforcing one another:

1. **Identity exposure** supplies a large candidate set of domain principals.
2. **Guest-accessible Kerberos activity** yields service-ticket material.
3. **Weak credential resistance** permits offline password recovery.
4. **Password reuse** expands the compromise to another identity.
5. **Excessive directory permissions** create a targeted credential-access opportunity.
6. **Interactive access** enables inspection of local automation files.
7. **Hardcoded credentials** expose a more privileged domain identity.
8. **Privileged account access** completes the domain-compromise path.

The primary lesson is that Active Directory security must be evaluated as a connected graph of identities, credentials, permissions, and resources.

---

## 23. Security Findings

The following findings are derived from the supplied lab walkthrough. Severity labels are qualitative portfolio assessments, not formal CVSS ratings. A production assessment should validate each finding's scope, exploitability, business impact, and environmental context before assigning a final rating.

### Finding 01 — Anonymous and Guest Access Exposure

**Severity:** High

**Category:** Identity and access control

**Issue**

Anonymous and Guest access provide useful Active Directory enumeration capabilities before a normal domain credential is obtained.

**Evidence**

- Guest SMB session testing.
- RID enumeration.
- Directory information disclosure.

**Security impact**

Unauthenticated or weakly authenticated users may gather domain identity information that supports subsequent Kerberos and password attacks.

**Root cause**

Insufficient restrictions on anonymous or Guest access to directory and Windows networking services.

**Recommendations**

- Disable Guest access where it is not operationally required.
- Restrict anonymous enumeration.
- Review SMB and RPC security settings.
- Limit directory information exposed to unauthenticated clients.
- Monitor unusual enumeration patterns.

---

### Finding 02 — LDAP Information Disclosure

**Severity:** High

**Category:** Directory information exposure

**Issue**

The supplied evidence demonstrates that LDAP queries can retrieve domain and user information without strong authentication.

**Evidence**

- Unauthenticated LDAP query against `DC=thm,DC=local`.
- Extraction of `userPrincipalName` entries.
- A resulting user list containing hundreds of principals.

**Security impact**

The exposed information reduces the effort required to identify candidate accounts for Kerberos-based credential attacks and other authentication testing.

**Root cause**

Directory query permissions or anonymous bind behavior permit excessive information disclosure.

**Recommendations**

- Review anonymous LDAP and unauthenticated query behavior.
- Restrict directory attributes available to unauthenticated clients.
- Monitor high-volume and unusual directory searches.
- Review whether each exposed attribute is required for legitimate anonymous operations.

---

### Finding 03 — Kerberos Service-Ticket Exposure

**Severity:** High

**Category:** Credential access

**Issue**

Guest-accessible directory information supports a Kerberoasting path that obtains a service-ticket hash associated with `CODY_ROY`.

**Evidence**

- Service-account discovery.
- Successful ticket-material retrieval.
- Successful offline password recovery reported by the source.

**Security impact**

Weak service-account passwords may be recovered offline, allowing authentication as the associated account.

**Root cause**

A combination of accessible service-ticket requests and insufficient password strength for the affected service identity.

**Recommendations**

- Audit service accounts and their SPNs.
- Use long, random, unique service-account credentials.
- Prefer gMSAs where appropriate.
- Review legacy encryption dependencies and reduce RC4 use where compatibility permits.
- Monitor unusual service-ticket request patterns.

---

### Finding 04 — Password Reuse Across Domain Identities

**Severity:** High

**Category:** Credential management

**Issue**

A password recovered from `CODY_ROY` is also valid for `ZACHARY_HUNT`.

**Evidence**

The supplied password-spraying results report successful authentication for both identities.

**Security impact**

A credential compromised from one account can be used to access another account with a different permission set, broadening the attacker's available paths.

**Root cause**

The same password is used by multiple domain identities.

**Recommendations**

- Enforce unique credentials across user and service accounts.
- Use managed identities for service workloads where possible.
- Deploy password screening to prevent known-compromised and commonly reused passwords.
- Investigate credential reuse after an account compromise.
- Rotate affected credentials and assess any dependent systems.

---

### Finding 05 — Excessive Active Directory `GenericWrite` Permission

**Severity:** Critical

**Category:** Authorization and privilege escalation

**Issue**

BloodHound identifies a `GenericWrite` relationship from `ZACHARY_HUNT` to `JERRI_LANCASTER`.

**Evidence**

- Supplied BloodHound relationship.
- Subsequent targeted Kerberoasting against `JERRI_LANCASTER`.
- Successful credential recovery reported by the source.

**Security impact**

An account with excessive object-level permissions can influence another identity in ways that enable credential access and further compromise.

**Root cause**

Directory object permissions grant more modification capability than the initiating identity requires.

**Recommendations**

- Audit `GenericWrite`, `GenericAll`, `WriteDACL`, `WriteOwner`, and related effective permissions.
- Remove unnecessary object-level write rights.
- Review delegated administration and inherited ACLs.
- Use BloodHound or equivalent graph analysis to identify risky privilege paths.
- Validate remediation by confirming that the unnecessary relationship no longer exists.

---

### Finding 06 — Hardcoded Domain Credentials in a PowerShell Script

**Severity:** Critical

**Category:** Secrets management

**Issue**

The readable `C:\Scripts\syncer.ps1` file contains credentials associated with `SANFORD_DAUGHERTY`.

**Evidence**

- Script discovery during local enumeration.
- Script inspection with a Windows command shell.
- Subsequent credential validation reported by the source.

**Security impact**

A user with permission to read an automation file can obtain credentials beyond that user's intended privilege level. If the exposed identity is privileged, the result can be a direct path to broader administrative access.

**Root cause**

Reusable credentials are stored in readable script content instead of being obtained through an appropriate identity or secrets-management mechanism.

**Recommendations**

- Remove embedded passwords from scripts.
- Use gMSAs for supported Windows services and scheduled tasks.
- Store necessary secrets in an approved secrets-management system.
- Apply restrictive access controls to automation files.
- Rotate exposed credentials immediately.
- Search other scripts and deployment artifacts for similar exposures.

---

### Finding 07 — Domain Administrator Credential Exposure

**Severity:** Critical

**Category:** Privileged access management

**Issue**

The recovered credentials are associated with a Domain Administrator identity, according to the supplied BloodHound evidence.

**Evidence**

- Successful credential validation.
- Administrative share access reported by the source.
- BloodHound relationship to Domain Admins and the domain controller.
- Reported final administrative access.

**Security impact**

Domain Administrator access can enable broad control over domain identities, systems, and directory resources. In a real environment, this should be treated as a domain-compromise incident.

**Root cause**

A privileged credential is exposed through an accessible automation file and can be reused for interactive authentication.

**Recommendations**

- Separate privileged identities from standard user and service workflows.
- Avoid storing Domain Administrator credentials in routine automation.
- Use least-privilege service identities.
- Restrict and monitor RDP for privileged accounts.
- Apply privileged access workstation practices where appropriate.
- Rotate compromised privileged credentials and investigate their usage.
- Review the domain for persistence or additional unauthorized changes following a suspected compromise.

---

## 24. MITRE ATT&CK Mapping

The following mapping relates the observed activities to relevant MITRE ATT&CK concepts. Technique selection should be verified against the current ATT&CK knowledge base before using this table in a formal detection-engineering deliverable.

| Assessment activity | ATT&CK concept | Relevance |
|---|---|---|
| Domain user enumeration | Account Discovery — T1087 | Identifying valid domain accounts |
| Domain group and permission analysis | Permission Groups Discovery — T1069 | Investigating group membership and privilege relationships |
| Kerberos service-ticket attacks | Steal or Forge Kerberos Tickets: Kerberoasting — T1558.003 | Obtaining service-ticket material for offline cracking |
| Password spraying | Brute Force: Password Spraying — T1110.003 | Testing a candidate password across accounts |
| Credential reuse | Valid Accounts — T1078 | Using recovered credentials to authenticate as another identity |
| Credentials embedded in a script | Unsecured Credentials: Credentials In Files — T1552.001 | Discovering credentials stored in a file |
| Password cracking | Password Cracking — T1110.002 | Recovering passwords from captured authentication material |
| Active Directory permission abuse | Permission Groups Discovery / Account Manipulation concepts, depending on the exact operation | Investigating and abusing directory object permissions |
| RDP access | Remote Services: Remote Desktop Protocol — T1021.001 | Obtaining an interactive session over RDP |

**Mapping note:** The assessment demonstrates credential access, account discovery, and remote-service use. The precise ATT&CK mapping for the `GenericWrite` exploitation step depends on the exact directory modification performed; the supplied source does not establish every low-level operation. Avoid assigning a technique solely because a tool is commonly used for it.

---

## 25. Defensive Recommendations

The findings indicate that remediation must address identity exposure, credential strength, directory permissions, secret storage, and privileged access together.

### 25.1 Identity and authentication

- Disable unnecessary Guest and anonymous access.
- Restrict unauthenticated directory enumeration.
- Enforce strong, unique passwords.
- Prevent password reuse through policy and screening.
- Review service identities and their authentication requirements.
- Investigate compromised credentials across all systems where they may have been reused.

### 25.2 Kerberos hardening

- Audit accounts that do not require Kerberos pre-authentication.
- Review all service accounts and their SPNs.
- Use long, randomly generated service-account passwords.
- Prefer gMSAs where operationally appropriate.
- Reduce legacy Kerberos encryption dependencies when feasible.
- Monitor unusual ticket-request patterns and repeated authentication failures.

### 25.3 Active Directory permissions

- Review effective permissions on users, groups, computers, and other directory objects.
- Audit dangerous rights such as `GenericWrite`, `GenericAll`, `WriteDACL`, and `WriteOwner`.
- Remove unnecessary delegated permissions.
- Review inherited ACLs and legacy delegation.
- Conduct regular attack-path analysis using BloodHound or an equivalent tool.
- Verify that remediation breaks the identified privilege path.

### 25.4 Secrets and automation

- Never embed reusable passwords in readable scripts.
- Use managed service identities or a dedicated secrets-management platform.
- Restrict access to synchronization scripts, scheduled-task definitions, and deployment artifacts.
- Scan repositories and automation files for exposed credentials.
- Rotate credentials immediately after exposure.
- Assess whether the same credential exists in other environments or services.

### 25.5 Privileged access

- Separate standard user accounts from administrative identities.
- Avoid using Domain Administrator accounts for routine automation.
- Restrict RDP to approved hosts and users.
- Apply privileged access workstation practices.
- Monitor privileged authentication and administrative share access.
- Investigate unexpected Domain Admin logons.
- Use appropriate privileged-access controls and incident-response procedures.

### 25.6 Detection and monitoring

Useful monitoring opportunities include:

- Unusual LDAP queries or high-volume directory enumeration.
- Unexpected Kerberos service-ticket request patterns.
- Authentication attempts involving many accounts.
- Successful authentication using credentials previously associated with another identity.
- Changes to sensitive Active Directory object permissions.
- Access to scripts and files that contain secrets.
- RDP sessions initiated by privileged identities from unexpected systems.

Detection rules should be tuned to the environment and validated against normal administrative activity to reduce false positives.

---

## 26. Lessons Learned

### 26.1 Enumeration creates leverage

Guest and directory information exposure may appear low impact when evaluated separately. In combination, these weaknesses provide the identity information needed for credential-access techniques.

### 26.2 Credential security is an identity-wide problem

The recovered password becomes more consequential because it is reused by a second domain identity. Credential compromise must therefore trigger investigation beyond the initially affected account.

### 26.3 Directory ACLs are part of the attack surface

Group membership alone does not explain every privilege path. Object-level permissions can create meaningful control relationships between identities that would otherwise appear unrelated.

### 26.4 Service accounts need deliberate design

Service identities should have strong unique credentials, limited permissions, and appropriate lifecycle management. Managed service accounts can reduce the operational need to maintain reusable passwords manually.

### 26.5 Automation files can become credential repositories

Scripts are often readable by users who need to understand or troubleshoot a workflow. Storing passwords directly in those files turns ordinary read access into a potential credential-disclosure vulnerability.

### 26.6 Privileged credentials require stronger boundaries

Domain Administrator credentials should not be exposed through routine scripts or accessible to standard user workflows. Privileged identities need separate controls, monitoring, and restricted use.

### 26.7 Findings should distinguish exposure from successful exploitation

The AS-REP roasting branch demonstrates a useful reporting principle: identifying a vulnerable configuration does not necessarily mean a credential was recovered. A professional assessment records both the confirmed weakness and the actual result of testing.

---

## 27. Evidence and Artifact Management

Evidence should substantiate each major conclusion without publishing unnecessary secrets.

### Figure 01 — GenericWrite Relationship

![BloodHound evidence showing GenericWrite from Zachary Hunt to Jerri Lancaster](docs/assets/figure-1-bloodhound-genericwrite.png)

*Evidence purpose:* Supports the identified Active Directory object-permission relationship.

### Figure 02 — Domain Administrator Relationship

![BloodHound evidence showing Sanford Daugherty associated with Domain Admins and the domain controller](docs/assets/figure-2-bloodhound-domain-admins.png)

*Evidence purpose:* Supports the reported final privilege relationship.

### Figure 03 — Room Completion

![TryHackMe Operation Endgame room completion evidence](docs/assets/figure-3-room-completion.png)

*Evidence purpose:* Intended to document completion of the lab.

**Artifact verification:** The image references above are based on the filenames in the supplied source. Ensure the corresponding files exist at those paths in the repository before publishing. A reference in Markdown does not establish that an image is present or that it proves the stated conclusion.

### Evidence integrity principles

- Use authentic screenshots and terminal output.
- Preserve sufficient context to understand each result.
- Redact challenge flags, passwords, and other unnecessary secrets.
- Do not fabricate terminal output or create synthetic screenshots that appear to be captured evidence.
- Avoid including unrelated personal information or host details.
- Ensure each figure supports the claim associated with it.

---

## 28. Challenge Flag Protection

```text
Final Flag: [REDACTED]
```

The final challenge answer and recovered passwords are excluded from this public portfolio edition.

This write-up focuses on the security methodology, attack-chain reasoning, impact, and defensive lessons rather than publishing answer-only secrets.

---

## 29. Conclusion

Operation Endgame demonstrates how a series of Active Directory weaknesses can combine into a path from unauthenticated information gathering to Domain Administrator access.

The successful chain begins with Guest-accessible identity information, progresses through Kerberos service-ticket recovery and password reuse, and then leverages an excessive `GenericWrite` permission to obtain another domain credential. Interactive access exposes a PowerShell script containing additional credentials, ultimately leading to a Domain Administrator identity.

The central defensive lesson is that domain security depends on the interaction of multiple control layers:

- Authentication must resist credential guessing and reuse.
- Directory information must be exposed only where necessary.
- Active Directory permissions must follow least privilege.
- Service identities must be managed securely.
- Automation must not contain reusable privileged credentials.
- Administrative access must be isolated, restricted, and monitored.

**Final attack-chain summary:**

**Identity enumeration → Kerberos credential access → password reuse → ACL analysis → targeted Kerberoasting → RDP access → script-based credential discovery → Domain Administrator validation**

From a professional assessment perspective, the most important outcome is not simply that administrative access was achieved. It is that the path can be explained, the contributing weaknesses can be tied to evidence, and each weakness can be translated into a practical defensive action.

---

## 30. References

The following resources provide background for the technologies and security concepts covered in this assessment.

- [TryHackMe](https://tryhackme.com/) — Authorized hands-on cybersecurity training.
- [Microsoft Learn: Active Directory Domain Services](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/) — Active Directory architecture and administration.
- [Microsoft Learn: Kerberos authentication overview](https://learn.microsoft.com/en-us/windows-server/security/kerberos/kerberos-authentication-overview) — Windows Kerberos authentication.
- [MITRE ATT&CK](https://attack.mitre.org/) — Adversary tactics and techniques.
- [BloodHound](https://github.com/SpecterOps/BloodHound) — Active Directory relationship and attack-path analysis.
- [Impacket](https://github.com/fortra/impacket) — Network protocol and Windows authentication tooling.
- [Kerbrute](https://github.com/ropnop/kerbrute) — Kerberos user enumeration and authentication testing.
- [Hashcat](https://hashcat.net/hashcat/) — Password recovery and hash-analysis utility.
- [NetExec](https://github.com/Pennyw0rth/NetExec) — Network service assessment and authentication testing.
- [FreeRDP](https://github.com/FreeRDP/FreeRDP) — Remote Desktop Protocol client implementation.

---

**Author:** Anurag Ravankar  
**Platform:** TryHackMe  
**Room:** Operation Endgame  
**Purpose:** Authorized lab research, Active Directory security learning, and cybersecurity portfolio documentation

*This document describes activity within an authorized training environment. Techniques must only be used on systems for which explicit authorization has been granted.*
