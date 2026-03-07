# Umami Deployment with Kamal

This guide explains how to deploy Umami (or any Docker-based project) using Kamal, with a Tailscale-connected Hetzner ARM VPS, Cloudflare proxy, and Bitwarden for secrets management.

## Infrastructure Overview

```
User → Cloudflare (SSL: Full) → Tailscale VPN → Hetzner CAX11 (ARM) → Kamal + Umami + PostgreSQL
```

## Prerequisites

- Kamal CLI installed: `gem install kamal`
- Docker desktop or **Orbstack** installed on local machine (Orbstack recommended for ARM Macs)
- Tailscale installed on local machine and server
- Bitwarden CLI installed: `bw`
- Access to a Hetzner Cloud account (CAX11 ARM VPS recommended)

---

## Local Development Setup

The `app/` folder is **gitignored** - it contains the actual application source code. This keeps the infra repo lightweight and allows building locally (faster, less server overhead).

### Initial Setup

```bash
# Clone the umami source (or your app)
git clone https://github.com/umami-software/umami.git app

# Or if migrating from existing deployment, copy from old server:
scp -r deploy@<OLD_SERVER_IP>:/var/lib/umami/app ./app
```

### Build & Run Locally (Orbstack)

```bash
cd app
docker build -t umami-local .
docker run -d -p 3000:3000 -e DATABASE_URL="postgresql://user:pass@localhost:5432/db" umami-local
```

---

## 1. Server Setup (Hetzner CAX11)

### Create VPS

1. Log into [Hetzner Cloud Console](https://console.hetzner.cloud)
2. Create a new project
3. Add a new server:
   - **Image**: Ubuntu 22.04 or 24.04 (ARM64)
   - **Type**: CAX11 (ARM, 2 vCPU, 2GB RAM)
   - **Location**: Choose nearest to you
   - **SSH Key**: Add your public key

### Install Docker

SSH into the server and install Docker:

```bash
ssh root@<SERVER_IP>
curl -fsSL https://get.docker.com | sh
usermod -aG docker root
```

### Create Deploy User

```bash
adduser deploy
usermod -aG docker deploy
```

### Install Tailscale on Server

```bash
curl -fsSL https://tailscale.com/install.sh | sh
tailscale up --operator=deploy --advertise-exit-node
```

**On your local machine**, connect to Tailscale and note the server's Tailscale IP (e.g., `100.112.248.60`).

---

## 2. Cloudflare Setup

1. Add your domain to Cloudflare
2. Create an A record pointing to your **VPS public IP** (not Tailscale IP):
   - **Type**: A
   - **Name**: umami (or your subdomain)
   - **Content**: <VPS_PUBLIC_IP>
   - **Proxy status**: Proxied (orange cloud)
3. Set **SSL/TLS Encryption Mode** to **Full**
   - Go to SSL/TLS → Overview → Select "Full"

---

## 3. Bitwarden Secrets Setup

### Create Bitwarden Item

1. Log into Bitwarden web vault
2. Create a new Login item named `umami-secrets`
3. Add these custom fields:
   - `KAMAL_REGISTRY_PASSWORD` - Your GitHub Container Registry token
   - `POSTGRES_PASSWORD` - A secure password for PostgreSQL

### Configure Bitwarden CLI

```bash
bw login
export BW_SESSION=$(bw unlock --raw)
```

### Update Secrets File

Edit `.kamal/secrets` with your Bitwarden account:

```bash
SECRETS=$(kamal secrets fetch --adapter bitwarden --account "your-email@example.com" --from umami-secrets KAMAL_REGISTRY_PASSWORD POSTGRES_PASSWORD)

KAMAL_REGISTRY_PASSWORD=$(kamal secrets extract KAMAL_REGISTRY_PASSWORD $SECRETS)
POSTGRES_PASSWORD=$(kamal secrets extract POSTGRES_PASSWORD $SECRETS)
DATABASE_URL="postgresql://umami:${POSTGRES_PASSWORD}@umami-db:5432/umami"
```

---

## 4. Kamal Configuration

### Update deploy.yml

Key settings in `config/deploy.yml`:

```yaml
service: umami

image: your-registry/umami

builder:
  arch: arm64          # Important for ARM VPS
  context: ./app
  dockerfile: Dockerfile

servers:
  web:
    - <TAILSCALE_IP>   # e.g., 100.112.248.60

proxy:
  ssl: true
  host: umami.yourdomain.com
  app_port: 3000
  healthcheck:
    path: /favicon.ico

registry:
  server: ghcr.io
  username: your-github-username
  password:
    - KAMAL_REGISTRY_PASSWORD

ssh:
  user: deploy

env:
  clear:
    PORT: 3000
  secret:
    - DATABASE_URL

accessories:
  db:
    image: postgres:16-alpine
    host: <TAILSCALE_IP>
    port: 5432
    env:
      clear:
        POSTGRES_DB: umami
        POSTGRES_USER: umami
      secret:
        - POSTGRES_PASSWORD
    volumes:
      - /var/lib/umami/postgres:/var/lib/postgresql/data
```

---

## 5. First Deployment

### Boot Accessories (Database)

```bash
kamal accessory boot db
```

### Deploy

```bash
kamal deploy
```

---

## 6. Database Migration (Optional)

If migrating from another server, follow these steps:

### Export from Old Server

SSH to old server and export the database:

```bash
# On old server
pg_dump -U umami -h localhost -d umami > umami_dump.sql
```

### Download Locally

```bash
# From your local machine
scp deploy@<OLD_SERVER_IP>:/home/deploy/umami_dump.sql ./
```

### Upload to New Server

```bash
# Upload to new server via Tailscale IP
scp umami_dump.sql deploy@<TAILSCALE_NEW_IP>:/home/deploy/
```

### Import to New Database

```bash
# SSH to new server
ssh deploy@<TAILSCALE_NEW_IP>

# Copy dump to db container
docker cp umami_dump.sql kamal-umami-db-1:/tmp/

# Import (using the container name from kamal)
docker exec kamal-umami-db-1 psql -U umami -d umami -f /tmp/umami_dump.sql

# Clean up
docker exec kamal-umami-db-1 rm /tmp/umami_dump.sql
rm umami_dump.sql
```

---

## Useful Commands

```bash
# Deploy with verbose output
kamal deploy -v

# Rollback to previous version
kamal rollback

# View logs
kamal logs

# SSH into server
ssh deploy@<TAILSCALE_IP>

# Check container status
docker ps

# Restart app container
kamal app restart
```

## Troubleshooting

- **Healthcheck failing**: Ensure the healthcheck path returns 200 OK
- **Database connection issues**: Use container name (`umami-db`) not `localhost` in DATABASE_URL
- **SSL issues**: Ensure Cloudflare SSL is set to "Full" not "Strict"
- **ARM build issues**: Ensure builder.arch is set to `arm64`
