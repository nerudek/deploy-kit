<div align="center">

# Deploy Kit 🚀

**Ship to Production in One Command**

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![GitHub Stars](https://img.shields.io/github/stars/nerudek/deploy-kit?style=flat-square)](https://github.com/nerudek/deploy-kit/stargazers)
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)](https://github.com/nerudek/deploy-kit/pulls)

One-command deployment to VPS, Railway, Fly.io, or Vercel — pick your target and go live in minutes with pre-built configurations.

</div>

---

## Table of Contents

- [Features](#features)
- [Quick Start](#quick-start)
- [Platform Decision Tree](#platform-decision-tree)
- [1. VPS Deployment](#1-vps-deployment-hetznernetcupdigitalocean)
- [2. Railway Deployment](#2-railway-deployment)
- [3. Fly.io Deployment](#3-flyio-deployment)
- [4. Vercel Deployment](#4-vercel-deployment)
- [Environment Variables](#environment-variables)
- [Health Check Endpoint](#health-check-endpoint)
- [Common Pitfalls](#common-pitfalls)
- [FAQ](#faq)
- [Contributing](#contributing)
- [License](#license)
- [Support](#support)

---

## Features

- **One-command deploy** — run a single command, see your app live
- **Four platforms** — VPS (Docker), Railway, Fly.io, Vercel
- **Zero config for static sites** — Vercel setup in 30 seconds
- **Full control** — VPS option with custom binaries, cron, GPU, Docker Compose
- **Budget-friendly** — cheapest option from $3.50/mo (Fly.io) to $5/mo (Hetzner VPS)
- **Pre-configured** — ship with `railway.toml`, `fly.toml`, `vercel.json`, `deploy.sh` ready to go
- **Health checks** — automatic verification after deployment
- **Auto HTTPS** — Caddy/Certbot on VPS, built-in on platforms

## Quick Start

```bash
# Pick your platform and go:

# Vercel (static sites — fastest)
npm i -g vercel && vercel

# Railway (full-stack apps)
brew install railway && railway login && railway init && railway up --deploy

# Fly.io (edge-deployed containers)
brew install flyctl && fly launch && fly deploy

# VPS (Docker — full control)
# See VPS section below
```

All major platforms require just 2-3 environment variables/credentials to get started.

## Platform Decision Tree

```
Is it purely static?                    -> Vercel
Does it need a database + server?        -> Railway or Fly.io
Need full control (cron, GPU, bin)?      -> VPS
On a tight budget?                       -> VPS ($5/mo Hetzner/Netcup)
```

| Platform | Best For | Starting Cost | Setup Time |
|----------|----------|--------------|------------|
| Vercel   | Static sites, SPAs | Free | 30 seconds |
| Railway  | Full-stack, databases | $5/mo + usage | 2 minutes |
| Fly.io   | Edge containers, global | ~$3.50/mo | 2 minutes |
| VPS      | Docker, GPU, cron | $4-6/mo | 5 minutes |

---

## 1. VPS Deployment (Hetzner/Netcup/DigitalOcean)

### One-Time Setup (~5 minutes)

```bash
# On your VPS (Ubuntu 24.04)
apt update && apt install -y docker.io git nginx certbot python3-certbot-nginx
systemctl enable --now docker

# Optional: Tailscale for secure networking
curl -fsSL https://tailscale.com/install.sh | sh
tailscale up
```

### Deploy Script

Create `deploy.sh` in your repository root:

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

Usage:

```bash
DEPLOY_HOST=vps.tailnet.ts.net APP_NAME=my-app PORT=5173 bash deploy.sh
```

### Reverse Proxy with Caddy (auto HTTPS)

```caddyfile
myapp.yourdomain.com {
    reverse_proxy localhost:5173
}
```

---

## 2. Railway Deployment

Railway auto-detects your app type and builds using Nixpacks.

### Setup

```bash
brew install railway     # or: npm i -g @railway/cli
railway login
railway init
```

### Configuration (`railway.toml`)

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

### Deploy

```bash
railway up                # preview deployment
railway up --deploy       # production deployment
```

---

## 3. Fly.io Deployment

Fly.io runs containers at the edge with global regions.

### Setup

```bash
brew install flyctl
fly auth signup
fly launch                 # creates fly.toml from your app
```

### Configuration (`fly.toml`)

```toml
app = "my-app"
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

### Deploy

```bash
fly deploy
fly scale memory 512    # reduce to ~$3.50/mo
```

---

## 4. Vercel Deployment

Best for static sites, SPAs, and frameworks like Next.js, Vite, or Remix.

### Setup

```bash
npm i -g vercel
vercel login
vercel                    # interactive setup
```

### Configuration (`vercel.json`)

```json
{
  "buildCommand": "npm run build",
  "outputDirectory": "dist",
  "installCommand": "npm ci",
  "framework": "vite"
}
```

### Deploy

```bash
vercel                    # preview deployment
vercel --prod             # production deployment
```

---

## Environment Variables

Never commit real secrets to your repository. Use platform secret managers.

```bash
# .env.example — safe to commit
PORT=3000
DATABASE_URL=postgresql://user:pass@host/db
API_KEY=your-api-key-here
```

| Platform | How to set secrets |
|----------|-------------------|
| Railway  | Dashboard → Variables or `railway variables set KEY=value` |
| Fly.io   | `fly secrets set KEY=value` |
| Vercel   | Dashboard → Settings → Environment Variables |
| VPS      | `.env` file (add to `.gitignore`) or systemd env file |

---

## Health Check Endpoint

All platforms support health checks. Add this to your app:

```javascript
// Express / Fastify / Node.js
app.get('/health', (req, res) => {
  res.json({ status: 'ok', uptime: process.uptime() });
});
```

```python
# FastAPI
@app.get("/health")
async def health():
    return {"status": "ok", "uptime": time.time() - start_time}
```

---

## Common Pitfalls

1. **Deploying `.env` file** — add to `.gitignore`, use platform secrets
2. **Port mismatch** — app listens on 3000 but platform expects 8080
3. **No health check** — platform can't detect crashes without it
4. **Build on M1 Mac, deploy on x86_64** — use `--platform linux/amd64` in Dockerfile
5. **DNS not propagated** — wait up to 48h, verify with `dig yourdomain.com`

---

## FAQ

**Q: Which platform should I pick?**
Static site = Vercel. Full-stack = Railway. Docker/GPU = VPS. Edge-first global = Fly.io.

**Q: How much does each cost?**
Vercel: free tier. Railway: $5/mo + usage. Fly.io: ~$3.50/mo. VPS: $4-6/mo (Hetzner/Netcup).

**Q: Do I need a domain?**
No — all platforms offer a free subdomain. Custom domain (~$10/year) for professionalism.

**Q: How do I set up a custom domain?**
Buy on Porkbun/Namecheap. Add a CNAME record pointing to the platform domain. Platforms auto-provision SSL.

**Q: Can I deploy a Python/FastAPI app?**
Yes — Railway and Fly.io auto-detect Python. For VPS, use `Dockerfile` with `python:3.12-slim`.

**Q: How to handle file uploads in production?**
Use S3-compatible storage (Cloudflare R2, Backblaze B2). Local disk is ephemeral on platforms.

**Q: Can I deploy multiple services from one repo?**
Railway: monorepo with multiple services. Fly.io: separate apps. VPS: Docker Compose.

**Q: What about staging environments?**
Railway: `railway environment create staging`. Vercel: preview deploys on PR. Fly.io: `fly deploy -a app-staging`.

**Q: How do I rollback?**
Railway: `railway rollback`. Fly.io: `fly deploy --image registry.fly.io/app:previous`. Vercel: promote previous deploy. VPS: `git checkout HEAD~1 && docker compose up -d --build`.

**Q: Can I run background jobs/cron?**
Railway: cron service. VPS: native crontab. Vercel: Vercel Cron Jobs. Fly.io: Machines API.

**Q: How to monitor my deployed app?**
All platforms have built-in logging. Add Uptime Robot (free) for uptime monitoring.

**Q: My deploy fails with "out of memory" — what now?**
Increase RAM in platform settings. For VPS: add swap space or upgrade the plan.

**Q: How to debug a production crash?**
`fly logs`, `railway logs`, or check application-level logs (not just platform logs).

**Q: Can I use WebSockets?**
Railway: yes. Fly.io: yes. Vercel: serverless only — use SSE or move to Railway. VPS: yes.

**Q: What about database backups?**
Railway: automatic daily backups. VPS: `pg_dump` to cron + rsync. Fly.io: `fly postgres connect` + manual dump.

---

## Contributing

Contributions are welcome! Here's how to help:

1. **Fork** the repository
2. **Create a feature branch:** `git checkout -b feat/my-feature`
3. **Commit your changes:** `git commit -am 'feat: add new platform config'`
4. **Push:** `git push origin feat/my-feature`
5. **Open a Pull Request**

Please ensure your changes follow the existing structure and include configuration for at least one deployment platform.

---

## License

This project is licensed under the MIT License. See the [LICENSE](LICENSE) file for details.

---

## Support

- **Documentation:** See [SKILL.md](./SKILL.md) for the comprehensive deployment guide
- **Issues:** [GitHub Issues](https://github.com/nerudek/deploy-kit/issues)
- **Author:** [@nerudek](https://github.com/nerudek) on GitHub

---

<div align="center">

If this saved you time: [PayPal.me/nerudek](https://www.paypal.me/nerudek)

⭐ Star the repo if you find this useful!

</div>
