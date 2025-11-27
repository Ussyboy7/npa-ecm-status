# Upptime Status Page Deployment Guide

A complete guide to deploying a status page like [status.cursor.com](https://status.cursor.com) using Upptime and GitHub.

**Live Example:** https://ussyboy7.github.io/npa-ecm-status/

---

## Table of Contents

1. [Overview](#overview)
2. [Architecture](#architecture)
3. [Prerequisites](#prerequisites)
4. [Step 1: Create GitHub Repository](#step-1-create-github-repository)
5. [Step 2: Create Configuration Files](#step-2-create-configuration-files)
6. [Step 3: Set Up Self-Hosted Runner](#step-3-set-up-self-hosted-runner-for-private-networks)
7. [Step 4: Configure GitHub Settings](#step-4-configure-github-settings)
8. [Step 5: Push and Deploy](#step-5-push-and-deploy)
9. [Customization](#customization)
10. [Incidents](#incidents) ← **NEW**
11. [Notifications](#notifications) ← **NEW**
12. [Troubleshooting](#troubleshooting)
13. [Adapting for Other Apps](#adapting-for-other-apps)

---

## Overview

This guide sets up a **free, automated status page** that:
- ✅ Monitors your services every 5 minutes
- ✅ Shows 90-day uptime bars (like Cursor's status page)
- ✅ Displays uptime percentages
- ✅ Creates incident reports automatically
- ✅ Hosted on GitHub Pages (free)
- ✅ Works with private/internal networks (via self-hosted runner)

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
│  │  - site/        │    │   (Public Status Page)         │  │
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
3. **Git** installed locally
4. **SSH access** to your server (for self-hosted runner)

---

## Step 1: Create GitHub Repository

1. Go to https://github.com/new
2. **Repository name:** `your-app-status` (e.g., `npa-ecm-status`)
3. **Visibility:** Public (required for free GitHub Pages)
4. **Do NOT** initialize with README
5. Click **Create repository**

---

## Step 2: Create Configuration Files

Create the folder structure:

```bash
mkdir -p your-app-status/.github/workflows
mkdir -p your-app-status/site
cd your-app-status
```

### File 1: `.upptimerc.yml`

```yaml
# Status Page Configuration
owner: YOUR_GITHUB_USERNAME
repo: your-app-status

# Sites to monitor
sites:
  - name: Frontend
    url: http://YOUR_SERVER_IP:PORT
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
      - 401  # 401 = service is up, just not logged in

# Add more services as needed...

status-website:
  name: Your App Status
  introTitle: "**Your App** Service Status"
  theme: auto

i18n:
  activeIncidents: Active Incidents
  allSystemsOperational: All Systems Operational
  up: Operational
  down: Major Outage

commitMessages:
  commitAuthorName: Upptime Bot
  commitAuthorEmail: upptime@your-app.com
```

### File 2: `.github/workflows/uptime.yml`

```yaml
name: Uptime CI
on:
  push:
    branches:
      - main
  workflow_dispatch:
  schedule:
    - cron: "*/5 * * * *"

jobs:
  uptime:
    runs-on: self-hosted  # For private networks
    # runs-on: ubuntu-latest  # For public URLs only
    permissions:
      contents: write
      issues: write
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 0
          
      - name: Setup Node.js 16
        uses: actions/setup-node@v4
        with:
          node-version: 16
          
      - name: Install Upptime
        run: npm install @upptime/uptime-monitor@v1.34.0
        
      - name: Run Upptime monitor
        run: node node_modules/@upptime/uptime-monitor/dist/index.js || true
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
        run: node node_modules/@upptime/uptime-monitor/dist/index.js graphs || true
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
    runs-on: ubuntu-latest
    permissions:
      contents: read
      pages: write
      id-token: write
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          ref: main
          fetch-depth: 0
          
      - name: Pull latest changes
        run: git pull origin main
          
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

### File 3: `site/index.html`

This is the custom status page UI (Cursor-style with 90-day uptime bars):

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Your App Status</title>
  <link href="https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700&display=swap" rel="stylesheet">
  <style>
    * { margin: 0; padding: 0; box-sizing: border-box; }
    body { 
      font-family: 'Inter', -apple-system, sans-serif; 
      background: #0f0f10;
      min-height: 100vh;
      color: #e5e5e5;
    }
    .container { max-width: 900px; margin: 0 auto; padding: 48px 24px; }
    
    header { margin-bottom: 32px; display: flex; align-items: center; justify-content: space-between; }
    .logo { display: flex; align-items: center; gap: 12px; }
    .logo-icon { 
      width: 42px; height: 42px; 
      background: linear-gradient(135deg, #3b82f6, #1d4ed8);
      border-radius: 10px;
      display: flex; align-items: center; justify-content: center;
      font-size: 1.25rem;
    }
    .logo-text { font-size: 1.25rem; font-weight: 600; color: #fafafa; }
    .header-links a { color: #a3a3a3; text-decoration: none; font-size: 0.875rem; margin-left: 16px; }
    .header-links a:hover { color: #fff; }
    
    .status-banner {
      background: #18181b;
      border: 1px solid #27272a;
      border-radius: 12px;
      padding: 24px;
      margin-bottom: 32px;
      display: flex;
      align-items: center;
      gap: 16px;
    }
    .status-banner.operational { border-left: 4px solid #22c55e; }
    .status-banner.down { border-left: 4px solid #ef4444; }
    .status-dot {
      width: 10px; height: 10px;
      border-radius: 50%;
      background: #22c55e;
      animation: pulse 2s infinite;
    }
    @keyframes pulse { 0%, 100% { opacity: 1; } 50% { opacity: 0.5; } }
    .status-banner.down .status-dot { background: #ef4444; }
    .status-content h2 { font-size: 1.125rem; font-weight: 600; color: #fafafa; }
    .status-content p { font-size: 0.875rem; color: #a3a3a3; }
    
    .section-header {
      display: flex;
      justify-content: space-between;
      margin-bottom: 12px;
      padding: 0 4px;
    }
    .section-title { font-size: 0.75rem; color: #71717a; text-transform: uppercase; }
    
    .services { 
      background: #18181b;
      border: 1px solid #27272a;
      border-radius: 12px;
      overflow: hidden;
    }
    .service {
      padding: 16px 20px;
      display: flex;
      justify-content: space-between;
      align-items: center;
      border-bottom: 1px solid #27272a;
    }
    .service:last-child { border-bottom: none; }
    .service:hover { background: #1f1f23; }
    
    .service-left { display: flex; align-items: center; gap: 12px; }
    .service-icon { font-size: 1rem; }
    .service-name { font-weight: 500; color: #fafafa; }
    .service-status { font-size: 0.75rem; color: #22c55e; margin-left: 8px; }
    .service-status.down { color: #ef4444; }
    
    .uptime-bars { display: flex; gap: 1px; }
    .uptime-bar {
      width: 3px; height: 24px;
      border-radius: 1px;
      background: #22c55e;
    }
    .uptime-bar:hover { transform: scaleY(1.2); }
    .uptime-bar.down { background: #ef4444; }
    .uptime-bar.partial { background: #eab308; }
    .uptime-bar.unknown { background: #3f3f46; }
    
    .uptime-percent { font-size: 0.8125rem; color: #a3a3a3; min-width: 80px; text-align: right; }
    
    .legend {
      display: flex;
      justify-content: center;
      gap: 24px;
      padding: 16px;
      font-size: 0.75rem;
      color: #71717a;
      border-top: 1px solid #27272a;
    }
    .legend-item { display: flex; align-items: center; gap: 6px; }
    .legend-dot { width: 10px; height: 10px; border-radius: 2px; }
    .legend-dot.operational { background: #22c55e; }
    .legend-dot.down { background: #ef4444; }
    
    .footer {
      margin-top: 48px;
      padding-top: 24px;
      border-top: 1px solid #27272a;
      display: flex;
      justify-content: space-between;
      color: #52525b;
      font-size: 0.8125rem;
    }
    .footer a { color: #71717a; text-decoration: none; }
    
    @media (max-width: 640px) {
      .uptime-bars { display: none; }
      .header-links { display: none; }
    }
  </style>
</head>
<body>
  <div class="container">
    <header>
      <div class="logo">
        <div class="logo-icon">🏢</div>
        <span class="logo-text">Your App Status</span>
      </div>
      <div class="header-links">
        <a href="http://YOUR_SERVER:PORT" target="_blank">Visit App</a>
        <a href="https://github.com/YOUR_USER/your-app-status">GitHub</a>
      </div>
    </header>
    
    <div class="status-banner operational" id="status-banner">
      <div class="status-dot"></div>
      <div class="status-content">
        <h2 id="status-title">All Systems Operational</h2>
        <p>Uptime over the past 90 days</p>
      </div>
    </div>
    
    <div class="section-header">
      <span class="section-title">Services</span>
      <span class="section-title">90 days ago → Today</span>
    </div>
    
    <div class="services" id="services-list">
      <div class="service"><span>Loading...</span></div>
    </div>
    
    <div class="legend">
      <div class="legend-item"><div class="legend-dot operational"></div> Operational</div>
      <div class="legend-item"><div class="legend-dot down"></div> Outage</div>
    </div>
    
    <footer class="footer">
      <span>Last updated: <span id="time"></span></span>
      <a href="https://upptime.js.org">Powered by Upptime</a>
    </footer>
  </div>
  
  <script>
    // UPDATE THESE VALUES:
    const GITHUB_USER = 'YOUR_GITHUB_USERNAME';
    const GITHUB_REPO = 'your-app-status';
    
    const icons = {
      'Frontend': '🖥️',
      'API Health': '⚡',
      'Authentication': '🔐',
      'default': '📦'
    };
    
    function generateUptimeBars(dailyMinutesDown = {}) {
      const bars = [];
      const today = new Date();
      for (let i = 89; i >= 0; i--) {
        const date = new Date(today);
        date.setDate(date.getDate() - i);
        const dateStr = date.toISOString().split('T')[0];
        const minutesDown = dailyMinutesDown[dateStr] || 0;
        let barClass = minutesDown === 0 ? '' : (minutesDown < 60 ? 'partial' : 'down');
        bars.push(`<div class="uptime-bar ${barClass}"></div>`);
      }
      return bars.join('');
    }
    
    async function loadStatus() {
      try {
        const r = await fetch(`https://raw.githubusercontent.com/${GITHUB_USER}/${GITHUB_REPO}/main/history/summary.json?t=${Date.now()}`);
        const services = await r.json();
        
        const allUp = services.every(s => s.status === 'up');
        const banner = document.getElementById('status-banner');
        const title = document.getElementById('status-title');
        
        banner.className = `status-banner ${allUp ? 'operational' : 'down'}`;
        title.textContent = allUp ? 'All Systems Operational' : 'System Issues Detected';
        
        document.getElementById('services-list').innerHTML = services.map(s => `
          <div class="service">
            <div class="service-left">
              <span class="service-icon">${icons[s.name] || icons['default']}</span>
              <span class="service-name">${s.name}</span>
              <span class="service-status ${s.status === 'down' ? 'down' : ''}">${s.status === 'up' ? 'Operational' : 'Down'}</span>
            </div>
            <div style="display:flex;align-items:center;gap:16px">
              <div class="uptime-bars">${generateUptimeBars(s.dailyMinutesDown)}</div>
              <span class="uptime-percent">${s.uptime} uptime</span>
            </div>
          </div>
        `).join('');
      } catch (e) {
        document.getElementById('status-title').textContent = 'Unable to load status';
      }
      document.getElementById('time').textContent = new Date().toLocaleString();
    }
    
    loadStatus();
    setInterval(loadStatus, 5 * 60 * 1000);
  </script>
</body>
</html>
```

### File 4: `README.md`

```markdown
# Your App Status Page

Real-time status monitoring.

**Live Status:** https://YOUR_USERNAME.github.io/your-app-status/

## Services Monitored

| Service | Endpoint |
|---------|----------|
| Frontend | http://YOUR_SERVER:PORT |
| API | http://YOUR_SERVER:PORT/api/health/ |

## Setup

See [DEPLOYMENT_GUIDE.md](./DEPLOYMENT_GUIDE.md)
```

---

## Step 3: Set Up Self-Hosted Runner (For Private Networks)

**Skip this if monitoring PUBLIC URLs only.**

On your server:

```bash
# 1. Create directory
mkdir -p ~/actions-runner-status
cd ~/actions-runner-status

# 2. Download runner
curl -o actions-runner-linux-x64-2.321.0.tar.gz -L \
  https://github.com/actions/runner/releases/download/v2.321.0/actions-runner-linux-x64-2.321.0.tar.gz

# 3. Extract
tar xzf actions-runner-linux-x64-2.321.0.tar.gz

# 4. Get token from GitHub:
#    Go to: https://github.com/YOUR_USER/your-app-status/settings/actions/runners/new
#    Copy the token

# 5. Configure
./config.sh --url https://github.com/YOUR_USER/your-app-status --token YOUR_TOKEN

# 6. Install as service
sudo ./svc.sh install
sudo ./svc.sh start

# 7. Verify
sudo ./svc.sh status
```

---

## Step 4: Configure GitHub Settings

### 4.1 Enable Actions Permissions

1. Go to: `https://github.com/YOUR_USER/your-app-status/settings/actions`
2. **Workflow permissions:** Select **"Read and write permissions"**
3. ✅ Check **"Allow GitHub Actions to create and approve pull requests"**
4. **Save**

### 4.2 Configure GitHub Pages Environment

1. Go to: `https://github.com/YOUR_USER/your-app-status/settings/environments`
2. Click **"github-pages"** (create if doesn't exist)
3. Under **"Deployment branches"**:
   - Click **"Add deployment branch or tag rule"**
   - Select **"All branches"** OR add **"main"**
4. **Save protection rules**

### 4.3 Enable GitHub Pages

1. Go to: `https://github.com/YOUR_USER/your-app-status/settings/pages`
2. **Source:** Select **"GitHub Actions"**
3. **Save**

---

## Step 5: Push and Deploy

```bash
cd your-app-status

# Initialize
git init
git add .
git commit -m "Initial status page setup"

# Push
git branch -M main
git remote add origin https://github.com/YOUR_USER/your-app-status.git
git push -u origin main
```

### Run Workflow

1. Go to: `https://github.com/YOUR_USER/your-app-status/actions`
2. Click **"Uptime CI"**
3. Click **"Run workflow"** → **"Run workflow"**
4. Wait ~3-5 minutes

### View Your Status Page

```
https://YOUR_USER.github.io/your-app-status/
```

---

## Customization

### Change Check Frequency

In `.upptimerc.yml`:
```yaml
workflowSchedule:
  uptime: "*/5 * * * *"    # Every 5 minutes
  uptime: "*/15 * * * *"   # Every 15 minutes
```

### Add More Services

In `.upptimerc.yml`:
```yaml
sites:
  - name: Database
    url: http://YOUR_SERVER:5432/health
    expectedStatusCodes:
      - 200
```

### Custom Domain

1. Create `site/CNAME` with your domain:
   ```
   status.yourapp.com
   ```
2. Add DNS CNAME: `status.yourapp.com` → `YOUR_USER.github.io`
3. Configure in GitHub Pages settings

---

## Incidents

### Automatic Incidents ✅

Upptime **automatically** creates and manages incidents:

```
Service goes DOWN
       ↓
GitHub Issue created automatically
"🚨 NPA ECM Frontend is down"
       ↓
Service comes back UP
       ↓
Issue closed automatically
"✅ Resolved in 2h 15m"
```

**View incidents:** `https://github.com/YOUR_USER/your-app-status/issues`

The workflow needs `issues: write` permission (already included in the workflow above).

### Manual Incidents

For **scheduled maintenance** or **custom announcements**:

1. Go to: `https://github.com/YOUR_USER/your-app-status/issues/new`
2. **Title:** `Scheduled Maintenance: Database Upgrade`
3. **Add label:** `maintenance` or `incident`
4. **Body:** Describe the maintenance window

```markdown
## Scheduled Maintenance

**When:** November 28, 2025 - 2:00 AM to 4:00 AM WAT

**Affected Services:**
- Database
- API

**What's happening:**
Database upgrade to improve performance.

**Expected Impact:**
Brief service interruption. Users may experience slow responses.
```

The incident will appear on your status page!

### Incident Labels

| Label | Purpose |
|-------|---------|
| `incident` | Unplanned outage |
| `maintenance` | Scheduled maintenance |
| `degraded` | Partial service issues |
| `resolved` | Issue has been fixed |

---

## Notifications

Get alerts when services go down. Add to `.upptimerc.yml`:

### Slack Notifications

```yaml
notifications:
  - type: slack
    channel: alerts
    webhookUrl: ${{ secrets.SLACK_WEBHOOK }}
```

**Setup:**
1. Create Slack webhook: https://api.slack.com/messaging/webhooks
2. Add secret in GitHub: `Settings → Secrets → Actions → New secret`
   - Name: `SLACK_WEBHOOK`
   - Value: Your webhook URL

### Email Notifications

```yaml
notifications:
  - type: email
    address: admin@yourcompany.com
```

### SMS Notifications (via Twilio)

```yaml
notifications:
  - type: sms
    accountSid: ${{ secrets.TWILIO_SID }}
    authToken: ${{ secrets.TWILIO_TOKEN }}
    phoneNumber: "+1234567890"
```

### Microsoft Teams

```yaml
notifications:
  - type: teams
    webhookUrl: ${{ secrets.TEAMS_WEBHOOK }}
```

### Discord

```yaml
notifications:
  - type: discord
    webhookUrl: ${{ secrets.DISCORD_WEBHOOK }}
```

### Multiple Notifications

You can combine multiple notification channels:

```yaml
notifications:
  - type: slack
    channel: alerts
    webhookUrl: ${{ secrets.SLACK_WEBHOOK }}
    
  - type: email
    address: admin@yourcompany.com
    
  - type: email
    address: devops@yourcompany.com
```

---

## Troubleshooting

### "Waiting for a runner to pick up this job..."

Runner not connected. On your server:
```bash
cd ~/actions-runner-status
sudo ./svc.sh status
sudo ./svc.sh start
```

### "Branch not allowed to deploy to github-pages"

1. Go to: `Settings → Environments → github-pages`
2. Add deployment branch rule for `main` or `All branches`

### "Missing environment" error

Add to workflow:
```yaml
environment:
  name: github-pages
  url: ${{ steps.deployment.outputs.page_url }}
```

### "npx: command not found" or "could not determine executable"

Use direct node command:
```yaml
- run: node node_modules/@upptime/uptime-monitor/dist/index.js || true
```

### Node module version mismatch

Use Node.js 16:
```yaml
- uses: actions/setup-node@v4
  with:
    node-version: 16
```

---

## Adapting for Other Apps

1. **Create new repo:** `my-other-app-status`

2. **Copy files:**
   ```bash
   cp -r npa-ecm-status/* my-other-app-status/
   ```

3. **Update `.upptimerc.yml`:**
   - Change `owner` and `repo`
   - Update `sites` with your endpoints

4. **Update `site/index.html`:**
   - Change `GITHUB_USER` and `GITHUB_REPO`
   - Update icons mapping
   - Update title and links

5. **Set up runner** (if private network)

6. **Configure GitHub settings** (Actions, Environment, Pages)

7. **Push and deploy**

---

## Quick Reference

| Setting | Location |
|---------|----------|
| Actions permissions | Settings → Actions → General |
| Environment rules | Settings → Environments → github-pages |
| Pages source | Settings → Pages → Source: GitHub Actions |
| Runner status | Settings → Actions → Runners |

| File | Purpose |
|------|---------|
| `.upptimerc.yml` | Sites to monitor, branding |
| `.github/workflows/uptime.yml` | Automation workflow |
| `site/index.html` | Custom status page UI |
| `history/summary.json` | Auto-generated uptime data |

---

*Last updated: November 2025*
