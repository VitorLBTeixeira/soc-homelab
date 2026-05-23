# 🛡️ SOC Home Lab

A hands-on Security Operations Center lab environment built from scratch to simulate real-world threat detection, incident investigation, and response workflows.

---

## 📋 Overview

This lab was designed to replicate — at a minimal scale — the environment a SOC analyst operates in at an MSSP. Every component was configured manually, without guided tutorials, to ensure a genuine understanding of each layer.

The lab evolves in phases, with each phase introducing new complexity and new detection scenarios.

---

## 🏗️ Architecture

### Phase 1 — SIEM Foundation
| Component | Technology | Role |
|---|---|---|
| SIEM Indexer | Splunk Enterprise 10.x | Log ingestion, search, detection |
| Endpoint | Windows 10 Pro | Log source via Universal Forwarder |
| Host OS | Ubuntu 24.04 Server (VirtualBox) | Splunk server |

### Phase 2 — Active Directory Environment
| Component | Technology | Role |
|---|---|---|
| SIEM Indexer | Splunk Enterprise 10.x | Log ingestion, search, detection |
| Domain Controller | Windows Server 2022 | AD DS, DNS, GPO, UF installed |
| Endpoint | Windows 10 Pro | Domain-joined client, UF installed |
| Attacker | Kali Linux | Simulated external threat actor |
| Host OS | Ubuntu 24.04 Server (VirtualBox) | Splunk server |

### Network
- Hypervisor: VirtualBox 7.2.x
- Network mode: NAT Network (`10.0.2.0/24`)
- Domain: `soc.lab`

---

## 📁 Repository Structure

```
soc-homelab/
│
├── README.md                   # This file
│
├── detections/                 # SPL queries and detection logic
│   ├── README.md
│   ├── brute_force_4625.spl
│   ├── brute_force_kerberos.spl
│   └── privilege_escalation.spl
│
├── writeups/                   # Lab scenario documentation
│   ├── README.md
│   └── ad_attack_kill_chain.md
│
└── architecture/               # Environment setup notes
    └── splunk_setup.md
```

---

## 🔬 Detection Scenarios

| Scenario | Event IDs | Technique | Writeup |
|---|---|---|---|
| Windows Brute Force | 4625 | Credential Access | Phase 1 |
| Kerberos Brute Force | 4771, 4768 | Credential Access | [AD Kill Chain](writeups/ad_attack_kill_chain.md) |
| Kerberoasting | 4769 | Credential Access | [AD Kill Chain](writeups/ad_attack_kill_chain.md) |
| Privilege Escalation | 4728 | Privilege Escalation | [AD Kill Chain](writeups/ad_attack_kill_chain.md) |
| Authenticated Enumeration | 4624 | Discovery | [AD Kill Chain](writeups/ad_attack_kill_chain.md) |

---

## 🧰 Tools & Technologies

**SIEM & Detection**
- Splunk Enterprise + Universal Forwarder
- Windows Security Event Logs
- GPO Audit Policy configuration

**Attack Simulation**
- Kali Linux
- Kerbrute — user enumeration and brute force
- Impacket (GetUserSPNs) — Kerberoasting
- CrackMapExec — SMB enumeration and lateral movement
- Hashcat — offline hash cracking

**Frameworks**
- MITRE ATT&CK
- Windows Event ID reference

---

## 📌 MITRE ATT&CK Coverage

| Tactic | Technique | ID |
|---|---|---|
| Discovery | Account Discovery: Domain Account | T1087.002 |
| Credential Access | Brute Force | T1110 |
| Credential Access | Kerberoasting | T1558.003 |
| Privilege Escalation | Account Manipulation | T1098 |

---

## 🗺️ Roadmap

- [x] Phase 1 — Splunk + Windows UF + brute force detection
- [x] Phase 2 — Active Directory + full kill chain simulation
- [ ] Phase 3 — Object Access Auditing (Event ID 4663)
- [ ] Phase 4 — Detection rules as Sigma format
- [ ] Phase 5 — Wazuh integration

---

> Built by [Vitor Teixeira](https://www.linkedin.com/in/vitor-lohan-teixeira) | Transitioning into SOC L1 | Blue Team focused
