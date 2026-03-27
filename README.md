# Nextcloud Production Setup & Troubleshooting Log (2026-03-26/27)

This document provides a comprehensive log of repairs, upgrades, and optimizations performed on the Nextcloud instance in the `homelab/nextcloud` stack.

## 🚀 Quick Status
- **Version**: Nextcloud 32.0.6
- **Database**: PostgreSQL 16
- **Proxy**: Caddy (with HSTS and Cloudflare loopback bypass)
- **Background Jobs**: Dedicated Cron container running every 5 minutes

---

## 🛠 Issues & Solutions

### 1. Nextcloud Office (Docs) Not Opening
- **Symptoms**: Loading screen stuck; logs showed `403 Forbidden` from `cloud.jeya.dev`.
- **Cause**: Loopback requests from Nextcloud to its own built-in CODE server were being routed through Cloudflare, which blocked the internal container traffic.
- **Solution**: 
    - Added `extra_hosts` to `docker-compose.yml` to point `cloud.jeya.dev` to the Caddy container IP (`172.19.0.15`).
    - Set `disable_certificate_verification` for the `richdocuments` app.
    - Result: Connection Successful (200 OK).

### 2. Major Version Upgrade (v30 → v31 → v32)
- **Action**: Sequential upgrades triggered by updating the Docker image tags.
- **Fixes**:
    - Handled `db:add-missing-indices` and `maintenance:repair --include-expensive` to satisfy Nextcloud 32's database requirements.
    - Resolved `needsDbUpgrade` state using `occ upgrade` manually when container automation stalled.

### 3. Background Jobs (Cron) Failures
- **Symptoms**: Warning "Last background job execution ran 10 hours ago".
- **Fix**: 
    - Added a dedicated `nextcloud-cron` service to `docker-compose.yml`.
    - Configured it to run as the `www-data` user (required for permission consistency).
    - Set entrypoint to a shell loop: `while true; do php -f /var/www/html/cron.php; sleep 300; done`.

### 4. 2FA Lockout (Enforcement without Configuration)
- **Symptoms**: Users seeing "Two-factor authentication is enforced but has not been configured" upon login.
- **Cause**: Global enforcement was set to `true` while the admin account had no 2FA methods active.
- **Solution**: 
    - Ran `occ twofactorauth:enforce --off`.
    - Set `twofactor_enforced` to `0` (integer) in `config.php` as string-based booleans were being misinterpreted by the system.

### 5. Missing 2FA TOTP QR Codes
- **Symptoms**: QR Code not appearing on the Security page; logs showed `ConnectException` in Guzzle.
- **Cause**: Nextcloud was unable to connect to its own local domain to verify setup/images due to security restrictions.
- **Solution**: 
    - Enabled `allow_local_remote_servers` => `true` in `config.php`.
    - Cleared app states by disabling/re-enabling `twofactor_totp`.

### 6. Git Push Rejections (Secrets scanning)
- **Symptoms**: `git push` was rejected by GitHub due to "repository rule violations".
- **Cause**: Sensitive Brevo (Sendinblue) SMTP keys were found in the tracked `.env` file.
- **Solution**: 
    - Removed `.env` from Git tracking: `git rm --cached .env`.
    - Corrected `.gitignore` to ensure `.env` stays local.
    - Reset the `main` branch to remove secret-containing commits from the history before successful push.

### 7. Caddy Infrastructure Cleanup
- **Action**: Cleaned up the `Caddyfile` by removing outdated proxy entries for `papra` and `baikal`.
- **Result**: Leaner configuration and faster reloads.

---

## 🔧 Production Configuration Reference

### Docker Compose Additions
```yaml
    extra_hosts:
      - "cloud.jeya.dev:172.19.0.15" # Bypasses Cloudflare for internal document processing
```

### Critical Config.php Settings
```php
  'overwriteprotocol' => 'https',
  'trusted_proxies' => ['172.19.0.0/16'],
  'allow_local_remote_servers' => true,
  'maintenance_window_start' => 1, // Runs heavy jobs at 1 AM
  'default_phone_region' => 'IE',
```

---
*Maintained by Antigravity AI Assistant.*
