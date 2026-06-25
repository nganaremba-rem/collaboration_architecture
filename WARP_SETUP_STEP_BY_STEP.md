# 🪜 WARP Setup — Step by Step (Click by Click)

> A complete, beginner-friendly guide to setting up **Cloudflare WARP** for the
> **dev servers (GitLab + test site)** — written so you can follow each click.
>
> 🎯 **Goal:** let your team reach the dev servers by **logging in** (not by IP
> address), **without ever giving WARP a path into production.**
>
> You will touch three places: **AWS**, the **dev server** (over SSH), and
> **Cloudflare**. Follow top to bottom. Don't skip the safety notes.

---

## ✅ Before you start

Have these ready:

- [ ] An **AWS Console login** (to see the dev server's address).
- [ ] A **company email** to make the Cloudflare account (not a personal Gmail).
- [ ] **SSH access** to the dev server (you can log into it from a terminal).
- [ ] About **30–45 minutes**, and a coffee ☕.

### ⚠️ Two safety rules (read once)

1. **Do NOT close any AWS ports during this guide.** Your current access keeps
   working the whole time as a fallback. Nobody gets locked out.
2. **Keep AWS Session Manager working** — it's your emergency way into the server
   if anything ever goes wrong.

> 🧭 **Dashboard note (read once):** Cloudflare renamed **Zero Trust** to
> **Cloudflare One**, and moved most settings around. The left menu may say either
> name. The big change: the old **Settings → WARP Client** and **Settings →
> Authentication** pages are gone. Those controls now live under **Team &
> Resources → Devices**, and tunnels live under **Networks → Connectors**. This
> guide uses the **current** paths. If a menu name looks slightly different, match
> on the words in **bold** — Cloudflare tweaks labels often.

---

## 🟦 Part A — Find your dev server's private address (in AWS)

We need the dev server's **private IP** (its address *inside* AWS, like `10.0.1.15`).

1. Go to **<https://console.aws.amazon.com>** and log in.
2. In the top search bar, type **EC2** → click **EC2**.
3. Left menu → click **Instances**.
4. Click the **dev server** (GitLab / test) in the list.
5. In the panel below, open the **Networking** tab.
6. Find **Private IPv4 address**. Write it down. Example: `10.0.1.15`.

> 📝 **Write it here:** dev private IP = `________________`

✅ **You should see:** a number like `10.0.1.x`. That's the address WARP will open.

---

## 🟦 Part B — (Recommended) Make sure that address won't change

Good news about AWS:

- The **private IP stays the same** when you **reboot or stop/start** the server.
- It only changes if the server is **deleted and rebuilt** from scratch.
- (The *public* IP changes on stop/start — but WARP doesn't use that one.)

So for normal restarts, **you're already fine** — the private IP is stable.

👉 **Only if** your team rebuilds the dev server often (not just reboots), do this
so the address is locked forever:

1. EC2 → **Instances** → click the dev server.
2. **Actions → Networking → Manage IP addresses**.
3. Under the network interface, **Assign a private IP** and tick **"Allow me to
   choose"**, then type the exact address you wrote in Part A.
4. **Save**.

✅ **You should see:** the same private IP listed as assigned. It won't drift now.

---

## 🟦 Part C — Create the Cloudflare account + Zero Trust team

1. Go to **<https://dash.cloudflare.com/sign-up>**.
2. Sign up with the **company email** → confirm the email.
3. After login, open the left menu → click **Zero Trust**.
4. It asks for a **team name** → pick something simple, e.g. `yourcompany`.
   - This becomes your login page: `yourcompany.cloudflareaccess.com`.
   - ⚠️ This is **just a login portal name** — it is **not** your real website
     domain, and it does **not** change your DNS.
5. Choose the **Free** plan (up to 50 users, $0). Confirm.

> 💳 **Heads-up:** during this signup Cloudflare often asks you to **add a payment
> method** (credit card, PayPal, or Google Pay) to confirm the plan — even for the
> **Free $0** plan. Adding it does **not** charge you. As long as you pick **Free**,
> you are **never billed** unless you deliberately upgrade later. The card is just
> kept on file.

✅ **You should see:** the Zero Trust dashboard open.

---

## 🟦 Part D — Confirm the login method (One-time PIN)

Good news: **nothing to switch on here.** One-time PIN is the **default** — when no
identity provider is connected, people log in by typing their email and getting a
code. You only need to *visit* this screen to confirm it.

1. Zero Trust → left menu → **Team & Resources** → **Devices**.
2. Click the **Management** tab (top of the page, next to "Device profiles").
3. In the **Device enrollment** section, find **Device enrollment permissions**.
   The text right there confirms: *"By default, users can log in with a one-time
   pin."* — that's your proof OTP is on. Then click **Manage**.
4. Open the **Login methods** tab. You'll see the **Authentication** panel (it now
   has a **`New`** badge and **Identity / MFA** tabs — Cloudflare redesigned this
   screen in mid-2025, so the old "one-time pin" sentence is no longer printed here).

✅ **You should see:** the Authentication panel with **"Accept all available
identity providers"** toggled **ON**. That ON state includes One-Time PIN, so OTP
login works. **Nothing to change here — do NOT click Save.**

> 💡 Want extra login options later (Google, Microsoft, etc.)? Those are added under
> **Integrations → Identity providers** — not needed for this guide.
>
> ⚠️ **If WARP login shows only "Sign in with Cloudflare" (no email box):** Cloudflare
> now connects a managed **Cloudflare** identity provider by default on new accounts.
> While it is present, email **One-Time PIN is hidden**. To get email OTP back, go to
> **Integrations → Identity providers**, open the **⋯** menu on **Cloudflare**, and
> **Delete** it. With no identity provider connected, OTP becomes the default again.
>
> 💡 Leave this **Manage** screen open — **Part E** is the next tab over.

---

## 🟦 Part E — Decide who is allowed to join

This stops random people from enrolling their laptop.

1. Same place as Part D: Zero Trust → **Team & Resources** → **Devices** →
   **Management** tab → **Device enrollment permissions** → **Manage**.
2. Open the **Policies** tab.
3. Click **Create new policy** (or **Add current policies**, if shown).
4. **Policy name:** `Company staff`.
5. **Action:** `Allow`.
6. Under **Include**, choose **Emails ending in** → type `@yourcompany.com`.
   - (Or choose **Emails** and paste each person's email.)
7. **Save**.

✅ **You should see:** your new policy listed. Only those emails can now enroll a
device. (At least one policy is required — without it, nobody can join.)

> ⚠️ **Do not skip or leave this unsaved.** If there is **no saved enrollment
> policy**, WARP login fails with **"Enrollment request is invalid"** — even though
> everything else looks correct. If you hit that error, come back here and confirm a
> policy is listed (not just drafted in the builder).

---

## 🟦 Part F — Install the tunnel on the dev server

The tunnel is a small program (`cloudflared`) that connects the server **outward**
to Cloudflare. No incoming door is opened on the server.

1. Zero Trust → left menu → **Networks** → **Connectors**.
2. Click **Add a tunnel**.
3. Choose **Cloudflared** → **Next**.
4. **Name** it `dev-tunnel` → **Save tunnel**.
5. Cloudflare now shows an **install command**. Choose your server's system (e.g.
   **Debian/Ubuntu 64-bit**). **Copy** the command shown.
6. **SSH into the dev server** from your terminal:
   ```
   ssh your-user@<dev-server>
   ```
7. **Paste and run** the command Cloudflare gave you (it starts with
   `curl ...` or `cloudflared service install ...`). Press Enter.
8. Back in the Cloudflare page, wait until the tunnel shows
   **Connected / 🟢 Healthy**.

✅ **You should see:** 🟢 **Healthy** next to your tunnel. (If not, re-check you
pasted the full command on the right server.)

---

## 🟥 Part G — Route ONLY the dev server (the important isolation step)

This is the step that **keeps production safe.** We tell WARP about the **one dev
address only** — never the whole network.

1. Zero Trust → **Networks** → **Routes**.
2. Stay on the **CIDR** tab → click **Add a route**.
3. In the **CIDR** box, enter the dev IP **as a single address** by adding `/32`:
   ```
   10.0.1.15/32
   ```
   👉 Use **your** address from Part A, followed by **`/32`**.
4. (Optional) **Description:** `dev server`.
5. For **Tunnel**, pick your `dev-tunnel` from the dropdown.
6. **Save**.

> 🛑 **Do NOT enter the whole range** like `10.0.1.0/24`. That would open the entire
> neighbourhood — and because the tunnel can reach **any host in the range it's
> given**, WARP users could then reach **production servers** sitting in the same
> network. The `/32` means "this one machine only."

✅ **You should see:** exactly one entry, ending in `/32`. Nothing wider.

---

## 🟦 Part H — Tell WARP to actually carry that address

By default, WARP **ignores** private addresses (like `10.x`). We must let our one
dev address through.

1. Zero Trust → **Team & Resources** → **Devices** → **Device profiles** tab.
2. Under **General profiles**, click the profile name **Default** → in the panel
   that slides in, click **Edit**.
3. Scroll down to the **Split Tunnels** section. You'll see four radio buttons:
   - **If "Exclude IPs and domains" is selected (the default):**
     click **Manage**, find any entry covering your address (e.g. `10.0.0.0/8`) and
     **remove just that one** — OR switch the mode to **Include** (next option).
   - **If you select "Include IPs and domains":**
     click **Manage**, remove the broad defaults, and **Add** only your dev address:
     `10.0.1.15/32`.
4. **Save** (top or bottom of the profile page).

> 💡 Simplest safe choice: switch to **Include** mode and add **only** your
> `/32` dev address. Then WARP carries *just* that — nothing else private.

✅ **You should see:** your dev `/32` address in the list (Include), or the broad
private range removed (Exclude).

---

## 🟥 Part I — (Extra safety) Block production explicitly

Belt-and-suspenders: add a rule that **denies** production, so even a future mistake
can't reach it.

1. Zero Trust → **Traffic policies** → **Firewall policies**.
2. Open the **Network** tab (next to DNS / HTTP) → click **Add a policy**.
3. **Name:** `Block production`.
4. **Action:** `Block`.
5. Build the rule: **Destination IP** → **in** → type your **production**
   address or range (e.g. `10.0.2.0/24` — the production subnet).
6. **Save**, then make sure the policy's **Status** toggle is **On** (and the policy
   sits **above** any allow rules — they run top to bottom).

✅ **You should see:** a `Block production` policy listed and enabled.

> 🔎 Don't know production's address? Find it the same way as Part A (EC2 →
> production instance → Networking → Private IPv4), then use its subnet.

---

## 🟦 Part J — Install WARP on a laptop & log in

Do this on each team member's computer.

1. Go to **<https://1.1.1.1/>** and download **Cloudflare WARP** (Windows / macOS).
2. Install and **open** the app.
3. Click the **gear ⚙️ (Settings)** → **Account** → **Login with Cloudflare Zero
   Trust**.
4. Type your **team name** from Part C (e.g. `yourcompany`).
5. A browser opens → type your **company email** → you get a **code** by email →
   type the code.
6. Back in the app, toggle WARP **On**.

✅ **You should see:** WARP says **Connected**.

---

## 🟩 Part K — Test (prove it works AND prove production is safe)

From the **enrolled laptop** (WARP Connected):

1. Open a browser to `https://10.0.1.15` (your dev IP) → **GitLab / test loads** ✅
2. Try `git clone` / `git push` to the dev server → **works** ✅
3. Open the **production reservation system** by its normal web link →
   **still loads as usual** ✅ (WARP doesn't block it)

Now prove the safety boundaries:

4. Try to reach a **production** private address (e.g. `https://10.0.2.20`) →
   **should NOT connect** ✅ (blocked by Part G's `/32` + Part I's deny rule)
5. On a laptop **without WARP / not logged in**, try the dev IP →
   **cannot reach it** ✅
6. Turn on phone hotspot (a different network) on the enrolled laptop → dev still
   works ✅ (proves it no longer depends on IP)

✅ If all six match, you're done.

---

## 🟦 Part L — Later hardening (optional, when ready)

Only after everyone is comfortably on WARP:

- 🔒 **Close the old doors:** in the AWS Security Group, close inbound `80/443/22`
  for the dev server and remove the old IP-allowlist rules.
  **Keep AWS Session Manager** as your emergency entrance first.
- 🛡️ **Add MFA:** require a passkey / Touch ID / Windows Hello (free), or add
  Google login, for an extra layer.

---

## 🟧 Part M — Reaching a site that is **public + IP-allowlisted** (important gotcha)

This is the trap that wastes the most time. Read it if your dev site has a **public
web address** (e.g. behind an **AWS load balancer / ALB**) that is locked down by
**whitelisting your office's static IP**.

**The problem in one sentence:** when WARP is on, your traffic leaves through a
**Cloudflare IP**, *not* your office IP — so the allowlist blocks you. It works in
the office *without* WARP only because you come from the whitelisted office IP.

So the public address (e.g. `gitlab-dev.example.com → 16.x.x.x`) is the **wrong door**
for WARP. WARP only carries the server's **private** address (the `/32` from Part G,
e.g. `172.31.x.x`). Two things must be true to reach the site **by name** over WARP:

1. **The private path must actually work.** Test it with WARP connected:
   ```
   curl -vk --max-time 10 https://<PRIVATE-IP>/
   ```
   - **Any HTTP reply** (even a `302` redirect to the hostname) → the tunnel works;
     go to step 2.
   - **Timeout / no route** → the tunnel is not delivering traffic. This is
     **server-side**, not in the Cloudflare dashboard. Ask whoever runs the server to:
     - enable **`warp-routing: { enabled: true }`** in the `cloudflared` config
       (the dashboard route alone is **not** enough), **and**
     - open the **AWS Security Group** on the dev box to allow the `cloudflared`
       host/SG on `443` (or `80`).

2. **DNS must point the hostname at the private IP for WARP users.** A normal public
   DNS lookup returns the *public* ALB IP, which WARP does not carry — and most apps
   (GitLab included) redirect raw-IP access back to their hostname. Fix it with
   **Cloudflare Zero Trust → Internal DNS** (create a record
   `gitlab-dev.example.com → <PRIVATE-IP>` served only over WARP).
   - ⚠️ **Local Domain Fallback does *not* fix this** for remote users — it hands DNS
     to the device's local resolver, which off-network has no idea what the private IP
     is.

> 💡 If you only have **read-only** access to AWS and **do not control the domain's
> DNS**, you cannot finish steps 1–2 yourself. Send the two bullet points in step 1
> (warp-routing + Security Group) to the AWS owner, and ask the DNS owner for an
> internal record (or to repoint the hostname to the tunnel).

---

## ✅ Final checklist

- [ ] Dev private IP found (Part A) — and pinned if you rebuild servers (Part B).
- [ ] Cloudflare account + Zero Trust team created, Free plan (Part C).
- [ ] One-time PIN login confirmed (Part D).
- [ ] Enrollment limited to company emails (Part E).
- [ ] Tunnel installed on dev server — shows 🟢 **Healthy** (Part F).
- [ ] **Only the `/32` dev address** routed — NOT the whole range (Part G).
- [ ] Split Tunnel lets that one address through (Part H).
- [ ] **Block-production** firewall policy enabled (Part I).
- [ ] WARP installed + logged in on a laptop — **Connected** (Part J).
- [ ] Dev reachable ✅ · production NOT reachable over WARP ✅ · reservation system
      still works ✅ · works on a different network ✅ (Part K).
- [ ] AWS ports still open as fallback (close only later — Part L).

---

## 🆘 If something goes wrong

- **Dev won't load after connecting WARP:** Part H is the usual cause — the address
  is still excluded. Switch Split Tunnel to **Include** and add the `/32`.
- **"Enrollment request is invalid" when logging in to WARP:** there is no **saved**
  device-enrollment policy. Go to Part E and confirm a policy is **listed** (a draft
  in the builder does not count).
- **WARP login shows only "Sign in with Cloudflare", no email box:** delete the
  managed **Cloudflare** identity provider (Part D note) so email OTP returns.
- **Site works without WARP but not with it (public + IP-allowlisted):** see **Part M**
  — you're hitting the public address; WARP only carries the private `/32`.
- **Can't log in to WARP:** check Part E includes your email, and the team name in
  Part J is spelled exactly right.
- **Production *is* reachable (it shouldn't be):** re-check Part G is a `/32` (not
  `/24`), and Part I's block policy is enabled and above any allow rules.
- **Locked out of the server:** use **AWS Session Manager** (that's why we kept it).
