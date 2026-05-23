# Active Directory Attack Kill Chain
**Lab Phase:** 2  
**Date:** May 2026  
**Environment:** Windows Server 2022 DC + Windows 10 endpoint + Kali Linux attacker  
**SIEM:** Splunk Enterprise with UF on DC-01  

---

## Scenario Overview

Simulated a full Active Directory attack kill chain from the perspective of an external threat actor with initial network access. The goal was to progress from zero knowledge of the environment to full domain compromise, while collecting forensic evidence at each stage via Splunk.

**Premise:** The attacker is already inside the network. No software vulnerabilities were exploited — only weak configurations, reusable passwords, and legitimate protocol abuse.

---

## Environment

| Host | OS | Role | IP |
|---|---|---|---|
| splunk-server | Ubuntu 24.04 Server | Splunk Indexer | 10.0.2.x |
| DC-01 | Windows Server 2022 | Domain Controller | 10.0.2.23 |
| DESKTOP-WIN10 | Windows 10 Pro | Domain-joined endpoint | 10.0.2.x |
| kali | Kali Linux | Attacker | 10.0.2.6 |

**Domain:** `soc.lab`  
**Target accounts:** `john.doe` (standard user), `soc.admin` (privileged)

---

## Kill Chain

### Phase 1 — User Enumeration

**Objective:** Identify valid domain accounts without credentials.

**Tool:** Kerbrute  
**Method:** The Kerberos protocol responds differently to valid and invalid usernames during pre-authentication. Kerbrute exploits this behavior to enumerate accounts without a password.

Initial attempt with generic wordlists found `administrator` (default account). A custom wordlist with common corporate username formats (`john`, `john.doe`, `johndoe`, `j.doe`) confirmed `john.doe` as a valid account — simulating OSINT-based username discovery.

```bash
kerbrute userenum --dc 10.0.2.23 -d soc.lab /tmp/users.txt
```

**Evidence collected:**
- Event ID **4768** with failure status — each Kerbrute attempt generates a failed TGT request on the DC

---

### Phase 2 — Credential Brute Force

**Objective:** Recover the password for `john.doe`.

**Tool:** Kerbrute  
**Method:** Password spraying and brute force against the Kerberos service using the rockyou wordlist. `Password123!` — a common corporate password pattern — was present in the list.

```bash
kerbrute bruteuser --dc 10.0.2.23 -d soc.lab /usr/share/wordlists/rockyou.txt john.doe
```

**Evidence collected:**
- Event ID **4771** (repeated) — Kerberos pre-authentication failed for each incorrect password attempt
- Event ID **4768** (success) — TGT granted when the correct password was submitted

**SOC Triage:**  
A high volume of 4771 events from a single `Client_Address` within a short timeframe is the classic brute force pattern. A subsequent 4768 success from the same IP confirms compromise. **Escalate immediately.**

---

### Phase 3 — Authenticated Enumeration

**Objective:** Map the environment using compromised credentials.

**Tool:** CrackMapExec  
**Method:** With valid credentials for `john.doe`, enumerated SMB shares and domain users.

```bash
crackmapexec smb 10.0.2.23 -u john.doe -p 'Password123!' --shares
crackmapexec smb 10.0.2.23 -u john.doe -p 'Password123!' --users
```

**Findings:**
- Accessible shares: `NETLOGON`, `SYSVOL` (READ) — standard for domain users
- `ADMIN$`, `C$` — access denied (john.doe is not an admin)
- Full user list obtained: `soc.admin`, `john.doe`, `krbtgt`, `Guest`, `Administrator`
- `soc.admin` identified as next target based on naming convention

**Evidence collected:**
- Event ID **4624** — successful network logon (Logon_Type 3) from Kali IP

---

### Phase 4 — Kerberoasting

**Objective:** Obtain the password hash of `soc.admin` for offline cracking.

**Tool:** Impacket (GetUserSPNs), Hashcat  
**Method:** Any authenticated domain user can request a Kerberos service ticket (TGS) for accounts with a registered Service Principal Name (SPN). The ticket is encrypted with the target account's password hash and can be cracked offline.

A SPN was configured on `soc.admin` to simulate a real service account:
```powershell
setspn -A HTTP/dc-01.soc.lab soc.admin
```

TGS requested using john.doe's credentials:
```bash
impacket-GetUserSPNs soc.lab/john.doe:'Password123!' -dc-ip 10.0.2.23 -request
```

Hash cracked offline:
```bash
hashcat -m 13100 /tmp/hash.txt /usr/share/wordlists/rockyou.txt
# Result: Password123! — cracked in 0 seconds
```

**Evidence collected:**
- Event ID **4769** — Kerberos service ticket requested for `soc.admin` by `john.doe` from `10.0.2.6`

**SOC Triage:**  
A 4769 event for a privileged account originating from a non-administrative IP is a mandatory investigation. Service accounts rarely receive TGS requests from standard users. Cross-reference with the requesting account's recent activity.

---

### Phase 5 — Privilege Escalation

**Objective:** Add `john.doe` to Domain Admins for persistent full domain access.

**Tool:** CrackMapExec  
**Method:** With `soc.admin` credentials, executed a remote command on the DC to add `john.doe` to the Domain Admins group.

```bash
crackmapexec smb 10.0.2.23 -u soc.admin -p 'Password123!' -x "net group \"Domain Admins\" john.doe /add /domain"
# Output: The command completed successfully.
```

**Evidence collected:**
- Event ID **4728** — member added to security-enabled global group
  - Subject (executor): `soc.admin`
  - Member added: `CN=John,OU=SOC-Users,DC=soc,DC=lab`
  - Group: `Domain Admins`

**SOC Triage:**  
Any modification to a privileged group is a critical alert. The combination of an unexpected executor account, off-hours timing, and a preceding 4769 event creates a clear attack narrative. **Incident confirmed. Escalate to L2/IR team.**

---

## Detection Summary

| Phase | Attack | Event ID | SPL Query |
|---|---|---|---|
| 1–2 | Recon + Brute Force | 4771, 4768 | [brute_force_kerberos.spl](../detections/brute_force_kerberos.spl) |
| 3 | Authenticated Enumeration | 4624 | — |
| 4 | Kerberoasting | 4769 | — |
| 5 | Privilege Escalation | 4728 | [privilege_escalation.spl](../detections/privilege_escalation.spl) |

---

## Key Takeaways

- **No CVEs required.** The entire kill chain relied on weak passwords and default configurations — the most common real-world scenario.
- **Every phase left evidence.** No stage was invisible to the SIEM. Detection failures in production are rarely about missing data — they are about missing detection logic or untriaged alerts.
- **Correlating events tells the story.** Individually, a 4771 is noise. Combined with a 4768 success from the same IP, followed by a 4769 and a 4728 — it is a full compromise narrative.

---

## MITRE ATT&CK Mapping

| Tactic | Technique | ID | Phase |
|---|---|---|---|
| Discovery | Account Discovery: Domain Account | T1087.002 | 1 |
| Credential Access | Brute Force | T1110 | 2 |
| Discovery | Account Discovery: Domain Account | T1087.002 | 3 |
| Credential Access | Kerberoasting | T1558.003 | 4 |
| Privilege Escalation | Account Manipulation | T1098 | 5 |
