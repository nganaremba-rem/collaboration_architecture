# 🗺️ Remote Access — Strategy, Status, Constraints & Options

A single source of truth: what we want, what works today, what's blocking us, and
how it would be solved with/without money and with/without a domain. Written plainly.

---

## 1. What we want to achieve

Let staff reach our **internal AWS apps from anywhere** (working from home), logging
in with the **company email**, while keeping those apps **off the public Internet** —
replacing the current "whitelist the office static IP" approach, which breaks the
moment someone is not in the office.

Apps in scope:
- **Web/HTTP** — GitLab, dashboards, internal admin tools.
- **SSH** — terminal to servers.
- **RDP** — Windows remote desktop.
- **Databases / raw TCP** — DB clients, etc.

Requirements: **secure**, ideally **free**, easy enough for non-technical staff.

---

## 2. What is working right now (verified this session)

| Piece | State |
|---|---|
| Cloudflare Zero Trust (free), team `l3-solution` | ✅ Working |
| WARP device enrollment | ✅ Policy: allow emails ending `@indiasolarwarehousing.jp` |
| Login method | ✅ Email One-Time PIN (the managed Cloudflare IdP was removed so OTP shows) |
| Tunnel `gitlab-tunnel` (runs **on** the GitLab box `172.31.35.68`) | ✅ Healthy (1/1) |
| Private route `172.31.35.68/32` + WARP split-tunnel **Include** | ✅ WARP reaches the box |
| GitLab reachable over WARP | ✅ Proven — returns HTTP 200 on port **80** with `X-Forwarded-Proto: https` |
| **Access to GitLab in a browser, today** | ✅ Local HTTPS proxy on the laptop: `https://localhost:8443` → GitLab over WARP (files in `~/gitlab-warp-proxy/`) |
| Team option (no per-user proxy) | ⏳ Admin runs the same shim on the box port 443 — see `GITLAB_WARP_PROXY_ADMIN.md` |

**Key technical facts discovered:**
- GitLab on the box serves plain **HTTP on port 80** (also 8060). **No 443 on the box** —
  HTTPS is terminated by the **AWS load balancer** (public IP `16.76.20.97`,
  hostname `gitlab-dev.l3-solution.com`, locked to the office IP).
- GitLab **forces HTTP→HTTPS**, so hitting the raw IP in a browser bounces to a
  non-existent `https://172.31.35.68` → `ERR_ADDRESS_UNREACHABLE`.
- GitLab serves **any hostname** (no `external_url` lock) as long as the request looks
  like HTTPS — so a proper TLS front-end "just works" with **no GitLab change**.

---

## 3. Constraints we are working under

1. **Full access to the GitLab instance.** The operator installed `cloudflared` on the
   box themselves and can make server-side changes there (shell/sudo). Broad AWS-console
   actions (security groups, load balancers) may be limited, but the dev box itself is in
   reach — e.g. the box-side 443 shim can be done without waiting on anyone.
2. **No control of the production domain's DNS.** Someone else manages it; we cannot
   change records (this is what `DNS_MIGRATION_QUESTIONS.md` is for).
3. **Won't move the existing/production domain to Cloudflare** without the owner doing it
   properly — risky if records aren't copied first (see §4).
4. **Prefer no cost** — free tier; reluctant (not unwilling) to buy a cheap domain.
5. **Technical operator, small team.** Comfortable with shell/SSH/running commands; the
   free plan (≤50 users) is plenty.
6. Cloudflare free plan caps: **50 users**, **24h** log retention, **3 locations**.

---

## 4. Risks

- **Moving the production domain to Cloudflare** = changing its nameservers, making
  Cloudflare authoritative for the **whole** domain (website **and email/MX**). If every
  record isn't copied first, production and email **break**. High risk → avoid.
- **Self-signed certs** (current proxy / box shim) = browser warnings; users must click
  through. Not ideal for a non-technical team at scale.
- **Per-user local proxy** = doesn't scale, fragile (no WebSocket/live-log features),
  each laptop needs setup.
- **Raw-IP access** = ugly, and the "who can get in" gate is only WARP enrollment, not a
  per-app login.
- **Dependency on the admin** for server-side hardening (closing the public ALB / IP
  allowlist) and for any box-side change.
- Free-plan limits above; if the team grows past 50 users, it becomes paid ($7/user/mo).

---

## 5. The core blocker (why this is fiddly)

Something must **terminate HTTPS** on a name the browser trusts. Only three things can:

| TLS terminator | Needs | Notes |
|---|---|---|
| The **public ALB** | office IP | The thing we're trying to escape |
| **Cloudflare** (Access) | a **domain on Cloudflare** | Cleanest; gives company-email login |
| The **GitLab box itself** | admin adds a 443 listener + cert | Free, but self-signed/raw-IP, needs admin |

Everything below is just different answers to "where does TLS terminate, and how do users
prove they're staff."

---

## 6. How it would be done — by scenario

### A. No constraints / money is fine (the clean answer)
**Cloudflare Zero Trust + a registered domain** (a separate one, ~$10/yr — trivial):
- **Tunnel** into the VPC (already have one) + **public hostname** per app.
- **Access** policy = allow `@company.com` (email OTP, or Google/Microsoft SSO).
- **WARP** only for databases / raw-TCP tools.
- Result: open a URL → company login → in, from anywhere, nothing public, no cert
  warnings. Add each new app in minutes. See `CLOUDFLARE_ACCESS_SETUP.md`.
- (Alternative paid/native: **AWS Verified Access**, or **ALB + Cognito/OIDC** — both
  keep things in AWS but cost money and/or leave apps public-but-auth-gated.)

### B. No money, **using the existing domain**
Put the domain (or a subdomain) on **Cloudflare**, then same as A but **$0** (you already
own the domain):
- **Whole domain on Cloudflare** = the risky migration in §4 — only do it if the DNS
  owner properly copies all records first. Production-affecting; needs the DNS owner.
- **Subdomain only** (e.g. `apps.l3-solution.com`) would be ideal and production-safe —
  **but** Cloudflare's subdomain-as-its-own-zone feature is **Enterprise-only**, so it is
  **not available on the free plan**. So in practice "use the existing domain" on free =
  the whole-domain migration, which we're avoiding.
- Net: this path is **blocked** for us today (no DNS control + don't want the risky move).

### C. Current reality — no domain, no money, read-only AWS (what we built)
**WARP private access + a local/box HTTPS shim:**
- **Today, per user:** the local proxy → `https://localhost:8443` (working now).
- **For the team:** admin runs the shim on the box port 443 → everyone opens
  `https://172.31.35.68` over WARP. See `GITLAB_WARP_PROXY_ADMIN.md`.
- Trade-offs: self-signed cert warnings, raw IP, per-app gating is only WARP enrollment.
- This is the **only fully-free, no-domain path** — it works, but it's the least clean.

---

## 7. If we DO get a domain, is WARP still needed?

**Partly.** A domain (with Cloudflare Access) removes WARP for most things, but not all:

| App type | With a domain + Access | WARP still needed? |
|---|---|---|
| **Web apps** (GitLab, dashboards) | Open URL → login in browser | ❌ No |
| **SSH** | Browser-rendered terminal after login | ❌ No (browser); WARP optional |
| **RDP** | Browser-rendered desktop after login | ❌ No (browser); WARP optional |
| **Databases / raw TCP** (DBeaver, psql, etc.) | Desktop tool → private IP | ✅ **Yes** — WARP carries raw TCP |

**Summary:** with a domain, **web + SSH + RDP need no client** (browser only). **WARP is
still required for databases and any non-browser raw-TCP tool.** So we keep WARP, but it
shrinks to "just the database/raw-TCP use case" instead of being the whole solution.

---

## 8. Recommendation & next steps

**Ranked:**
1. **Register one cheap separate domain in Cloudflare (~$10/yr).** Removes every blocker
   cleanly (no production risk, no DNS-owner dependency, mostly self-service). Then follow
   `CLOUDFLARE_ACCESS_SETUP.md`. ← best value.
2. **Ask the DNS owner** whether `l3-solution.com` can move to Cloudflare *properly* (all
   records copied). If yes, it's $0 and clean. If no, this stays blocked.
3. **Stay on the free shim** (Option C) for now: per-user proxy today, admin box-shim for
   the team. Accept self-signed warnings + raw IP. Documented and working.

**Immediate, no-decision-needed:** keep using the local proxy
(`~/gitlab-warp-proxy/`, `https://localhost:8443`). It works today.

**One small ask to the admin (independent of the above):** run the box-side shim
(`GITLAB_WARP_PROXY_ADMIN.md`) so the whole team gets `https://172.31.35.68` without each
person running a proxy.

---

## 9. Related documents

- `CLOUDFLARE_ACCESS_SETUP.md` / `_JP.md` — full domain-based Access setup (the clean goal).
- `GITLAB_WARP_PROXY_ADMIN.md` — box-side 443 shim for the team (no domain).
- `~/gitlab-warp-proxy/README.md` — the local per-user proxy (working today).
- `WARP_SETUP_STEP_BY_STEP.md` / `_JP.md` — original WARP enrollment + routing guide.
