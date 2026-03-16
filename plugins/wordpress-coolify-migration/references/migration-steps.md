# WordPress Migration Steps

## Architecture Overview

```
Browser --HTTPS--> Cloudflare --tunnel--> localhost:80 (Traefik) --HTTP--> WordPress Container
                   (SSL termination)       (host)                          (port 80)
```

Cloudflare handles HTTPS. Traefik routes by hostname. WordPress serves HTTP only.

## Step 1: Add Domain to Cloudflare

1. Add site in Cloudflare dashboard (Free plan is fine)
2. Select "Import DNS records automatically"
3. **Do NOT change nameservers yet** - complete all setup first so there's zero downtime

## Step 2: Create WordPress Service in Coolify

1. Create a new WordPress + MariaDB service in Coolify
2. Set the domain to `http://DOMAIN.COM` (**NOT** `https://` - this causes Traefik redirect loops)
3. Save and deploy
4. Verify both containers are running (healthy)

**Via Coolify API:**
```bash
# List services
curl -s -H "Authorization: Bearer $COOLIFY_TOKEN" "$COOLIFY_URL/api/v1/services"

# Check service status
curl -s -H "Authorization: Bearer $COOLIFY_TOKEN" "$COOLIFY_URL/api/v1/services/$SERVICE_UUID"

# Restart service
curl -s -X POST -H "Authorization: Bearer $COOLIFY_TOKEN" "$COOLIFY_URL/api/v1/services/$SERVICE_UUID/restart"
```

If Coolify is behind Cloudflare Access, add these headers to all API calls:
```
-H "CF-Access-Client-Id: $CF_ACCESS_CLIENT_ID"
-H "CF-Access-Client-Secret: $CF_ACCESS_CLIENT_SECRET"
```

## Step 3: Add Cloudflare Tunnel Route

1. Cloudflare Zero Trust -> Networks -> Tunnels -> select tunnel -> Public Hostname
2. Add route: Domain: `DOMAIN.COM`, Path: `*`, Service: `http://localhost:80`
3. **The domain won't appear in the dropdown until nameservers are changed and domain is active on Cloudflare**

## Step 4: Change Nameservers

1. Update nameservers at old registrar to the Cloudflare-assigned nameservers
2. Click "Check nameservers now" in Cloudflare overview to speed up activation
3. Wait for green checkmark (usually 5-30 minutes)

## Step 5: Increase PHP Upload Limits

Default WordPress Docker image has a 2MB upload limit. Update the docker compose via Coolify API:

```yaml
services:
  wordpress:
    image: 'wordpress:latest'
    volumes:
      - 'wordpress-files:/var/www/html'
    environment:
      - SERVICE_URL_WORDPRESS
      - WORDPRESS_DB_HOST=mariadb
      - WORDPRESS_DB_USER=$SERVICE_USER_WORDPRESS
      - WORDPRESS_DB_PASSWORD=$SERVICE_PASSWORD_WORDPRESS
      - WORDPRESS_DB_NAME=wordpress
    entrypoint:
      - bash
      - -c
      - |
        echo "upload_max_filesize = 512M" > /usr/local/etc/php/conf.d/uploads.ini
        echo "post_max_size = 512M" >> /usr/local/etc/php/conf.d/uploads.ini
        echo "memory_limit = 512M" >> /usr/local/etc/php/conf.d/uploads.ini
        echo "max_execution_time = 600" >> /usr/local/etc/php/conf.d/uploads.ini
        echo "max_input_time = 600" >> /usr/local/etc/php/conf.d/uploads.ini
        exec docker-entrypoint.sh apache2-foreground
    depends_on:
      - mariadb
    healthcheck:
      test: ["CMD", "curl", "-f", "http://127.0.0.1"]
      interval: 2s
      timeout: 10s
      retries: 10
  mariadb:
    image: 'mariadb:11'
    volumes:
      - 'mariadb-data:/var/lib/mysql'
    environment:
      - MYSQL_ROOT_PASSWORD=$SERVICE_PASSWORD_ROOT
      - MYSQL_DATABASE=wordpress
      - MYSQL_USER=$SERVICE_USER_WORDPRESS
      - MYSQL_PASSWORD=$SERVICE_PASSWORD_WORDPRESS
    healthcheck:
      test: ["CMD", "healthcheck.sh", "--connect", "--innodb_initialized"]
      interval: 5s
      timeout: 20s
      retries: 10
```

**To update via API**, base64-encode the compose and PATCH:
```bash
ENCODED=$(echo "$COMPOSE_YAML" | base64 -w 0)
curl -s -X PATCH -H "Authorization: Bearer $COOLIFY_TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"docker_compose_raw\":\"$ENCODED\"}" \
  "$COOLIFY_URL/api/v1/services/$SERVICE_UUID"
```

Restart the service after updating.

## Step 6: Install WordPress & All-in-One WP Migration

### Install WP-CLI (required, not included in WordPress Docker image)
```bash
docker exec $WP_CONTAINER bash -c 'curl -sO https://raw.githubusercontent.com/wp-cli/builds/gh-pages/phar/wp-cli.phar && chmod +x wp-cli.phar && mv wp-cli.phar /usr/local/bin/wp'
```

**Note:** WP-CLI disappears on container restart (image rebuild). Reinstall after each restart.

### Install WordPress core and plugin
```bash
docker exec $WP_CONTAINER wp core install \
  --url=http://DOMAIN.COM \
  --title=SITENAME \
  --admin_user=admin \
  --admin_password=TEMP_PASSWORD \
  --admin_email=admin@DOMAIN.COM \
  --allow-root

docker exec $WP_CONTAINER wp plugin install all-in-one-wp-migration --activate --allow-root
```

## Step 7: Restore Backup via SSH Tunnel

Cloudflare free plan has a **100MB upload limit**. Large .wpress files will hang at ~1%. Bypass by SSH tunneling directly to the WordPress container.

### Set up SSH tunnel
```bash
# Get container internal IP
docker inspect $WP_CONTAINER --format '{{range .NetworkSettings.Networks}}{{.IPAddress}} {{end}}'

# Start SSH tunnel (use unique local port per site: 8888, 8889, etc.)
ssh -i $SSH_KEY -L LOCAL_PORT:CONTAINER_IP:80 -N -f root@SERVER_IP
```

### Add temporary WP_HOME override
WordPress will redirect localhost to the real domain without this:
```bash
docker exec $WP_CONTAINER sed -i '/^<?php/a define("WP_HOME", "http://localhost:LOCAL_PORT"); define("WP_SITEURL", "http://localhost:LOCAL_PORT");' /var/www/html/wp-config.php
```

### Upload backup
1. Open `http://localhost:LOCAL_PORT/wp-admin/admin.php?page=ai1wm_import` in browser
2. Log in with WordPress admin credentials
3. Click Import From -> FILE and select the .wpress file
4. Wait for import to complete, confirm when prompted

### Alternative: SCP + direct restore (if backup < 512MB)
```bash
# Upload to server
scp -i $SSH_KEY BACKUP.wpress root@SERVER_IP:/tmp/

# Copy into container backup directory
docker cp /tmp/BACKUP.wpress $WP_CONTAINER:/var/www/html/wp-content/ai1wm-backups/
docker exec $WP_CONTAINER chown www-data:www-data /var/www/html/wp-content/ai1wm-backups/BACKUP.wpress
```
Note: The "Restore" button on the Backups page requires the paid Unlimited Extension.

## Step 8: Post-Import Cleanup

### Remove localhost override
```bash
docker exec $WP_CONTAINER sed -i '/WP_HOME.*localhost/d' /var/www/html/wp-config.php
```

### Replace all localhost URLs in database
```bash
# Reinstall WP-CLI if container was restarted
docker exec $WP_CONTAINER bash -c 'curl -sO https://raw.githubusercontent.com/wp-cli/builds/gh-pages/phar/wp-cli.phar && chmod +x wp-cli.phar && mv wp-cli.phar /usr/local/bin/wp'

# Replace localhost URLs
docker exec $WP_CONTAINER wp search-replace 'http://localhost:LOCAL_PORT' 'https://DOMAIN.COM' --all-tables --allow-root

# Ensure all http URLs are https
docker exec $WP_CONTAINER wp search-replace 'http://DOMAIN.COM' 'https://DOMAIN.COM' --all-tables --allow-root
```

### Kill SSH tunnel
```bash
# Windows
taskkill //F //IM ssh.exe

# Linux/Mac
kill $(lsof -t -i:LOCAL_PORT)
```

## Step 9: Fix HTTPS / Redirect Loop

**The Problem:** Cloudflare tunnel sends HTTP to localhost:80. Traefik forwards to WordPress. WordPress sees HTTP but its siteurl is `https://`, so it redirects to HTTPS. Cloudflare receives the redirect, sends it back through the tunnel as HTTP again. Infinite loop.

**The Fix:** Force `$_SERVER['HTTPS'] = 'on'` in wp-config.php.

Find the proxy detection section in wp-config.php (around line 120-126) and replace the conditional check:

```php
// OLD (does NOT work reliably through tunnel+Traefik):
if (isset($_SERVER['HTTP_X_FORWARDED_PROTO']) && strpos($_SERVER['HTTP_X_FORWARDED_PROTO'], 'https') !== false) {
    $_SERVER['HTTPS'] = 'on';
}

// NEW (force HTTPS since always behind Cloudflare):
// Force HTTPS - always behind Cloudflare tunnel
$_SERVER['HTTPS'] = 'on';
```

**Best method to edit wp-config.php remotely:** Write a PHP fix script locally, SCP to server, docker cp into container, execute with `php /tmp/fix.php`. Avoid complex sed commands - shell escaping with `$_SERVER` variables through SSH + docker exec is extremely error-prone.

Example fix script pattern:
```php
<?php
$file = '/var/www/html/wp-config.php';
$content = file_get_contents($file);
$content = str_replace($old_string, $new_string, $content);
file_put_contents($file, $content);
```

## Step 10: Deactivate Hosting-Specific Plugins

Old hosting plugins cause critical errors on Coolify/Docker:

| Plugin | Package | Issue |
|--------|---------|-------|
| SiteGround Optimizer | `sg-cachepress` | Fatal errors (dynamic property deprecation) |
| Starter Templates | Host-specific | May conflict |
| Any host caching plugin | Varies | Incompatible with Docker environment |

```bash
docker exec $WP_CONTAINER wp plugin deactivate sg-cachepress --allow-root
docker exec $WP_CONTAINER wp plugin deactivate starter-templates --allow-root
# List all plugins to check for others:
docker exec $WP_CONTAINER wp plugin list --allow-root
```

## Step 11: Email - Cloudflare Email Routing

Free email forwarding without self-hosting (no SMTP server needed, no port 25 approval from Hetzner):

1. Cloudflare dashboard -> select domain -> Email -> Email Routing -> Enable
2. Add destination email address (can be any email provider - Gmail, Yahoo, etc.)
3. Verify destination via confirmation email
4. Create routing rules:
   - Specific addresses (e.g., `info@DOMAIN.COM` -> personal email)
   - Or **Catch-all** to forward all @DOMAIN.COM emails
5. Delete old MX and mail-related DNS records from previous host

## Key Gotchas

| Issue | Cause | Fix |
|-------|-------|-----|
| ERR_TOO_MANY_REDIRECTS | WordPress HTTPS redirect + tunnel sends HTTP | Force `$_SERVER['HTTPS'] = 'on'` in wp-config.php |
| Upload stuck at ~1% | Cloudflare free plan 100MB upload limit | SSH tunnel to bypass Cloudflare |
| Domain not in tunnel dropdown | Domain not yet active on Cloudflare | Change nameservers first, wait for activation |
| Critical error after import | Old hosting plugins (SiteGround etc.) | Deactivate via WP-CLI |
| localhost URLs after import | WP_HOME override left in wp-config | Remove override + wp search-replace |
| wp-config.php syntax errors | Shell escaping with sed through SSH+docker | Use PHP fix scripts instead of sed |
| WP-CLI disappears after restart | Container rebuilt from image | Reinstall after each container restart |
| Coolify service FQDN via API | Services don't expose FQDN update in API | Change via Coolify UI (service -> app -> Settings -> Domains) |
| Coolify FQDN http vs https | `https://` triggers Traefik redirect middleware | Use `http://` - Cloudflare handles SSL |
| Cloudflare SSL mode | Must be "Full" for tunnel to work | Set in Cloudflare -> SSL/TLS -> Overview |
| Old site still showing | Browser cache or DNS cache | Incognito window + `ipconfig /flushdns` |
