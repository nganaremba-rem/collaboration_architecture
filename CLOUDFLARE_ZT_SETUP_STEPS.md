# 🚀 Setup Steps — Cloudflare Zero Trust for GitLab & the Test Site

> **Goal:** people log in with their company email (one-time PIN) to reach
> `gitlab-company.com` and `test-sites-company.com`. Works from any IP (fixes the
> dynamic-IP problem).
>
> **This guide assumes you already have** (from `INFO_TO_REQUEST_*`):
> - The domain + access to change nameservers at the registrar.
> - A company-owned email to register Cloudflare.
> - SSH + sudo on both servers.
> - The list of allowed emails (or `@company.com` domain).
>
> **Decisions baked in (from our discussion):**
> - Login = **One-time PIN** (email code). Free. No OAuth, no extra account.
> - **Ports stay open for now** → old access keeps working in parallel. No breakage.
> - Only the **2 hostnames** get the login. Public sites stay public.
> - We **test on a temporary hostname first**, then cut over.

---

## ⚠️ Before you start — two safety rules

1. **Do not close any AWS ports during this guide.** The old way keeps working as
   a fallback the whole time.
2. **Keep an emergency entrance** (AWS Session Manager) so you can never lock
   yourself out of a server.

---

## Step 1 — Create Cloudflare account & add the domain

1. Go to <https://dash.cloudflare.com/sign-up>. Sign up with the **company shared
   email** (not personal Gmail).
2. Click **Add a site / domain** → type `company.com` → continue.
3. Pick the **Free** plan.
4. Cloudflare **scans your existing DNS records**. ⚠️ **Important:** carefully
   **check the imported list against your current DNS.** Every existing record
   (website `A`, mail `MX`, `TXT`, etc.) must be present. Add any that are missing
   **now** — this is what prevents breakage.
5. Cloudflare shows you **2 nameservers** (e.g. `xxx.ns.cloudflare.com`). Copy them.

> 💡 For any **public service that should NOT go through Cloudflare's proxy** (e.g.
> a public site with big uploads), set its record to **"DNS only" (grey cloud)**.
> Then it behaves exactly as today.

---

## Step 2 — Change nameservers at the registrar

1. Log into your **domain registrar** (where the domain was bought — GoDaddy,
   Route 53, Onamae, etc.).
2. Find **Nameservers** → replace the current ones with **Cloudflare's 2
   nameservers** from Step 1.
3. Save. Activation takes **a few minutes to ~24 hours**.
4. Cloudflare emails you when the domain is **Active**.
5. ✅ **Verify nothing broke:** open your public website and check email still
   flows. (Because records were imported in Step 1, this should be unchanged.)

> At this point **no login is on anything yet.** Cloudflare is just doing DNS.

---

## Step 3 — Turn on Zero Trust

1. In the Cloudflare dashboard, open **Zero Trust** (left menu).
2. Choose a **team name** (becomes `your-team.cloudflareaccess.com`).
3. Select the **Free** plan (up to 50 users, **$0**). A card may be requested for
   signup but the plan is free.

---

## Step 4 — Confirm the login method (One-time PIN)

1. Zero Trust → **Settings → Authentication**.
2. Under **Login methods**, make sure **One-time PIN** is present (it's on by
   default). Nothing else needed — users will type their email and get a code.

---

## Step 5 — Install the tunnel on each server

Do this **once per server** (GitLab server, test-site server).

1. Zero Trust → **Networks → Tunnels → Create a tunnel**.
2. Choose **Cloudflared** → name it (e.g. `gitlab-tunnel`) → **Save**.
3. Cloudflare shows an **install command**. **SSH into that server** and run it
   (it installs `cloudflared` and connects — outbound only, no inbound port
   needed). Wait until the tunnel shows **Connected / Healthy**.
4. Add a **Public Hostname** for the tunnel — **use a temporary test name first**:
   - **Subdomain:** `gitlab-zt`  → **Domain:** `company.com`
   - **Service:** `HTTP` → `localhost:80` (or whatever port GitLab listens on;
     `https://localhost:443` if it serves HTTPS locally).
5. Repeat Steps 1–4 on the **test-site server** with `test-sites-zt.company.com`.

> 💡 `cloudflared` connects **outbound** to Cloudflare, so the servers' inbound
> ports can stay open now and be closed later with no change to this setup.

---

## Step 6 — Put the login in front (Access application + policy)

Do this for **each** test hostname.

1. Zero Trust → **Access → Applications → Add an application → Self-hosted**.
2. **Application name:** `GitLab` → **Session duration:** e.g. `24 hours` (or `1
   week` for fewer logins).
3. **Public hostname:** `gitlab-zt.company.com`.
4. **Add a policy:**
   - **Name:** `Allow team`
   - **Action:** `Allow`
   - **Include →** either **Emails ending in** `@company.com`, **or** **Emails**
     and paste the allowed list.
5. Save. Repeat for `test-sites-zt.company.com`.

---

## Step 7 — Test with the dynamic-IP team

1. Open `https://gitlab-zt.company.com` → you should see the **Cloudflare login**.
2. Enter a **company email** → get the **PIN by email** → type it → you're in.
3. Try a **non-allowed email** → should be **blocked**.
4. Ask a teammate to log in **from a different network (phone tethering)** → still
   works → confirms it's **IP-independent** (the dynamic-IP fix).
5. Test `test-sites-zt.company.com` the same way.
6. For GitLab: test a normal `git clone` / `push` over HTTPS through the new
   hostname after logging in via browser once.

---

## Step 8 — Cut over to the real hostnames

Once the test hostnames work:

1. In each tunnel, **add a Public Hostname** for the real name
   (`gitlab-company.com`, `test-sites-company.com`) → same local service.
2. In **Access → Applications**, add an app + policy for each **real** hostname
   (same as Step 6).
3. Tell the team the real addresses.
4. ✅ The **old IP-allowlist path is still alive** (ports open) as a fallback — no
   one is cut off.

---

## Step 9 — Later hardening (optional, when ready)

- 🔧 **Git large files / Docker registry / >100 MB pushes:** the Cloudflare proxy
  rejects single HTTP uploads over 100 MB. If you hit this, switch git to **SSH**
  (`cloudflared access ssh`) or push large artifacts to **S3** directly.
- 🔒 **Close the ports:** once everyone uses the Cloudflare hostnames, close
  inbound `80/443` (and `22` if used) in the AWS Security Group and delete the old
  IP-allow rules. **Keep AWS Session Manager** as the emergency entrance.
- 🛡️ **Add MFA:** require a **passkey / Touch ID / Windows Hello** in the Access
  policy (free, no hardware to buy), or connect Google login for SSO.

---

## ✅ Final verification checklist

- [ ] Public website + email still work after nameserver change.
- [ ] `gitlab-company.com` / `test-sites-company.com` show the Cloudflare login.
- [ ] Allowed email gets in; unknown email is blocked.
- [ ] Login works from a new location / changed IP.
- [ ] `git push` / `pull` works after login (watch the 100 MB limit).
- [ ] AWS Session Manager emergency access confirmed working.

---

## 🆘 If something goes wrong

- **Locked out of the web app:** check the Access policy email rule; check the
  tunnel is **Healthy**; you can delete the Access app to remove the gate
  temporarily.
- **Locked out of a server:** use **AWS Session Manager**.
- **Public site broke after NS change:** a DNS record was missed in Step 1 — re-add
  it in Cloudflare DNS, or set it to **grey cloud (DNS only)**.
