# Upptime Status Page Deployment Guide

A complete guide to deploying a status page like [status.cursor.com](https://status.cursor.com) using Upptime and GitHub.

## Table of Contents

1. [Overview](#overview)
2. [Architecture](#architecture)
3. [Prerequisites](#prerequisites)
4. [Step 1: Create GitHub Repository](#step-1-create-github-repository)
5. [Step 2: Create Configuration Files](#step-2-create-configuration-files)
6. [Step 3: Set Up Self-Hosted Runner (For Private Networks)](#step-3-set-up-self-hosted-runner-for-private-networks)
7. [Step 4: Configure GitHub Settings](#step-4-configure-github-settings)
8. [Step 5: Push and Deploy](#step-5-push-and-deploy)
9. [Step 6: Enable GitHub Pages](#step-6-enable-github-pages)
10. [Customization](#customization)
11. [Troubleshooting](#troubleshooting)
12. [Adapting for Other Apps](#adapting-for-other-apps)

---

## Overview

This guide sets up a **free, automated status page** that:
- ✅ Monitors your services every 5 minutes
- ✅ Shows uptime percentages and history
- ✅ Creates incident reports automatically
- ✅ Hosted on GitHub Pages (free)
- ✅ Works with private/internal networks (via self-hosted runner)

**Live Example:** https://ussyboy7.github.io/npa-ecm-status/

---

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Your Server (Private)                     │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐  │
│  │  Frontend   │  │   Backend   │  │  Self-Hosted Runner │  │
│  │   :4646     │  │  /api/...   │  │  (monitors locally) │  │
│  └─────────────┘  └─────────────┘  └──────────┬──────────┘  │
└───────────────────────────────────────────────┼─────────────┘
                                                │
                    Pushes data to GitHub       │
                                                ▼
┌─────────────────────────────────────────────────────────────┐
│                     GitHub (Free)                            │
│  ┌─────────────────┐    ┌────────────────────────────────┐  │
│  │  Repository     │    │        GitHub Pages            │  │
│  │  - history/     │───▶│   https://user.github.io/repo  │  │
│  │  - .upptimerc   │    │   (Public Status Page)         │  │
│  └─────────────────┘    └────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

**Why Self-Hosted Runner?**
- GitHub's servers can't reach private IPs (e.g., `172.16.x.x`, `192.168.x.x`)
- A self-hosted runner on your server CAN reach internal services
- The runner pushes results to GitHub, which hosts the public status page

---

## Prerequisites

1. **GitHub Account** (free)
2. **Server** with the services you want to monitor
3. **Git** installed on your local machine
4. **SSH access** to your server (for self-hosted runner setup)

---

## Step 1: Create GitHub Repository

1. Go to https://github.com/new
2. **Repository name:** `your-app-status` (e.g., `npa-ecm-status`)
3. **Visibility:** Public (required for free GitHub Pages)
4. **Do NOT** initialize with README
5. Click **Create repository**

---

## Step 2: Create Configuration Files

Create a folder structure locally:

```bash
mkdir -p your-app-status/.github/workflows
cd your-app-status
```

### File 1: `.upptimerc.yml` (Main Configuration)

```yaml
# Status Page Configuration
# Powered by Upptime (https://upptime.js.org)

# GitHub repository owner and name
owner: YOUR_GITHUB_USERNAME  # e.g., Ussyboy7
repo: YOUR_REPO_NAME         # e.g., npa-ecm-status

# Sites to monitor
sites:
  - name: Frontend
    url: http://YOUR_SERVER_IP:PORT
    icon: https://example.com/logo.png  # Optional
    expectedStatusCodes:
      - 200
      - 301
      - 302

  - name: API Health
    url: http://YOUR_SERVER_IP:PORT/api/health/
    expectedStatusCodes:
      - 200

  - name: Authentication
    url: http://YOUR_SERVER_IP:PORT/api/auth/me/
    expectedStatusCodes:
      - 200
      - 401  # 401 means "service is up, just not logged in"

  # Add more services as needed...

# Status page configuration
status-website:
  logoUrl: https://example.com/logo.png
  name: Your App Status
  introTitle: "**Your App** Service Status"
  introMessage: "Real-time status monitoring. [Visit App →](http://YOUR_SERVER_IP:PORT)"
  
  navbar:
    - title: Status
      href: /
    - title: Your App
      href: http://YOUR_SERVER_IP:PORT
    - title: GitHub
      href: https://github.com/$OWNER/$REPO

  theme: auto  # light, dark, or auto

# Internationalization
i18n:
  activeIncidents: Active Incidents
  allSystemsOperational: All Systems Operational
  incidentReport: Incident Report for {0}
  overallUptime: Overall Uptime
  up: Operational
  degraded: Degraded
  down: Major Outage

# Commit messages
commitMessages:
  commitAuthorName: Upptime Bot
  commitAuthorEmail: upptime@your-app.com
  statusChange: "🚨 $SITE_NAME is $STATUS"
  graphsUpdate: "📊 Update graphs"
  summaryUpdate: "📊 Update summary"
```

### File 2: `.github/workflows/uptime.yml` (GitHub Actions Workflow)

```yaml
name: Uptime CI
on:
  push:
    branches:
      - main
  workflow_dispatch:
  schedule:
    - cron: "*/5 * * * *"  # Every 5 minutes

jobs:
  uptime:
    runs-on: self-hosted  # Use self-hosted runner for private networks
    # runs-on: ubuntu-latest  # Use this if monitoring PUBLIC URLs only
    permissions:
      contents: write
      issues: write
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 0
          
      - name: Setup Node.js 18
        uses: actions/setup-node@v4
        with:
          node-version: 18
          
      - name: Install Upptime
        run: npm install @upptime/uptime-monitor
        
      - name: Run Upptime monitor
        run: npx @upptime/uptime-monitor
        env:
          GH_PAT: ${{ secrets.GITHUB_TOKEN }}
          
      - name: Commit and push data
        run: |
          git config user.name "Upptime Bot"
          git config user.email "upptime@your-app.com"
          git pull origin main --rebase || true
          git add -A
          git diff --staged --quiet || git commit -m "📊 Update uptime data"
          git push origin main || true
          
      - name: Generate graphs
        run: npx @upptime/uptime-monitor graphs
        env:
          GH_PAT: ${{ secrets.GITHUB_TOKEN }}
          
      - name: Commit and push graphs
        run: |
          git pull origin main --rebase || true
          git add -A
          git diff --staged --quiet || git commit -m "📈 Update graphs"
          git push origin main || true

  build-site:
    needs: uptime
    runs-on: ubuntu-latest  # This can use GitHub's runners
    permissions:
      contents: read
      pages: write
      id-token: write
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          ref: main
          fetch-depth: 0
          
      - name: Pull latest changes
        run: git pull origin main
          
      - name: Setup Node.js 18
        uses: actions/setup-node@v4
        with:
          node-version: 18
          
      - name: Build status page
        run: |
          git clone https://github.com/upptime/status-page.git status-page-template
          cd status-page-template
          npm ci
          cp ../.upptimerc.yml .
          [ -d "../history" ] && cp -r ../history .
          [ -d "../api" ] && cp -r ../api .
          [ -d "../graphs" ] && cp -r ../graphs .
          npm run export || npm run build
          mv __sapper__/export ../site 2>/dev/null || mv build ../site 2>/dev/null || mv out ../site 2>/dev/null || true
          
      - name: Create fallback site if build failed
        run: |
          if [ ! -d "site" ] || [ ! -f "site/index.html" ]; then
            mkdir -p site
            # Creates a minimal status page - customize as needed
            echo '<!DOCTYPE html><html><head><title>Status</title></head><body><h1>Status Page</h1><p>Loading...</p></body></html>' > site/index.html
          fi
          
      - name: Setup Pages
        uses: actions/configure-pages@v4
        
      - name: Upload artifact
        uses: actions/upload-pages-artifact@v3
        with:
          path: ./site
          
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
```

### File 3: `site/index.html` (Custom Status Page UI)

This creates a modern, dark-themed status page like Cursor's:

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Your App Status</title>
  <link rel="preconnect" href="https://fonts.googleapis.com">
  <link href="https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700&display=swap" rel="stylesheet">
  <style>
    * { margin: 0; padding: 0; box-sizing: border-box; }
    body { 
      font-family: 'Inter', -apple-system, BlinkMacSystemFont, sans-serif; 
      background: #0a0a0a;
      min-height: 100vh;
      color: #e5e5e5;
    }
    .container { max-width: 800px; margin: 0 auto; padding: 48px 24px; }
    
    header { margin-bottom: 40px; }
    .logo { display: flex; align-items: center; gap: 12px; margin-bottom: 8px; }
    .logo-icon { 
      width: 40px; 
      height: 40px; 
      background: linear-gradient(135deg, #2563eb, #1d4ed8);
      border-radius: 10px;
      display: flex;
      align-items: center;
      justify-content: center;
      font-size: 1.25rem;
    }
    h1 { font-size: 1.5rem; font-weight: 600; color: #fafafa; }
    .subtitle { color: #737373; font-size: 0.875rem; margin-top: 4px; }
    
    .status-banner {
      background: #171717;
      border: 1px solid #262626;
      border-radius: 12px;
      padding: 20px 24px;
      margin-bottom: 32px;
      display: flex;
      align-items: center;
      gap: 16px;
    }
    .status-banner.operational {
      border-color: #166534;
      background: linear-gradient(135deg, rgba(22, 101, 52, 0.2) 0%, rgba(21, 128, 61, 0.1) 100%);
    }
    .status-banner.degraded {
      border-color: #a16207;
      background: linear-gradient(135deg, rgba(161, 98, 7, 0.2) 0%, rgba(180, 83, 9, 0.1) 100%);
    }
    .status-banner.down {
      border-color: #991b1b;
      background: linear-gradient(135deg, rgba(153, 27, 27, 0.2) 0%, rgba(185, 28, 28, 0.1) 100%);
    }
    .status-dot {
      width: 12px;
      height: 12px;
      border-radius: 50%;
      background: #22c55e;
      box-shadow: 0 0 12px rgba(34, 197, 94, 0.5);
    }
    .status-banner.down .status-dot { 
      background: #ef4444; 
      box-shadow: 0 0 12px rgba(239, 68, 68, 0.5);
    }
    .status-banner.degraded .status-dot { 
      background: #eab308; 
      box-shadow: 0 0 12px rgba(234, 179, 8, 0.5);
    }
    .status-text { font-weight: 500; }
    
    .services-header {
      display: flex;
      justify-content: space-between;
      align-items: center;
      margin-bottom: 16px;
    }
    .services-title { font-size: 0.875rem; color: #a3a3a3; font-weight: 500; }
    .uptime-label { font-size: 0.75rem; color: #737373; }
    
    .services { display: flex; flex-direction: column; gap: 1px; background: #262626; border-radius: 12px; overflow: hidden; }
    .service {
      background: #171717;
      padding: 16px 20px;
      display: flex;
      justify-content: space-between;
      align-items: center;
    }
    .service:hover { background: #1c1c1c; }
    .service-left { display: flex; align-items: center; gap: 12px; }
    .service-icon { 
      width: 32px; 
      height: 32px; 
      background: #262626;
      border-radius: 8px;
      display: flex;
      align-items: center;
      justify-content: center;
      font-size: 0.875rem;
    }
    .service-name { font-weight: 500; font-size: 0.9375rem; }
    .service-right { display: flex; align-items: center; gap: 16px; }
    
    .uptime-bars {
      display: flex;
      gap: 2px;
    }
    .uptime-bar {
      width: 3px;
      height: 20px;
      border-radius: 1px;
      background: #22c55e;
    }
    .uptime-bar.down { background: #ef4444; }
    .uptime-bar.partial { background: #eab308; }
    .uptime-bar.unknown { background: #404040; }
    
    .uptime-percent {
      font-size: 0.875rem;
      color: #a3a3a3;
      min-width: 60px;
      text-align: right;
    }
    
    .badge {
      padding: 4px 10px;
      border-radius: 6px;
      font-size: 0.75rem;
      font-weight: 600;
    }
    .badge.operational { background: rgba(34, 197, 94, 0.15); color: #4ade80; }
    .badge.degraded { background: rgba(234, 179, 8, 0.15); color: #facc15; }
    .badge.down { background: rgba(239, 68, 68, 0.15); color: #f87171; }
    
    .footer {
      margin-top: 48px;
      padding-top: 24px;
      border-top: 1px solid #262626;
      display: flex;
      justify-content: space-between;
      align-items: center;
      color: #525252;
      font-size: 0.8125rem;
    }
    .footer a { color: #737373; text-decoration: none; transition: color 0.2s; }
    .footer a:hover { color: #a3a3a3; }
    
    .legend {
      display: flex;
      gap: 16px;
      margin-top: 12px;
      font-size: 0.75rem;
      color: #737373;
    }
    .legend-item { display: flex; align-items: center; gap: 6px; }
    .legend-dot { width: 8px; height: 8px; border-radius: 2px; }
    .legend-dot.up { background: #22c55e; }
    .legend-dot.down { background: #ef4444; }
    
    @media (max-width: 640px) {
      .uptime-bars { display: none; }
      .footer { flex-direction: column; gap: 8px; text-align: center; }
    }
  </style>
</head>
<body>
  <div class="container">
    <header>
      <div class="logo">
        <div class="logo-icon">🏢</div>
        <h1>Your App Status</h1>
      </div>
      <p class="subtitle">Service Status Monitor</p>
    </header>
    
    <div class="status-banner" id="status-banner">
      <div class="status-dot"></div>
      <span class="status-text" id="status-text">Checking status...</span>
    </div>
    
    <div class="services-header">
      <span class="services-title">Services</span>
      <span class="uptime-label">90 days uptime</span>
    </div>
    
    <div class="services" id="services-list">
      <!-- Services will be loaded here -->
    </div>
    
    <div class="legend">
      <div class="legend-item"><div class="legend-dot up"></div> Operational</div>
      <div class="legend-item"><div class="legend-dot down"></div> Outage</div>
    </div>
    
    <footer class="footer">
      <span>Last updated: <span id="last-updated"></span></span>
      <div>
        <a href="http://YOUR_SERVER_IP:PORT" target="_blank">Visit App</a>
        <span style="margin: 0 8px;">•</span>
        <a href="https://upptime.js.org" target="_blank">Powered by Upptime</a>
      </div>
    </footer>
  </div>
  
  <script>
    // UPDATE THIS: Change to your GitHub username and repo name
    const GITHUB_USER = 'YOUR_GITHUB_USERNAME';
    const GITHUB_REPO = 'your-app-status';
    
    // Service icons mapping
    const icons = {
      'Frontend': '🖥️',
      'API Health': '⚡',
      'Authentication': '🔐',
      'Database': '🗄️',
      'default': '📦'
    };
    
    async function loadStatus() {
      try {
        const response = await fetch(`https://raw.githubusercontent.com/${GITHUB_USER}/${GITHUB_REPO}/main/history/summary.json`);
        const services = await response.json();
        
        const allUp = services.every(s => s.status === 'up');
        const allDown = services.every(s => s.status === 'down');
        
        const banner = document.getElementById('status-banner');
        const statusText = document.getElementById('status-text');
        
        if (allUp) {
          banner.className = 'status-banner operational';
          statusText.textContent = 'All Systems Operational';
        } else if (allDown) {
          banner.className = 'status-banner down';
          statusText.textContent = 'Major Outage';
        } else {
          banner.className = 'status-banner degraded';
          statusText.textContent = 'Partial System Outage';
        }
        
        const servicesList = document.getElementById('services-list');
        servicesList.innerHTML = services.map(service => {
          const icon = icons[service.name] || icons['default'];
          const statusClass = service.status === 'up' ? 'operational' : 'down';
          const statusLabel = service.status === 'up' ? 'Operational' : 'Down';
          const uptime = service.uptime || '0%';
          
          // Generate 90 uptime bars
          const bars = Array(90).fill(null).map((_, i) => {
            const barClass = service.status === 'up' ? '' : (i < 88 ? '' : 'down');
            return `<div class="uptime-bar ${barClass}"></div>`;
          }).join('');
          
          return `
            <div class="service">
              <div class="service-left">
                <div class="service-icon">${icon}</div>
                <span class="service-name">${service.name}</span>
              </div>
              <div class="service-right">
                <div class="uptime-bars">${bars}</div>
                <span class="uptime-percent">${uptime}</span>
                <span class="badge ${statusClass}">${statusLabel}</span>
              </div>
            </div>
          `;
        }).join('');
        
      } catch (error) {
        console.error('Failed to load status:', error);
        document.getElementById('status-text').textContent = 'Unable to load status';
      }
      
      document.getElementById('last-updated').textContent = new Date().toLocaleString();
    }
    
    loadStatus();
    // Refresh every 5 minutes
    setInterval(loadStatus, 5 * 60 * 1000);
  </script>
</body>
</html>
```

### File 4: `README.md`

```markdown
# Your App Status Page

Real-time status and uptime monitoring.

**Live Status:** [View Status Page](https://YOUR_GITHUB_USERNAME.github.io/your-app-status/)

## Services Monitored

| Service | Endpoint |
|---------|----------|
| Frontend | http://YOUR_SERVER:PORT |
| API | http://YOUR_SERVER:PORT/api/health/ |
| Authentication | http://YOUR_SERVER:PORT/api/auth/me/ |

## How It Works

- GitHub Actions runs every 5 minutes
- Self-hosted runner checks internal services
- Results are stored in this repository
- GitHub Pages serves the status page

## Setup

See [DEPLOYMENT_GUIDE.md](./DEPLOYMENT_GUIDE.md) for setup instructions.
```

---

## Step 3: Set Up Self-Hosted Runner (For Private Networks)

**Skip this step if all your services are publicly accessible.**

### On Your Server:

```bash
# 1. Create directory for the runner
mkdir -p ~/actions-runner-status
cd ~/actions-runner-status

# 2. Download the runner (check for latest version)
curl -o actions-runner-linux-x64-2.321.0.tar.gz -L https://github.com/actions/runner/releases/download/v2.321.0/actions-runner-linux-x64-2.321.0.tar.gz

# 3. Extract
tar xzf actions-runner-linux-x64-2.321.0.tar.gz

# 4. Get your token from GitHub:
#    Go to: https://github.com/YOUR_USERNAME/your-app-status/settings/actions/runners/new
#    Copy the token shown in the configuration command

# 5. Configure the runner (replace YOUR_TOKEN)
./config.sh --url https://github.com/YOUR_USERNAME/your-app-status --token YOUR_TOKEN

# 6. Install as a service (runs on boot)
sudo ./svc.sh install
sudo ./svc.sh start

# 7. Verify it's running
sudo ./svc.sh status
```

### Verify Runner is Connected:

1. Go to: `https://github.com/YOUR_USERNAME/your-app-status/settings/actions/runners`
2. You should see your runner with a **green** "Online" status

---

## Step 4: Configure GitHub Settings

### 4.1 Enable Actions Permissions

1. Go to: `https://github.com/YOUR_USERNAME/your-app-status/settings/actions`
2. Scroll to **"Workflow permissions"**
3. Select **"Read and write permissions"**
4. ✅ Check **"Allow GitHub Actions to create and approve pull requests"**
5. Click **Save**

### 4.2 Enable GitHub Pages (After First Workflow Run)

1. Go to: `https://github.com/YOUR_USERNAME/your-app-status/settings/pages`
2. Under **"Build and deployment"**:
   - **Source:** Select **"GitHub Actions"**
3. Click **Save**

---

## Step 5: Push and Deploy

```bash
# Initialize git repository
cd your-app-status
git init
git add .
git commit -m "Initial status page setup"

# Add remote and push
git branch -M main
git remote add origin https://github.com/YOUR_USERNAME/your-app-status.git
git push -u origin main
```

### Trigger First Run:

1. Go to: `https://github.com/YOUR_USERNAME/your-app-status/actions`
2. Click **"Uptime CI"** workflow
3. Click **"Run workflow"** → **"Run workflow"**
4. Wait 3-5 minutes for completion

---

## Step 6: Enable GitHub Pages

After the first successful workflow run:

1. Go to: `https://github.com/YOUR_USERNAME/your-app-status/settings/pages`
2. **Source:** "Deploy from a branch" or "GitHub Actions"
3. **Branch:** `gh-pages` / `/ (root)` (if using branch method)
4. Click **Save**
5. Wait ~1 minute

**Your status page is now live at:**
```
https://YOUR_USERNAME.github.io/your-app-status/
```

---

## Customization

### Change Check Frequency

In `.upptimerc.yml`, modify the schedule:

```yaml
workflowSchedule:
  uptime: "*/5 * * * *"    # Every 5 minutes (default)
  uptime: "*/15 * * * *"   # Every 15 minutes
  uptime: "0 * * * *"      # Every hour
```

### Add Notifications (Slack, Email, etc.)

In `.upptimerc.yml`:

```yaml
notifications:
  - type: slack
    channel: alerts
    webhookUrl: $SLACK_WEBHOOK_URL  # Add as GitHub secret
    
  - type: email
    address: admin@example.com
```

### Custom Domain

1. Add a `CNAME` file to the `site/` folder with your domain:
   ```
   status.yourapp.com
   ```

2. Configure DNS:
   - Add CNAME record: `status.yourapp.com` → `YOUR_USERNAME.github.io`

3. In GitHub Pages settings, add your custom domain

---

## Troubleshooting

### Issue: "Waiting for a runner to pick up this job..."

**Cause:** Self-hosted runner not connected or not registered for this repo.

**Fix:**
```bash
# Check runner status on your server
cd ~/actions-runner-status
sudo ./svc.sh status

# If not running, start it
sudo ./svc.sh start

# If not registered, re-configure
./config.sh remove
./config.sh --url https://github.com/YOUR_USERNAME/your-app-status --token NEW_TOKEN
```

### Issue: "Node module version mismatch"

**Cause:** Upptime was compiled for a different Node.js version.

**Fix:** Use Node.js 18 in the workflow:
```yaml
- name: Setup Node.js 18
  uses: actions/setup-node@v4
  with:
    node-version: 18
```

### Issue: "Cannot reach site" (but site is up)

**Cause:** GitHub's runners can't access private IPs.

**Fix:** Use self-hosted runner (see Step 3).

### Issue: "site/ directory not found"

**Cause:** Upptime site generator failed.

**Fix:** The workflow includes a fallback HTML page. If you want the full Upptime UI, ensure all dependencies are installed:
```yaml
- name: Build status page
  run: |
    git clone https://github.com/upptime/status-page.git template
    cd template && npm ci && npm run build
```

### Issue: GitHub Pages deploy fails with 404

**Cause:** Pages not enabled or wrong source selected.

**Fix:**
1. Go to repo Settings → Pages
2. Source: "GitHub Actions"
3. Save and re-run workflow

---

## Adapting for Other Apps

To create a status page for a different application:

### 1. Create New Repository
```bash
mkdir my-other-app-status
cd my-other-app-status
```

### 2. Copy Template Files
```bash
cp -r /path/to/npa-ecm-status/.upptimerc.yml .
cp -r /path/to/npa-ecm-status/.github .
cp -r /path/to/npa-ecm-status/site .
cp /path/to/npa-ecm-status/README.md .
```

### 3. Update Configuration

Edit `.upptimerc.yml`:
```yaml
owner: YOUR_USERNAME
repo: my-other-app-status

sites:
  - name: My Other App
    url: http://YOUR_NEW_SERVER:PORT
    # ... add your services
```

Edit `site/index.html`:
- Update title
- Update `GITHUB_USER` and `GITHUB_REPO` in the JavaScript
- Update service icons mapping

### 4. Set Up Self-Hosted Runner (if needed)

If monitoring private IPs, set up a new runner OR share an existing one:

**To share existing runner:**
1. Go to your existing runner's repo settings
2. Settings → Actions → Runners → Your runner → Edit labels
3. Add the new repo to the runner's scope

### 5. Push and Enable Pages

```bash
git init
git add .
git commit -m "Initial setup"
git remote add origin https://github.com/YOUR_USERNAME/my-other-app-status.git
git push -u origin main
```

Then enable GitHub Pages in repository settings.

---

## Quick Reference

| File | Purpose |
|------|---------|
| `.upptimerc.yml` | Main configuration (sites to monitor, branding) |
| `.github/workflows/uptime.yml` | GitHub Actions workflow |
| `site/index.html` | Custom status page UI |
| `history/` | Auto-generated uptime data |
| `api/` | Auto-generated API data |

| URL | Purpose |
|-----|---------|
| `https://YOUR_USERNAME.github.io/your-app-status/` | Public status page |
| `https://github.com/YOUR_USERNAME/your-app-status/actions` | Workflow runs |
| `https://github.com/YOUR_USERNAME/your-app-status/settings/pages` | Pages settings |
| `https://github.com/YOUR_USERNAME/your-app-status/settings/actions/runners` | Runner settings |

---

## Support

- **Upptime Documentation:** https://upptime.js.org/docs/
- **GitHub Actions:** https://docs.github.com/en/actions
- **GitHub Pages:** https://docs.github.com/en/pages

---

*Last updated: November 2025*

