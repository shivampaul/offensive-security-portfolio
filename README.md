# Offensive Security Portfolio

Offensive security writeups and labs from a network & cloud security engineer moving into **agentic-AI and offensive security** — bringing infrastructure, identity (OAuth2/OIDC, IAM), and network-protocol depth to AppSec and AI red teaming.

Each writeup explains **why**, not just **how**: enumeration reasoning, root-cause analysis, and a defender's remediation view. The focus is repeatable methodology over flag-collecting.

---

## Method

**Enumerate → Hypothesise → Exploit → Escalate → Document → Defender's View**

Enumerate fully before exploiting. Write the hypothesis before trying it. Translate every attack chain into detection + remediation — the security-engineering judgment that matters in production.

---

## Writeups

| Box | Difficulty | Focus |
|-----|-----------|-------|
| [Planning](writeups/planning.md) | Easy Linux | Grafana CVE-2024-9264 (SQLi→RCE), container→host pivot via cleartext creds + password reuse, SUID privesc |
| [Secret](writeups/secret.md) | Easy Linux | Exposed `.git` → leaked JWT secret → forged admin token → command injection, SUID binary + core-dump privesc |
| [Nibbles](writeups/nibbles.md) | Easy Linux | Nibbleblog CVE-2015-6967 file-upload RCE, sudo-script privilege escalation |

*Credentials, tokens, target IPs, and flags are redacted throughout. All work performed on authorised, sandboxed platforms (Hack The Box).*

---

## Focus areas

- **AppSec / web** — access control, SSRF, IDOR, JWT/OAuth abuse (identity-engineering lens)
- **Cloud red teaming** — AWS trust boundaries, IAM privilege escalation, container/K8s escapes
- **Agentic-AI security** — OWASP ASI 2026 Top 10: goal hijack, tool misuse, inter-agent comms, and MCP attack surface

## Why the cross-over

Most people enter AI security from an ML background. The angle here is the reverse: bringing network-protocol, identity, and cloud-infrastructure depth to agentic-AI attack surfaces — where tool misuse, privilege abuse, and inter-agent communication are, underneath, classic security problems in new clothing.

---

## Repo layout

```
├── writeups/     HTB machine writeups
├── templates/    Writeup structure
└── notes/        Enumeration / privesc checklists
```
