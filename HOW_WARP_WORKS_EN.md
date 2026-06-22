# 🔌 How Cloudflare WARP Works — in plain language

> A simple, no-jargon explanation of how people will reach GitLab and the test
> site after the WARP setup. Read top to bottom — each part builds on the last.

---

## 🎯 The problem we're solving

- Your servers live in Amazon (AWS). Today people reach them by **IP address** (a
  street address on the internet).
- That address **keeps changing** — your team's address changes when they move
  (home, office, café, phone), and the server's address can change when it
  restarts. "Allow only this address" breaks constantly.
- Leaving the server open to the whole internet so any address can reach it is
  risky.

**The idea:** stop trusting *addresses*. Trust the *person* instead. You log in
once with your company email, and then you can reach the servers from anywhere.

---

## 🏠 The analogy (remember this and the rest is easy)

Picture your server as a **shop inside a locked warehouse** with no public door.

- 🚇 **The tunnel (`cloudflared`)** — a small program on the server. It digs a
  **private corridor from the inside outward** to a guarded checkpoint. Because
  it's dug from the inside, the warehouse never needs a public door.
- 🛂 **Cloudflare** — the **guarded checkpoint**. It has branches all over the
  world, so there's always one near you. Nobody passes without showing ID.
- 📞 **The WARP app** (on each person's laptop) — a **special phone line** that
  connects your laptop to that same checkpoint.
- 🪪 **The login (email + PIN)** — showing your **ID at the checkpoint**.

When your ID checks out, the checkpoint connects **your laptop's line** to **the
shop's corridor**. Now you're talking to the shop as if you were standing inside
the warehouse — no matter where you actually are.

---

## 🧩 The four pieces

| Piece | What it is | Where it lives |
|---|---|---|
| **`cloudflared` (tunnel)** | Small program that connects the server outward to Cloudflare | On each server |
| **Cloudflare** | The global checkpoint / switchboard in the middle | The internet (Cloudflare's network) |
| **WARP app** | App that connects a person's PC to Cloudflare | On each user's laptop |
| **Login (one-time PIN)** | Email + a code, proves who you are | Done once per session |

---

## 🚶 What actually happens, step by step

When a teammate opens GitLab:

1. **The server already dug its corridor.** `cloudflared` keeps a steady
   outward connection to Cloudflare. (Set up once; reconnects by itself.)
2. **The person turns on WARP and logs in.** They type their company email, get a
   code by email, type it. ✅ ID confirmed.
3. **Their laptop is now linked to the checkpoint.** It behaves as if it's sitting
   inside the company's private network.
4. **They open GitLab by its private address (or an internal name).** The request
   travels: laptop → WARP → nearest Cloudflare checkpoint → the server's corridor
   → GitLab.
5. **GitLab answers back the same way.** To the user it just feels like a normal
   website. They work as usual — browse, `git push`, `git pull`.

---

## 💡 Why this fixes the changing-IP problem

- **You're identified by your login, not your address.** Your IP can change every
  day — you still get in, because the checkpoint checks *you*, not where you are.
- **The server's address can change too.** The server reaches the checkpoint
  through its **own outward corridor**, so its public IP doesn't matter. And we
  point WARP at the server's **whole private neighborhood (subnet)**, not one exact
  house — so even if the server's private address shifts, it's still found.
- **After a restart, the corridor re-digs itself.** `cloudflared` runs as a
  background service and reconnects automatically.

---

## 🔒 Why it's safer

- The server **never opens a public door.** The only way in is through the
  checkpoint, and the checkpoint demands ID. Random people scanning the internet
  find nothing to knock on.
- Later, you can **fully lock the old doors** (close the AWS ports) and the
  checkpoint is the *only* way in.
- (For now we leave the old doors open so nothing breaks during the switch — both
  ways work in parallel.)

---

## ⚖️ The trade-offs (honest)

- 👍 Free for up to 50 people. No domain or DNS changes. No 100 MB upload limit
  (big files and Docker images work fine).
- 👎 Each person must **install the WARP app once** and log in (it's not just a
  browser). After that it's mostly invisible.
- 👎 Access now depends on Cloudflare being up (like depending on any one service).

---

## 🧠 One-sentence summary

> The server quietly connects *out* to a global checkpoint; each person connects
> their laptop to the same checkpoint and shows ID; the checkpoint links them
> together — so the **right person reaches the server from anywhere, even as
> addresses change, without leaving the server exposed.**

---

# ⚙️ Configuration steps (how to set it up)

> Assumes you already have: a company email for Cloudflare, SSH + sudo on both
> servers, the servers' private IP/subnet, and the list of allowed emails.

> ⚠️ **Two safety rules:** (1) Don't close any AWS ports during setup — the old
> access stays as a fallback. (2) While ports stay open you can always SSH in the
> normal way, so you can't get locked out.

## Step 1 — Create the Cloudflare account & Zero Trust team
1. Sign up at <https://dash.cloudflare.com/sign-up> with the **company shared
   email** (not personal Gmail).
2. Open **Zero Trust** (left menu).
3. Pick a **team name** → becomes `your-team.cloudflareaccess.com` (this is just
   your login portal, **not** your real domain).
4. Choose the **Free** plan (up to 50 users, $0).

> 💡 You never add your domain or change nameservers anywhere in this guide.

## Step 2 — Set the login method (One-time PIN)
1. Zero Trust → **Settings → Authentication**.
2. Confirm **One-time PIN** is on (default). Users type their company email and
   get a code.

## Step 3 — Decide who may install/enroll WARP
1. Zero Trust → **Settings → WARP Client → Device enrollment permissions →
   Manage / Add a rule**.
2. **Action: Allow**, **Include → Emails ending in** `@company.com` (or paste the
   list).

## Step 4 — Run the tunnel on each server (as a private network)
1. Zero Trust → **Networks → Tunnels → Create a tunnel → Cloudflared**.
2. Name it (e.g. `aws-private`) → **Save**.
3. Copy the **install command** → **SSH into the server** and run it. Wait for
   **Connected / Healthy**. (It connects **outbound only** — no inbound port.)
4. Open the tunnel → **Private Network** tab → **Add a private network** → enter
   the server's **private CIDR** (e.g. `10.0.1.0/24`). Route the **subnet**, not a
   single IP, so it keeps working if the IP changes.
5. Make sure `cloudflared` **auto-starts on boot** (it installs as a service) so
   it reconnects after the daily restart.

## Step 5 — Make WARP route those private IPs (important gotcha)
WARP **excludes** private ranges by default, so allow your server range through:
1. Zero Trust → **Settings → WARP Client → Split Tunnels**.
2. If mode is **Exclude** (default): remove any entry covering your range (e.g.
   `10.0.0.0/8`), OR switch to **Include** and add only your server CIDR.

## Step 6 — Install WARP on a user device & enroll
1. Download **Cloudflare WARP** (1.1.1.1 app): <https://1.1.1.1/> (Windows/macOS).
2. Open it → **Settings → Account → Login with Cloudflare Zero Trust**.
3. Enter the **team name** → browser opens → enter **company email** → type the
   **PIN** from email. WARP shows **Connected**.

## Step 7 — Test
1. From the enrolled PC: open `https://<GitLab-private-IP>` → GitLab loads.
2. `git clone` / `push` over HTTPS or SSH → works (no 100 MB limit).
3. A PC **not** running WARP → can't reach it.
4. Log in from a **different network** (phone tethering) → still works (proves
   it's IP-independent).

## Step 8 — (Optional) Use names instead of raw IPs
So people type `gitlab` not `10.0.1.15`:
- Add an **internal DNS record** (Route 53 private hosted zone), or
- Use Zero Trust **Gateway → Resolver / local domains**.
This also shields users if the server's private IP ever changes.

## Step 9 — (Optional) Limit who reaches which server
- Zero Trust → **Gateway → Firewall policies → Network** → allow/deny by user
  group + destination IP/port.

## Step 10 — Later hardening (when ready)
- 🔒 **Close the ports** (inbound `80/443/22`) once everyone uses WARP, and remove
  old IP-allow rules. Keep an emergency path (AWS Session Manager) **before** doing
  this — it's only needed once ports are closed.
- 🛡️ **Add MFA:** require a passkey / Touch ID / Windows Hello (free), or add
  Google login for SSO.

## ✅ Final checklist
- [ ] Public sites + email unaffected (you never touched DNS).
- [ ] Tunnel **Healthy**; private subnet added; `cloudflared` auto-starts.
- [ ] Enrolled + logged-in PC reaches GitLab and the test site.
- [ ] Non-enrolled PC cannot reach them.
- [ ] Works from a different network / changed IP.
- [ ] `git push` / `pull` works (large files OK).
