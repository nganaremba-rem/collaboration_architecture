# 🌐 nginx Reverse Proxy for GitLab over WARP — Full Guide

A from-scratch, line-by-line guide to replacing the little Python proxy with **nginx**.
It explains **what a reverse proxy is**, **why we need one here**, and gives **two ways**
to set it up:

- **Approach A — dedicated port:** open GitLab at `https://localhost:8443`
- **Approach B — path, no port:** open GitLab at `https://localhost/gitlab/` (port 443)

Every config line is explained. Read top to bottom once; after that, the config blocks
are copy-paste.

---

## 1. The problem (why we need a proxy at all)

Our GitLab dev box (`172.31.35.68`) serves GitLab as **plain HTTP on port 80**. The
HTTPS (the padlock) is normally added by the **AWS load balancer** sitting in front of
it — and that load balancer is locked to the office IP. When we connect over **WARP**,
we reach the box's **private IP directly**, bypassing the load balancer. So:

1. There is **no HTTPS on the box** — port 443 is closed there.
2. GitLab is configured to **force HTTP → HTTPS**. So when a browser asks for
   `http://172.31.35.68`, GitLab replies "go to `https://172.31.35.68` instead."
3. But nothing answers on `https://172.31.35.68` (no 443), so the browser dies with
   `ERR_ADDRESS_UNREACHABLE`.

We are stuck in a loop: GitLab insists on HTTPS, but the HTTPS endpoint doesn't exist
once we're past the load balancer.

**The fix:** put a tiny HTTPS endpoint *on our side* that:
- speaks **HTTPS** to the browser (so the browser is happy), and
- forwards the request as **plain HTTP** to GitLab on port 80, and
- tells GitLab *"this request already arrived over HTTPS"* so GitLab stops redirecting.

That "tiny HTTPS endpoint that forwards to a backend" **is a reverse proxy**. nginx is
the standard tool for it.

---

## 2. What is a reverse proxy (plain terms)

- A **forward proxy** sits in front of *you* (the client) and fetches the wider Internet
  on your behalf (e.g. a corporate web filter).
- A **reverse proxy** sits in front of a *server* and accepts requests on the server's
  behalf, then forwards them to the real server behind it.

```
  Browser  ──HTTPS──▶  [ nginx reverse proxy ]  ──HTTP──▶  GitLab :80
                          (terminates TLS,
                           adds X-Forwarded-Proto)
```

The browser thinks it is talking to a normal HTTPS website. It never knows GitLab is
really plain HTTP behind nginx. nginx **"terminates TLS"** — it is the place where the
encryption stops and plain HTTP begins.

The single most important thing nginx does for us is add one header:

```
X-Forwarded-Proto: https
```

This header is the standard way a proxy tells the app behind it *"the original request
came in over HTTPS, even though I'm forwarding it to you as plain HTTP."* GitLab trusts
this header, decides "this user is already on HTTPS," and **stops redirecting**. That
single header breaks the loop from §1.

---

## 3. Why nginx instead of the Python proxy

The Python proxy works, but nginx is better here because:

- **WebSockets** — GitLab's live CI job logs stream over WebSockets. nginx forwards
  these correctly (the Python proxy did not).
- **Stable & supervised** — runs as a systemd service, restarts on crash/reboot.
- **Battle-tested** — handles large git pushes, gzip, timeouts, many users at once.
- **One config file** — easier to read and change than proxy code.

Same job, sturdier tool.

---

## 4. Prerequisites (do these once)

1. **WARP must be connected.** nginx can only reach `172.31.35.68` through the WARP
   tunnel. No WARP → nginx gets "no route to host."
2. **nginx installed.** Check:
   ```bash
   nginx -v
   ```
   If missing: `sudo pacman -S nginx` (Arch) / `sudo apt install nginx` (Debian/Ubuntu).
3. **A self-signed certificate** for nginx to present to the browser. Make one:
   ```bash
   sudo mkdir -p /etc/nginx/ssl
   sudo openssl req -x509 -newkey rsa:2048 -nodes -days 825 \
     -keyout /etc/nginx/ssl/gitlab-warp.key \
     -out    /etc/nginx/ssl/gitlab-warp.crt \
     -subj "/CN=localhost"
   ```
   - `-x509` → make a finished certificate (not a signing request).
   - `-newkey rsa:2048` → also generate a fresh 2048-bit private key.
   - `-nodes` → "no DES," i.e. **don't password-protect the key** (so nginx can start
     unattended).
   - `-days 825` → valid ~2 years (825 is the max most browsers accept).
   - `-keyout` / `-out` → where to write the private key and the certificate.
   - `-subj "/CN=localhost"` → the certificate's name. Browsers will still warn because
     **nobody trusted** signed it (self-signed) — that warning is expected, click through
     once.

---

## 5. Approach A — dedicated port (`https://localhost:8443`)

The simplest, most reliable layout. nginx listens on its **own port** (`8443`) and
forwards everything to GitLab. Because GitLab lives at the **root** of this port (`/`),
none of GitLab's internal links break. **Recommended.**

Create `/etc/nginx/conf.d/gitlab-warp.conf`:

```nginx
# ─────────────────────────────────────────────────────────────────────────
# WebSocket upgrade helper.
# GitLab's live CI logs use WebSockets. A WebSocket starts as a normal HTTP
# request that asks to "Upgrade" the connection. This map sets a variable we
# reuse below: if the client sent an Upgrade header, pass "upgrade"; if not,
# pass "close". Without this, live logs hang.
# ─────────────────────────────────────────────────────────────────────────
map $http_upgrade $connection_upgrade {
    default upgrade;
    ''      close;
}

server {
    # ── Listen on HTTPS, port 8443, localhost only ──
    # "ssl" makes nginx terminate TLS here. We bind 127.0.0.1 so only this
    # laptop can reach it (it is a personal proxy, not a public service).
    listen 127.0.0.1:8443 ssl;
    server_name localhost;

    # ── The certificate nginx shows the browser (from §4) ──
    ssl_certificate     /etc/nginx/ssl/gitlab-warp.crt;
    ssl_certificate_key /etc/nginx/ssl/gitlab-warp.key;

    # ── Allow arbitrarily large uploads ──
    # Default nginx caps request bodies at 1 MB, which breaks large git
    # pushes and file uploads. 0 = no limit.
    client_max_body_size 0;

    location / {
        # ── The backend: GitLab's plain HTTP on the box, over WARP ──
        proxy_pass http://172.31.35.68:80;

        # ── Pass the browser's Host header through unchanged ──
        # GitLab builds its links (redirects, asset URLs) from the Host it
        # receives. $http_host is exactly what the browser sent
        # ("localhost:8443"), so generated links point back to this proxy.
        proxy_set_header Host $http_host;

        # ── THE KEY LINE ──
        # Tell GitLab the original request was HTTPS. This stops GitLab's
        # forced HTTP→HTTPS redirect, which otherwise loops forever.
        proxy_set_header X-Forwarded-Proto https;

        # ── Pass the real client IP along (good hygiene, used in logs) ──
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Real-IP       $remote_addr;

        # ── WebSocket support (pairs with the map at the top) ──
        # WebSockets need HTTP/1.1 and the Upgrade/Connection headers
        # forwarded. This is what makes live CI logs stream.
        proxy_http_version 1.1;
        proxy_set_header Upgrade    $http_upgrade;
        proxy_set_header Connection $connection_upgrade;

        # ── Streaming-friendly settings ──
        # Turn off response buffering so streamed logs appear immediately,
        # and allow long-lived connections (1 hour) for log tailing.
        proxy_buffering   off;
        proxy_read_timeout 3600s;
    }
}
```

Enable it:

```bash
sudo nginx -t                 # test the config syntax — fix errors before reloading
sudo systemctl enable --now nginx
sudo systemctl reload nginx   # if nginx was already running
```

Open **`https://localhost:8443`** → accept the self-signed warning once → GitLab loads.

### Why Approach A "just works"
GitLab sits at the **root** (`/`) of port 8443. Every link GitLab generates
(`/users/sign_in`, `/-/assets/...`, `/your-group/your-repo`) is an absolute path from
root, and root is GitLab. Nothing has to be rewritten.

---

## 6. Approach B — path, no port (`https://localhost/gitlab/`)

Here nginx listens on the **standard HTTPS port 443** (so no `:8443` in the URL) and
serves GitLab under a **sub-path** like `/gitlab/`. This is what people mean by "reverse
proxy by path." It is how you'd host **several apps on one hostname**:

```
  https://localhost/gitlab/     → GitLab
  https://localhost/grafana/    → Grafana
  https://localhost/wiki/       → something else
```

### ⚠️ The catch you must understand first

When GitLab lives under `/gitlab/` but **GitLab itself still thinks it lives at root**,
its links break. GitLab will generate a login link like `/users/sign_in` — but the real
URL is `/gitlab/users/sign_in`. The browser asks for `/users/sign_in`, nginx has no rule
for it, and you get a 404. Assets (CSS/JS) 404 too, so the page looks broken.

There are **two ways** to deal with this. Pick one.

### Option B1 — tell GitLab it lives under the path (the correct, robust way)

Configure GitLab's **relative URL root** so GitLab itself generates `/gitlab/...` links.
On the GitLab box, edit `/etc/gitlab/gitlab.rb`:

```ruby
external_url "http://172.31.35.68/gitlab"
```

Then `sudo gitlab-ctl reconfigure`. Now GitLab builds every link under `/gitlab/`, and a
plain path proxy works cleanly.

⚠️ **This changes GitLab server-side** and affects how the load balancer path must look
too — it is a real change to a shared dev box. Coordinate before doing it. This is the
*only* fully-correct way to run GitLab under a sub-path; nginx tricks (below) are
band-aids.

nginx side for B1:

```nginx
map $http_upgrade $connection_upgrade {
    default upgrade;
    ''      close;
}

server {
    listen 127.0.0.1:443 ssl;       # standard HTTPS port → no :port in the URL
    server_name localhost;

    ssl_certificate     /etc/nginx/ssl/gitlab-warp.crt;
    ssl_certificate_key /etc/nginx/ssl/gitlab-warp.key;
    client_max_body_size 0;

    # Everything under /gitlab/ goes to GitLab.
    location /gitlab/ {
        proxy_pass http://172.31.35.68:80;   # NOTE: no trailing path here on purpose
        # ^ Because GitLab now expects /gitlab/ in the path (via external_url),
        #   we forward the path AS-IS, including the /gitlab/ prefix.

        proxy_set_header Host              $http_host;
        proxy_set_header X-Forwarded-Proto https;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Real-IP         $remote_addr;

        proxy_http_version 1.1;
        proxy_set_header Upgrade    $http_upgrade;
        proxy_set_header Connection $connection_upgrade;
        proxy_buffering    off;
        proxy_read_timeout 3600s;
    }
}
```

Open **`https://localhost/gitlab/`**. Clean and stable.

### Option B2 — strip the prefix at nginx, no GitLab change (band-aid)

If you **can't** change GitLab, you can make nginx **strip** `/gitlab/` before forwarding,
so GitLab still thinks it's at root:

```nginx
location /gitlab/ {
    # Trailing slash on BOTH location and proxy_pass = strip the /gitlab/ prefix.
    # Browser asks /gitlab/users/sign_in  →  nginx forwards /users/sign_in to GitLab.
    proxy_pass http://172.31.35.68:80/;

    proxy_set_header Host              $http_host;
    proxy_set_header X-Forwarded-Proto https;
    proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
    proxy_set_header X-Real-IP         $remote_addr;

    proxy_http_version 1.1;
    proxy_set_header Upgrade    $http_upgrade;
    proxy_set_header Connection $connection_upgrade;
    proxy_buffering    off;
    proxy_read_timeout 3600s;
}
```

**The key detail:** the **trailing slash** on `proxy_pass http://172.31.35.68:80/;`.

- `proxy_pass http://172.31.35.68:80;`  (no slash) → forwards the **whole** path,
  including `/gitlab/`. GitLab gets `/gitlab/users/sign_in` and 404s (it's at root).
- `proxy_pass http://172.31.35.68:80/;` (with slash) → nginx **replaces** the matched
  `/gitlab/` with `/`. GitLab gets `/users/sign_in`. ✅

**But B2 is fragile.** GitLab *still generates root links* (`/users/sign_in`), so the
moment GitLab sends an absolute link or a redirect, the browser leaves `/gitlab/` and
breaks. To patch that you'd need response rewriting:

```nginx
    # Rewrite absolute paths in HTML so links stay under /gitlab/.
    # This is brittle — it only catches HTML, not JS-built URLs or JSON.
    sub_filter 'href="/'  'href="/gitlab/';
    sub_filter 'src="/'   'src="/gitlab/';
    sub_filter_once off;

    # Fix the Location header on redirects (e.g. after login).
    proxy_redirect / /gitlab/;
```

Even with these, single-page-app routes and some assets can still slip out. **For GitLab,
B2 is not recommended** — GitLab is a big app with many absolute URLs. Use **B1** (real
`external_url`) if you need a sub-path, or just use **Approach A** (dedicated port), which
sidesteps the whole problem.

---

## 7. Port vs. path — which to use

| | Approach A — port `:8443` | Approach B — path `/gitlab/` |
|---|---|---|
| URL | `https://localhost:8443` | `https://localhost/gitlab/` |
| GitLab change needed | **None** | **Yes** for the clean way (B1: `external_url`) |
| Link breakage risk | None | High unless B1 is used |
| Good for | one app, fastest setup | many apps under one hostname |
| Recommended here | ✅ **Yes** | Only with B1, only if you need one hostname |

**Rule of thumb:** different app → give it a **different port** (Approach A). It is the
least fragile. Only reach for path-based routing when you must serve several apps on a
single hostname (e.g. you have exactly one domain name and want
`example.com/gitlab`, `example.com/grafana`). For GitLab specifically, the port approach
is almost always the right call.

---

## 8. Testing & troubleshooting

Test from the command line first (no browser cache to confuse you):

```bash
# Approach A
curl -skI https://localhost:8443/users/sign_in | head -5    # expect: HTTP/2 200

# Approach B
curl -skI https://localhost/gitlab/users/sign_in | head -5  # expect: HTTP/2 200
```
`-s` silent, `-k` accept the self-signed cert, `-I` headers only.

| Symptom | Cause | Fix |
|---|---|---|
| `curl: (7) Failed to connect ... 172.31.35.68` in nginx error log | WARP not connected | Connect WARP |
| Endless redirect / `ERR_TOO_MANY_REDIRECTS` | `X-Forwarded-Proto https` missing | Add that header line |
| Page loads but **no CSS**, links 404 (Approach B) | GitLab generating root links | Use B1 (`external_url`), not B2 |
| Live CI logs hang | WebSocket lines missing | Add the `map` + `Upgrade`/`Connection` lines |
| 413 Request Entity Too Large on push | body cap | `client_max_body_size 0;` |
| Browser warns "Not secure" | self-signed cert | Expected; click through, or get a real domain |

Watch nginx's own logs while testing:
```bash
sudo tail -f /var/log/nginx/error.log
```

---

## 9. Caveats (the honest limits)

- **Self-signed = browser warnings, forever.** nginx cannot fix trust. Only a real domain
  with a real certificate (e.g. Cloudflare Access — see `CLOUDFLARE_ACCESS_SETUP.md`)
  removes the warning.
- **WARP is still mandatory.** nginx only reaches the box through the tunnel.
- **Per-laptop setup.** This config runs on *your* machine. For the whole team without
  per-laptop work, run the equivalent shim **on the box** (`GITLAB_WARP_PROXY_ADMIN.md`)
  or move to the domain-based setup.
- **The gate is only WARP enrollment.** This proxy adds no per-app login. Company-email
  login per app needs Cloudflare Access (a domain).

---

## 10. Related documents

- `GITLAB_WARP_PROXY_ADMIN.md` — run the same shim **on the box** (port 443) for the team.
- `CLOUDFLARE_ACCESS_SETUP.md` — the clean long-term answer: real HTTPS + company-email
  login, no proxy, no cert warnings (needs a domain on Cloudflare).
- `REMOTE_ACCESS_STRATEGY.md` — the big picture: where this proxy fits and what replaces it.
