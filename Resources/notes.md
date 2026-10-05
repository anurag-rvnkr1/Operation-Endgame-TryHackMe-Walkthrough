# 🐍 Operation Endgame — Technical Notes

## Attack Chain

```text
Guest exposure
→ LDAP enumeration
→ Kerberos assessment
→ Guest Kerberoasting
→ credential recovery
→ password reuse
→ BloodHound
→ GenericWrite
→ targeted Kerberoasting
→ RDP
→ hardcoded privileged credentials
→ Domain Administrator
```

## Core Lessons

### LDAP
Unauthenticated directory disclosure can turn a domain into a large, actionable username database.

### Kerberos
Kerberoasting and AS-REP roasting rely on different account conditions and should be assessed separately.

### Password Reuse
A weak service credential becomes substantially more valuable when it is reused by other identities.

### ACL Review
`GenericWrite` and similar rights should be treated as potential attack-path enablers.

### Secret Management
Readable PowerShell scripts are not safe credential stores.

### Privileged Access
Domain Administrator accounts require strict separation and monitoring.
