# 🔍 Detections

SPL queries developed and tested in the lab environment. Each query includes context, the threat it detects, and triaging guidance.

---

## Index

| File | Threat | Event IDs |
|---|---|---|
| [brute_force_4625.spl](brute_force_4625.spl) | Windows login brute force | 4625 |
| [brute_force_kerberos.spl](brute_force_kerberos.spl) | Kerberos brute force against AD | 4771, 4768 |
| [privilege_escalation.spl](privilege_escalation.spl) | Addition to privileged group | 4728 |

---

> All queries were developed iteratively based on simulated attack data collected in the lab.
