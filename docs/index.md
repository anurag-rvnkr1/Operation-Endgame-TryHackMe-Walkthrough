<div class="ctf-hero">

  <h1>Operation Endgame</h1>

  <p>
    An Active Directory security assessment demonstrating how directory information disclosure, Kerberos weaknesses, password reuse, excessive object permissions, and exposed automation credentials can combine into a path from low-privilege domain access to Domain Administrator.
  </p>

  <div class="ctf-badges">
    <span class="ctf-badge">TryHackMe</span>
    <span class="ctf-badge">Hard</span>
    <span class="ctf-badge">Windows</span>
    <span class="ctf-badge">Active Directory</span>
    <span class="ctf-badge">Kerberos</span>
    <span class="ctf-badge">Privilege Escalation</span>
  </div>

</div>

> **Publication notice:** This portfolio edition redacts challenge flags, recovered passwords, and complete crackable hashes. Account names and technical evidence are retained where needed to explain the documented attack path. The assessment is limited to the supplied TryHackMe lab evidence.

## Documentation Map

<div class="ctf-toc">
<div class="ctf-toc-title">Documentation Map</div>
<ul>
<li><a href="#mission-and-executive-summary">Mission and Executive Summary</a></li>
<li><a href="#quick-overview">Quick Overview</a></li>
<li><a href="#assessment-objectives-and-scope">Assessment Objectives and Scope</a></li>
<li><a href="#methodology-and-tooling">Methodology and Tooling</a></li>
<li><a href="#attack-surface-overview">Attack-Surface Overview</a></li>
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
<li><a href="#targeted-ticket-cracking">Targeted Ticket Cracking</a></li>
<li><a href="#interactive-access-through-rdp">Interactive Access Through RDP</a></li>
<li><a href="#local-enumeration-and-credential-discovery">Local Enumeration and Credential Discovery</a></li>
<li><a href="#domain-administrator-validation">Domain Administrator Validation</a></li>
<li><a href="#end-to-end-attack-chain">End-to-End Attack Chain</a></li>
<li><a href="#security-findings">Security Findings</a></li>
<li><a href="#mitre-attck-mapping">MITRE ATT&CK Mapping</a></li>
<li><a href="#defensive-recommendations">Defensive Recommendations</a></li>
<li><a href="#lessons-learned">Lessons Learned</a></li>
<li><a href="#evidence-and-artifact-management">Evidence and Artifact Management</a></li>
<li><a href="#challenge-flag-protection">Challenge Flag Protection</a></li>
<li><a href="#conclusion">Conclusion</a></li>
<li><a href="#references">References</a></li>
<li><a href="#responsible-use">Responsible Use</a></li>
</ul>
</div>

---

## Mission and Executive Summary

Operation Endgame is a hard-difficulty Active Directory lab focused on identifying and validating the relationships between identity exposure, Kerberos authentication, credential security, directory permissions, and privileged access.

The assessment begins with network reconnaissance of a Windows Domain Controller. The identified attack surface includes DNS, HTTP/S, Kerberos, RPC, LDAP/LDAPS, SMB, Global Catalog, and Remote Desktop Protocol (RDP).

Anonymous and Guest access provide opportunities for domain identity and directory enumeration. The resulting information supports further investigation of Kerberos attack surfaces. The AS-REP roasting branch identifies accounts with a potentially vulnerable pre-authentication configuration, although the supplied wordlist does not recover a usable password through that branch.

A more successful credential-access path emerges through Guest-accessible Kerberoasting. Service-ticket material associated with `CODY_ROY` is recovered and cracked. The recovered password also authenticates as `ZACHARY_HUNT`, demonstrating password reuse between distinct domain identities.

BloodHound analysis then identifies a `GenericWrite` relationship from `ZACHARY_HUNT` to `JERRI_LANCASTER`. The documented workflow uses this permission relationship to pursue targeted Kerberoasting. A subsequently recovered credential provides RDP access to the Windows host, where a readable PowerShell synchronization script exposes credentials associated with `SANFORD_DAUGHERTY`.

The supplied room material reports that the final identity is associated with Domain Admins and that administrative access to the target is achieved.

### Executive assessment

The principal weaknesses demonstrated by this lab are:

- Excessive domain information exposure through anonymous and Guest access.
- Exposure of user and service-account information through directory enumeration.
- Kerberos service-ticket material susceptible to offline password recovery.
- Password reuse across domain identities.
- Excessive Active Directory object permissions.
- Reusable credentials stored in a readable PowerShell script.
- A credential-discovery path leading to a privileged domain identity.

<div class="key-finding">

<div class="key-finding-title">Primary Security Outcome</div>

The supplied evidence describes a chained Active Directory compromise path from unauthenticated information gathering through multiple domain identities to Domain Administrator access. The result illustrates how weaknesses across authentication, authorization, and secret management can compound into a domain-wide security risk.

</div>

The most important defensive lesson is that Active Directory security must be assessed as a connected system of identities, credentials, permissions, and resources. Correcting one isolated weakness may not eliminate other viable paths to privileged access.

## Quick Overview

<div class="ctf-card-grid">

  <div class="ctf-card">
    <div class="ctf-card-title">Platform</div>
    <div class="ctf-card-value">TryHackMe</div>
  </div>

  <div class="ctf-card">
    <div class="ctf-card-title">Difficulty</div>
    <div class="ctf-card-value">Hard</div>
  </div>

  <div class="ctf-card">
    <div class="ctf-card-title">Environment</div>
    <div class="ctf-card-value">Windows / Active Directory</div>
  </div>

  <div class="ctf-card">
    <div class="ctf-card-title">Domain</div>
    <div class="ctf-card-value">`thm.local`</div>
  </div>

  <div class="ctf-card">
    <div class="ctf-card-title">Target Role</div>
    <div class="ctf-card-value">Domain Controller</div>
  </div>

  <div class="ctf-card">
    <div class="ctf-card-title">Initial Exposure</div>
    <div class="ctf-card-value">Anonymous / Guest Enumeration</div>
  </div>

  <div class="ctf-card">
    <div class="ctf-card-title">Credential Access</div>
    <div class="ctf-card-value">Kerberoasting / Password Reuse</div>
  </div>

  <div class="ctf-card">
    <div class="ctf-card-title">Reported Outcome</div>
    <div class="ctf-card-value">Domain Administrator Access</div>
  </div>

</div>

### Key technical capabilities demonstrated

The supplied walkthrough documents experience with network and service enumeration, Windows domain identity discovery, Kerberos attack-surface assessment, offline password recovery, Active Directory relationship analysis, object-permission investigation, RDP access, Windows command-line enumeration, and credential-exposure analysis.

The successful credential path depends on the combined effect of several findings rather than a single isolated vulnerability.

## Assessment Objectives and Scope

### Objectives

The assessment aims to:

1. Identify exposed network services and establish the target's likely role.
2. Evaluate anonymous and Guest access.
3. Enumerate domain users and directory information.
4. Assess Kerberos pre-authentication and service-ticket attack surfaces.
5. Determine whether recovered authentication material can be cracked offline.
6. Validate whether credentials are reused across domain identities.
7. Map Active Directory relationships and permissions using BloodHound.
8. Investigate the security impact of excessive directory object permissions.
9. Obtain an interactive session using a recovered domain credential.
10. Identify additional credential material exposed through local files.
11. Validate the reported final privilege level using the supplied evidence.
12. Derive practical defensive recommendations from the documented attack path.

### Scope

| Attribute | Assessment details |
|---|---|
| Platform | TryHackMe |
| Room | Operation Endgame |
| Domain | `thm.local` |
| Target address in supplied scan | `10.49.128.29` |
| Target role | Domain Controller |
| Assessment type | Authorized lab-based security assessment |
| Publication edition | Challenge secrets redacted |

The target address is an instance-specific lab address and may change between sessions.

This document covers the designated training target and the evidence included in the source material. It does not claim that the same weaknesses exist in a production environment or in infrastructure outside the lab.

### Assessment boundaries

The findings describe the supplied walkthrough's reported observations and outcomes. Where the source establishes exposure but does not document a complete validation procedure, the finding is described accordingly rather than treated as independently verified production evidence.

## Methodology and Tooling

The assessment follows a progressive workflow: map the external attack surface, enumerate domain identities, assess Kerberos credential-access opportunities, investigate permission relationships, validate recovered credentials, examine resources available to an authenticated user, and document the final privilege path.

| Phase | Tools documented in the source | Purpose |
|---|---|---|
| Network reconnaissance | RustScan, Nmap | Identify exposed TCP ports and services |
| Domain user enumeration | Kerbrute, LDAP utilities | Discover domain identities and naming information |
| RPC and SMB assessment | `rpcclient`, `smbclient`, NetExec | Test anonymous and Guest access |
| Kerberos assessment | Impacket, NetExec | Assess AS-REP roasting and Kerberoasting paths |
| Offline password recovery | Hashcat | Evaluate captured ticket material |
| Relationship analysis | BloodHound, `bloodhound-python` | Investigate object permissions and potential privilege paths |
| Targeted credential access | `targetedKerberoast.py` | Follow the documented permission-based attack path |
| Interactive access | FreeRDP | Validate RDP access |
| Windows enumeration | PowerShell, Windows command shell | Inspect local scripts and identify credential exposure |

A tool reporting success does not, by itself, establish the impact of a security issue. The relevant result must be interpreted in context and tied to the observed access, permission relationship, or credential exposure.

## Attack-Surface Overview

The initial reconnaissance identifies services consistent with a Windows Active Directory Domain Controller.

| Port | Service | Assessment relevance |
|---:|---|---|
| 53/tcp | DNS | Domain name resolution |
| 80/tcp | HTTP / IIS | Web-service identification |
| 88/tcp | Kerberos | Domain authentication |
| 135/tcp | MSRPC | Windows remote procedure calls |
| 139/tcp | NetBIOS | Legacy Windows networking |
| 389/tcp | LDAP | Directory queries and enumeration |
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
| 49xxx/tcp | Windows RPC | Dynamic RPC endpoints reported in the source |

The source does not enumerate every individual dynamic RPC port. The `49xxx/tcp` entry therefore represents the dynamic-port range described in the supplied material rather than one confirmed port number.

The most consequential attack surface is not the number of open ports alone. It is the interaction between identity services, directory queries, Kerberos ticket requests, credentials, and object permissions.

## Reconnaissance and Service Enumeration

### Full TCP scan

RustScan is used to discover open TCP ports and invoke Nmap service detection and default scripts.

```bash
rustscan -a 10.49.128.29 --range 1-65535 -- -A -sC -Pn
```

**Purpose:** Scan the target's full TCP port range and request detailed service identification.

**Assessment value:** The results identify a broad Windows domain service footprint, directing later testing toward Active Directory enumeration and authentication mechanisms.

### Service interpretation

The combination of Kerberos, LDAP, Global Catalog, SMB, and Windows RPC strongly indicates an Active Directory Domain Controller.

This observation informs the next stages of the assessment, which prioritize:

- Domain identity exposure.
- Authentication and credential weaknesses.
- Service-account security.
- Directory object permissions.
- Relationships between ordinary and privileged identities.
- Potential paths to administrative domain access.

**Result:** Reconnaissance establishes the Windows domain attack surface and identifies the services relevant to the documented credential-compromise path.

## DNS and Web Service Assessment

### DNS

Port 53 is identified in the supplied material as running Simple DNS Plus. The provided enumeration does not reveal additional useful subdomains.

DNS therefore contributes environmental context, but the source does not establish a successful DNS-based compromise path.

### HTTP and HTTPS

Port 80 exposes Microsoft IIS 10.0 and a default Windows landing page. The supplied material does not establish an application-specific vulnerability on the HTTPS service.

The web services help characterize the target, but they do not form the principal credential-compromise path documented in this assessment.

**Result:** DNS and web enumeration provide contextual information. The more consequential findings emerge from domain identity and credential analysis.

## Kerberos User Enumeration

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

### Security significance

Validated usernames provide inputs for later assessment activities, including:

- AS-REP roasting checks.
- Kerberos service-account investigation.
- Controlled password-reuse validation.
- Authentication testing.
- BloodHound relationship analysis.

User enumeration does not automatically grant authentication or privileged access. Its significance is that it reduces uncertainty and supports more targeted investigation of domain authentication and authorization.

**Result:** Domain account enumeration establishes the identity set used for subsequent testing.

## RPC Null Authentication

Unauthenticated RPC access is tested using `rpcclient`.

```bash
rpcclient -U "" -N 10.49.128.29
```

The supplied SID and share enumeration attempts return:

```text
NT_STATUS_ACCESS_DENIED
```

### Interpretation

The output indicates that the tested RPC operations are denied in the supplied unauthenticated context. It does not demonstrate unrestricted anonymous RPC enumeration.

Successful session establishment and permission to execute specific RPC operations are separate conditions. This distinction is important when assessing the actual extent of unauthenticated access.

**Result:** The tested RPC enumeration operations are restricted, while other identity-exposure paths remain relevant to the broader assessment.

## SMB and Guest Access

SMB enumeration indicates that the server accepts sessions using empty credentials. Guest access is tested using:

```bash
smbclient //TARGET/ -U 'guest'
```

RID enumeration is also performed:

```bash
nxc smb TARGET -u 'guest' -p '' --rid-brute
```

### Security significance

Guest access supplies additional domain identity and group information before a normal domain credential is obtained.

Combined with directory information disclosure, this reduces the effort required to identify service accounts and candidate identities for subsequent authentication testing.

**Result:** Guest access contributes to initial information gathering and supports the later Kerberos investigation.

## LDAP Information Disclosure

Unauthenticated LDAP querying is tested against the domain naming context.

```bash
ldapsearch -x -H ldap://TARGET -b "DC=thm,DC=local"
```

The supplied response confirms the `thm.local` naming context and standard Active Directory directory structures.

User principals are extracted using:

```bash
ldapsearch -x -H ldap://TARGET \
  -b "DC=thm,DC=local" | grep userPrincipalName
```

The source reports hundreds of user entries.

### Security significance

Bulk directory information disclosure creates a larger candidate set for authentication testing and reveals domain identity information.

In this assessment, LDAP-derived user information supports Kerberos investigation and credential-reuse testing.

<div class="key-finding">

<div class="key-finding-title">Finding — Directory Information Exposure</div>

The supplied evidence shows that LDAP queries can retrieve domain and user information without the authentication restriction expected in a hardened environment. The precise set of attributes exposed and the operational reason for the access should be confirmed when validating this behavior in a real assessment.

</div>

### Defensive priority

Review anonymous LDAP behavior, restrict unnecessary directory visibility, and monitor unusual bulk directory queries.

## AS-REP Roasting Assessment

AS-REP roasting targets accounts configured not to require Kerberos pre-authentication. Depending on the password strength and the recovered authentication material, an attacker may be able to perform offline password guessing.

The supplied username list is tested using Impacket:

```bash
impacket-GetNPUsers thm.local/ \
  -dc-ip TARGET \
  -usersfile users.txt \
  -format hashcat \
  -outputfile hashes.txt
```

The source reports that multiple roastable accounts are identified and ticket material is recovered.

However, the selected wordlist does not recover a usable credential from this branch.

### Assessment result

AS-REP roasting identifies a configuration weakness, but it is **not the successful credential-acquisition branch** used to complete the documented compromise.

This distinction is important in professional reporting: confirming an exposed configuration does not necessarily mean that a password has been recovered or that the same path has led to access.

### Defensive recommendations

- Audit accounts that do not require Kerberos pre-authentication.
- Enable pre-authentication wherever operationally appropriate.
- Document and review exceptions that have a justified business requirement.
- Enforce strong, unique passwords for domain accounts.
- Monitor unusual Kerberos authentication activity.

## Guest-Accessible Kerberoasting

Kerberoasting targets service accounts associated with Service Principal Names (SPNs). A principal that can request a service ticket may obtain ticket material suitable for offline password analysis.

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

The source reports a Kerberos service-ticket hash using the `$krb5tgs$23$` format.

### Security significance

This stage turns directory information exposure into credential-attack material.

The security impact depends on the service account's password strength, the ticket's encryption type, the account's privileges, and whether its password is used elsewhere.

**Result:** The Guest-accessible Kerberoasting branch produces the ticket material used in the successful credential-recovery path.

## Service-Ticket Cracking

Hashcat is used to evaluate the recovered service-ticket material against a wordlist.

```bash
hashcat -m 13100 cody.hash /usr/share/wordlist/rockyou.txt
```

The supplied source reports successful credential recovery. The recovered password is intentionally omitted from this public edition.

### Impact

A recovered service-account password may enable authentication as that account. The practical impact depends on the account's permissions, group membership, service relationships, and whether the credential has been reused by other identities.

In this case, the recovered credential becomes the starting point for further identity and permission analysis.

**Result:** Offline password recovery produces a usable domain credential and advances the assessment beyond unauthenticated enumeration.

## BloodHound Collection and Analysis

BloodHound is used to investigate Active Directory relationships and identify potential privilege paths.

The supplied collection workflow is:

```bash
bloodhound-python -c All \
  -u CODY_ROY \
  -p 'REDACTED' \
  -d thm.local
```

The initial graph does not reveal a direct privileged path from the recovered account. The investigation therefore continues by examining whether the credential is reused by other identities.

### Why graph analysis matters

Active Directory privilege is determined not only by group membership, but also by object-level permissions, delegated control, and relationships between identities and resources.

BloodHound helps visualize these relationships and prioritize paths that warrant further validation.

**Result:** The initial collection supplies relationship data. Subsequent credential-reuse testing reveals a more useful identity for additional analysis.

## Password Reuse and Credential Validation

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

### Security significance

The second authentication result establishes that a password recovered from one identity is also valid for another domain user.

Password reuse allows the impact of one compromised account to extend beyond that account's intended privileges. This makes further authorization analysis important: a reused credential may expose another set of directory permissions and access paths.

**Result:** `ZACHARY_HUNT` becomes the identity used to investigate the next privilege-escalation opportunity.

### Defensive priority

Enforce unique credentials, use managed service identities where appropriate, and investigate cross-account credential reuse after any password compromise.

## GenericWrite Attack Path

BloodHound identifies the following relationship:

```text
ZACHARY_HUNT
     |
     | GenericWrite
     v
JERRI_LANCASTER
```

**Figure 01 — BloodHound relationship reported in the supplied source.** The original source identifies this relationship but does not include an explicit Markdown image path for the screenshot.

### Understanding `GenericWrite`

`GenericWrite` is an Active Directory permission that can allow an identity to modify properties of a target directory object, subject to the effective permissions and object configuration.

This permission can become security-critical when it permits modification of an account or service-related property in a way that enables further credential access.

In this lab, the identified relationship supports the documented targeted Kerberoasting workflow against `JERRI_LANCASTER`.

### Security impact

Object-level permissions can create privilege paths that are not obvious from ordinary group-membership review. An account that appears to be a standard domain user may still control a security-relevant property of another identity through delegated permissions.

<div class="key-finding">

<div class="key-finding-title">Finding — Excessive Object Permissions</div>

The supplied BloodHound evidence identifies a `GenericWrite` relationship from `ZACHARY_HUNT` to `JERRI_LANCASTER`. The walkthrough then uses this relationship in a targeted credential-access workflow. The precise underlying directory modification should be recorded from the original operation when producing a formal assessment report.

</div>

### Defensive priority

Review directory ACLs, remove unnecessary write permissions, and validate effective permissions rather than relying solely on group membership.

## Targeted Kerberoasting

The identified `GenericWrite` relationship is used in a targeted Kerberoasting workflow.

```bash
targetedKerberoast.py \
  -d thm.local \
  -u ZACHARY_HUNT \
  -p 'REDACTED' \
  --dc-ip TARGET \
  --request-user JERRI_LANCASTER
```

The supplied evidence reports that Kerberos service-ticket material is obtained for `JERRI_LANCASTER`. The complete hash is omitted from the public portfolio edition.

### Assessment significance

The relationship identified through BloodHound supplies the basis for targeted credential access against another domain identity.

This illustrates why directory ACL review is essential. An authorization weakness can create a path to further credential access even when the initiating account is not directly privileged.

**Result:** The targeted Kerberoasting branch produces ticket material for offline password analysis.

## Targeted Ticket Cracking

Hashcat is used to evaluate the captured ticket material:

```bash
hashcat -m 13100 \
  jerri.hash \
  /usr/share/wordlists/rockyou.txt
```

The supplied source reports successful password recovery. The recovered password is redacted.

### Impact

The recovered credential provides access as `JERRI_LANCASTER`, allowing the investigation to progress from directory-level permission abuse to an interactive Windows session.

**Result:** The targeted Kerberoasting branch supplies the credential used for the next stage of the documented attack chain.

## Interactive Access Through RDP

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

The source reports that access to several Windows interfaces is restricted. According to the supplied walkthrough, a command shell can still be opened using:

```text
Windows + R → cmd
```

### Assessment result

The credential enables an interactive Windows desktop session.

This is a meaningful transition in the attack chain: the authenticated user can investigate resources available on the host, including scripts and files that may expose additional credentials.

### Security significance

RDP access can expose a wider range of local resources than a network authentication check alone. The risk depends on the connecting identity's permissions, the host configuration, and the resources readable by that identity.

## Local Enumeration and Credential Discovery

During local enumeration, the following PowerShell synchronization script is discovered:

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

The source reports that the script contains a domain credential pair associated with an Active Directory synchronization task. The password is not reproduced in this document.

### Security finding

The script functions as a credential store despite being intended for automation.

When a user can read a script containing reusable credentials, file access can become credential access. If the embedded credential belongs to a privileged identity, the impact can extend far beyond the permissions of the user who reads the file.

### Assessment result

The script exposes credentials associated with `SANFORD_DAUGHERTY`, providing the next identity for validation.

### Defensive recommendations

- Remove plaintext credentials from scripts and configuration files.
- Use Group Managed Service Accounts (gMSAs) where applicable.
- Use an approved secrets-management solution for credentials required by automation.
- Restrict access to scripts containing sensitive operational logic.
- Rotate exposed credentials.
- Audit repositories, scheduled tasks, deployment files, and synchronization scripts for embedded secrets.

## Domain Administrator Validation

The recovered `SANFORD_DAUGHERTY` credential is validated over SMB:

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

### BloodHound evidence

**Figure 02 — BloodHound relationship associated with `SANFORD_DAUGHERTY`, Domain Admins, and the domain controller, as described in the source material.**

The supplied source states that BloodHound confirms the recovered identity's association with the Domain Admins group and the domain controller.

### Final administrative access

The credentials are subsequently used for RDP access, and the supplied room material reports administrative access to the Windows host.

The final challenge answer remains redacted.

<div class="key-finding">

<div class="key-finding-title">Reported Final Outcome</div>

The supplied walkthrough reports that the chain reaches Domain Administrator privileges. This conclusion is based on the source's described BloodHound relationship and final access result; the public edition does not reproduce privileged credentials or the final challenge answer.

</div>

## End-to-End Attack Chain

The following sequence summarizes the principal compromise path documented by the source.

<div class="attack-chain">
  <div class="attack-step">Domain Controller Discovery</div>
  <div class="attack-arrow">→</div>
  <div class="attack-step">Guest / LDAP Enumeration</div>
  <div class="attack-arrow">→</div>
  <div class="attack-step">Kerberos Ticket Recovery</div>
  <div class="attack-arrow">→</div>
  <div class="attack-step">Offline Password Cracking</div>
  <div class="attack-arrow">→</div>
  <div class="attack-step">Password Reuse</div>
  <div class="attack-arrow">→</div>
  <div class="attack-step">GenericWrite Relationship</div>
  <div class="attack-arrow">→</div>
  <div class="attack-step">Targeted Kerberoasting</div>
  <div class="attack-arrow">→</div>
  <div class="attack-step">RDP Access</div>
  <div class="attack-arrow">→</div>
  <div class="attack-step">Script Credential Exposure</div>
  <div class="attack-arrow">→</div>
  <div class="attack-step">Domain Administrator</div>
</div>

### Attack-chain interpretation

The successful path depends on multiple weaknesses reinforcing one another:

1. **Identity exposure** provides candidate domain principals.
2. **Guest-accessible Kerberos activity** yields service-ticket material.
3. **Insufficient credential resistance** permits offline password recovery.
4. **Password reuse** expands access to another identity.
5. **Excessive directory permissions** create an additional credential-access opportunity.
6. **Interactive access** enables investigation of local automation files.
7. **Hardcoded credentials** expose a more privileged domain identity.
8. **Privileged account access** completes the reported domain-compromise path.

The AS-REP roasting branch is a separate assessment result: it identifies roastable accounts, but the supplied wordlist does not recover a usable password through that branch.

The primary lesson is that Active Directory security must be evaluated as a connected graph of identities, credentials, permissions, and resources.

## Security Findings

The findings below are based on the supplied lab walkthrough. Severity labels are preliminary qualitative assessment judgments rather than formal CVSS ratings. They should not be interpreted as independently validated production ratings.

| ID | Finding | Source-reported evidence | Preliminary severity |
|---|---|---|---|
| F-01 | Anonymous and Guest access exposure | Guest SMB testing, RID enumeration, and directory information exposure | High |
| F-02 | LDAP information disclosure | Unauthenticated LDAP query and extraction of user-principal information | High |
| F-03 | Kerberos service-ticket exposure | Guest-accessible Kerberoasting followed by reported offline password recovery | High |
| F-04 | Password reuse across domain identities | Recovered password reported as valid for `CODY_ROY` and `ZACHARY_HUNT` | High |
| F-05 | Excessive `GenericWrite` permission | BloodHound relationship and subsequent targeted Kerberoasting workflow | Critical |
| F-06 | Credentials embedded in PowerShell script | Readable `C:\Scripts\syncer.ps1` reported to contain reusable credentials | Critical |
| F-07 | Domain Administrator credential exposure | Credential validation, reported Domain Admins relationship, and final access | Critical |

The source does not include CVSS vectors, a production impact analysis, or evidence from an independent assessment. The ratings above are retained as qualitative portfolio assessments from the supplied material; a formal engagement should independently validate exploitability, affected assets, business impact, and remediation status.

### F-01 — Anonymous and Guest Access Exposure

**Category:** Identity and access control  
**Preliminary severity:** High

**Issue**

Anonymous and Guest access provide useful Active Directory enumeration capabilities before a normal domain credential is obtained.

**Evidence in the supplied source**

- Guest SMB session testing.
- RID enumeration.
- Directory information disclosure.

**Potential security impact**

An unauthenticated or weakly authenticated user may gather domain identity information that supports subsequent Kerberos and password attacks.

**Likely control weakness**

Anonymous or Guest access to Windows networking and directory services may be broader than necessary.

**Recommendations**

- Disable Guest access where it is not operationally required.
- Restrict anonymous enumeration.
- Review SMB and RPC security settings.
- Limit directory information exposed to unauthenticated clients.
- Monitor unusual identity-enumeration patterns.

### F-02 — LDAP Information Disclosure

**Category:** Directory information exposure  
**Preliminary severity:** High

**Issue**

The supplied evidence demonstrates that LDAP queries can retrieve domain and user information without strong authentication.

**Evidence in the supplied source**

- Unauthenticated query against `DC=thm,DC=local`.
- Extraction of `userPrincipalName` entries.
- A reported result set containing hundreds of user entries.

**Potential security impact**

Exposed directory data reduces the effort required to identify candidate accounts for Kerberos-based credential attacks and other authentication testing.

**Likely control weakness**

Directory query permissions or unauthenticated query behavior may permit excessive information disclosure.

**Recommendations**

- Review anonymous LDAP and unauthenticated query behavior.
- Restrict directory attributes available to unauthenticated clients where feasible.
- Monitor high-volume and unusual directory searches.
- Verify whether exposed attributes are required for legitimate anonymous operations.

### F-03 — Kerberos Service-Ticket Exposure

**Category:** Credential access  
**Preliminary severity:** High

**Issue**

Guest-accessible directory information supports a Kerberoasting path that obtains service-ticket material associated with `CODY_ROY`.

**Evidence in the supplied source**

- Service-account discovery.
- Successful ticket-material retrieval.
- Successful offline password recovery reported by the walkthrough.

**Potential security impact**

A weak service-account password may be recovered offline, potentially enabling authentication as the associated identity.

**Likely contributing factors**

- Service-ticket material available to a low-privilege principal.
- Insufficient password strength for the affected service identity.

**Recommendations**

- Audit service accounts and their SPNs.
- Use long, random, unique service-account credentials.
- Prefer gMSAs where appropriate.
- Review legacy Kerberos encryption dependencies and reduce RC4 use where compatibility permits.
- Monitor unusual service-ticket request patterns.

### F-04 — Password Reuse Across Domain Identities

**Category:** Credential management  
**Preliminary severity:** High

**Issue**

A password recovered from `CODY_ROY` is also reported as valid for `ZACHARY_HUNT`.

**Evidence in the supplied source**

The password-spraying results report successful authentication for both identities.

**Potential security impact**

A credential compromised from one account can provide access to another account with a different permission set, broadening the available attack paths.

**Likely control weakness**

The same password is used across multiple domain identities.

**Recommendations**

- Enforce unique credentials across user and service accounts.
- Use managed identities for service workloads where possible.
- Apply password screening to block commonly reused and known-compromised passwords.
- Investigate credential reuse after an account compromise.
- Rotate affected credentials and assess dependent systems.

### F-05 — Excessive Active Directory `GenericWrite` Permission

**Category:** Authorization and privilege escalation  
**Preliminary severity:** Critical

**Issue**

BloodHound identifies a `GenericWrite` relationship from `ZACHARY_HUNT` to `JERRI_LANCASTER`.

**Evidence in the supplied source**

- Documented BloodHound relationship.
- Subsequent targeted Kerberoasting against `JERRI_LANCASTER`.
- Successful credential recovery reported by the source.

**Potential security impact**

Excessive object-level permissions may allow an account to manipulate another identity in a way that supports further credential access.

**Likely control weakness**

Directory object permissions grant more modification capability than the initiating identity requires.

**Recommendations**

- Audit `GenericWrite`, `GenericAll`, `WriteDACL`, `WriteOwner`, and other consequential effective permissions.
- Remove unnecessary object-level write rights.
- Review delegated administration and inherited ACLs.
- Use BloodHound or equivalent graph analysis to identify risky privilege paths.
- Verify remediation by confirming that the unnecessary relationship has been removed.

### F-06 — Hardcoded Domain Credentials in a PowerShell Script

**Category:** Secrets management  
**Preliminary severity:** Critical

**Issue**

The readable `C:\Scripts\syncer.ps1` file is reported to contain credentials associated with `SANFORD_DAUGHERTY`.

**Evidence in the supplied source**

- Script discovery during local enumeration.
- Inspection using a Windows command shell.
- Subsequent credential validation reported by the walkthrough.

**Potential security impact**

A user permitted to read an automation file may obtain credentials beyond that user's intended privilege level. If the identity is privileged, the exposure can lead to broader administrative access.

**Likely control weakness**

Reusable credentials are embedded in readable script content rather than obtained through an appropriate identity or secrets-management mechanism.

**Recommendations**

- Remove embedded passwords from scripts.
- Use gMSAs for supported Windows services and scheduled tasks.
- Store necessary secrets in an approved secrets-management system.
- Apply restrictive access controls to automation files.
- Rotate exposed credentials.
- Search other scripts and deployment artifacts for similar disclosures.

### F-07 — Domain Administrator Credential Exposure

**Category:** Privileged access management  
**Preliminary severity:** Critical

**Issue**

The recovered credentials are associated with a Domain Administrator identity according to the supplied BloodHound evidence.

**Evidence in the supplied source**

- Successful credential validation.
- Administrative share access reported by the source.
- BloodHound relationship to Domain Admins and the domain controller.
- Reported final administrative access.

**Potential security impact**

Domain Administrator access can enable broad control over domain identities, systems, and directory resources. In a real environment, this condition should be treated as a potential domain-compromise incident.

**Likely control weakness**

A privileged credential is exposed through an accessible automation file and can be used for interactive authentication.

**Recommendations**

- Separate privileged identities from standard user and service workflows.
- Do not store Domain Administrator credentials in routine automation.
- Use least-privilege service identities.
- Restrict and monitor RDP for privileged accounts.
- Apply privileged access workstation practices where appropriate.
- Rotate compromised privileged credentials and investigate their use.
- Assess the environment for persistence or unauthorized changes following a suspected compromise.

## MITRE ATT&CK Mapping

The following mappings relate the source-reported assessment activities to relevant MITRE ATT&CK concepts. They are analytical mappings based on the documented actions, not claims that every underlying technique was independently verified. Consult the current ATT&CK knowledge base before using this table in a formal detection-engineering deliverable.

| Assessment activity | ATT&CK technique | ID | Evidence and qualification |
|---|---|---|---|
| Domain user enumeration | Account Discovery | T1087 | User-principal discovery is documented; the domain-account sub-technique may be applicable. |
| Domain group and permission analysis | Permission Groups Discovery | T1069 | BloodHound is used to investigate domain group and privilege relationships. |
| Kerberoasting | Steal or Forge Kerberos Tickets: Kerberoasting | T1558.003 | The source reports service-ticket retrieval for offline password analysis. |
| Password spraying | Brute Force: Password Spraying | T1110.003 | The documented workflow tests a candidate password against enumerated users. |
| Credential reuse | Valid Accounts | T1078 | The source reports authentication as another identity using a recovered password. |
| Credentials stored in a script | Unsecured Credentials: Credentials In Files | T1552.001 | A readable PowerShell script is reported to contain reusable credentials. |
| Password cracking | Brute Force: Password Cracking | T1110.002 | Hashcat is used for offline password recovery. |
| RDP access | Remote Services: Remote Desktop Protocol | T1021.001 | The recovered credential is used to obtain an RDP session. |
| Active Directory object-permission abuse | Exact technique not assigned | — | The source identifies `GenericWrite` and a targeted Kerberoasting workflow but does not fully document the low-level directory modification. |

The precise ATT&CK mapping for the `GenericWrite` exploitation step depends on the actual directory modification performed. A technique should not be assigned solely because a particular tool is commonly used for it.

## Defensive Recommendations

The findings indicate that remediation must address identity exposure, credential strength, directory permissions, secret storage, and privileged access as a coordinated program.

### Identity and authentication

- Disable unnecessary Guest and anonymous access.
- Restrict unauthenticated directory enumeration.
- Enforce strong, unique credentials.
- Prevent password reuse through policy and password screening.
- Review service identities and their authentication requirements.
- Investigate compromised credentials across systems where they may have been reused.

### Kerberos hardening

- Audit accounts that do not require Kerberos pre-authentication.
- Review service accounts and their SPNs.
- Use long, randomly generated service-account passwords.
- Prefer gMSAs where operationally appropriate.
- Reduce legacy Kerberos encryption dependencies when feasible.
- Monitor suspicious ticket-request patterns and repeated authentication failures.

### Active Directory permissions

- Review effective permissions on users, groups, computers, and other directory objects.
- Audit consequential rights such as `GenericWrite`, `GenericAll`, `WriteDACL`, and `WriteOwner`.
- Remove unnecessary delegated permissions.
- Review inherited ACLs and legacy delegation.
- Conduct regular attack-path analysis using BloodHound or an equivalent tool.
- Verify that remediation breaks the identified privilege path.

### Secrets and automation

- Never embed reusable passwords in readable scripts.
- Use managed service identities or a dedicated secrets-management platform.
- Restrict access to synchronization scripts, scheduled-task definitions, and deployment artifacts.
- Scan repositories and automation files for exposed credentials.
- Rotate credentials immediately after exposure.
- Assess whether exposed credentials exist in other environments or services.

### Privileged access

- Separate standard user accounts from administrative identities.
- Avoid using Domain Administrator accounts for routine automation.
- Restrict RDP to approved hosts and users.
- Apply privileged access workstation practices.
- Monitor privileged authentication and administrative share access.
- Investigate unexpected Domain Admin logons.
- Follow appropriate privileged-access and incident-response procedures after suspected compromise.

### Detection and monitoring

Useful monitoring opportunities include:

- Unusual LDAP queries and high-volume directory enumeration.
- Unexpected Kerberos service-ticket request patterns.
- Authentication attempts involving many accounts.
- Successful authentication with credentials known to be exposed elsewhere.
- Changes to sensitive Active Directory object permissions.
- Access to scripts and files that contain secrets.
- RDP sessions initiated by privileged identities from unexpected systems.

Detection rules should be tuned and validated against normal administrative activity to reduce false positives.

## Lessons Learned

### Enumeration creates leverage

Guest and directory information exposure may appear low impact when considered separately. In combination, these weaknesses provide the identity information needed for later credential-access techniques.

### Credential security is an identity-wide problem

The recovered password becomes more consequential because it is reported as valid for a second domain identity. Credential compromise should therefore trigger investigation beyond the initially affected account.

### Directory ACLs are part of the attack surface

Group membership alone does not explain every privilege path. Object-level permissions can create meaningful control relationships between identities that otherwise appear unrelated.

### Service accounts require deliberate design

Service identities should have strong, unique credentials, limited permissions, and appropriate lifecycle management. Managed service accounts reduce the operational need to maintain reusable passwords manually.

### Automation files can become credential repositories

Scripts are often readable by users who need to understand or troubleshoot a workflow. Storing passwords directly in those files turns ordinary read access into a potential credential-disclosure vulnerability.

### Privileged credentials need stronger boundaries

Domain Administrator credentials should not be exposed through routine scripts or made accessible to standard user workflows. Privileged identities require separate controls, monitoring, and restricted use.

### Exposure and successful exploitation are different findings

The AS-REP roasting branch demonstrates an important reporting principle: identifying a vulnerable configuration does not necessarily mean that a password was recovered. A professional assessment records both the observed weakness and the actual result of testing.

## Evidence and Artifact Management

Evidence should substantiate each major conclusion without publishing unnecessary secrets.

### Figure 01 — `GenericWrite` relationship

**Evidence purpose:** Supports the reported Active Directory object-permission relationship between `ZACHARY_HUNT` and `JERRI_LANCASTER`.

The source describes the screenshot but does not provide an explicit image path for this figure.

### Figure 02 — Domain Administrator relationship

**Evidence purpose:** Supports the reported final privilege relationship involving `SANFORD_DAUGHERTY`, Domain Admins, and the domain controller.

The source describes the screenshot but does not provide an explicit image path for this figure.

### Figure 03 — Room completion

**Evidence purpose:** Intended to document completion of the lab.

The source does not provide an explicit image path for this figure.

### Artifact verification

The original source contains figure descriptions but no actual Markdown image references or filenames for these figures. No image paths have been invented. Add authentic repository-relative image references only after verifying the filenames and locations.

### Evidence integrity principles

- Use authentic screenshots and terminal output.
- Preserve sufficient context to understand each result.
- Redact challenge flags, passwords, and other unnecessary secrets.
- Do not fabricate terminal output or present synthetic images as captured evidence.
- Avoid including unrelated personal information or host details.
- Ensure each figure actually supports the associated claim.

## Challenge Flag Protection

```text
Final Flag: [REDACTED]
```

The final challenge answer and recovered passwords are excluded from this public portfolio edition.

This write-up focuses on security methodology, attack-chain reasoning, impact, and defensive lessons rather than publishing answer-only secrets. The original redaction policy is preserved.

## Conclusion

Operation Endgame demonstrates how several Active Directory weaknesses can combine into a path from unauthenticated information gathering to Domain Administrator access.

The documented chain begins with Guest-accessible identity information, progresses through Kerberos service-ticket recovery and password reuse, and then uses an excessive `GenericWrite` permission to pursue another domain credential. Interactive access exposes a PowerShell script containing additional credentials, ultimately leading to a privileged domain identity.

The central defensive lesson is that domain security depends on the interaction of multiple control layers:

- Authentication must resist credential guessing and reuse.
- Directory information must be exposed only where necessary.
- Active Directory permissions must follow least privilege.
- Service identities must be managed securely.
- Automation must not contain reusable privileged credentials.
- Administrative access must be isolated, restricted, and monitored.

**Final attack-chain summary**

**Identity enumeration → Kerberos credential access → password reuse → ACL analysis → targeted Kerberoasting → RDP access → script-based credential discovery → Domain Administrator validation**

The key professional outcome is not simply that administrative access was reported. It is that the path can be explained, its contributing weaknesses can be tied to the supplied evidence, and each weakness can be translated into a practical defensive action.

## References

The following resources are preserved from the source material and provide background for the technologies and security concepts covered in this assessment.

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

## Responsible Use

This documentation describes activity within an authorized TryHackMe training environment. The techniques discussed should be used only against systems for which explicit permission to test has been granted. Findings in this lab should not be assumed to exist in other environments without independent validation.
