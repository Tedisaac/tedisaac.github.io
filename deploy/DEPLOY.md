# isaacted.dev — setup

One static site on a box that already runs two other projects. Every step is
written so neither of them is disturbed.

**What this does NOT do:** no port is opened, no Docker, no container, no
firewall change, and no other project's config is opened. nginx reads the files
off disk. The only shared resource is nginx itself.

Legend: 💻 = your laptop · 🖥 = the server · 🌐 = a browser

---

## 0 🖥 What is already on that server

| In use | By |
|---|---|
| 80, 443 | nginx (Certbot-managed) — Chekit's 7 `uat*.chekit.shop` hosts |
| 3000–3003, 4000, 4001, 8000, 8080 | Chekit UAT containers (loopback only) |
| 8081 | Carniva's Caddy edge (loopback only) |
| 22 | sshd |

**This site adds nothing to that list.** Confirm before and after:

```bash
sudo ss -tlnp | sort -k4
```

---

## 1 🌐 DNS at Spaceship

| Type | Host | Value |
|---|---|---|
| A | `@` | `212.47.79.124` |
| A | `www` | `212.47.79.124` |

Then wait for it, and check — a wrong record means a failed challenge in step 5,
and repeated failures hit Let's Encrypt's rate limit:

```bash
dig +short isaacted.dev
dig +short www.isaacted.dev
```

Both must print `212.47.79.124`.

> **`.dev` is on the HSTS preload list.** Browsers refuse plain HTTP on it
> outright, so the domain is unreachable until step 5 issues the certificate.
> That is the TLD, not a broken setup — Certbot is unaffected, because Let's
> Encrypt fetches the challenge directly rather than through a browser.

---

## 2 🖥 The deploy account

A dedicated account with **no sudo and no docker group**. That is the point: the
CI key lands here, so even if it leaks it cannot reach Chekit or Carniva.

```bash
sudo adduser --system --group --home /srv/isaacted --shell /bin/bash isaacted
sudo passwd -l isaacted      # key auth only; no password login exists
```

`--shell /bin/bash` is required — the forced command in step 4 runs through the
user's login shell, and `nologin` would refuse it.

Confirm it has no privileges:

```bash
groups isaacted        # expect: isaacted  (NOT sudo, NOT docker)
```

### The directory layout matters

```bash
sudo install -d -o isaacted -g isaacted -m 755 /srv/isaacted
sudo install -d -o isaacted -g isaacted -m 755 /srv/isaacted/public
sudo install -d -o isaacted -g isaacted -m 700 /srv/isaacted/.ssh
```

The site is served from **`/srv/isaacted/public`**, not from `/srv/isaacted`.
The home directory holds `.ssh/authorized_keys`; if nginx served the home, that
file would be fetchable by anyone who guessed the path. `public/` keeps the
served tree and the account's secrets apart, and the nginx config denies
dotfiles as a second line.

`755` so nginx (`www-data`) can read the site; `700` on `.ssh` so only the
account can read its keys.

---

## 3 🖥 nginx

**Baseline the other project first** and keep the output:

One line on purpose — line-continuations get mangled on paste:

```bash
for h in $(sudo nginx -T 2>/dev/null | awk '/server_name/{for(i=2;i<=NF;i++){gsub(/;/,"",$i); if($i!="_"&&$i!="")print $i}}' | sort -u | grep -v isaacted); do printf '%-32s %s\n' "$h" "$(curl -s -o /dev/null --max-time 10 -w '%{http_code}' "https://$h/")"; done
```

If that is awkward, list the hosts by hand instead:

```bash
for h in uat.chekit.shop uat-api.chekit.shop; do printf '%-32s %s\n' "$h" "$(curl -s -o /dev/null --max-time 10 -w '%{http_code}' "https://$h/")"; done
```

Some hosts legitimately return 401/403/404 at `/`. What matters afterwards is
that the codes are **unchanged**, not that they are all 200.

This config is **not** deployed by CI, and that is deliberate: the deploy key is
confined by `rrsync` to `/srv/isaacted/public` and the account has no sudo, so
nothing coming from GitHub can write to `/etc/nginx`. A key that could rewrite
nginx config could take Chekit and Carniva down. It is a one-time bootstrap —
copy it by hand from your laptop:

```bash
scp deploy/isaacted.dev.conf isaac@212.47.79.124:/tmp/      # 💻
```

Then, on the server:

```bash
sudo install -o root -g root -m 644 /tmp/isaacted.dev.conf /etc/nginx/sites-available/isaacted.dev.conf
sudo ln -sfn /etc/nginx/sites-available/isaacted.dev.conf /etc/nginx/sites-enabled/isaacted.dev.conf

sudo nginx -t                 # MUST pass before the next line
sudo systemctl reload nginx
```

If you ever edit the config, repeat those four lines. It happens roughly never.

> **After step 5, the repo copy and the server copy diverge.** Certbot edits the
> installed file in place — it adds `listen 443 ssl`, the certificate paths, and
> a `:80` → `:443` redirect. Re-running the `install` command above overwrites
> those edits and the site drops to plain HTTP, which on a `.dev` domain means
> browsers refuse it entirely. If you do have to reinstall, run
> `sudo certbot --nginx -d isaacted.dev -d www.isaacted.dev` again straight
> after; it is idempotent and will reuse the existing certificate.

`nginx -t` validates the whole config *before* anything reloads, so a mistake
here means "the portfolio does not go live" — it cannot take Chekit down. The
reload is graceful: in-flight requests drain rather than drop.

Re-run the baseline. Every code must match.

---

## 4 The deploy key

### 4a 💻 Generate it — on your laptop

```bash
ssh-keygen -t ed25519 -f ~/.ssh/isaacted-deploy -C isaacted-deploy -N ''
cat ~/.ssh/isaacted-deploy.pub
```

`-N ''` means no passphrase, and it is not optional: GitHub Actions runs
non-interactively and cannot type one. The private half never goes on the server.

### 4b 🖥 Find rrsync

`rrsync` ships with rsync and confines a key to a single directory. The path
moved between releases:

```bash
command -v rrsync || ls /usr/share/rsync/scripts/rrsync
```

If it is missing entirely: `sudo apt-get install -y rsync`.

### 4c 🖥 Install the key, locked to one directory

Two variables so nothing can wrap or mis-quote. Paste your public key and the
rrsync path you just found:

```bash
PUB='ssh-ed25519 AAAA...PASTE_YOUR_PUBLIC_KEY... isaacted-deploy'
OPTS='command="/usr/bin/rrsync /srv/isaacted/public",no-agent-forwarding,no-port-forwarding,no-pty,no-user-rc,no-X11-forwarding'

echo "$OPTS $PUB" | sudo tee /srv/isaacted/.ssh/authorized_keys >/dev/null
sudo chown isaacted:isaacted /srv/isaacted/.ssh/authorized_keys
sudo chmod 600 /srv/isaacted/.ssh/authorized_keys
```

It must be **one unbroken line** — sshd reads this file line by line, so a
wrapped entry becomes two invalid ones:

```bash
sudo awk 'END{print NR" line(s)"}' /srv/isaacted/.ssh/authorized_keys
# expect exactly:  1 line(s)
```

Only the line COUNT matters. Do not count fields — the space inside
`command="... /srv/isaacted/public"` is inside quotes, but awk splits on it
anyway, so a correct entry looks like 5 fields, not 4.

### 4d 🖥 Does sshd allow this user?

If `/etc/ssh/sshd_config.d/99-hardening.conf` has an `AllowUsers` line, add
`isaacted` to it or the key can never connect:

```bash
sudo sshd -T | grep -i allowusers
```

If it lists users and `isaacted` is absent, append it, then:

```bash
sudo sshd -t && sudo systemctl reload ssh    # -t FIRST. Never reload untested.
```

### 4e 💻 Prove the restriction works

```bash
ssh -i ~/.ssh/isaacted-deploy isaacted@212.47.79.124 'cat /etc/passwd'
```

**This must be refused.** If it prints the file, the forced command is not in
effect — stop and fix 4c before going further.

---

## 5 🖥 Certificate

```bash
sudo certbot --nginx -d isaacted.dev -d www.isaacted.dev
```

Certbot rewrites the `listen` lines in `isaacted.dev.conf` and reloads nginx — a
second graceful reload, so run the step 3 baseline once more afterwards.

---

## 6 🌐 GitHub secrets

Repo → Settings → Secrets and variables → Actions:

| Secret | Value |
|---|---|
| `DEPLOY_HOST` | `212.47.79.124` |
| `DEPLOY_SSH_KEY` | the **private** key from 4a, whole file including header and footer |
| `DEPLOY_KNOWN_HOSTS` | `ssh-keyscan -t ed25519 212.47.79.124 \| grep -v '^#'` |

The host key is **pinned** rather than scanned at deploy time, so nothing can
impersonate the server mid-deploy. Confirm you scanned the real machine — these
fingerprints must match:

```bash
sudo ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub        # 🖥
ssh-keyscan -t ed25519 212.47.79.124 | ssh-keygen -lf -      # 💻
```

They will look different at a glance — one prints the raw key, the other a
fingerprint. Compare the `SHA256:...` part.

---

## 7 💻 First deploy

```bash
git push origin main
```

The workflow rsyncs the site, then checks `https://isaacted.dev/` returns 200.

Then confirm nothing else moved:

```bash
sudo ss -tlnp | sort -k4        # 🖥 same list as step 0 - no new listener
```

and re-run the Chekit baseline from step 3.

---

## Ongoing

Push to `main`. That is the whole workflow.

`--delete` keeps the server matching the repo, so removing a file here removes
it there. Safe, because `rrsync` confines every path to `/srv/isaacted/public`
and has no way to delete outside it.

| Task | Command |
|---|---|
| Deploy | `git push origin main` |
| What is on the server | `sudo ls -la /srv/isaacted/public` |
| Deploy attempts | `sudo journalctl -u ssh \| grep isaacted` |
| Rotate the key | redo 4a and 4c, update `DEPLOY_SSH_KEY` |

## Worth knowing

`index.html` is **1.9 MB** because every image is inlined as base64. Repeat
visits are cheap — `no-cache` plus ETag gets a 304 rather than a re-download —
but a first load on mobile data is a real wait. Splitting the images out to
files would cut the initial payload a long way, if that ever matters.
