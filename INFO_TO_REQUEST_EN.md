# 📝 Information to Request — secure access to GitLab & the test site

> Goal: let people reach **`gitlab-company.com` and `test-sites-company.com`**
> through a **Cloudflare login**, and close the servers' open ports.
> (Replace IP-allowlisting with per-person login = fixes the "dynamic IP" problem.)
> Each row: **What / Why / Who**. 🔴 Required / 🟡 Optional.

> 🙂 **Easy:** Only the 6 things below are needed. Migration / separate account are
> **not needed** for this.

---

## 1. 🌐 Domain & DNS

| # | What | Why | Req | Fill in |
|---|---|---|---|---|
| 1.1 | **Domain name** (e.g. `company.com`) + who controls **DNS / the registrar** | To point nameservers at Cloudflare | 🔴 | ________ |
| 1.2 | The **hostnames** to use (`gitlab-company.com` / `test-sites-company.com`) | These become the entry addresses | 🔴 | ________ |

---

## 2. 🟠 Cloudflare account

> ⚠️ The account must be **company-owned** — not a personal Gmail. Ideally register
> with a company **shared mailbox** (`infra@…`) and invite the builder as admin.

| # | What | Why | Req | Fill in |
|---|---|---|---|---|
| 2.1 | A company **shared email** to register, OR admin access to an existing account | So the account stays company-owned | 🔴 | ________ |

---

## 3. 🖥️ The two servers

| # | What | Why | Req | Fill in |
|---|---|---|---|---|
| 3.1 | The **EC2** instances running GitLab and the test site (which instance, OS) | To know where to install the tunnel | 🔴 | ________ |
| 3.2 | **SSH / admin + sudo** on each server | To install & run `cloudflared` (the tunnel) | 🔴 | ________ |

---

## 4. ☁️ AWS access — NOT needed now (ports stay open)

> ⚠️ **Security note:** Not closing ports means the servers are **still reachable
> by their raw IP**. The Cloudflare login only protects the hostname path
> (`gitlab-company.com`); anyone with the IP can still hit the server directly and
> bypass it. This phase adds a login door without locking the old doors.
> **Plan to close inbound 80/443 (and 22 if used) later** to get the real benefit.
> `cloudflared` is outbound-only, so no inbound AWS rule is needed to set it up.

| # | What | Why | Req | Fill in |
|---|---|---|---|---|
| 4.1 | (Later) Console / IAM rights to **edit Security Groups** | To close ports once you're ready | 🟡 | ________ |
| 4.2 | Is **AWS Session Manager** available? | Emergency entrance for when ports do get closed | 🟡 | ________ |

---

## 5. 👥 Who gets in (the allowlist)

| # | What | Why | Req | Fill in |
|---|---|---|---|---|
| 5.1 | List of allowed **emails** OR an **email domain** (`@company.com`) | Access is granted per person | 🔴 | ________ |
| 5.2 | **Headcount** (confirm **under 50**) | Free Zero Trust cap | 🔴 | ________ |
| 5.3 | Want **Google login / MFA**? | Optional, can add later | 🟡 | ________ |

---

## 6. 🔧 GitLab git workflow (100 MB limit)

| # | What | Why | Req | Fill in |
|---|---|---|---|---|
| 6.1 | Git over **HTTPS** or **SSH**? Using **Git LFS / Docker images / files > 100 MB**? | HTTPS uploads over 100 MB fail; if so, use SSH to bypass | 🔴 | ________ |

---

> ✅ Once every 🔴 is filled, the build can start.
