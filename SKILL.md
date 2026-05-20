---
name: deploy-kit
description: One-command deployment to VPS, Railway, Fly.io, or Vercel — pick your target and go live in minutes with pre-built configurations
version: 1.0.0
author: nerudek
compatible-with: hermes-agent, claude-code, openclaw
---

# Deploy Kit — Ship to Production in One Command

## Problem

You built something cool but have no idea how to put it online. VPS setup takes days. Platform docs are 200 pages. Every deployment tutorial assumes you already know DNS, SSL, and Linux admin. You just want `deploy` and see it live.

## Solution

Pre-configured deployment configs for four platforms. Pick your target, set 2-3 secrets, run one command. Works for static sites, Node.js APIs, Python backends, and Docker apps.

## Platform Decision Tree

```
Is it purely static? -> Vercel
Does it need a database + custom server? -> Railway or Fly.io
Do you need full control (cron, custom binaries, GPU)? -> VPS
Are you on a tight budget? -> VPS ($5/mo Hetzner/Netcup)
```

## 1. VPS Deployment (Hetzner/Netcup/DigitalOcean)

### Setup (one-time, 5 minutes)

```bash
# On your VPS (Ubuntu 24.04)
apt update && apt install -y docker.io git nginx certbot python3-certbot-nginx
systemctl enable --now docker
curl -fsSL https://tailscale.com/install.sh | sh
tailscale up
```

### Deploy Script

Create `deploy.sh` in your repo:

```bash
#!/bin/bash
set -e
HOST=${DEPLOY_HOST:-your-vps.tailnet.ts.net}
APP_DIR=/app/$APP_NAME

rsync -avz --delete --exclude '.git' --exclude 'node_modules' ./ $HOST:$APP_DIR/
ssh $HOST "cd $APP_DIR && docker compose up -d --build"
sleep 3
curl -sf http://$HOST:$PORT/health && echo " OK" || echo " FAIL"
```

Usage: `DEPLOY_HOST=vps.tailnet.ts.net APP_NAME=banner-editor PORT=5173 bash deploy.sh`

### Reverse Proxy with Caddy (auto HTTPS)

```caddyfile
banner.yourdomain.com {
    reverse_proxy localhost:5173
}
```

## 2. Railway Deployment

```bash
brew install railway
railway login
railway init
```

`railway.toml`:
```toml
[build]
builder = "nixpacks"
buildCommand = "npm run build"

[deploy]
startCommand = "node dist/server.js"
healthcheckPath = "/health"

[service]
port = 3000

[tools.node]
version = "22"
```

```bash
railway up          # preview
railway up --deploy # production
```

## 3. Fly.io Deployment

```bash
brew install flyctl
fly auth signup
fly launch
```

`fly.toml` (auto-generated, tweak as needed):
```toml
app = "banner-editor"
primary_region = "ams"

[build]
  [build.args]
    NODE_VERSION = "22"

[http_service]
  internal_port = 5173
  force_https = true
  auto_stop_machines = "stop"
  auto_start_machines = true

[[vm]]
  memory = "1gb"
  cpu_kind = "shared"
  cpus = 1
```

```bash
fly deploy
fly scale memory 512  # ~$3.50/mo
```

## 4. Vercel Deployment

```bash
npm i -g vercel
vercel login
vercel
```

`vercel.json`:
```json
{
  "buildCommand": "npm run build",
  "outputDirectory": "dist",
  "installCommand": "npm ci",
  "framework": "vite"
}
```

```bash
vercel            # preview
vercel --prod     # production
```

## Environment Variables

```bash
# .env.example (never commit real values)
PORT=3000
DATABASE_URL=postgresql://user:pass@host/db
API_KEY=your-api-key-here
```

## Health Check Endpoint

```javascript
app.get('/health', (req, res) => {
  res.json({ status: 'ok', uptime: process.uptime() });
});
```

## Common Pitfalls

1. Deploying `.env` file -- add to `.gitignore`, use platform secrets
2. Port mismatch -- app listens on 3000 but platform expects 8080
3. No health check -- platform can't detect crashes
4. Build on M1 Mac, deploy on x86_64 -- use `--platform linux/amd64` in Dockerfile
5. DNS not propagated -- wait up to 48h, verify with `dig yourdomain.com`

## FAQ

**Q: Which platform should I pick?**
Static site = Vercel. Full-stack = Railway. Docker/GPU = VPS. Edge-first global = Fly.io.

**Q: How much does each cost?**
Vercel: free tier. Railway: $5/mo + usage. Fly.io: ~$3.50/mo. VPS: $4-6/mo (Hetzner/Netcup).

**Q: Do I need a domain?**
No -- all platforms give free subdomain. Custom domain ($10/year) for professionalism.

**Q: How do I set up a custom domain?**
Buy on Porkbun/Namecheap. Add CNAME pointing to platform domain. Platform auto-provisions SSL.

**Q: Can I deploy a Python/FastAPI app?**
Yes -- Railway and Fly.io auto-detect. For VPS, use `Dockerfile` with `python:3.12-slim`.

**Q: How to handle file uploads in production?**
Use S3-compatible storage (Cloudflare R2, Backblaze B2). Local disk is ephemeral on platforms.

**Q: Can I deploy multiple services from one repo?**
Railway: monorepo with multiple services. Fly.io: separate apps. VPS: docker compose.

**Q: What about staging environments?**
Railway: `railway environment create staging`. Vercel: preview deploys on PR. Fly.io: `fly deploy -a app-staging`.

**Q: How do I rollback?**
Railway: `railway rollback`. Fly.io: `fly deploy --image registry.fly.io/app:previous`. Vercel: Promote previous deploy. VPS: `git checkout HEAD~1 && docker compose up -d --build`.

**Q: Can I run background jobs/cron?**
Railway: cron service. VPS: native crontab. Vercel: Vercel Cron Jobs. Fly.io: Machines API.

**Q: How to monitor my deployed app?**
All platforms have built-in logs. Add Uptime Robot (free). See `skill-post-deploy` for comprehensive monitoring.

**Q: My deploy fails with "out of memory" -- what now?**
Increase RAM in platform settings. For VPS: add swap or upgrade plan.

**Q: How to debug a production crash?**
`fly logs`, `railway logs`, check application logs (not just platform logs).

**Q: Can I use WebSockets?**
Railway: yes. Fly.io: yes. Vercel: serverless only -- use SSE or move to Railway. VPS: yes.

**Q: What about database backups?**
Railway: automatic daily. VPS: `pg_dump` to cron + rsync. Fly.io: `fly postgres connect` + manual dump.

---

If this saved you time: [PayPal.me/nerudek](https://www.paypal.me/nerudek)
GitHub: [github.com/nerudek](https://github.com/nerudek)
