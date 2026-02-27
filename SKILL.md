---
name: deploy-vercel-cloudflare
description: Create a new web project, push to GitHub private repo, deploy to Vercel, and bind a Cloudflare subdomain. Use when the user wants to create and deploy a new web app, set up a project with Vercel hosting, configure a custom domain via Cloudflare DNS, or any combination of these deployment steps.
---

# Deploy Project to Vercel + Cloudflare Domain

End-to-end workflow: create project → GitHub private repo → Vercel deploy → Cloudflare subdomain.

## Prerequisites

Verify these CLI tools are authenticated before starting:

```bash
gh auth status        # GitHub CLI
vercel whoami         # Vercel CLI (if expired: vercel login --github)
wrangler whoami       # Cloudflare Wrangler (if needed: wrangler login)
```

If any tool is not logged in, run the corresponding login command first.

## Workflow

### Step 1: Create Project

Scaffold the project in the target directory. Adapt the command to the framework the user wants:

```bash
# Next.js example
npx create-next-app@latest . --typescript --tailwind --eslint --app --src-dir --no-import-alias --use-npm
```

Other common starters:
- Nuxt: `npx nuxi@latest init .`
- Vite + React: `npm create vite@latest . -- --template react-ts`
- Astro: `npm create astro@latest -- --template minimal .`

Build to verify before proceeding:

```bash
npm run build
```

### Step 2: Push to GitHub Private Repo

```bash
git add -A
git commit -m "feat: initial commit"
gh repo create <repo-name> --private --source=. --push --description "<description>"
```

- `<repo-name>`: infer from directory name or ask the user
- The `--source=. --push` flags handle remote setup and initial push in one command

### Step 3: Deploy to Vercel

```bash
vercel --prod --yes
```

This will:
- Auto-detect the framework
- Link/create the Vercel project
- Connect the GitHub repo for automatic deployments
- Deploy to production

Note the production URL from the output (e.g. `https://<project>-<suffix>.vercel.app`).

### Step 4: Bind Cloudflare Subdomain

This step requires two things: adding the domain in Vercel, and creating a CNAME record in Cloudflare.

#### 4a. Add custom domain to Vercel project

```bash
vercel domains add <subdomain>.<domain>
```

If `vercel domains add` reports a permission error on the inspect step but says "Success! Domain ... added", it still worked — verify via API:

```bash
# Get project ID from .vercel/project.json
cat .vercel/project.json

# Check domain status via Vercel API
VERCEL_TOKEN=$(python3 -c "import json; print(json.load(open('<path-to-vercel-auth.json>'))['token'])")
curl -s "https://api.vercel.com/v10/projects/<projectId>/domains?teamId=<orgId>" \
  -H "Authorization: Bearer $VERCEL_TOKEN" | python3 -m json.tool
```

The Vercel auth.json location varies by system. Common paths:
- `~/.local/share/com.vercel.cli/auth.json`
- `~/Library/Application Support/com.vercel.cli/auth.json`

#### 4b. Create CNAME record in Cloudflare

The wrangler OAuth token typically lacks DNS write scope. Use a Cloudflare API Token with **Zone > DNS > Edit** permission.

```bash
# 1. Get zone ID
CF_TOKEN="<cloudflare-api-token>"
curl -s "https://api.cloudflare.com/client/v4/zones?name=<domain>" \
  -H "Authorization: Bearer $CF_TOKEN" | python3 -c "
import sys, json
data = json.load(sys.stdin)
print(data['result'][0]['id'])
"

# 2. Create CNAME record
ZONE_ID="<zone-id>"
curl -s -X POST "https://api.cloudflare.com/client/v4/zones/$ZONE_ID/dns_records" \
  -H "Authorization: Bearer $CF_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "type": "CNAME",
    "name": "<subdomain>",
    "content": "cname.vercel-dns.com",
    "ttl": 1,
    "proxied": false
  }'
```

**Important**: Set `proxied: false` (DNS only / grey cloud) to let Vercel handle SSL.

#### 4c. Verify

```bash
# DNS resolution
dig <subdomain>.<domain> CNAME +short
# Should return: cname.vercel-dns.com.

# HTTP access
curl -sI https://<subdomain>.<domain> | head -5
# Should return: HTTP/2 200 with server: Vercel
```

## Asking the User

If information is missing, ask for:

| Info | Example |
|------|---------|
| Project framework | Next.js, Nuxt, Vite, Astro |
| GitHub repo name | `my-awesome-app` |
| Root domain managed in Cloudflare | `xzd.me` |
| Desired subdomain | `my-app` → `my-app.xzd.me` |
| Cloudflare API Token (DNS edit) | User provides or creates at dash.cloudflare.com/profile/api-tokens |

## Partial Execution

The user may only need a subset of steps. Each step is independent — skip steps that are already done or not requested:

- "Deploy to Vercel" → Steps 3 only
- "Add a domain" → Steps 4 only
- "Create and push to GitHub" → Steps 1-2 only
- "Full setup" → All steps
