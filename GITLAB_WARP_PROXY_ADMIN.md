# 🔧 Admin: make GitLab reachable over WARP by IP (no domain)

**For:** the AWS/server admin. **Goal:** let WARP-connected staff open
`https://172.31.35.68` and reach GitLab — **without** a domain, and **without**
touching GitLab's config or the AWS load balancer.

## Why this is needed

GitLab on the dev box serves plain HTTP on **port 80**; HTTPS is terminated by the
**AWS load balancer** (the box has **no 443**). WARP carries staff straight to the
box's private IP, so GitLab's forced HTTP→HTTPS redirect lands on a missing port 443
and the browser fails (`ERR_ADDRESS_UNREACHABLE`).

Fix: run a tiny **HTTPS shim on the box, port 443**, that forwards to the local
GitLab `:80` and adds `X-Forwarded-Proto: https`. GitLab then serves real pages.

```
WARP user  ->  https://172.31.35.68   (this shim on the box, port 443, self-signed)
           ->  http://127.0.0.1:80     (GitLab's existing nginx)
```

This is **additive and safe**:
- Does **not** change GitLab config, `external_url`, or the load balancer.
- Port 443 on the box is currently unused (`ss -ltnp` shows nginx on 80 and 8060 only).
- WARP reaches it because `cloudflared` already runs on the box; no Security Group
  change is needed (it forwards to localhost).

## Steps (run on the GitLab box, `172.31.35.68`)

```bash
sudo mkdir -p /opt/gitlab-warp-proxy && cd /opt/gitlab-warp-proxy

# 1. Self-signed cert for the private IP
sudo openssl req -x509 -newkey rsa:2048 -nodes -days 825 \
  -keyout key.pem -out cert.pem \
  -subj "/CN=172.31.35.68" -addext "subjectAltName=IP:172.31.35.68"

# 2. Put gitlab_proxy_box.py here (from ~/gitlab-warp-proxy/gitlab_proxy_box.py).
#    It listens 0.0.0.0:443 -> 127.0.0.1:80. No edits needed.

# 3. Run it as a service so it stays up across reboots
sudo tee /etc/systemd/system/gitlab-warp-proxy.service >/dev/null <<'UNIT'
[Unit]
Description=GitLab WARP TLS shim (443 -> local 80)
After=network.target
[Service]
WorkingDirectory=/opt/gitlab-warp-proxy
ExecStart=/usr/bin/python3 /opt/gitlab-warp-proxy/gitlab_proxy_box.py
Restart=always
AmbientCapabilities=CAP_NET_BIND_SERVICE   # allow binding 443 without full root
[Install]
WantedBy=multi-user.target
UNIT

sudo systemctl daemon-reload
sudo systemctl enable --now gitlab-warp-proxy
sudo systemctl status gitlab-warp-proxy --no-pager
```

## Verify

```bash
# On the box:
curl -skI https://127.0.0.1/users/sign_in | head -5      # expect: HTTP/1.1 200 OK

# From a WARP-connected laptop:
curl -skI https://172.31.35.68/users/sign_in | head -5    # expect: HTTP/1.1 200 OK
```

Then staff open **https://172.31.35.68** in a browser (accept the self-signed cert
warning once) and log in.

## Notes & caveats

- **Cert warning:** self-signed, so browsers warn once. To remove it, install a cert
  trusted by your org, or move to the domain-based Cloudflare Access setup
  (`CLOUDFLARE_ACCESS_SETUP.md`) which gives clean HTTPS + company-email login.
- **WebSockets:** live CI log streaming may not update through this simple shim;
  normal browsing/login/repos/MRs/issues work.
- **If GitLab is the Omnibus package** (likely — bundled nginx on 80 and 8060): do
  **not** add a second system nginx on 443; this standalone shim avoids that conflict.
  Alternatively GitLab's own HTTPS can be enabled in `/etc/gitlab/gitlab.rb`, but that
  changes `external_url` and affects the load-balancer path — discuss before doing it.
- **Better long-term:** a cheap domain + Cloudflare Access removes the proxy, the cert
  warning, and the raw-IP entirely, and adds company-email login. See
  `CLOUDFLARE_ACCESS_SETUP.md`.
