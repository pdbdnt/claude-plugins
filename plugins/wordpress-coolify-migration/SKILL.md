---
name: wordpress-coolify-migration
description: Migrate WordPress sites from any hosting provider to a self-hosted Coolify server (Hetzner or similar VPS) with Cloudflare DNS, Cloudflare Tunnel, and Cloudflare Email Routing. Use when user asks to migrate WordPress, set up WordPress on Coolify, restore a .wpress backup, configure Cloudflare tunnel for WordPress, fix WordPress redirect loops behind a reverse proxy, or set up email forwarding with Cloudflare. Covers the full workflow from DNS setup through backup restoration to post-migration URL fixes.
---

# WordPress Migration to Coolify + Cloudflare Tunnel

Migrate WordPress sites from old hosting to a self-hosted Coolify instance behind a Cloudflare Tunnel.

## Prerequisites

Gather from the user before starting:
- **Domain name** to migrate
- **VPS SSH access** (IP, SSH key path)
- **Coolify dashboard URL** and API token
- **Cloudflare Access credentials** (if Coolify is behind CF Access)
- **Cloudflare tunnel name** (existing tunnel on the server)
- **Backup file** (.wpress from All-in-One WP Migration plugin)

## Workflow

Follow the steps in [references/migration-steps.md](references/migration-steps.md) in order. Key phases:

1. **Cloudflare DNS** - Add domain, import records, defer nameserver change
2. **Coolify setup** - Create WordPress+MariaDB service, set domain as `http://` (NOT `https://`)
3. **Tunnel route** - Add public hostname in existing Cloudflare tunnel
4. **Nameserver switch** - Change at old registrar, wait for activation
5. **PHP limits** - Update compose with 512MB upload entrypoint
6. **Backup restore** - SSH tunnel to bypass Cloudflare 100MB upload limit
7. **Post-import cleanup** - URL replacement, HTTPS fix, plugin cleanup
8. **Email** - Cloudflare Email Routing (free)

## Critical Gotchas

Read the **Key Gotchas** table in [references/migration-steps.md](references/migration-steps.md) before starting. The most common issues:

- **ERR_TOO_MANY_REDIRECTS**: Force `$_SERVER['HTTPS'] = 'on'` in wp-config.php. The conditional `HTTP_X_FORWARDED_PROTO` check does NOT work reliably through tunnel+Traefik.
- **Upload stuck at ~1%**: Cloudflare free plan has 100MB upload limit. Use SSH tunnel to bypass.
- **Coolify FQDN must be `http://`**: Using `https://` triggers Traefik redirect middleware causing loops.
- **Old hosting plugins crash WordPress**: Deactivate SiteGround Optimizer and similar host-specific plugins via WP-CLI after import.
