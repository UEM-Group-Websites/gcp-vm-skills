---
name: gcp-node-vm-deploy
description: >
  Deploy a Node.js backend to a GCP Compute Engine VM and set up automated GitHub Actions deploys.
  Use this skill whenever the user wants to: provision a GCP VM for a Node.js/Express app, migrate
  from Vercel/serverless to a persistent VM, set up PM2 + Caddy + PostgreSQL on a GCP instance,
  configure SSH-based GitHub Actions CD, or perform any "deploy my backend to GCP" request.
  Also use when the user mentions deploy scripts (setup-vm.sh, deploy.sh, Caddyfile, ecosystem.config.cjs,
  backup.sh) in a GCP context. Even if the user only says "help me set up auto-deploy to GCP", use this skill.
---

# GCP Node.js VM Deploy

This skill walks through provisioning a GCP Compute Engine VM, bootstrapping it with Node.js +
PostgreSQL + Caddy + PM2, deploying a Node.js app, and wiring up GitHub Actions for automated
deploys on every push to `main`.

**Stack (opinionated defaults, all overridable):**
- Machine: `e2-small` (1 vCPU / 2 GB RAM), Ubuntu 24.04 LTS
- Node.js 24.x (via NodeSource), Reverse proxy: Caddy (auto Let's Encrypt HTTPS)
- Process manager: PM2 (systemd-managed, survives reboots)
- Database: PostgreSQL 16 on the same VM (skip if the app uses an external DB)
- Auto-deploy: raw SSH + runner-built `.env` via GitHub Actions on push to `main`
- 2 GB swap (kernel-tuned), UFW firewall, unattended security upgrades, nightly DB backup

---

## Phase 0 — Gather information

Before touching any infrastructure, collect these values. Ask the user for anything not obvious
from context:

| Variable | Example | Notes |
|---|---|---|
| `GCP_PROJECT` | `hosting-vms` | GCP project ID |
| `GCP_ZONE` | `us-central1-a` | Pick the zone where most other VMs live |
| `VM_NAME` | `myapp-backend-vm` | Kebab-case, descriptive |
| `STATIC_IP_NAME` | `myapp-backend` | Used as the address name in gcloud |
| `APP_NAME` | `myapp-api` | PM2 process name, log prefix |
| `APP_DIR` | `/var/www/myapp` | Where the app lives on the VM |
| `PORT` | `3000` | The port Node.js listens on |
| `MAIN_SCRIPT` | `src/server.js` | Entry point passed to PM2 |
| `DOMAIN` | `api.example.com` | The public hostname (for Caddy TLS) |
| `REPO_URL` | `https://github.com/org/repo.git` | HTTPS clone URL |
| `IS_PRIVATE_REPO` | `false` | If true, need a GitHub PAT for cloning |

Also ask: does the project already have `deploy/setup-vm.sh`, `deploy/deploy.sh`,
`deploy/Caddyfile`, `deploy/backup.sh`, `.github/workflows/deploy.yml`, and `ecosystem.config.cjs`?

---

## Phase 1 — Scaffold deploy files (if missing)

**Read `references/templates.md`** for the full content of each file. Every template uses
`{{PLACEHOLDER}}` syntax — find-and-replace all placeholders with real values before writing.

Files to create if any are missing:

| File | Purpose |
|---|---|
| `ecosystem.config.cjs` | PM2 process config (name, script, memory limit, log paths) |
| `deploy/setup-vm.sh` | One-shot idempotent bootstrap — run once as root on the VM |
| `deploy/deploy.sh` | Per-deploy: git pull, `npm ci`, Prisma, PM2 reload; supports `SKIP_GIT=1` |
| `deploy/Caddyfile` | Reverse proxy: TLS, gzip, 20 MB upload cap, JSON access log |
| `deploy/backup.sh` | Nightly `pg_dump` with dual-bound retention (14d age + 10 count) |
| `.github/workflows/deploy.yml` | Runner builds `.env` from GitHub secrets, ships via `scp`, calls `deploy.sh SKIP_GIT=1` |

After writing all files:
```bash
git add deploy/ .github/ ecosystem.config.cjs
git commit -m "chore: add gcp vm deploy infrastructure"
```

---

## Phase 2 — Push all commits to remote

```bash
git push origin main
```

---

## Phase 3 — Provision the VM

### 3a. Reserve a static external IP

```bash
gcloud compute addresses create {{STATIC_IP_NAME}} \
  --project={{GCP_PROJECT}} \
  --region={{GCP_REGION}}

gcloud compute addresses describe {{STATIC_IP_NAME}} \
  --project={{GCP_PROJECT}} \
  --region={{GCP_REGION}} \
  --format="value(address)"
```

### 3b. Create the VM

```bash
gcloud compute instances create {{VM_NAME}} \
  --project={{GCP_PROJECT}} \
  --zone={{GCP_ZONE}} \
  --machine-type=e2-small \
  --image-family=ubuntu-2404-lts-amd64 \
  --image-project=ubuntu-os-cloud \
  --boot-disk-size=20GB \
  --boot-disk-type=pd-standard \
  --tags=http-server,https-server \
  --address={{STATIC_IP_NAME}}
```

Add GCP-level firewall rules if they don't exist (setup-vm.sh also enables UFW — both layers needed):
```bash
gcloud compute firewall-rules create default-allow-http  --project={{GCP_PROJECT}} --direction=INGRESS --action=ALLOW --rules=tcp:80  --target-tags=http-server
gcloud compute firewall-rules create default-allow-https --project={{GCP_PROJECT}} --direction=INGRESS --action=ALLOW --rules=tcp:443 --target-tags=https-server
```

---

## Phase 4 — Bootstrap the VM

Copy scripts then run the bootstrap:
```bash
gcloud compute scp --recurse deploy/ {{VM_NAME}}:/tmp/deploy-scripts \
  --project={{GCP_PROJECT}} --zone={{GCP_ZONE}}

gcloud compute ssh {{VM_NAME}} --project={{GCP_PROJECT}} --zone={{GCP_ZONE}} \
  --command="sudo bash /tmp/deploy-scripts/setup-vm.sh 2>&1"
```

**Capture the DB password** from the output line: `-> Generated DB password for {{DB_USER}}: <hex-string>`

---

## Phase 5 — Generate GitHub Actions SSH key

```bash
ssh-keygen -t ed25519 -C "github-actions@{{VM_NAME}}" -f /tmp/deploy-key -N ""
PUB_KEY=$(cat /tmp/deploy-key.pub)
gcloud compute ssh {{VM_NAME}} --project={{GCP_PROJECT}} --zone={{GCP_ZONE}} \
  --command="echo '${PUB_KEY}' | sudo tee -a /home/deploy/.ssh/authorized_keys \
             && sudo chmod 600 /home/deploy/.ssh/authorized_keys"
```

---

## Phase 6 — Clone the repo and write the initial .env

```bash
gcloud compute ssh {{VM_NAME}} \
  --project={{GCP_PROJECT}} --zone={{GCP_ZONE}} \
  --command="sudo -u deploy git clone {{REPO_URL}} {{APP_DIR}}"
```

Write the production `.env` for the first deploy (after this, CI delivers it on every push):
```bash
gcloud compute ssh {{VM_NAME}} \
  --project={{GCP_PROJECT}} --zone={{GCP_ZONE}} \
  --command="sudo tee {{APP_DIR}}/.env > /dev/null << 'ENVEOF'
PORT={{PORT}}
NODE_ENV=production
DATABASE_URL=postgresql://{{DB_USER}}:{{DB_PASS}}@localhost:5432/{{DB_NAME}}
DIRECT_URL=postgresql://{{DB_USER}}:{{DB_PASS}}@localhost:5432/{{DB_NAME}}
JWT_SECRET={{JWT_SECRET}}
CORS_ORIGINS={{CORS_ORIGINS}}
UPLOAD_DIR={{APP_DIR}}/uploads
ENVEOF
sudo chown deploy:deploy {{APP_DIR}}/.env && sudo chmod 600 {{APP_DIR}}/.env"
```

---

## Phase 7 — First deploy + smoke test + backup cron

```bash
SSH="gcloud compute ssh {{VM_NAME}} --project={{GCP_PROJECT}} --zone={{GCP_ZONE}} --command"

$SSH "sudo -u deploy bash {{APP_DIR}}/deploy/deploy.sh 2>&1"
$SSH "curl -s http://localhost:{{PORT}}/healthz"

# Nightly backup cron (runs at 02:00 as root)
$SSH "(crontab -l 2>/dev/null; echo '0 2 * * * {{APP_DIR}}/deploy/backup.sh >> {{LOG_DIR}}/backup.log 2>&1') | crontab -"
```

---

## Phase 8 — Wire up GitHub Actions

The workflow builds `.env` from GitHub secrets on every deploy, so **every env var the app needs must be a secret**. Set them all:

```bash
PRIVATE_KEY=$(cat /tmp/deploy-key)

gh secret set VM_HOST     --repo {{GITHUB_REPO}} --body "{{STATIC_IP}}"
gh secret set VM_USER     --repo {{GITHUB_REPO}} --body "deploy"
gh secret set VM_SSH_KEY  --repo {{GITHUB_REPO}} --body "${PRIVATE_KEY}"
gh secret set VM_SSH_PORT --repo {{GITHUB_REPO}} --body "22"

# One command per app env var — match exactly what's in the deploy.yml heredoc:
gh secret set PORT         --repo {{GITHUB_REPO}} --body "{{PORT}}"
gh secret set NODE_ENV     --repo {{GITHUB_REPO}} --body "production"
gh secret set UPLOAD_DIR   --repo {{GITHUB_REPO}} --body "{{APP_DIR}}/uploads"
gh secret set JWT_SECRET   --repo {{GITHUB_REPO}} --body "{{JWT_SECRET}}"
gh secret set CORS_ORIGINS --repo {{GITHUB_REPO}} --body "{{CORS_ORIGINS}}"
gh secret set DATABASE_URL --repo {{GITHUB_REPO}} --body "postgresql://{{DB_USER}}:{{DB_PASS}}@localhost:5432/{{DB_NAME}}"
gh secret set DIRECT_URL   --repo {{GITHUB_REPO}} --body "postgresql://{{DB_USER}}:{{DB_PASS}}@localhost:5432/{{DB_NAME}}"
# Add any other vars the app needs (EMAIL_USER, ADMIN_PASSWORD, etc.)

rm -f /tmp/deploy-key /tmp/deploy-key.pub
```

---

## Phase 9 — Verify automated deployments

```bash
gh workflow run "Deploy to GCP VM" --repo {{GITHUB_REPO}} --ref main
gh run watch $(gh run list --repo {{GITHUB_REPO}} --limit 1 --json databaseId -q '.[0].databaseId') --repo {{GITHUB_REPO}}
```

---

## Phase 10 — Communicate remaining manual steps

Always tell the user:

1. **DNS** — add an `A` record: `{{DOMAIN}}` → `{{STATIC_IP}}`. Caddy cannot issue a TLS cert until this resolves.
2. **HTTPS check** — after DNS propagates: `curl https://{{DOMAIN}}/healthz`
3. **CORS** — update `CORS_ORIGINS` GitHub secret; the next push will pick it up automatically
4. **Other secrets** — any blank env vars (email, extra admin fields) must be added as GitHub secrets and reflected in the `deploy.yml` heredoc
5. **GCS backups** — when ready to graduate from local-only backups, uncomment the `gsutil` block in `backup.sh` and provision a service account + bucket

---

## Troubleshooting: SSH Permission denied (publickey)

When GitHub Actions fails with `Permission denied (publickey)`:

1. Verify public key is in `/home/deploy/.ssh/authorized_keys` on the VM
2. Check permissions: `.ssh/` must be `700`, `authorized_keys` must be `600`
3. Confirm `VM_SSH_KEY` secret contains the **private** key (not the `.pub` file)
4. Test manually: `ssh -i /tmp/deploy-key deploy@<VM_IP> "echo OK"`
5. Key must be ed25519 or RSA — not DSA or an unrecognized format
