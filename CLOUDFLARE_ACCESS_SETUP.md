# 🔐 Company Remote Access — Cloudflare Zero Trust (Free) Setup

**Goal:** reach our internal AWS apps (GitLab, dashboards, servers, databases) **from
anywhere**, log in with the **company email**, and keep everything **off the public
Internet** — replacing the old "office IP whitelist" that breaks when working from home.
All on Cloudflare's **free** plan (up to 50 users, $0).

This guide is self-contained. An admin makes the few AWS changes; every Cloudflare step
is click-by-click so a non-technical person can follow along.

---

## 🧭 The big picture (read this first)

We put one small program (**`cloudflared`**, the "connector") inside our AWS network. It
makes an **outbound-only** connection to Cloudflare — **no inbound ports are opened** on
our servers. Cloudflare then becomes the **front door**: when someone opens an app's URL,
Cloudflare shows a **login page**, checks their **company email**, and only then lets
them through to the app.

**Which method for which kind of app:**

| What you're accessing | How it's done | App on the user's device? |
|---|---|---|
| Web apps (GitLab, Grafana, admin panels) | Tunnel **public hostname** + **Access** login | **None** — just a browser |
| **SSH** (terminal to a server) | **Browser-rendered SSH** + Access | None (in browser) |
| **RDP** (Windows remote desktop) | **Browser-rendered RDP** + Access | None (in browser) |
| **Databases / other raw connections** | **WARP app** + private route + policy | **WARP** installed |

So **web, SSH, and RDP need no install** — users just open a URL and log in. Only
**databases / raw-TCP tools** need the **WARP** app on the laptop.

**Three Cloudflare pieces, all free:**
- **Tunnel** — the secure connector into AWS.
- **Access** — the login gate (company email) in front of each app.
- **WARP** — the device app, only for databases / raw-TCP.

---

## ✅ Before you start (prerequisites)

1. **A domain on Cloudflare.** Access needs a domain that Cloudflare manages. Either move
   our domain's DNS to Cloudflare (free), or delegate a subdomain such as
   `apps.ourcompany.com`. Apps will then live at addresses like
   `gitlab.apps.ourcompany.com`.
2. **A way to log in with company email** (set up in Step A).
3. **An admin with AWS access** to install the connector and later lock the apps down.

> ⚠️ Without a Cloudflare-managed domain, Access cannot put a login page in front of the
> apps. Sort this out first.

---

## 🟦 Step A — Turn on company-email login (you, in Cloudflare)

1. Go to **Zero Trust** (<https://one.dash.cloudflare.com>).
2. **Settings → Authentication → Login methods → Add new.**
3. Choose how staff prove their company email:
   - **Recommended:** **Google Workspace** or **Microsoft Entra / 365** — if our company
     email is Google or Microsoft, this is the cleanest (real single sign-on, ties
     directly to the company account).
   - **Simplest, no setup:** **One-time PIN** — staff type their email and get a code.
4. Click the method → follow the prompts → **Save** → use **Test** to confirm it works.

✅ **You should see:** your login method listed and the Test succeeds. Every app below
will require this login.

---

## 🟦 Step B — Install the connector into AWS (admin does this; you watch the dashboard)

The connector is one small program in our VPC that links AWS to Cloudflare.

1. Zero Trust → **Networks → Connectors → Create a tunnel → Cloudflared**.
2. **Name** it `aws-apps` → **Save tunnel**.
3. Cloudflare shows an **install command**. **The admin** runs it on a small EC2 instance
   (or ECS task) inside the same VPC as our apps. It only makes outbound connections.
4. Wait until the tunnel shows **🟢 Healthy**.
5. For database/raw-TCP access later, the admin enables WARP routing in the connector
   config:
   ```
   warp-routing:
     enabled: true
   ```

✅ **You should see:** the `aws-apps` tunnel **🟢 Healthy**. One connector can serve
**all** our apps.

> 💡 We may already have a healthy tunnel (e.g. `gitlab-tunnel`). It can be reused instead
> of creating `aws-apps`.

---

## 🟩 Step C — Add a WEB app (the repeatable pattern)

Do this once per web app. Example: GitLab.

1. **Publish the hostname.** Networks → **Connectors** → open the tunnel → **Public
   hostname / Published application routes → Add**:
   - **Subdomain/Domain:** `gitlab.apps.ourcompany.com`
   - **Service:** `https://` + the app's **internal** address (the ALB or instance the
     connector can reach inside AWS).
   - **Save**.
2. **Put a login gate in front.** Access → **Applications → Add an application →
   Self-hosted** → enter `gitlab.apps.ourcompany.com` → **Next**.
3. **Add the policy:**
   - **Action:** `Allow`
   - **Include:** **Emails ending in** → `@ourcompany.com`
   - (Optional: also require MFA, or limit to a group.)
   - **Save / Add application.**

✅ **Test:** open `https://gitlab.apps.ourcompany.com` → a Cloudflare login page appears →
sign in with a company email → GitLab loads. A non-company email is **refused**.

🔁 **To add the next web app**, repeat Step C with its own hostname and internal address.

---

## 🟦 Step D — Add SSH or RDP (no client, runs in the browser)

1. On the same tunnel, add a route for the server:
   - **SSH:** add a **browser-rendered SSH / terminal** application (or "Access for
     Infrastructure" for SSH) pointing at the server's internal address + port 22.
   - **RDP:** add a **browser-rendered RDP** application pointing at the Windows host +
     port 3389.
2. Access → attach a policy: **Allow**, **Emails ending in** `@ourcompany.com`.

✅ **Test:** open the app's URL → log in with company email → a **terminal** (SSH) or a
**Windows desktop** (RDP) opens **inside the browser**. No PuTTY / RDP client needed.

> ⚠️ For these browser-rendered apps the policy must be **Allow** or **Block** only
> (Bypass / Service Auth are not supported).

---

## 🟧 Step E — Add a database / raw-TCP tool (needs the WARP app)

Databases and other non-browser tools can't be rendered in a page, so the user's laptop
runs the **WARP** app (one-time install + company-email login).

1. **Route the database's private IP.** Networks → **Routes → Add CIDR route** →
   the DB host's private address as a `/32` (e.g. `172.31.40.10/32`) → via the tunnel.
2. **Let WARP carry it.** Team & Resources → **Devices → Device profiles → Default →
   Split Tunnels (Include)** → **Manage** → add the same `/32`.
3. **Gate it by identity.** Add a Gateway / Access **network policy** allowing only
   `@ourcompany.com` to reach that address.
4. **User side:** install the **WARP** app, log in with the team name + company email,
   **Connect**. Then point the DB tool at the private IP.

✅ **Test:** with WARP **connected**, the DB tool reaches the private IP. With WARP
**off**, it cannot. (This is the same routing pattern already used for the GitLab dev
server example, `172.31.35.68/32`.)

---

## 🔒 Step F — Close the public doors (admin, after each app works)

Once an app is reachable and login-gated through Cloudflare:

1. Make its **ALB internal**, or restrict the ALB **security group** to allow **only the
   `cloudflared` connector** — and **remove the old office-IP allowlist** rules.
2. **Keep AWS Session Manager** as an emergency entrance before closing SSH ports.

✅ **Net effect:** the app now answers **only** through the authenticated Cloudflare
hostname. It is no longer on the public Internet, and "office IP vs home IP" stops
mattering entirely.

---

## 🔁 Per-app onboarding checklist (copy-paste for every new app)

- [ ] Decide the hostname, e.g. `appname.apps.ourcompany.com`.
- [ ] **Web:** add Public hostname → internal address (Step C-1).
- [ ] **SSH/RDP:** add browser-rendered app (Step D).
- [ ] **DB/TCP:** add CIDR route + Split Tunnel include (Step E).
- [ ] Add Access policy: **Allow**, **Emails ending in** `@ourcompany.com` (Steps C-3 / D-2 / E-3).
- [ ] Test from a **non-office network** with a company email.
- [ ] Admin locks the ALB/SG to the connector and removes the IP allowlist (Step F).

---

## ⚠️ Free-plan limits & things to know

- **Up to 50 users** free; **24-hour** log retention; **3-location** cap; community
  support. Fine for our use.
- **Web, SSH, RDP** need **no client** — browser only. **Databases / raw-TCP** still need
  the **WARP** app.
- Browser-rendered SSH/RDP policies must be **Allow/Block** only.
- The domain hosting the app hostnames must be **managed by Cloudflare** (or a delegated
  subdomain).

---

## 🔎 Verify the whole thing

1. From **home / mobile data**, open an app URL → Cloudflare login appears.
2. Company email → app loads. **Non-company email → denied.**
3. After Step F: hitting the old public ALB address directly → **blocked**; only the
   Cloudflare hostname works.
4. SSH/RDP open in the browser after login. The DB tool works over WARP and fails with
   WARP off.

---

## 🆘 If something goes wrong

- **No login page, app is just open / or just public-blocked:** the Access policy isn't
  attached to that hostname — re-check Step C-2/3.
- **Login works but app won't load:** the tunnel **Service** address (Step C-1) can't be
  reached from the connector inside AWS — check the internal address/port and the
  server's security group.
- **"This domain isn't on Cloudflare":** finish the prerequisite — the app hostname must
  be under a Cloudflare-managed domain.
- **DB won't connect:** confirm WARP is **Connected**, the `/32` is in **both** the CIDR
  route and the Split Tunnel **Include** list, and `warp-routing` is enabled on the
  connector.

---

## 📚 References

- Zero Trust plans & limits — <https://www.cloudflare.com/plans/zero-trust-services/> ·
  <https://developers.cloudflare.com/cloudflare-one/account-limits/>
- Publish a self-hosted app —
  <https://developers.cloudflare.com/cloudflare-one/access-controls/applications/http-apps/self-hosted-public-app/>
- Browser-rendered SSH / RDP —
  <https://developers.cloudflare.com/cloudflare-one/access-controls/applications/non-http/browser-rendering/>
