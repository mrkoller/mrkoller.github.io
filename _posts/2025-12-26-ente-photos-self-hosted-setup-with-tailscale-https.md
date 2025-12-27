---
title: "Ente Photos Self-Hosted Setup with Tailscale"
date: 2025-12-26
tags: [ente, docker, tailscale, homelab, privacy]
description: "Deploy Ente Photos self-hosted stack with Tailscale, PostgreSQL, and MinIO for privacy-focused photo backup"
image: /assets/img/ente-photos-header.png
---

## Overview

This app is something I have wanted to try out for some time, but failed several times to implement fully in the past. There was always some login/upload/permisssions issues I couldn't seem to crack even with AI help. I recently had some success and wanted to document this. Make sure you see the Mobile section for some learnings if you go this route.

[Ente Photos](https://ente.io) is a self-hosted, end-to-end encrypted photo backup solution - think Google Photos, but you control the infrastructure and encryption keys. This guide documents deploying the full Ente stack using Docker, Tailscale for HTTPS access, PostgreSQL for metadata, and MinIO for S3-compatible object storage.

This deployment follows Ente's [official self-hosting documentation](https://ente.io/help/self-hosting/), adapted for a homelab environment with Tailscale networking and NFS storage.

**What you'll end up with:**
- Ente Photos API server (Museum) accessible via Tailscale HTTPS
- PostgreSQL database for user accounts and metadata
- MinIO S3-compatible storage for photos and videos
- Mobile app support (iOS/Android)
- Zero reverse proxy complexity thanks to Tailscale serve

## Prerequisites

Before starting, ensure you have:

- Docker host with Docker Compose
- NFS mount for persistent storage (or local storage)
- Tailscale network configured
- Portainer (optional, but makes stack management easier)
- Tailscale auth key with `tag:container` permission

**Software versions used:**
- PostgreSQL 15
- Ente Server (ghcr.io/ente-io/server:latest)
- MinIO (latest)
- Tailscale (latest)

## Architecture

The stack consists of four main components sharing a single Tailscale network namespace:

- **Museum** - Ente Photos API server
- **PostgreSQL** - Primary database storing user accounts and photo metadata
- **MinIO** - S3-compatible object storage for actual photos and videos
- **Tailscale** - Provides secure HTTPS access via MagicDNS and automatic certificates

All containers use `network_mode: service:ente-tailscale`, meaning they share the Tailscale container's network namespace. This eliminates the need for a reverse proxy while providing automatic HTTPS via Tailscale's certificates.

## Directory Structure

On your Docker host, create the required directory structure:

```bash
# Create all required directories
sudo mkdir -p /mnt/docker-data/ente/{config,museum,postgres,minio,tailscale}

# Set permissions (adjust UID/GID as needed)
sudo chown -R 1000:1000 /mnt/docker-data/ente/
sudo chmod 755 /mnt/docker-data/ente/*
```

**Directory purposes:**
- `config/` - Contains credentials.yaml and museum.yaml
- `museum/` - Museum application data (read-only mount)
- `postgres/` - PostgreSQL database files
- `minio/` - MinIO object storage data
- `tailscale/` - Tailscale state and configuration

## Secret Configuration

Ente uses a two-file approach for secrets:

1. **`.env`** - Docker environment variables (database password, MinIO credentials)
2. **`credentials.yaml`** - Ente-specific secrets (encryption keys, S3 configuration)

**Important: Some values must match between these two files.**

### Generate Secrets

First, generate four unique secrets for credentials:

```bash
# Generate 4 unique base64 secrets
openssl rand -base64 32  # SECRET_1 - PostgreSQL password
openssl rand -base64 32  # SECRET_2 - Encryption key
openssl rand -base64 32  # SECRET_3 - Hash key
openssl rand -base64 32  # SECRET_4 - JWT secret
```

Also choose:
- MinIO admin username (e.g., `minio-admin`)
- MinIO admin password (use `openssl rand -base64 24`)

Get a Tailscale auth key from https://login.tailscale.com/admin/settings/keys

### Create .env File

Create `.env` with your generated values:

```env
# Tailscale
TS_AUTHKEY=tskey-auth-xxxxxxxxxxxxxxxxxxxxxxxxxxxxx

# PostgreSQL Database
POSTGRES_PASSWORD=SECRET_1  # Your generated SECRET_1
POSTGRES_DB=ente_db
POSTGRES_USER=pguser

# MinIO S3 Storage
MINIO_ROOT_USER=minio-admin       # Your chosen username
MINIO_ROOT_PASSWORD=SECRET_5      # Your chosen MinIO password
```

### Create credentials.yaml

This file contains Ente-specific encryption keys and S3 configuration. **The encryption keys in this file are immutable** - changing them after creating accounts will break all logins and make photos undecryptable.

```yaml
# Database credentials - MUST MATCH .env POSTGRES_PASSWORD
db:
  host: localhost
  port: 5432
  name: ente_db
  user: pguser
  password: SECRET_1  # MUST MATCH .env POSTGRES_PASSWORD

# Encryption keys - IMMUTABLE after first account creation
key:
  encryption: SECRET_2  # Encrypts all photo metadata
  hash: SECRET_3        # Hashes user emails for lookups

# JWT authentication
jwt:
  secret: SECRET_4

# S3 bucket configuration
# All three buckets point to the same MinIO instance
s3:
  b2-eu-cen:
    key: minio-admin           # MUST MATCH .env MINIO_ROOT_USER
    secret: SECRET_5           # MUST MATCH .env MINIO_ROOT_PASSWORD
    endpoint: https://ente.your-tailnet.ts.net:3200
    region: eu-cen
    bucket: b2-eu-cen

  wasabi-eu-central-2-v3:
    key: minio-admin           # MUST MATCH .env MINIO_ROOT_USER
    secret: SECRET_5           # MUST MATCH .env MINIO_ROOT_PASSWORD
    endpoint: https://ente.your-tailnet.ts.net:3200
    region: eu-central-2
    bucket: wasabi-eu-central-2-v3

  scw-eu-fr-v3:
    key: minio-admin           # MUST MATCH .env MINIO_ROOT_USER
    secret: SECRET_5           # MUST MATCH .env MINIO_ROOT_PASSWORD
    endpoint: https://ente.your-tailnet.ts.net:3200
    region: fr-par
    bucket: scw-eu-fr-v3
```

**Why three buckets?** Ente's architecture supports multiple S3 providers for redundancy. In a homelab setup, all three are on the same MinIO server - they're just different buckets. In a production deployment, these would be different S3 providers like Backblaze B2, Wasabi, and Scaleway.

Upload credentials.yaml to your Docker host:

```bash
scp credentials.yaml your-docker-host:/mnt/docker-data/ente/config/
```

### Important: credentials.yaml is Immutable

The encryption keys in `credentials.yaml` (`key.encryption`, `key.hash`, and `jwt.secret`) are cryptographic keys that cannot be changed after creating user accounts:

- **`key.hash`** - Hashes user emails for database lookups. Changing this means no existing users can log in.
- **`key.encryption`** - Encrypts all photo metadata. Changing this makes existing photos undecryptable.
- **`jwt.secret`** - Signs authentication tokens. Changing this invalidates all sessions.

**If you lose or change these keys:** All users lose access to their accounts and uploaded photos become permanently undecryptable.

**Backup strategy:**

```bash
# Create timestamped backup after initial setup
sudo cp /mnt/docker-data/ente/config/credentials.yaml \
       /mnt/docker-data/ente/config/credentials.yaml.backup-$(date +%Y%m%d-%H%M%S)

# Store backups in multiple locations:
# 1. NFS mount (already backed up to NAS)
# 2. Password manager (1Password, Bitwarden, etc.)
# 3. Encrypted USB drive in safe location
```

### Create DNS Configuration

Create `resolv.conf` that uses Tailscale MagicDNS instead of Docker DNS:

```bash
echo 'nameserver 100.100.100.100' | sudo tee /mnt/docker-data/ente/resolv.conf
```

**Why Tailscale DNS?** Containers using `network_mode: service:tailscale` share the Tailscale container's network namespace. Docker DNS (127.0.0.11) doesn't have access to Tailscale MagicDNS from within this shared namespace, so museum can't resolve `*.ts.net` hostnames for S3 access. Using Tailscale MagicDNS (100.100.100.100) solves this.

## Docker Compose Configuration

Create `docker-compose.yml` with the following configuration:

```yaml
services:
  ente-tailscale:
    image: tailscale/tailscale:latest
    container_name: ente-tailscale
    hostname: ente
    command: sh -c "tailscaled & sleep 5 && tailscale up --authkey=$$TS_AUTHKEY --hostname=ente --advertise-tags=tag:container --accept-dns=false && tailscale serve --bg --https=443 --set-path=/ 3000 && tailscale serve --bg --https=443 --set-path=/api 8080 && tailscale serve --bg --https=3200 9000 && sleep infinity"
    environment:
      - TS_AUTHKEY=${TS_AUTHKEY}
      - TS_STATE_DIR=/var/lib/tailscale
    volumes:
      - /dev/net/tun:/dev/net/tun
      - /mnt/docker-data/ente/tailscale:/var/lib/tailscale
      - /mnt/docker-data/ente/resolv.conf:/etc/resolv.conf:ro
    cap_add:
      - NET_ADMIN
      - SYS_MODULE
    restart: unless-stopped

  web:
    image: ghcr.io/ente-io/web:latest
    container_name: ente-web
    environment:
      - ENTE_API_ORIGIN=http://localhost:8080
    volumes:
      - /mnt/docker-data/ente/resolv.conf:/etc/resolv.conf:ro
    network_mode: service:ente-tailscale
    depends_on:
      - ente-tailscale
      - museum
    restart: unless-stopped

  museum:
    image: ghcr.io/ente-io/server:latest
    container_name: ente-museum
    environment:
      - ENTE_DB_HOST=localhost
      - ENTE_DB_PORT=5432
      - ENTE_DB_NAME=${POSTGRES_DB:-ente_db}
      - ENTE_DB_USER=${POSTGRES_USER:-pguser}
      - ENTE_DB_PASSWORD=${POSTGRES_PASSWORD}
      - ENTE_DB_SSLMODE=disable
    volumes:
      - /mnt/docker-data/ente/museum:/data:ro
      - /mnt/docker-data/ente/config/museum.yaml:/museum.yaml:ro
      - /mnt/docker-data/ente/config/credentials.yaml:/credentials.yaml:ro
      - /mnt/docker-data/ente/resolv.conf:/etc/resolv.conf:ro
    network_mode: service:ente-tailscale
    depends_on:
      - ente-tailscale
      - postgres
      - minio
    restart: unless-stopped

  postgres:
    image: postgres:15
    container_name: ente-postgres
    environment:
      - POSTGRES_DB=${POSTGRES_DB:-ente_db}
      - POSTGRES_USER=${POSTGRES_USER:-pguser}
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
    volumes:
      - /mnt/docker-data/ente/postgres:/var/lib/postgresql/data
      - /mnt/docker-data/ente/resolv.conf:/etc/resolv.conf:ro
    network_mode: service:ente-tailscale
    depends_on:
      - ente-tailscale
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER:-pguser} -d ${POSTGRES_DB:-ente_db}"]
      interval: 5s
      timeout: 3s
      retries: 10
    restart: unless-stopped

  minio:
    image: minio/minio:latest
    container_name: ente-minio
    command: server /data --address :9000 --console-address :9001
    environment:
      - MINIO_ROOT_USER=${MINIO_ROOT_USER:-changeme}
      - MINIO_ROOT_PASSWORD=${MINIO_ROOT_PASSWORD:-changeme1234}
    volumes:
      - /mnt/docker-data/ente/minio:/data
      - /mnt/docker-data/ente/resolv.conf:/etc/resolv.conf:ro
    network_mode: service:ente-tailscale
    depends_on:
      - ente-tailscale
    restart: unless-stopped

  minio-setup:
    image: minio/mc:latest
    container_name: ente-minio-setup
    entrypoint: >
      /bin/sh -c "
      sleep 10;
      /usr/bin/mc alias set myminio http://localhost:9000 ${MINIO_ROOT_USER:-changeme} ${MINIO_ROOT_PASSWORD:-changeme1234};
      /usr/bin/mc mb --ignore-existing myminio/b2-eu-cen;
      /usr/bin/mc mb --ignore-existing myminio/wasabi-eu-central-2-v3;
      /usr/bin/mc mb --ignore-existing myminio/scw-eu-fr-v3;
      exit 0;
      "
    network_mode: service:ente-tailscale
    depends_on:
      - ente-tailscale
      - minio
    restart: "no"
```

**Key configuration details:**

- **Tailscale serve commands** - Map HTTPS ports:
  - Port 443 / → Port 3000 (web interface)
  - Port 443 /api → Port 8080 (museum API)
  - Port 3200 → Port 9000 (MinIO S3, needs browser access for presigned URLs)

- **Network mode sharing** - All containers use `network_mode: service:ente-tailscale` to share the Tailscale network namespace

- **DNS resolution** - Custom `resolv.conf` mounted with `nameserver 100.100.100.100` for Tailscale MagicDNS

- **MinIO setup** - One-time container creates three buckets and exits

## Deployment

### Via Portainer (Optional)

1. Log into Portainer
2. Navigate to **Stacks** → **Add stack**
3. Choose **Repository** as build method
4. Configure:
   - Name: `ente-photos`
   - Repository URL: Your homelab repo
   - Repository reference: `refs/heads/main`
   - Compose path: `ente/docker-compose.yml`
5. Add environment variables from `.env`
6. Click **Deploy the stack**

### Via Docker Compose

Alternatively, deploy directly:

```bash
# Navigate to stack directory
cd /path/to/ente/

# Deploy stack
docker compose up -d

# Watch startup logs
docker compose logs -f
```

## Post-Deployment Verification

Check that all containers are running:

```bash
docker ps | grep ente
```

You should see 5 containers:
- ente-tailscale (running)
- ente-web (running)
- ente-museum (running)
- ente-postgres (running)
- ente-minio (running)
- ente-minio-setup (exited - this is expected)

Check logs for each service:

```bash
# Museum startup
docker logs ente-museum

# PostgreSQL
docker logs ente-postgres

# MinIO
docker logs ente-minio

# Tailscale connection
docker logs ente-tailscale
```

Verify MinIO buckets were created:

```bash
docker logs ente-minio-setup
```

Should show three buckets created:
- b2-eu-cen
- wasabi-eu-central-2-v3
- scw-eu-fr-v3

Test Tailscale HTTPS access from any device on your Tailnet:

```
https://ente.your-tailnet.ts.net
```

The first HTTPS request may take 1-2 minutes while Tailscale provisions an SSL certificate.

**Important:** After initial deployment, restart Tailscale to refresh peer discovery:

```bash
docker restart ente-tailscale
docker restart ente-museum
```

## First Account Setup

Navigate to `https://ente.your-tailnet.ts.net`

Click the login page 8 times to expose the developer settings (or this may pop-up as you try to fill out the create account details). This lets you set API for your self-hosted instance. You'll need to do this for the mobile/desktop apps as well and any time you access it from a new device or after clearing browser caches for the site.

The text to enter would be: https://ente.your-tailnet.ts.net/api

### Getting the Verification Code

Without SMTP configured (common for homelab), verification codes appear in museum logs instead of email:

```bash
docker logs ente-museum 2>&1 | grep "Verification code" | tail -1
```

You'll see: `Verification code: 123456`

Enter this 6-digit code to complete signup and access your account.

## Admin Configuration

To grant unlimited storage to your account, you need **both** of these steps:

### Step 1: Find Your User ID

```bash
docker exec -it ente-postgres psql -U pguser -d ente_db \
  -c "SELECT user_id FROM users WHERE email='your@email.com';"
```

Note the `user_id` returned (e.g., 12345).

### Step 2: Update Database Storage

```bash
# Set storage to 100TB (109951162777600 bytes)
docker exec ente-postgres psql -U pguser -d ente_db \
  -c "UPDATE subscriptions SET storage = 109951162777600 WHERE user_id = 12345;"
```

### Optional: Add to museum.yaml Admin List

Create or edit `/mnt/docker-data/ente/config/museum.yaml`:

```yaml
internal:
  admins:
    - 12345  # Your user_id from step 1
```

Restart museum:

```bash
docker restart ente-museum
```

**Why both steps?** The `museum.yaml` admins list grants admin privileges (user management, etc.), but the UI reads storage quota from the `subscriptions.storage` database field. You need to update the database for unlimited storage to actually display and be enforced.

Refresh the Ente web page - you should now see "100 TB" instead of "10 GB" storage.

## Troubleshooting

### Museum Won't Start

**Check database connection:**

```bash
docker exec ente-museum cat /credentials.yaml
```

Verify `db.password` matches your `.env` `POSTGRES_PASSWORD`.

**Check logs for database errors:**

```bash
docker logs ente-museum | grep -i error
```

### Upload Failures at 97-99%

**Most common cause:** DNS resolution issue.

**Symptoms:**
- Uploads stall at 97-99%
- Museum logs show `nslookup: connection timed out; no servers could be reached`
- Error: `OBJECT_SIZE_FETCH_FAILED`

**Solution:**

```bash
# Verify resolv.conf uses Tailscale DNS
cat /mnt/docker-data/ente/resolv.conf

# Should be: nameserver 100.100.100.100
# NOT: nameserver 127.0.0.11

# If wrong, recreate:
echo 'nameserver 100.100.100.100' | sudo tee /mnt/docker-data/ente/resolv.conf

# Restart containers
docker restart ente-tailscale ente-museum ente-postgres ente-minio
```

**Test DNS resolution from museum container:**

```bash
docker exec ente-museum nslookup ente.your-tailnet.ts.net
# Should return Tailscale IP (100.x.x.x)
```

### Login Returns 404 Error

**Symptoms:**
- Login fails with HTTP 404
- Museum logs show: `GetUserIDWithEmail → sql: no rows in result set`
- User exists in database but can't log in

**Root cause:** The `key.hash` value in credentials.yaml changed after the account was created. Email lookups hash the email using this key, then search for that hash in the database. If the key changed, hashes don't match.

**Verification:**

```bash
docker logs ente-museum 2>&1 | grep "GetUserIDWithEmail"
# You'll see: "Caused by: sql: no rows in result set"
```

**Solution 1: Create New Account**

The simplest solution is to create a fresh account with the current (stable) credentials.yaml:

1. Go to `https://ente.your-tailnet.ts.net`
2. Click "Sign Up" with same or different email
3. New account will work with current hash key
4. Follow admin storage steps above

**Solution 2: Restore Original credentials.yaml**

If you have a backup of the original credentials.yaml from when the account was created:

```bash
# Restore backup
sudo cp /mnt/docker-data/ente/config/credentials.yaml.backup-YYYYMMDD \
       /mnt/docker-data/ente/config/credentials.yaml

# Restart museum
docker restart ente-museum
```

### Verification Code Not Received

Without SMTP configured, codes only appear in logs:

```bash
docker logs ente-museum 2>&1 | grep "Verification code" | tail -1
```

Format is a 6-digit numeric code.

### MinIO Connection Errors

**Symptoms:**
- Can't upload photos
- Errors about S3 connection failures

**Causes and solutions:**

**1. MinIO credentials mismatch**

```bash
# Verify MinIO credentials in credentials.yaml match .env
docker exec ente-museum cat /credentials.yaml | grep -A 10 "s3:"
```

All S3 `key` fields must match `.env` `MINIO_ROOT_USER`.
All S3 `secret` fields must match `.env` `MINIO_ROOT_PASSWORD`.

**2. Buckets not created**

```bash
# Verify buckets exist
docker exec ente-minio ls /data

# Should see: b2-eu-cen/ wasabi-eu-central-2-v3/ scw-eu-fr-v3/
```

If missing, recreate by restarting minio-setup:

```bash
docker compose up -d ente-minio-setup
```

**3. MinIO endpoint not browser-accessible**

Museum generates **presigned S3 URLs** that the browser uses to upload directly to MinIO. The endpoint in credentials.yaml must be accessible from the browser (not just from within containers):

- Error: `endpoint: http://localhost:9000` (browser can't reach localhost in container)
- Solution: `endpoint: https://ente.your-tailnet.ts.net:3200` (browser can reach via Tailscale)

The Tailscale serve command proxies port 3200 → MinIO port 9000 with HTTPS.

### Tailscale Connection Timeout

**Symptoms:** Can access Ente but webhooks/OIDC fail

**Solution:**

```bash
# Refresh peer discovery
docker restart ente-tailscale
docker restart ente-museum

# Verify Tailscale can reach other nodes
docker exec ente-tailscale ping -c 3 another-device.your-tailnet.ts.net
```

## Mobile Apps

1. **iOS**: Download Ente Photos from App Store
2. **Android**: Download from Play Store or F-Droid

**Configuration:**

Open the app and click somewhere oon the login page 8 times:
- Custom server endpoint: `https://ente.your-tailnet.ts.net/api`
- Log in with account created above

Your photos sync across all devices using the same account.

**NOTE:** 

Be cautious for iOS on uploading your whole camera roll at once. I have not devled into this fully, but the app creates encrypted versions of your photos on the device (doubling the space your photos take up). You can supposedly reclaim the space afterwards, but in my experience this tried to delete my iCloud versions for Apple Photos, which I did not want. Better to upload via macOS web app or desktop app and then log into your mobile account. Then it will recognize you already have everything backed up and you can start from new photos. You may want this behavior if you are fully migrating from iCloud to Ente Photos, but I am still testing things so this was not desirable for me.

## Key Lessons

### Lesson 1: Encryption Keys Are Immutable

The encryption keys in `credentials.yaml` (`key.encryption`, `key.hash`, and `jwt.secret`) cannot be changed after creating user accounts.

**What each key does:**
- `key.hash` - Hashes user emails for database lookups
- `key.encryption` - Encrypts all photo metadata
- `jwt.secret` - Signs authentication tokens

**What happens if you change them:**
- All users can't log in (email hash mismatch)
- All photos become undecryptable
- Complete data loss with no recovery

**Protection strategy:**
- Generate once during initial setup
- Backup to 3+ locations immediately (NFS, password manager, encrypted USB)
- Never modify after first account creation

### Lesson 2: DNS Resolution with Network Mode

Containers using `network_mode: service:tailscale` have unique DNS requirements.

**The issue:** Docker DNS (127.0.0.11) can't resolve `*.ts.net` hostnames from within the shared Tailscale network namespace.

**The solution:** Use Tailscale MagicDNS (100.100.100.100) by mounting a custom `resolv.conf`.

**Symptoms when wrong:**
- Uploads fail at 97-99%
- Error: `OBJECT_SIZE_FETCH_FAILED`
- Logs show: `nslookup: connection timed out; no servers could be reached`

**Fix:** Mount `/etc/resolv.conf` with `nameserver 100.100.100.100` in all containers.

### Lesson 3: MinIO Endpoint Must Be Browser-Accessible

Museum generates **presigned S3 URLs** that the browser uses to upload directly to MinIO.

**The issue:** The S3 endpoint in `credentials.yaml` must be reachable from the user's browser, not just from containers.

**Wrong configuration:**
```yaml
endpoint: http://localhost:9000  # Browser can't reach this
```

**Right configuration:**
```yaml
endpoint: https://ente.your-tailnet.ts.net:3200  # Browser can reach via Tailscale
```

The Tailscale serve command proxies port 3200 → MinIO port 9000 with HTTPS, making MinIO accessible to browsers on your Tailnet.

### Lesson 4: Admin Storage Requires Two Steps

Setting yourself as admin in `museum.yaml` alone doesn't grant unlimited storage.

**Wrong assumption:** Adding your user_id to `museum.yaml` admins list gives unlimited storage.

**Reality:** The UI reads storage quota from the `subscriptions.storage` database field.

**Both steps required:**
1. Add to `museum.yaml` admins list (grants admin privileges)
2. Update database: `UPDATE subscriptions SET storage = 109951162777600`

Without step 2, the UI still shows "10 GB free plan" even though you're listed as admin.

### Lesson 5: File vs Directory Mount Issues

Docker doesn't fail gracefully when mounting files as directories.

**The issue:** If `credentials.yaml` gets created as a directory instead of a file (e.g., during testing), Docker silently mounts it but museum can't read the config.

**Symptoms:**
- 404 errors on login
- No obvious errors in logs
- Container seems to be running fine

**Verification:**

```bash
# Check if it's actually a file
file /mnt/docker-data/ente/config/credentials.yaml
# Should say: "ASCII text", not "directory"
```

**Prevention:** Always verify files are files before first deployment.

## Limitations

**This setup works well for:**
- Tailscale-accessible users only
- Homelab/personal use cases
- Users who value privacy and data sovereignty
- Mobile photo backup across multiple devices

**Not suitable for:**
- Public-facing photo sharing service
- Users without Tailscale access
- (Note: Ente supports external S3 providers like Backblaze B2 or Wasabi for production deployments)

Useful database queries for administration:

```bash
# List all users
docker exec ente-postgres psql -U pguser -d ente_db \
  -c "SELECT user_id, email FROM users;"

# Check storage quota for user
docker exec ente-postgres psql -U pguser -d ente_db \
  -c "SELECT user_id, storage FROM subscriptions WHERE user_id = 12345;"

# View recent museum logs
docker logs ente-museum --tail 50

# Check MinIO disk usage
docker exec ente-minio du -sh /data/*

# Backup credentials.yaml
sudo cp /mnt/docker-data/ente/config/credentials.yaml \
       /mnt/docker-data/ente/config/credentials.yaml.backup-$(date +%Y%m%d-%H%M%S)
```


---

AI Influence Level: [AIL2](https://danielmiessler.com/blog/ai-influence-level-ail)
---

**Resources:**
- Official Ente self-hosting docs: https://ente.io/help/self-hosting/
- Ente GitHub: https://github.com/ente-io/ente
- Tailscale serve guide: https://ente.io/help/self-hosting/guides/tailscale
