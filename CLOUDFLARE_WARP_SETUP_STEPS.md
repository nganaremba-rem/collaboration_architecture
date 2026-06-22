# 🚀 Setup Steps — Cloudflare WARP (Private Network) for GitLab & the Test Site

> **Approach:** WARP / private-network mode. **No changes to your domain, no
> nameserver move, no DNS risk.** Users install the free WARP app, log in with
> their company email, and reach the servers by their **private address** — from
> any location.
>
> This is the safest option. (The nameserver-move version is in
> `CLOUDFLARE_ZT_SETUP_STEPS.md` — you do **not** need it with this guide.)

> **How it works in one line:** the server runs a small program (`cloudflared`)
> that dials out to Cloudflare; each user's PC runs **WARP** and logs in; once
> logged in, the PC behaves as if it's inside your private network and can reach
> GitLab/test directly. Identity is checked, not IP.

---

## ✅ Why this avoids the risks you worried about

- **No nameserver change** — your domain and public sites are never touched.
- **No 100 MB upload limit** — WARP routes traffic at the network level (not the
  HTTP proxy), so big `git push`, LFS, and Docker images work normally.
- **Free** — same Zero Trust free plan (up to 50 users), OTP login, $0.
- Trade-off: each user installs the **WARP app once** (it's not browser-only).

---

## ⚠️ Before you start — two safety rules

1. **Do not close any AWS ports during this guide.** The old access keeps working
   as a fallback the whole time.
2. **Keep AWS Session Manager** working so you can never lock yourself out.

You'll also need: the **private IPs / subnet** of the GitLab and test servers
(e.g. `10.0.1.0/24`). Get this from AWS (EC2 → instance → Private IPv4).

---

## Step 1 — Create the Cloudflare account & Zero Trust team

1. Sign up at <https://dash.cloudflare.com/sign-up> with the **company shared
   email** (not personal Gmail).
2. Open **Zero Trust** (left menu).
3. Pick a **team name** → becomes `your-team.cloudflareaccess.com`. (This is just
   your login portal — **not** your real domain.)
4. Choose the **Free** plan (up to 50 users, $0).

> 💡 You do **not** add your domain or change nameservers anywhere in this guide.

---

## Step 2 — Set the login method (One-time PIN)

1. Zero Trust → **Settings → Authentication**.
2. Confirm **One-time PIN** is on (default). Users will type their company email
   and get a code.

---

## Step 3 — Decide who may install/enroll WARP

1. Zero Trust → **Settings → WARP Client → Device enrollment permissions →
   Manage / Add a rule**.
2. **Action: Allow**, **Include → Emails ending in** `@company.com` (or paste the
   email list).
3. Save. Now only your team can enroll their PCs.

---

## Step 4 — Run the tunnel on each server (as a private network)

Do this on **each** server (or once if both share the same VPC subnet — then one
tunnel can cover both).

1. Zero Trust → **Networks → Tunnels → Create a tunnel → Cloudflared**.
2. Name it (e.g. `aws-private`) → **Save**.
3. Cloudflare shows an **install command**. **SSH into the server** and run it.
   Wait until the tunnel is **Connected / Healthy**. (`cloudflared` connects
   **outbound only** — no inbound port needed.)
4. Open the tunnel → **Private Network** tab → **Add a private network** → enter
   the server's **private CIDR**, e.g. `10.0.1.0/24` (or a single IP like
   `10.0.1.15/32`).
5. If GitLab and the test site are on different subnets, add each CIDR.

---

## Step 5 — Make WARP route those private IPs (important gotcha)

By default WARP **excludes** private IP ranges, so you must allow your server
range through:

1. Zero Trust → **Settings → WARP Client → Split Tunnels**.
2. If mode is **Exclude IPs** (default): find any entry covering your server range
   (e.g. `10.0.0.0/8`) and **remove it**, OR switch to **Include** mode and add
   only your server CIDR.
3. Save. This is what lets enrolled PCs actually reach the private servers.

---

## Step 6 — Install WARP on a user device & enroll

1. Download **Cloudflare WARP** (the 1.1.1.1 app) on the PC:
   <https://1.1.1.1/> (Windows / macOS available).
2. Open it → **Settings → Account → Login with Cloudflare Zero Trust**.
3. Enter the **team name** from Step 1 → a browser opens → enter **company email**
   → type the **PIN** from email → done.
4. WARP shows **Connected**.

---

## Step 7 — Test

From the **enrolled** device:
1. Open a browser to `https://<GitLab-private-IP>` → GitLab loads.
2. `git clone` / `git push` over HTTPS or SSH to the private IP → works (no 100 MB
   limit).
3. Open the **test site** by its private IP → loads.

Then confirm the gate:
4. From a device **not** running WARP / not logged in → the private IPs are
   **unreachable**.
5. Log in from a **different network** (phone tethering) → still works → confirms
   it's **IP-independent** (the dynamic-IP fix).

---

## Step 8 — (Optional) Use names instead of raw IPs

So people type `gitlab` instead of `10.0.1.15`:
- Add an **internal DNS record** in your VPC (Route 53 private hosted zone), or
- Use Zero Trust **Gateway → Resolver / local domains** to map a name to the
  private IP.

Optional comfort only — raw private IPs already work.

---

## Step 9 — (Optional) Tighten which user reaches which server

Basic setup: any enrolled `@company.com` user can reach the private network. To
limit specific people to specific servers/ports:
- Zero Trust → **Gateway → Firewall policies → Network** → allow/deny by user
  group + destination IP/port.

---

## Step 10 — Later hardening (when ready)

- 🔒 **Close the ports:** once everyone uses WARP, close inbound `80/443/22` in the
  AWS Security Group and remove old IP-allow rules. **Keep AWS Session Manager.**
- 🛡️ **Add MFA:** require a **passkey / Touch ID / Windows Hello** (free) via a
  device-posture or Access rule, or add Google login for SSO.

---

## ✅ Final verification checklist

- [ ] Public sites + email unaffected (you never touched DNS — confirm anyway).
- [ ] Tunnel shows **Healthy**; private network CIDR added.
- [ ] Enrolled + logged-in PC reaches GitLab and the test site by private IP.
- [ ] Non-enrolled PC cannot reach them.
- [ ] Works from a different network / changed IP.
- [ ] `git push` / `pull` works (large files OK — no 100 MB limit on WARP).
- [ ] AWS Session Manager emergency access confirmed.

---

## 🆘 If something goes wrong

- **Can't reach the server after connecting WARP:** the Split Tunnel step (Step 5)
  is the usual cause — the private range is still excluded. Fix the exclude/include
  list.
- **WARP won't log in:** check the device-enrollment rule (Step 3) includes the
  email; check the team name is correct.
- **Locked out of a server:** use **AWS Session Manager**.
