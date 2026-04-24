# Deploy Troy to Hetzner Cloud

> [!TIP]
> Sign up for [Hetzner Cloud](https://hetzner.cloud/?ref=V6DnI7GDHM4N) through the Roots referral link to receive $20 in cloud credits.

> [!WARNING]
> The deploy workflow prints the generated WordPress admin password in the run summary and logs. If your fork is public, anyone can read that password from your Actions tab until you change it. Either keep the fork private, or change the password via `wp-admin` immediately after first login.

One-click **Hetzner Cloud** deploy for [Troy Server](https://deploytroy.org/) — a permanent, self-hosted WordPress install with the Troy Server plugin pre-activated, fronted by Caddy with automatic Let's Encrypt on a custom domain.

Fork, add one secret, click **Run workflow**, paste the returned IP into your DNS. Done in ~3–5 min.

[![Deploy Troy](https://img.shields.io/badge/Deploy-Troy-success?style=for-the-badge&logo=github)](../../actions/workflows/deploy.yml)

## What you get

- Ubuntu 24.04 on a Hetzner Cloud VM (default `cx23` — Intel shared, 2 vCPU / 4 GB / 40 GB NVMe, ~€3.99/mo, EU locations only; use `cpx22` if you want US or Singapore)
- Caddy v2 (auto HTTPS via Let's Encrypt, auto-renewing)
- PHP 8.5-FPM (from `ondrej/php` PPA) + MariaDB — all on-box, persistent
- Latest WordPress, auto-installed via WP-CLI
- Latest [Troy Server](https://github.com/sybrew/troy/releases) and Troy Client plugins, downloaded from GitHub Releases and activated (Troy Client is required to update Troy Server)
- UFW firewall limited to SSH + 80 + 443

## Prerequisites

1. A **Hetzner Cloud** project and API token (`HETZNER_TOKEN`) with read/write scope
2. A domain you can create an A record on (any DNS provider — no API integration needed)
3. (Optional) An SSH key uploaded to your Hetzner project, so you can SSH in to debug

## Secret to add to your fork

**Settings → Secrets and variables → Actions → New repository secret**:

| Secret | Where to find it |
| --- | --- |
| `HETZNER_TOKEN` | Hetzner Cloud Console → your project → Security → API tokens |

## Deploy

1. **Actions → Deploy Troy → Run workflow**
2. Fill in:
   - **fqdn** — e.g. `troy.example.com`
   - **admin_email** — used for WP admin + Let's Encrypt renewal notices
   - Optional: `site_title`, `server_type`, `location`, `plugin_repo`, `ssh_key_names`
3. **Run workflow**
4. The job summary will show the server IP. **Create an A record** `<fqdn> → <ip>` at your DNS provider.
5. Wait ~30 seconds after DNS propagates; Caddy will fetch the cert on the first HTTPS hit. Admin URL + password are in the summary — **save the password, it is not stored anywhere else.**

## How it works

- `cloud-init.yaml.tmpl` — cloud-init template with `{{PLACEHOLDERS}}` for domain, credentials, plugin repo, and a Caddyfile
- `.github/workflows/deploy.yml` — on `workflow_dispatch`, generates passwords, renders the template, and calls `POST /v1/servers` on the Hetzner API with the rendered YAML as `user_data`
- On first boot, cloud-init installs packages (Caddy + PHP 8.5 from ondrej's PPA), runs `/root/install-troy.sh` which: sets up MariaDB, downloads WP + WP-CLI, installs WP, pulls the latest `troy-server-*.zip` and `troy-client-*.zip` from the configured GitHub release, activates Troy Server first then Troy Client (order matters — the client is what updates the server), and starts Caddy. Caddy handles TLS on its own once the A record resolves.

Logs land in `/var/log/troy-install.log` on the server.

## Customizing

- **Different plugin repo** — override `plugin_repo` at dispatch time (default `sybrew/troy`)
- **Automate the DNS step** — add a step calling your provider's API between *Create Hetzner server* and *Summary* in `deploy.yml` using `${{ steps.hetzner.outputs.ip }}`
- **Bigger box** — `cx33`, `cx43` (EU Intel), or `cpx32`, `cpx42` (AMD, any region).
- **ARM** — `cax11` is ~€4.49/mo in EU locations; everything in the stack (Caddy, PHP 8.5 via ondrej PPA, MariaDB) runs fine on arm64.
- **Non-EU location** — `cx*` and `cax*` are EU-only. For `ash`/`hil`/`sin`, use `cpx22` or larger.
- **Different admin username** — edit `ADMIN_USER: admin` in `.github/workflows/deploy.yml`

## Troubleshooting

- **Site loads on HTTP but not HTTPS** — DNS hasn't propagated yet, or is pointing elsewhere. Check with `dig +short <fqdn>`. Caddy will keep retrying; SSH in and watch `journalctl -u caddy -f`.
- **Plugin not activated** — check `/var/log/troy-install.log`; the release must contain an asset matching `troy-server-*.zip`.
- **`user_data size exceeded`** — you've added too much to the template. Hetzner caps `user_data` at 32 KiB.
