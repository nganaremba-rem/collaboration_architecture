# Separating Test from Production + Locking Down Access — Explained Simply

This guide is written for someone who has **not** done cloud/networking before.
It explains *what each thing is*, *why you need it*, and *how to set it up step by step*.
Everything here can be done for **$0**.

---

## Part 0 — The plain-English glossary (read this first)

| Term | What it actually means (analogy) |
|---|---|
| **AWS** | Amazon's cloud. You rent computers (servers) from Amazon instead of buying them. |
| **Server / instance** | A computer running in Amazon's data center. Your GitLab and your test site each run on one. |
| **Production ("prod")** | The **real, live** system your users actually use. It must never go down. |
| **Test environment** | A **copy/sandbox** where you try changes safely. If it breaks, no real users are affected. |
| **VPC** (Virtual Private Cloud) | Think of it as a **private fenced-off building** inside Amazon where your servers live. Servers in the same VPC can talk to each other; the fence keeps others out. |
| **Two separate VPCs** | Two separate buildings. A problem in the "test building" can't leak into the "prod building." This is the separation you want. |
| **AWS Account** | Your whole login/workspace at Amazon. You can have **more than one** (e.g. one for prod, one for test) — even stronger separation than two VPCs, and it's free to add. |
| **Security Group** | A **firewall / door policy** on a server. It decides who's allowed to knock and on which doors (ports). |
| **IP address** | The "street address" of a device on the internet. Your current access rule = "only let in people coming from these addresses." The problem: addresses change (home, office, phone, café), so this is fragile. |
| **GitLab** | Where your team stores and collaborates on code. You run your own copy ("self-managed"). |
| **IdP** (Identity Provider) | The thing that proves **who a person is** when they log in (like "Sign in with Google"). You don't have one yet — that's fine, there's a free way around it. |
| **Zero Trust / Access proxy** | A modern security idea: **trust the verified person, not their IP address.** Before anyone reaches GitLab or the test site, they must prove who they are (log in), every time. |
| **Cloudflare** | A free service that sits **in front of** your sites. It checks identity before letting anyone through, and hides your server from the open internet. This is the tool we'll use. |
| **Tunnel (`cloudflared`)** | A small program on your server that makes a **private outbound connection** to Cloudflare. Because the connection goes *out*, you can **shut every incoming door** — nobody on the internet can reach your server directly anymore. They can only come in through Cloudflare's checkpoint. |

---

## What you're trying to achieve (in one breath)

1. **Keep the test stuff in its own "building" so it can never disturb the live production stuff.**
2. **Stop using IP addresses to control who gets in.** Instead, make each person **log in as themselves**, so access follows the *person*, not their location.
3. **Make sure the test site isn't just sitting open on the internet** for anyone to find.
4. Do all of this **without spending money**.

---

## The recommendation (short version)

- **Separation:** Put the test environment in its **own VPC** (its own building). Better yet, its own **AWS account**. Don't connect it to production. Free.
- **Access control:** Put **Cloudflare** in front of both GitLab and the test site. People log in (even just with an email code or "Sign in with Google") instead of being filtered by IP. The server's incoming doors get **closed**, so it's no longer exposed. **Free for up to 50 users** — and you have under 50.

That's it. Two ideas: *separate the buildings*, and *put a guarded checkpoint with ID checks in front of the doors.*

---

## Part A — Separate Test from Production

### Why
Right now, if test and prod share the same "building" (VPC/account), a mistake or a hack in
test could spill into production. Production can't be stopped, so we **never touch it** — we
just build test somewhere separate.

### The two options

**Option 1 (simplest): A new, separate VPC for test.**
- A second fenced-off building inside your existing AWS account.
- Do **not** "peer" (connect) it to the production VPC.
- Give it its own firewall rules (security groups).

**Option 2 (strongest, also free): A new AWS *account* just for test.**
- Completely separate workspace. Even logins and permissions are separate.
- Use **AWS Organizations** (free) to keep one bill for both accounts.

Either is fine. Option 1 is easier to start with; Option 2 is the gold standard.

### Steps (Option 1 — new VPC)
1. Log in to the **AWS Console** (the website where you manage Amazon's cloud).
2. Search for **"VPC"** in the top search bar → open the VPC service.
3. Click **"Create VPC"** → choose **"VPC and more"** (it sets up the networking pieces for you).
4. Name it something obvious like `test-vpc`. Accept the default sizes if unsure.
5. **Do not** create a peering connection to your production VPC.
6. Launch your **test site** and a **test copy of GitLab** (or test runners) **inside this new VPC**.
7. Leave production exactly as it is. You never stopped or changed it. ✅

> If you'd rather do Option 2 (separate account): in the AWS Console search **"Organizations"**,
> create the organization, then **"Add an AWS account"**, and build the test VPC inside that new
> account using the same steps above.

---

## Part B — Replace IP-based access with "log in as yourself" (Cloudflare)

### Why
"Allow these IP addresses" breaks every time someone's address changes and is easy to get wrong.
Instead, we make people **prove who they are**. Bonus: we also **hide the servers** so they
aren't openly reachable on the internet.

### What we'll set up
- A free Cloudflare account.
- A small program (**`cloudflared`**, the "tunnel") on your GitLab server and on your test-site server.
- Login rules ("only people with these email addresses can enter").

### Steps

**1. Get a domain name pointed at Cloudflare**
   - You need a domain like `yourcompany.com` (if you already have one, good).
   - Make a **free Cloudflare account** at cloudflare.com, click **"Add a site,"** enter your domain,
     and follow the prompts to point your domain's "nameservers" to Cloudflare. (Cloudflare shows
     you exactly what to copy-paste.)

**2. Turn on Zero Trust (the free security dashboard)**
   - In the Cloudflare dashboard, open **"Zero Trust."**
   - Pick the **Free** plan when asked (it covers **up to 50 users**, forever).

**3. Install the tunnel on each server**
   - In Zero Trust → **Networks → Tunnels → "Create a tunnel."**
   - Cloudflare gives you a **copy-paste command** to run on the server. Run it on the **GitLab**
     server, then again on the **test-site** server (each gets its own tunnel).
   - For each tunnel, set the **"Public hostname,"** e.g.:
     - `gitlab.yourcompany.com` → points to GitLab
     - `test.yourcompany.com` → points to the test site

**4. Add the login rules (this is what replaces IP filtering)**
   - In Zero Trust → **Access → Applications → "Add an application" → Self-hosted.**
   - Enter the hostname (e.g. `test.yourcompany.com`).
   - Add a **policy**: "Allow" → and choose **Emails** (list your team's emails) or
     **Emails ending in** `@yourcompany.com`.
   - Repeat for `gitlab.yourcompany.com`.

**5. Choose how people prove who they are (no IdP needed)**
   - In Zero Trust → **Settings → Authentication**, the simplest free option is **"One-time PIN"**:
     a person enters their email, gets a code, types it in. Done — no extra accounts to buy.
   - Optional upgrade (still free): add **"Sign in with Google"** so people use their Google login.

**6. Close the doors (remove the exposure + the old IP rules)**
   - Now that all traffic comes through Cloudflare, go to AWS → your servers' **Security Groups**.
   - **Remove the old IP-allowlist rules**, and **close the public incoming ports** (80/443/22) to
     the internet. The tunnel doesn't need them — it dials *out* to Cloudflare.
   - Result: someone typing your server's raw address into a browser gets **nothing**; the only way
     in is the Cloudflare login page. The test site is no longer exposed. ✅

**7. (Recommended) Require a second factor (MFA)**
   - In the Access policy you can require an extra verification step for stronger security.

---

## Why this tool and not the others (plain English)

- **Tailscale / a VPN:** A VPN is like giving everyone a key to get inside the building network.
  Tailscale's free version now only allows **6 people** (changed April 2026) — too few for your team.
  And a plain VPN trusts the device, not the person.
- **Cloudflare (recommended):** Free for **50 people**, checks the **person** every time, needs **no
  identity system to start**, and **hides your servers**. Best fit for "no budget, under 50 users."

---

## What it costs
- Separate VPC or extra AWS account: **$0** (you only pay for the test server itself, just like any test machine).
- Cloudflare Zero Trust for under 50 people: **$0**, permanently.
- Email-code or Google login: **$0**.
- **Total new cost: nothing.**

---

## How to check it's actually working (do these after setup)
1. **Separation:** From the test server, confirm you **cannot** reach production's private resources.
   Confirm production is still running and unchanged.
2. **Not exposed:** From your phone (off Wi-Fi), type the server's **raw address** — it should be
   **unreachable**. Type `test.yourcompany.com` — it should show the **Cloudflare login page**, not the site.
3. **Login works:** Log in with an **allowed** email → you get in. Try a **random** email → **blocked**.
4. **No more IP dependence:** Log in successfully from a **new location** (phone hotspot) to prove
   access now follows the *person*, not the IP address.

## Two things to keep in mind
- **Git over SSH:** If your team pushes code using SSH (not the website), that needs one extra small
  setup step (`cloudflared access ssh`). Mention this to whoever sets it up.
- **Don't lock yourself out:** Keep one emergency admin way into your servers (AWS "Session Manager"
  works and needs no open doors), so a Cloudflare hiccup never traps you outside.

## Sources
- Cloudflare Zero Trust free plan (up to 50 users): https://www.cloudflare.com/plans/zero-trust-services/
- Tailscale free tier is now 6 users (April 2026): https://tailscale.com/docs/account/manage-plans/free-plans-discounts
