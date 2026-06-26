# 📨 Questions to ask the DNS owner (about moving DNS to Cloudflare)

Goal: find out whether we can put our domain's DNS on **Cloudflare**, so we can use
**Cloudflare Access** to give staff secure, company-email login to internal tools from
anywhere — without exposing anything publicly or relying on office-IP whitelists.

Copy-paste the message below. The decision guide after it explains how to read the
answers.

---

## The message to send

> Hi — I'm setting up secure remote access to our internal tools (GitLab, dashboards)
> using Cloudflare. To do that, our domain's **DNS needs to be hosted on Cloudflare**.
> A few questions:
>
> 1. **Where is our domain's DNS currently managed?** (i.e. which provider answers DNS
>    for the domain — this can be different from where we *registered* it.)
> 2. **Who can change the domain's nameservers?**
> 3. **Can we move the DNS hosting to Cloudflare?** To be clear about what this involves:
>    - We add the domain to a Cloudflare account (free).
>    - Cloudflare **imports all existing records**; we verify them first.
>    - We then switch the domain's **nameservers** to Cloudflare's.
>    - The **registrar does not change**, we do **not** transfer the domain, and we do
>      **not** lose any records — website and **email (MX)** keep working as long as the
>      records are copied correctly. Done properly there is **no downtime**.
>
> **Why we want this:** staff work from home, and the current office-IP allowlist blocks
> them. Cloudflare Access lets us require a **company-email login** in front of each tool,
> keep the tools **off the public Internet**, and it's **free** for our team size — but it
> only works if the domain's DNS is on Cloudflare.
>
> 4. **If you'd rather not move the whole domain:** can we instead **delegate a subdomain**
>    (e.g. `apps.ourdomain.com`) to Cloudflare? (We'll confirm the exact records needed.)
>
> Thanks!

---

## How to read their answer

**If they say "DNS is at <provider X>" (e.g. Route 53, GoDaddy, Sakura, Onamae, etc.):**
- Ask: **"Can <provider X> point the domain's nameservers to Cloudflare's?"** Almost all
  providers allow this — it's a standard nameserver change, not a domain transfer.
- They may ask **"can't we just keep DNS here and add records?"** → **No.** Cloudflare
  Access needs Cloudflare to be the **authoritative DNS** (the nameservers) so it can put
  the login gate and proxy in front. Adding individual records at another provider is
  **not enough**. So it's either: move the nameservers to Cloudflare, **or** delegate a
  subdomain to Cloudflare, **or** we use a separate domain instead.

**If they're worried about breaking the website / email:**
- Reassure: nothing is deleted. Cloudflare imports the current records; we **check them
  against the live ones** before flipping nameservers. Email (MX), web, everything is
  preserved. The switch is reversible (point nameservers back).

**If they say no to moving the whole domain:**
- Fallback A — **subdomain delegation** (`apps.ourdomain.com`): keeps the main domain
  untouched, only the subdomain goes to Cloudflare. ⚠️ Note for us: subdomain-as-its-own
  zone is a **paid** Cloudflare feature, so on the **free** plan this usually isn't an
  option — confirm before promising it.
- Fallback B — **separate domain**: we register a cheap, separate domain just for these
  app logins (~$10/yr). Zero impact on the production domain. This is the clean
  no-dependency option if they can't/won't move DNS.

---

## What we need, in one line

**Move the domain's nameservers to Cloudflare (free, registrar unchanged, all records
copied first, no downtime).** If that's not possible, we'll use a separate cheap domain.
