# Freshsite Site Setup Instructions

## Overview

These are the canonical steps for setting up a new client website under the freshsite studio operation. Follow this guide in order. All new sites use **Astro.build** as the framework and **GitHub Actions** for CI/CD.

---

## Step 1 — GitHub App Setup (one-time)

### 1a — Create the GitHub App

Go to: **https://github.com/organizations/freshsitestudio/settings/apps**

Click **"New GitHub App"**

| Field | Value |
|---|---|
| **GitHub App name** | `orca-freshsite` |
| **Description** | `Orca automation for freshsite studio` |
| **Homepage URL** | `https://freshsite.studio` |
| **Webhook > Active** | ❌ Uncheck |
| **Where can this GitHub App be installed?** | Only on this account |

**Permissions — Repository permissions:**

| Permission | Access |
|---|---|
| Contents | Read and write |
| Metadata | Read only |
| Pages | Read and write |
| Pull requests | Read and write |
| Administration | Read and write |

**Permissions — Organization permissions:**

| Permission | Access |
|---|---|
| Administration | Read and write |

### 1b — Generate a private key

On the App page → **"Private keys"** → **"Generate a private key"**

Save the downloaded `.pem` file to a secure location, e.g. `~/.ssh/orca-freshsite.pem`

### 1c — Install the App on the org

Go to the App's page → **"Install"** → select **"All repositories"**

### 1d — Store credentials in 1Password

Create an item called **`orca-freshsite github app`** in the shared vault (`walters-shared-passwords`) with three fields:

- `app_id` — shown at top of App page
- `installation_id` — found in URL when viewing the installation: `https://github.com/organizations/freshsitestudio/settings/installations/143053223`
- `private_key` — paste the full `.pem` file contents (field type: SSH Key)

### 1e — Cloudflare API token (one-time)

Create a Cloudflare API token at **https://dash.cloudflare.com/profile/api-tokens**

Template: **"Edit zone DNS"** — scope to `freshsite.studio` zone only.

Store in 1Password as **`Cloudflare - Local/admin automation token Freshsite.studio`** in the shared vault. The field name for the token must be `credential`.

> Note: FreeParking manages `corda.co.nz` DNS externally. For sites on other registrars, DNS must be migrated to Cloudflare or manual records must be added by the client.

---

## Step 2 — Astro Project Setup (local)

### 2a — Create Astro project

```bash
cd /mnt/ai/freshsite/mangrove/
npm create astro@latest companyname-astro -- --template minimal --no-git --no-install -y
cd companyname-astro
npm install
npx astro add tailwind -y
npm run build   # verify it builds
```

### 2b — Add GitHub Actions workflow

```bash
mkdir -p companyname-astro/.github/workflows
```

Create `.github/workflows/deploy.yml`:

```yaml
name: Deploy to GitHub Pages

on:
  push:
    branches: [main]
  workflow_dispatch:

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: pages
  cancel-in-progress: false

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      - name: Install, build, upload
        uses: actions/upload-pages-artifact@v3
        with:
          path: dist

  deploy:
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    needs: build
    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
```

---

## Step 3 — Create the GitHub Repository

### 3a — Create public repo via GitHub App

GitHub Pages requires the repo to be **public** (this is a GitHub limitation, not a billing issue — Enterprise is NOT required).

Use the GitHub App JWT to get an installation token, then create the repo:

```python
import jwt, time, urllib.request, json

with open('/path/to/orca-freshsite.pem') as f:
    pk = f.read()

APP_ID = 'APP_ID'
INSTALLATION_ID = 'INSTALLATION_ID'

jwt_token = jwt.encode({'iat': int(time.time()), 'exp': int(time.time()) + 600, 'iss': APP_ID}, pk, algorithm='RS256')

req = urllib.request.Request(
    'https://api.github.com/app/installations/INSTALLATION_ID/access_tokens',
    method='POST',
    headers={'Authorization': 'Bearer ' + jwt_token, 'Accept': 'application/vnd.github+json'}
)
with urllib.request.urlopen(req) as r:
    install_token = json.loads(r.read())['token']

# Create public repo (required for GitHub Pages)
result = urllib.request.Request(
    'https://api.github.com/orgs/freshsitestudio/repos',
    method='POST',
    headers={'Authorization': 'token ' + install_token, 'Accept': 'application/vnd.github+json', 'Content-Type': 'application/json', 'User-Agent': 'orca-freshsite'},
    data=json.dumps({
        'name': 'COMPANY-www',
        'description': 'Company Name — type of business',
        'private': False,  # MUST be public for Pages
        'auto_init': False
    }).encode()
)
with urllib.request.urlopen(result) as r:
    print(json.loads(r.read())['html_url'])
```

**Conventions:**
- Repo name: `companyname-www` (all lowercase, hyphens only)
- Description: `{Company Name} — {type of business}`
- **Always `private: False`** (public required for GitHub Pages)
- Org: `freshsitestudio`

### 3b — Add collaborators

Invite `walterclawford` and `KiwiGP-Code` as **Admin** to every new repo:

```python
def api(url, method='PUT', data=None):
    headers = {'Authorization': 'token ' + install_token, 'Accept': 'application/vnd.github+json', 'Content-Type': 'application/json', 'User-Agent': 'orca-freshsite'}
    if data:
        data = json.dumps(data).encode()
    req = urllib.request.Request(url, method=method, headers=headers, data=data)
    with urllib.request.urlopen(req) as r:
        return r.status

for user in ['walterclawford', 'KiwiGP-Code']:
    api(f'https://api.github.com/repos/freshsitestudio/COMPANY-www/collaborators/{user}', data={'permission': 'admin'})
```

> Note: These are one-time invites that need to be accepted. Future repos should use a GitHub Team for easier management.

### 3c — Push initial Astro project

Two-part push: contents API for most files, git push for workflow files (`.github/` is restricted via contents API).

**Part 1 — Push Astro files via Contents API:**

```python
import base64, os

# Get current file SHAs (to include sha for updates)
req = urllib.request.Request(
    f'https://api.github.com/repos/freshsitestudio/COMPANY-www/git/trees/HEAD?recursive=1',
    headers={'Authorization': 'token ' + install_token, 'Accept': 'application/vnd.github+json'}
)
with urllib.request.urlopen(req) as r:
    tree = json.loads(r.read())
shas = {item['path']: item['sha'] for item in tree.get('tree', []) if item['type'] == 'blob'}

def put_file(path, content, message):
    payload = {'message': message, 'content': base64.b64encode(content).decode()}
    if path in shas:
        payload['sha'] = shas[path]
    url = f'https://api.github.com/repos/freshsitestudio/COMPANY-www/contents/{path}'
    data = json.dumps(payload).encode()
    req = urllib.request.Request(url, data=data, method='PUT')
    req.add_header('Authorization', 'token ' + install_token)
    req.add_header('Accept', 'application/vnd.github+json')
    req.add_header('Content-Type', 'application/json')
    try:
        with urllib.request.urlopen(req) as r:
            return json.loads(r.read())['commit']['sha'][:8]
    except urllib.error.HTTPError as e:
        return f'ERR {e.code}: {e.read().decode()[:100]}'

# Files to push
files = {
    'AGENTS.md': 'AGENTS.md',
    'astro.config.mjs': 'astro.config.mjs',
    'package.json': 'package.json',
    'package-lock.json': 'package-lock.json',
    'tsconfig.json': 'tsconfig.json',
    'README.md': 'README.md',
    '.gitignore': '.gitignore',
    'src/pages/index.astro': 'src/pages/index.astro',
    'src/styles/global.css': 'src/styles/global.css',
    'public/favicon.ico': 'public/favicon.ico',
    'public/favicon.svg': 'public/favicon.svg',
    '.vscode/extensions.json': '.vscode/extensions.json',
    '.vscode/launch.json': '.vscode/launch.json',
}

os.chdir('/path/to/companyname-astro')
for local_path, repo_path in files.items():
    with open(local_path, 'rb') as f:
        content = f.read()
    result = put_file(repo_path, content, f'feat: add {repo_path}')
    print(f'  {repo_path}: {result}')
```

**Part 2 — Push workflow file via git + GIT_ASKPASS:**

```python
import subprocess, json, time, jwt, urllib.request

# Write a git credential helper that generates a fresh install token
helper = '''#!/usr/bin/env python3
import sys, subprocess, json, time, jwt, urllib.request
APP_ID = 'APP_ID'; INSTALLATION_ID = 'INSTALLATION_ID'
pem = subprocess.run(['op', 'read', 'op://walters-shared-passwords/Orca-freshsite Github App/private key'],
    capture_output=True, text=True).stdout.rstrip()
now = int(time.time()); jwt_t = jwt.encode({'iat': now, 'exp': now + 600, 'iss': APP_ID}, pem, algorithm='RS256')
req = urllib.request.Request(f'https://api.github.com/app/installations/{INSTALLATION_ID}/access_tokens', data=b'{}', method='POST')
req.add_header('Authorization', f'Bearer {jwt_t}'); req.add_header('Accept', 'application/vnd.github+json')
with urllib.request.urlopen(req) as r: print(json.loads(r.read())['token'])
'''
with open('/tmp/gh_askpass.py', 'w') as f:
    f.write(helper)
subprocess.run(['chmod', '+x', '/tmp/gh_askpass.py'])

# Set git remote and push
subprocess.run(['git', 'remote', 'set-url', 'origin', 'https://github.com/freshsitestudio/COMPANY-www.git'], cwd='/path/to/companyname-astro')
subprocess.run(['git', 'checkout', '-b', 'master'], cwd='/path/to/companyname-astro')  # ensure master branch exists
result = subprocess.run(
    ['git', 'push', 'origin', 'master'],
    cwd='/path/to/companyname-astro',
    env={**os.environ, 'GIT_ASKPASS': '/tmp/gh_askpass.py'},
    capture_output=True, text=True
)
print(result.stdout + result.stderr)
```

---

## Step 4 — Enable GitHub Pages

After the initial push, enable Pages via API:

```python
# Enable Pages
pages = urllib.request.Request(
    'https://api.github.com/repos/freshsitestudio/COMPANY-www/pages',
    method='POST',
    headers={'Authorization': 'token ' + install_token, 'Accept': 'application/vnd.github+json', 'Content-Type': 'application/json', 'User-Agent': 'orca-freshsite'},
    data=json.dumps({'source': {'branch': 'main', 'path': '/'}}).encode()
)
with urllib.request.urlopen(pages) as r:
    pages_data = json.loads(r.read())
    print(f"Pages URL: {pages_data.get('html_url')}")

# Set custom domain
req = urllib.request.Request(
    'https://api.github.com/repos/freshsitestudio/COMPANY-www/pages',
    method='POST',
    headers={'Authorization': 'token ' + install_token, 'Accept': 'application/vnd.github+json', 'Content-Type': 'application/json', 'User-Agent': 'orca-freshsite'},
    data=json.dumps({'cname': 'companyname.freshsite.studio', 'source': {'branch': 'main', 'path': '/'}}).encode()
)
with urllib.request.urlopen(req) as r:
    print(json.loads(r.read()))
```

> Note: The GitHub Actions workflow must have run at least once before Pages can be enabled, because GitHub needs the built artifact to exist. Make sure the workflow succeeds first.

---

## Step 5 — Add DNS Record on Cloudflare

```python
import urllib.request, json

# Get Cloudflare token from 1Password
result = subprocess.run(
    ['op', 'item', 'get', 'Cloudflare - Local/admin automation token Freshsite.studio',
     '--vault', 'walters-shared-passwords', '--fields', 'credential', '--format', 'json'],
    capture_output=True, text=True
)
CF_TOKEN = json.loads(result.stdout)['credential']

headers = {'Authorization': 'Bearer ' + CF_TOKEN, 'Content-Type': 'application/json', 'User-Agent': 'orca-freshsite'}
ZONE_ID = 'df6f34da75569461f6a97469a4f3b339'  # freshsite.studio

def cf(url, method='GET', data=None):
    if data:
        data = json.dumps(data).encode()
    req = urllib.request.Request(url, method=method, headers=headers, data=data)
    with urllib.request.urlopen(req) as r:
        return json.loads(r.read())

# Create CNAME
result = cf('https://api.cloudflare.com/client/v4/zones/' + ZONE_ID + '/dns_records', method='POST', data={
    'type': 'CNAME',
    'name': 'companyname',
    'content': 'freshsitestudio.github.io',
    'proxied': False,
    'ttl': 3600
})
print('DNS created:', result['success'], result.get('result', {}).get('name', ''))
```

For sites on external registrars (e.g. FreeParking for `corda.co.nz`):
- GitHub Pages must use the full domain (e.g. `corda.co.nz`)
- Client adds DNS CNAME records at their registrar manually
- Give them the exact records to add (see Step 7)

---

## Step 6 — BMAD Agent Workflow

Once the repo is set up with a basic landing page and Pages is live, use the BMAD agents to build the full site.

**Sequence:**
1. **Analyst** → briefs the project, scopes requirements, identifies audience and goals
2. **Tech Writer** → writes all copy: headings, body text, calls to action, about content
3. **UX Designer** → designs the site: layout, visual direction, component structure
4. **Developer** → builds the site: implements the design in code

**BMAD party mode:** Always set `BMAD_PARTY_MODE=true` in the `.env` file before running.

**BMAD config** (`_bmad/core/config.yaml`):
```yaml
user: Walter
project: COMPANY
company: Company Name
industry: industry type
tone: tone guidance
description: one sentence description
```

**Tool/model:** Codex CLI (`tool: codex`, `model: codex`)

Each agent runs as a Codex CLI session. See the BMAD skills in `.agents/skills/` for detailed instructions.

---

## Step 7 — Client DNS Instructions Template

When a client's domain is hosted elsewhere, give them these exact records:

```
Domain: companyname.co.nz (or .com)

Add these DNS records at your registrar:

Type    Name    Value                           TTL
CNAME   www     freshsitestudio.github.io       3600
CNAME   @       freshsitestudio.github.io       3600   (if supported by your registrar)

Note: If your registrar does not allow a bare "@" CNAME record,
only add the "www" record and use www.companyname.co.nz as your URL.
```

---

## Key Conventions

| Rule | Value |
|---|---|
| Repo naming | `companyname-www` (lowercase, hyphens) |
| Repo visibility | **Always public** (required for GitHub Pages — no way around this without Enterprise) |
| GitHub org | `freshsitestudio` |
| Vanity URL pattern | `companyname.freshsite.studio` |
| Collaborators on every repo | `walterclawford`, `KiwiGP-Code` (both Admin) |
| Framework | **Astro.build** (minimal template + Tailwind CSS v4) |
| CI/CD | GitHub Actions (`pages/build` + `pages/deploy`) |
| BMAD party mode | `BMAD_PARTY_MODE=true` in `.env` |
| BMAD tool | `tool: codex`, `model: codex` |

---

## Credentials Reference

All stored in 1Password shared vault (`walters-shared-passwords`):

| Item | Fields |
|---|---|
| `orca-freshsite github app` | `app_id`, `installation_id`, `private_key` (SSH Key) |
| `Cloudflare - Local/admin automation token Freshsite.studio` | `credential` (API token) |
| `ops@freshsite CloudFlare UT` | CloudFlare account-level token (backup) |

---

## Sites Live

| Site | Repo | Vanity URL | Client Domain | Status |
|---|---|---|---|---|
| Mangrove | `freshsitestudio/mangrove-www` | `mangrove.freshsite.studio` | — | Basic landing page live |
| Corda NZ | `freshsitestudio/corda-www` | `corda.freshsite.studio` | `corda.co.nz` | Basic Astro site live; FreeParking DNS pending |

---

## Astro Build Notes

**Tailwind CSS v4:** Installed via `npx astro add tailwind`. Config is in `astro.config.mjs` using `@tailwindcss/vite` plugin. No separate `tailwind.config.js` — v4 uses CSS imports.

**Global CSS:** `src/styles/global.css` contains `@import "tailwindcss";` — Tailwind is applied globally.

**Layouts:** Astro minimal template has no default layout. Create `src/layouts/Layout.astro` and import `../styles/global.css` in the frontmatter to apply Tailwind globally.

**Build output:** `dist/` directory — this is what GitHub Pages serves. The Actions workflow uploads this artifact and deploys it.

**node_modules:** Not pushed to GitHub. GitHub Actions installs from `package.json` during the build step.

---

## Push Troubleshooting

### "Resource not accessible by integration" on Contents API
This happens with `.github/` path files via Contents API. Use the git + `GIT_ASKPASS` method instead (see Step 3c Part 2).

### Git push fails with "Authentication failed"
Make sure the remote URL is `https://github.com/freshsitestudio/COMPANY-www.git` (not `x-access-token:` prefix). The `GIT_ASKPASS` script handles token injection.

### Git push fails with "refspec does not match"
The Astro project was created with `master` branch (not `main`). Use `master` as the branch name when pushing, then GitHub Actions will trigger on `main` once the workflow file is pushed.

### GitHub Actions workflow not triggering
The workflow file (`.github/workflows/deploy.yml`) must exist in the repo before the first push. Push workflow files separately after pushing other Astro files. Also check: Settings → Actions → General → Workflow permissions allows "Read and write permissions".
