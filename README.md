# 🦦 Stoat Self-Hosted — Coolify Deployment Guide

This guide explains how to deploy a self-hosted Stoat instance using **Coolify** on a **Hetzner VPS**, as an alternative to the standard SSH-based deployment in the [official README](https://github.com/stoatchat/self-hosted).

---

## 📋 Table of Contents

- [Prerequisites](#prerequisites)
- [How This Differs from the Official Guide](#how-this-differs-from-the-official-guide)
- [Step 1 — Set Up Your VPS](#step-1--set-up-your-vps)
- [Step 2 — Configure DNS](#step-2--configure-dns)
- [Step 3 — Generate Config Files via SSH](#step-3--generate-config-files-via-ssh)
- [Step 4 — Modify compose.yml for Traefik](#step-4--modify-composeyml-for-traefik)
- [Step 5 — Connect GitHub to Coolify](#step-5--connect-github-to-coolify)
- [Step 6 — Create a Docker Compose Resource in Coolify](#step-6--create-a-docker-compose-resource-in-coolify)
- [Step 7 — Set the Domain on the Caddy Service](#step-7--set-the-domain-on-the-caddy-service)
- [Step 8 — Mount Config Files as Persistent Storage](#step-8--mount-config-files-as-persistent-storage)
- [Step 9 — Add Environment Variables](#step-9--add-environment-variables)
- [Step 10 — Deploy](#step-10--deploy)
- [Post-Deploy — Create Your Account](#post-deploy--create-your-account)
- [Optional — Make Your Instance Invite-Only](#optional--make-your-instance-invite-only)
- [Optional — Configure SMTP for Email Verification](#optional--configure-smtp-for-email-verification)
- [Troubleshooting](#troubleshooting)

---

## Prerequisites

- A **Hetzner VPS** (CX22 or higher recommended — 2 vCPU, 4GB RAM)
- **Ubuntu 24.04** as the OS
- **Coolify** installed on the VPS ([installation guide](https://coolify.io/docs/installation))
- A domain or subdomain pointing to your VPS IP
- A DNS provider (e.g. Porkbun, Cloudflare)

---

## How This Differs from the Official Guide

The official guide runs Stoat with **Caddy** handling both routing and SSL directly on ports 80/443. In Coolify, **Traefik** is already acting as the reverse proxy, so we need to:

1. Move Caddy off ports 80/443 to an internal port (`1234`)
2. Let Traefik handle SSL and route traffic to Caddy
3. Mount config files via Coolify's persistent storage instead of the filesystem directly

Think of it like this: in the official guide, Caddy is the front door. In Coolify, Traefik is the front door, and Caddy becomes an internal router sitting behind it.

---

## Step 1 — Set Up Your VPS

SSH into your VPS and run basic setup:

```bash
# Update the system
apt-get update && apt-get upgrade -y

# Configure firewall
ufw allow ssh
ufw allow http
ufw allow https
ufw allow 7881/tcp
ufw allow 50000:50100/udp
ufw default deny
ufw enable

# Reboot to apply changes
reboot
```

Then install Coolify:

```bash
curl -fsSL https://cdn.coollabs.io/coolify/install.sh | bash
```

Access Coolify at `http://<your-vps-ip>:8000` and complete the setup wizard.

---

## Step 2 — Configure DNS

Add the following DNS records at your DNS provider:

```
Type    Host         Answer              TTL
A       @            <your-vps-ip>       600
A       *            <your-vps-ip>       600
A       <subdomain>  <your-vps-ip>       600
```

For example, if you want Stoat at `chat.yourdomain.com`:

```
A    chat    <your-vps-ip>    600
```

Verify propagation before continuing:

```bash
dig chat.yourdomain.com +short
# Should return your VPS IP
```

---

## Step 3 — Generate Config Files via SSH

SSH into your VPS and generate the Stoat config:

```bash
git clone https://github.com/stoatchat/self-hosted stoat
cd stoat
chmod +x ./generate_config.sh
./generate_config.sh your.domain
```

This creates `Revolt.toml`, `.env.web`, `Caddyfile`, and `livekit.yml`. You'll need the contents of all of these later.

Create the data directories:

```bash
mkdir -p ~/stoat/data/db
mkdir -p ~/stoat/data/rabbit
mkdir -p ~/stoat/data/minio
mkdir -p ~/stoat/data/caddy-data
mkdir -p ~/stoat/data/caddy-config
```

---

## Step 4 — Modify compose.yml for Traefik

The `compose.yml` needs two changes to work behind Traefik.

**Change 1:** Move Caddy to an internal port instead of 80/443:

```yaml
caddy:
  ports:
    - "1234:80"    # was "80:80" and "443:443"
```

**Change 2:** Remove the `env_file: .env.web` lines from both the `caddy` and `web` services (these will be set as environment variables in Coolify instead):

```yaml
# Remove this line from caddy service:
env_file: .env.web

# Remove this line from web service:
env_file: .env.web
```

**Change 3:** Set `createbuckets` to not restart after it finishes:

```yaml
createbuckets:
  restart: "no"
```

---

## Step 5 — Connect GitHub to Coolify

> **Note:** This step is only needed if you want to deploy from a Git repo. For the Docker Compose Empty approach (recommended), skip to Step 6.

In Coolify, go to **Settings → Sources → Add GitHub App** and follow the OAuth wizard. When asked for a Webhook Endpoint, choose the `https://` URL of your Coolify instance.

---

## Step 6 — Create a Docker Compose Resource in Coolify

1. In Coolify, go to your project → **New Resource**
2. Search for `docker` and select **Docker Compose Empty**
3. In the text editor that appears, paste the full contents of your modified `compose.yml`
4. Click **Save**

---

## Step 7 — Set the Domain on the Caddy Service

On the service stack page, find **Caddy** and click **Settings**:

- **Domains:** `https://your.domain`
- Uncheck **Strip Prefixes** (Stoat needs path prefixes like `/api`, `/ws` intact)
- Click **Save**

---

## Step 8 — Mount Config Files as Persistent Storage

Coolify auto-creates some storage entries from the compose volumes. You need to convert the config files from directories to actual files with content.

### Caddyfile

In the Caddy service → **Persistent Storages → Directories**, find the `Caddyfile` entry and click **Convert to file**. Confirm by typing the filepath shown, then paste your `Caddyfile` contents:

```
{$HOSTNAME} {
    route /api* {
        uri strip_prefix /api
        reverse_proxy http://api:14702 {
            header_down Location "^/" "/api/"
        }
    }
    route /ws {
        uri strip_prefix /ws
        reverse_proxy http://events:14703 {
            header_down Location "^/" "/ws/"
        }
    }
    route /autumn* {
        uri strip_prefix /autumn
        reverse_proxy http://autumn:14704 {
            header_down Location "^/" "/autumn/"
        }
    }
    route /january* {
        uri strip_prefix /january
        reverse_proxy http://january:14705 {
            header_down Location "^/" "/january/"
        }
    }
    route /gifbox* {
        uri strip_prefix /gifbox
        reverse_proxy http://gifbox:14706 {
            header_down Location "^/" "/gifbox/"
        }
    }
    route /livekit* {
        uri strip_prefix /livekit
        reverse_proxy http://livekit:7880 {
            header_down Location "^/" "/livekit/"
        }
    }
    route /ingress* {
        uri strip_prefix /ingress
        reverse_proxy http://voice-ingress:8500 {
            header_down Location "^/" "/ingress/"
        }
    }
    reverse_proxy http://web:5000
}
```

### Revolt.toml

For each of these services: **Api, Events, Autumn, January, Gifbox, Crond, Pushd, Voice Ingress** — go to their Persistent Storage, convert the `Revolt.toml` directory entry to a file, and paste your generated `Revolt.toml` contents (from `cat ~/stoat/Revolt.toml` on your VPS).

### livekit.yml

For the **Livekit** service, convert its storage entry to a file and paste the contents of `~/stoat/livekit.yml`.

---

## Step 9 — Add Environment Variables

Go to the main service stack → **Environment Variables** and add:

```
HOSTNAME=:80
REVOLT_PUBLIC_URL=https://your.domain/api
VITE_API_URL=https://your.domain/api
VITE_WS_URL=wss://your.domain/ws
VITE_MEDIA_URL=https://your.domain/autumn
VITE_PROXY_URL=https://your.domain/january
RABBITMQ_DEFAULT_USER=rabbituser
RABBITMQ_DEFAULT_PASS=rabbitpass
MINIO_ROOT_USER=minioautumn
MINIO_ROOT_PASSWORD=minioautumn
```

Replace `your.domain` with your actual domain.

---

## Step 10 — Deploy

Click **Deploy** and watch the logs. You should see all containers start up:

```
Container web-xxx Started
Container rabbit-xxx Started
Container database-xxx Started
Container caddy-xxx Started
Container api-xxx Started
Container events-xxx Started
Container autumn-xxx Started
...
```

Visit `https://your.domain` — you should see the Stoat login page.

> **Note:** The `createbuckets` container will show as "Restarting" if you didn't set `restart: "no"`. This is harmless — it's a one-time setup job — but setting it to `no` keeps things clean.

---

## Post-Deploy — Create Your Account

### If email verification is not configured

Stoat requires email verification by default, but without SMTP configured the verification email won't send. You can manually verify your account via the database.

In Coolify, go to **Terminal**, select the `database` container, click **Connect**, then run:

```
mongosh
use revolt
db.accounts.updateOne({ email: "your@email.com" }, { $set: { verification: { status: "Verified" } } })
```

Then log in normally at `https://your.domain`.

### If you want email verification to work

See [Configure SMTP](#optional--configure-smtp-for-email-verification) below.

---

## Optional — Configure SMTP for Email Verification

Add an `[email]` section to your `Revolt.toml`:

```toml
[email]
smtp_host = "smtp.yourprovider.com"
smtp_port = 587
smtp_username = "your@email.com"
smtp_password = "yourpassword"
smtp_from = "Stoat <noreply@yourdomain.com>"
```

Recommended free SMTP providers: **Resend**, **Brevo**, **Mailgun**.

---

## Troubleshooting

### `.env.web not found` error on deploy

You left `env_file: .env.web` in the compose file. Remove those lines from the `caddy` and `web` services — the variables are set in Coolify's Environment Variables instead.

### 503 on the domain

- Check Traefik is running: **Coolify → Servers → Proxy**
- Verify the Caddy service has the correct domain set in its Settings
- Make sure **Strip Prefixes** is unchecked on the Caddy service

### Containers failing to start

Check that all `Revolt.toml` file mounts are set as **Files** (not Directories) in Persistent Storage. If they're directories, Caddy/API/etc. can't read them as config files.

### `createbuckets` stuck in restart loop

Add `restart: "no"` to the `createbuckets` service in the compose file and redeploy.

### Can't log in with existing stoat.chat account

Your self-hosted instance has its own separate database. You need to create a new account on your instance — it's completely independent from stoat.chat.

---

## Updating

Pull the latest compose config from the official repo, check the [Notices](https://github.com/stoatchat/self-hosted#notices) for breaking changes, then update the compose file in Coolify via **Edit Compose File** and click **Deploy**.

---

**Last Updated:** February 2026  
**Based on:** Stoat Self-Hosted v0.11.1  
**Deployment Platform:** Coolify + Hetzner VPS