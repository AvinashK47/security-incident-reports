# Security Incident Reports & Forensic Post-Mortems

A centralized repository documenting real-world security incidents, forensic investigations, threat modeling, and production hardening implementations across self-hosted and cloud-native infrastructure.

---

## 📑 Incident Directory

| Incident ID | Date | Target / Context | Attack Vector | Severity | Status |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **[Attack-1: The r2sh Redis-to-Shell Botnet & Monero Miner Intrusion](./Attack-1/README.md)** | Oct 2026 | Oracle Cloud ARM VM (`exec-d` & Portfolio) | Unauthenticated Redis TCP port `6379` exposure $\rightarrow$ `CONFIG SET` arbitrary file overwrite (`authorized_keys`) | **CRITICAL** | **RESOLVED & HARDENED** |

---

## 🎯 Purpose & Methodology

Each post-mortem follows industry standard incident response methodologies (NIST SP 800-61 / SANS):

1. **Identification & Detection**: How anomalies were initially detected (network timeouts, build failures, process behavior).
2. **Containment**: Immediate actions to isolate compromised workloads without destroying volatile forensic evidence.
3. **Eradication**: Systematic removal of backdoors, unauthorized credentials, rogue crontabs, and malicious binaries.
4. **Recovery & Hardening**: Verification of clean operational state and implementation of immutable defense controls.
5. **Lessons Learned**: Architecture improvements, automated linting/firewalling, and policy changes to eliminate root causes.

---

## 👤 Author

Maintained by **[Avinash Kushwaha](https://github.com/AvinashK47)** ([https://avinashk47.me](https://avinashk47.me)).
