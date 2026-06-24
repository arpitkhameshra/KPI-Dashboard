# KPI Dashboard Deployment Guide

Complete step-by-step guide for deploying the KPI Dashboard on Ubuntu using PM2, Nginx, SSL, and Bun.

---

## Table of Contents

- [Server Details](#server-details)
- [Prerequisites](#prerequisites)
- [Clone Repository](#1-clone-repository)
- [Environment Variables](#2-configure-environment-variables)
- [Install Dependencies](#3-install-dependencies)
- [Build Application](#4-build-application)
- [Start with PM2](#5-start-application-with-pm2)
- [Verify Deployment](#6-verify-deployment)
- [Configure Nginx](#7-configure-nginx)
- [Configure SSL](#8-configure-ssl)
- [Automated Deployment](#automated-deployment)
- [Troubleshooting](#troubleshooting)

---

## Server Details

| Item | Value |
|--------|--------|
| Server IP | `13.200.74.191` |
| Domain | `kpi.webledger.in` |
| DNS | Route53 A Record → `13.200.74.191` |
| Application Port | `4000` |
| Project Path | `/home/ubuntu/kpi-dashboard` |
| Process Manager | PM2 |
| Reverse Proxy | Nginx |
| SSL | Let's Encrypt |
| Package Manager | Bun |
| Runtime | Node.js (Nitro `node-server`) |

---

## Prerequisites

Ensure the following are installed:

- Ubuntu Server
- Node.js
- Bun
- PM2
- Nginx
- Git

Install PM2:

```bash
npm install -g pm2
```

---

## 1. Clone Repository

```bash
cd /home/ubuntu
git clone <repo-url> kpi-dashboard
cd kpi-dashboard
```

---

## 2. Configure Environment Variables

Create:

```bash
/home/ubuntu/kpi-dashboard/.env
```

```env
SUPABASE_PROJECT_ID="your_project_id"
SUPABASE_PUBLISHABLE_KEY="your_anon_key"
SUPABASE_URL="https://your_project_id.supabase.co"

VITE_SUPABASE_PROJECT_ID="your_project_id"
VITE_SUPABASE_PUBLISHABLE_KEY="your_anon_key"
VITE_SUPABASE_URL="https://your_project_id.supabase.co"
```

### Why both versions are required?

#### Build-Time Variables

Used by Vite during frontend build:

```env
VITE_SUPABASE_PROJECT_ID
VITE_SUPABASE_PUBLISHABLE_KEY
VITE_SUPABASE_URL
```

#### Runtime Variables

Loaded by the Node.js server via dotenv:

```env
SUPABASE_PROJECT_ID
SUPABASE_PUBLISHABLE_KEY
SUPABASE_URL
```

> ⚠️ `.env` is ignored by Git and must be created manually on the server.

---

## 3. Install Dependencies

```bash
bun install
```

---

## 4. Build Application

```bash
bun run build
```

Verify output:

```bash
ls -la .output/server/index.mjs
```

Expected:

```text
.output/server/index.mjs
```

---

## 5. Start Application with PM2

Delete existing process:

```bash
pm2 delete kpi-dashboard 2>/dev/null || true
```

Start server:

```bash
PORT=4000 pm2 start .output/server/index.mjs \
  --name kpi-dashboard \
  --cwd /home/ubuntu/kpi-dashboard
```

Save PM2 configuration:

```bash
pm2 save
```

> ⚠️ The `--cwd` flag is mandatory. Without it, dotenv may not find the `.env` file.

---

## 6. Verify Deployment

Check status:

```bash
pm2 ls
```

View logs:

```bash
pm2 logs kpi-dashboard --lines 20
```

Test locally:

```bash
curl http://localhost:4000
```

Expected logs:

```text
[dotenv] loaded env vars...
Listening on: http://localhost:4000/
```

---

## 7. Configure Nginx

Create:

```bash
sudo nano /etc/nginx/sites-available/kpi.webledger.in
```

Add:

```nginx
server {
    server_name kpi.webledger.in;

    location / {
        proxy_pass http://localhost:4000;
        proxy_http_version 1.1;

        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";

        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;

        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    listen 443 ssl;

    ssl_certificate /etc/letsencrypt/live/kpi.webledger.in/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/kpi.webledger.in/privkey.pem;

    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;
}

server {
    listen 80;
    server_name kpi.webledger.in;

    return 301 https://$host$request_uri;
}
```

Enable:

```bash
sudo ln -sf /etc/nginx/sites-available/kpi.webledger.in \
/etc/nginx/sites-enabled/kpi.webledger.in
```

Validate:

```bash
sudo nginx -t
```

Reload:

```bash
sudo systemctl reload nginx
```

---

## 8. Configure SSL

```bash
sudo apt update
sudo apt install certbot python3-certbot-nginx -y

sudo certbot --nginx -d kpi.webledger.in
```

Verify renewal:

```bash
sudo certbot renew --dry-run
```

---

## Automated Deployment

Run:

```bash
bash /home/ubuntu/kpi-dashboard/deploy.sh
```

### Deployment Workflow

1. Loads Bun and Node paths
2. Loads `.env`
3. Pulls latest code
4. Checks dependency changes
5. Updates repository
6. Runs `bun install` if needed
7. Builds application
8. Restarts PM2
9. Reloads Nginx

---

## Troubleshooting

### 502 Bad Gateway

```bash
pm2 ls
pm2 logs kpi-dashboard --lines 50
```

Restart:

```bash
pm2 delete kpi-dashboard

PORT=4000 pm2 start .output/server/index.mjs \
  --name kpi-dashboard \
  --cwd /home/ubuntu/kpi-dashboard
```

### Missing Supabase Variables

Error:

```text
[Supabase] Missing Supabase environment variable(s)
```

Verify:

```bash
cat .env
```

Rebuild:

```bash
bun run build
```

Restart PM2.

### Clear PM2 Logs

```bash
pm2 flush kpi-dashboard
```

### Verify Port

```bash
sudo lsof -i :4000
```

---

## Important Architecture Decision

### Correct Configuration

```ts
export default defineConfig({
  nitro: {
    preset: "node-server",
  },

  tanstackStart: {
    server: {
      entry: "server",
    },
  },
});
```

### Incorrect Configuration

```ts
nitro: true
```

### Why?

Lovable defaults to the Cloudflare preset:

```text
cloudflare-module
```

Cloudflare Workers do not support:

- `process.env`
- `fs`
- Node.js runtime APIs

This breaks:

- dotenv
- runtime environment variables
- Supabase configuration

For Ubuntu + PM2 deployments, use:

```ts
preset: "node-server"
```

---

## Deployment Checklist

- [ ] Route53 record created
- [ ] Repository cloned
- [ ] `.env` file created
- [ ] Dependencies installed
- [ ] Application built
- [ ] PM2 running
- [ ] Nginx configured
- [ ] SSL issued
- [ ] Domain accessible
- [ ] PM2 saved

---

## Deployment Complete 🚀
