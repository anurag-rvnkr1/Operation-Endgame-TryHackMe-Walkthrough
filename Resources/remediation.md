# Remediation

## Identity
- Enforce strong, unique passwords.
- Eliminate password reuse.
- Prefer gMSA for service identities.
- Restrict Guest and anonymous access.

## Kerberos
- Audit SPNs.
- Reduce RC4 usage.
- Review accounts lacking pre-authentication.
- Monitor abnormal ticket requests.

## Active Directory ACLs
- Remove unnecessary `GenericWrite` and similar rights.
- Apply least privilege.
- Regularly review ACL attack paths.

## Secrets
- Remove passwords from scripts.
- Use secure secret stores or managed identities.
- Rotate exposed credentials immediately.

## Privileged Accounts
- Separate administrative accounts from standard workflows.
- Restrict RDP.
- Monitor Domain Admin authentication.
