# Offensive Security Portfolio — Method, Writeups & Agentic AI Red Teaming

Security engineer with ~6 years in network, cloud, and identity security (AWS, IAM, OAuth2/OIDC, container security), building depth in **offensive security, AppSec, and agentic-AI red teaming**.

This repo is a working portfolio: HTB machine writeups, threat models, and agentic-AI security experiments — each written to explain **why**, not just **how**. The focus is repeatable methodology and defensible root-cause analysis, not flag-collecting.

---

## Approach

Every machine follows and documents the same loop:

**Enumerate → Hypothesise → Exploit → Escalate → Document → Method-Diff → Defender's View**

The last two steps are the point: comparing my path against the intended one to tighten method, and translating each attack chain into detection + remediation — the security-engineering judgment that matters in production.

---

## Contents

### `/writeups`
HTB machine writeups. Each covers enumeration reasoning, the exploit chain with root-cause analysis, a method-diff, and a defender's remediation view.

| Box | Difficulty | Focus | Status |
|-----|-----------|-------|--------|
| Planning | Easy Linux | Grafana CVE-2024-9264, container→host, SUID privesc | complete (user + root) |
| Nibbles | Easy Linux | Nibbleblog CVE-2015-6967, sudo script privesc | complete (user + root) |

### `/templates`
The writeup structure I use for consistency.

### `/notes`
Personal checklists (enumeration, Linux/Windows privesc) built from experience — not copied.

---

## Focus areas
- **AppSec / web:** access control, SSRF, IDOR, JWT/OAuth abuse (identity-engineering lens)
- **Cloud red teaming:** AWS trust boundaries, IAM privilege escalation, container/K8s escapes
- **Agentic-AI security:** OWASP ASI 2026 Top 10 — goal hijack, tool misuse, inter-agent comms — red teaming tool-using LLM agents

---

## Why the cross-over
Most people enter AI security from an ML background. The differentiator here is the reverse: bringing network-protocol, identity, and cloud-infrastructure depth to agentic-AI attack surfaces — where tool misuse, privilege abuse, and inter-agent communication are, underneath, classic security problems in new clothing.

---
*Practice is conducted exclusively on authorised, sandboxed platforms (Hack The Box) and self-built lab environments.*
