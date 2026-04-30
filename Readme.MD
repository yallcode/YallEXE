<div align="center">

<img src="icon.png" width="80" height="80" alt="YallEXE Logo" />

# YallEXE

**Turn any HTML project into a native app — Android, Windows, macOS, and iOS.**

[![Live Demo](https://img.shields.io/badge/Live%20Demo-yallcode.github.io%2FYallEXE-ff6b35?style=flat-square&logo=github)](https://yallcode.github.io/YallEXE/)
[![Version](https://img.shields.io/badge/version-2.0.0--beta-00d4aa?style=flat-square)](https://github.com/yallcode/YallEXE/releases)
[![License](https://img.shields.io/badge/license-MIT-white?style=flat-square)](LICENSE)
[![YallCode](https://img.shields.io/badge/by-YallCode-ff6b35?style=flat-square)](https://yallcode.github.io/YallaYCode/)

</div>

---

## What is YallEXE?

YallEXE is a web-based compiler frontend that takes a **zipped HTML project** and converts it into native installable apps across all major platforms:

| Platform | Output | Engine |
|----------|--------|--------|
| 🤖 Android | `.apk` | Capacitor + Gradle |
| 🪟 Windows | `.exe` | Electron + NSIS |
| 🍎 macOS | `.dmg` | Electron + DMG |
| 📱 iOS | `.ipa` | Capacitor + Xcode |

The **frontend** is already live at [yallcode.github.io/YallEXE](https://yallcode.github.io/YallEXE/). This README documents the full project — what's built, what's mocked, and **how to wire up the real backend** using GitHub Actions.

---

## Table of Contents

- [Project Structure](#project-structure)
- [Frontend Features](#frontend-features)
- [How the Backend Works (Architecture)](#how-the-backend-works-architecture)
- [Full Backend Tutorial](#full-backend-tutorial)
  - [Step 1 — GitHub Actions Workflows](#step-1--github-actions-workflows)
  - [Step 2 — Cloudflare Worker API](#step-2--cloudflare-worker-api)
  - [Step 3 — Connect the Frontend](#step-3--connect-the-frontend)
- [iOS Code Signing Guide](#ios-code-signing-guide)
- [Deploying to GitHub Pages](#deploying-to-github-pages)
- [Roadmap](#roadmap)
- [Credits](#credits)

---

## Project Structure

```
YallEXE/
├── index.html          # Entire frontend (single file)
├── icon.png            # App logo / favicon
├── README.md           # This file
└── .github/
    └── workflows/
        ├── build-android.yml
        ├── build-windows.yml
        ├── build-mac.yml
        └── build-ios.yml
```

---

## Frontend Features

- **Drag & Drop Upload** — drop a `.zip` file containing your HTML project
- **Platform Selector** — toggle Android, Windows, macOS, iOS independently
- **iOS Advanced Settings** — toggle between YallEXE-managed signing and manual certificate upload (`.mobileprovision` + `.p12`)
- **App Metadata** — set app name, version, icon, and minimum OS version
- **Live Build Console** — terminal-style log viewer with color-coded output levels
- **Per-Platform Progress Bars** — individual fill bars for each build target
- **Download Panel** — download buttons for each compiled artifact
- **Dark Terminal UI** — built with Tailwind CDN + JetBrains Mono + Syne fonts, zero dependencies

> **Note:** The current frontend simulates builds with mock timers. The sections below show you how to connect it to real GitHub Actions builds.

---

## How the Backend Works (Architecture)

```
User Browser
    │
    │  1. Upload .zip + config (POST multipart)
    ▼
Cloudflare Worker  (yallexe.workers.dev)
    │
    │  2. Store zip in R2 bucket
    │  3. Trigger GitHub Actions via workflow_dispatch API
    │  4. Return a build_id to the browser
    ▼
GitHub Actions Runner
    │
    │  5. Download zip from R2
    │  6. Run Capacitor / Electron build
    │  7. Upload artifact (.apk / .exe / .dmg / .ipa)
    │  8. Call webhook back → Cloudflare Worker
    ▼
Cloudflare Worker  (webhook receiver)
    │
    │  9. Store artifact URL keyed by build_id
    ▼
User Browser
    │  10. Poll GET /status/:build_id every 3s
    │  11. When done → show download link
    ▼
  ✅ User downloads native app
```

---

## Full Backend Tutorial

### Prerequisites

Before starting, make sure you have:

- A [GitHub](https://github.com) account (you already have `yallcode`)
- A [Cloudflare](https://cloudflare.com) account (free tier works)
- [Node.js](https://nodejs.org) installed locally (or use GitHub Codespaces)
- [Wrangler CLI](https://developers.cloudflare.com/workers/wrangler/) — `npm install -g wrangler`

---

### Step 1 — GitHub Actions Workflows

Create these four workflow files inside `.github/workflows/` in your `YallEXE` repo.

#### `build-android.yml`

```yaml
name: Build Android APK

on:
  workflow_dispatch:
    inputs:
      build_id:
        required: true
        type: string
      zip_url:
        required: true
        type: string
      app_name:
        required: true
        type: string
      app_version:
        required: true
        type: string
      webhook_url:
        required: true
        type: string

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - name: Setup Java
        uses: actions/setup-java@v3
        with:
          java-version: '17'
          distribution: 'temurin'

      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'

      - name: Install Capacitor CLI
        run: npm install -g @capacitor/cli

      - name: Download project ZIP
        run: |
          curl -L "${{ github.event.inputs.zip_url }}" -o project.zip
          mkdir project && cd project
          unzip ../project.zip

      - name: Scaffold Capacitor Android project
        run: |
          cd project
          npm init -y
          npm install @capacitor/core @capacitor/android
          npx cap init "${{ github.event.inputs.app_name }}" \
            "com.yallexe.$(echo ${{ github.event.inputs.app_name }} | tr '[:upper:]' '[:lower:]' | tr -d ' ')" \
            --web-dir .
          npx cap add android

      - name: Sync web assets
        run: |
          cd project
          npx cap sync android

      - name: Build APK
        run: |
          cd project/android
          chmod +x gradlew
          ./gradlew assembleDebug

      - name: Upload artifact
        uses: actions/upload-artifact@v3
        with:
          name: android-apk-${{ github.event.inputs.build_id }}
          path: project/android/app/build/outputs/apk/debug/app-debug.apk

      - name: Notify webhook (success)
        if: success()
        run: |
          ARTIFACT_URL="https://github.com/${{ github.repository }}/actions/runs/${{ github.run_id }}"
          curl -X POST "${{ github.event.inputs.webhook_url }}" \
            -H "Content-Type: application/json" \
            -d "{\"build_id\":\"${{ github.event.inputs.build_id }}\",\"platform\":\"apk\",\"status\":\"done\",\"run_id\":\"${{ github.run_id }}\"}"

      - name: Notify webhook (failure)
        if: failure()
        run: |
          curl -X POST "${{ github.event.inputs.webhook_url }}" \
            -H "Content-Type: application/json" \
            -d "{\"build_id\":\"${{ github.event.inputs.build_id }}\",\"platform\":\"apk\",\"status\":\"error\"}"
```

#### `build-windows.yml`

```yaml
name: Build Windows EXE

on:
  workflow_dispatch:
    inputs:
      build_id:
        required: true
        type: string
      zip_url:
        required: true
        type: string
      app_name:
        required: true
        type: string
      app_version:
        required: true
        type: string
      webhook_url:
        required: true
        type: string

jobs:
  build:
    runs-on: windows-latest

    steps:
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'

      - name: Download project ZIP
        shell: bash
        run: |
          curl -L "${{ github.event.inputs.zip_url }}" -o project.zip
          mkdir project
          cd project
          unzip ../project.zip

      - name: Setup Electron project
        shell: bash
        run: |
          cd project
          cat > package.json << EOF
          {
            "name": "${{ github.event.inputs.app_name }}",
            "version": "${{ github.event.inputs.app_version }}",
            "main": "main.js",
            "scripts": { "build": "electron-builder --win" },
            "build": {
              "appId": "com.yallexe.app",
              "productName": "${{ github.event.inputs.app_name }}",
              "win": { "target": "nsis" },
              "directories": { "output": "dist" }
            }
          }
          EOF

          cat > main.js << 'EOF'
          const { app, BrowserWindow } = require('electron')
          const path = require('path')
          function createWindow() {
            const win = new BrowserWindow({ width: 1200, height: 800 })
            win.loadFile('index.html')
          }
          app.whenReady().then(createWindow)
          EOF

          npm install electron electron-builder --save-dev

      - name: Build EXE
        shell: bash
        run: |
          cd project
          npm run build

      - name: Upload artifact
        uses: actions/upload-artifact@v3
        with:
          name: windows-exe-${{ github.event.inputs.build_id }}
          path: project/dist/*.exe

      - name: Notify webhook (success)
        if: success()
        shell: bash
        run: |
          curl -X POST "${{ github.event.inputs.webhook_url }}" \
            -H "Content-Type: application/json" \
            -d "{\"build_id\":\"${{ github.event.inputs.build_id }}\",\"platform\":\"exe\",\"status\":\"done\",\"run_id\":\"${{ github.run_id }}\"}"

      - name: Notify webhook (failure)
        if: failure()
        shell: bash
        run: |
          curl -X POST "${{ github.event.inputs.webhook_url }}" \
            -H "Content-Type: application/json" \
            -d "{\"build_id\":\"${{ github.event.inputs.build_id }}\",\"platform\":\"exe\",\"status\":\"error\"}"
```

#### `build-mac.yml`

```yaml
name: Build macOS DMG

on:
  workflow_dispatch:
    inputs:
      build_id:
        required: true
        type: string
      zip_url:
        required: true
        type: string
      app_name:
        required: true
        type: string
      app_version:
        required: true
        type: string
      webhook_url:
        required: true
        type: string

jobs:
  build:
    runs-on: macos-latest

    steps:
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'

      - name: Download and setup project
        run: |
          curl -L "${{ github.event.inputs.zip_url }}" -o project.zip
          mkdir project && cd project && unzip ../project.zip

          cat > package.json << EOF
          {
            "name": "${{ github.event.inputs.app_name }}",
            "version": "${{ github.event.inputs.app_version }}",
            "main": "main.js",
            "scripts": { "build": "electron-builder --mac" },
            "build": {
              "appId": "com.yallexe.app",
              "productName": "${{ github.event.inputs.app_name }}",
              "mac": { "target": "dmg" },
              "directories": { "output": "dist" }
            }
          }
          EOF

          cat > main.js << 'EOF'
          const { app, BrowserWindow } = require('electron')
          function createWindow() {
            const win = new BrowserWindow({ width: 1200, height: 800 })
            win.loadFile('index.html')
          }
          app.whenReady().then(createWindow)
          EOF

          npm install electron electron-builder --save-dev

      - name: Build DMG
        run: cd project && npm run build

      - name: Upload artifact
        uses: actions/upload-artifact@v3
        with:
          name: mac-dmg-${{ github.event.inputs.build_id }}
          path: project/dist/*.dmg

      - name: Notify webhook (success)
        if: success()
        run: |
          curl -X POST "${{ github.event.inputs.webhook_url }}" \
            -H "Content-Type: application/json" \
            -d "{\"build_id\":\"${{ github.event.inputs.build_id }}\",\"platform\":\"mac\",\"status\":\"done\",\"run_id\":\"${{ github.run_id }}\"}"

      - name: Notify webhook (failure)
        if: failure()
        run: |
          curl -X POST "${{ github.event.inputs.webhook_url }}" \
            -H "Content-Type: application/json" \
            -d "{\"build_id\":\"${{ github.event.inputs.build_id }}\",\"platform\":\"mac\",\"status\":\"error\"}"
```

#### `build-ios.yml`

```yaml
name: Build iOS IPA

on:
  workflow_dispatch:
    inputs:
      build_id:
        required: true
        type: string
      zip_url:
        required: true
        type: string
      app_name:
        required: true
        type: string
      app_version:
        required: true
        type: string
      webhook_url:
        required: true
        type: string
      bundle_id:
        required: false
        type: string
        default: 'com.yallexe.app'

jobs:
  build:
    runs-on: macos-latest

    steps:
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'

      - name: Install Capacitor
        run: npm install -g @capacitor/cli

      - name: Download project
        run: |
          curl -L "${{ github.event.inputs.zip_url }}" -o project.zip
          mkdir project && cd project && unzip ../project.zip

      - name: Scaffold Capacitor iOS
        run: |
          cd project
          npm init -y
          npm install @capacitor/core @capacitor/ios
          npx cap init "${{ github.event.inputs.app_name }}" "${{ github.event.inputs.bundle_id }}" --web-dir .
          npx cap add ios
          npx cap sync ios

      - name: Build IPA (unsigned)
        run: |
          cd project/ios/App
          xcodebuild \
            -workspace App.xcworkspace \
            -scheme App \
            -configuration Release \
            -archivePath build/App.xcarchive \
            archive \
            CODE_SIGN_IDENTITY="" \
            CODE_SIGNING_REQUIRED=NO \
            CODE_SIGNING_ALLOWED=NO

          xcodebuild \
            -exportArchive \
            -archivePath build/App.xcarchive \
            -exportPath build/output \
            -exportOptionsPlist exportOptions.plist

      - name: Upload artifact
        uses: actions/upload-artifact@v3
        with:
          name: ios-ipa-${{ github.event.inputs.build_id }}
          path: project/ios/App/build/output/*.ipa

      - name: Notify webhook
        if: always()
        run: |
          STATUS=${{ job.status == 'success' && 'done' || 'error' }}
          curl -X POST "${{ github.event.inputs.webhook_url }}" \
            -H "Content-Type: application/json" \
            -d "{\"build_id\":\"${{ github.event.inputs.build_id }}\",\"platform\":\"ipa\",\"status\":\"${STATUS}\",\"run_id\":\"${{ github.run_id }}\"}"
```

> **After creating these files**, go to your repo → **Settings → Actions → General** → set "Workflow permissions" to **Read and write**.

---

### Step 2 — Cloudflare Worker API

This Worker is the bridge between your frontend and GitHub Actions. It handles:
- Receiving the uploaded ZIP
- Storing it in R2
- Triggering the correct GitHub Actions workflow
- Receiving the webhook callback
- Serving build status to the frontend

#### Setup R2 Bucket

```bash
# Login to Cloudflare
wrangler login

# Create the R2 bucket
wrangler r2 bucket create yallexe-builds
```

#### Create the Worker project

```bash
mkdir yallexe-worker && cd yallexe-worker
wrangler init --type javascript
```

#### `wrangler.toml`

```toml
name = "yallexe-api"
main = "src/index.js"
compatibility_date = "2024-01-01"

[[r2_buckets]]
binding = "BUILDS"
bucket_name = "yallexe-builds"

[vars]
GITHUB_OWNER = "yallcode"
GITHUB_REPO = "YallEXE"
```

#### Add secrets (run these in terminal)

```bash
# Your GitHub Personal Access Token
# Go to: GitHub → Settings → Developer settings → Personal access tokens → Fine-grained
# Permissions needed: Actions (write), Contents (read)
wrangler secret put GITHUB_TOKEN

# A random string you invent — used to verify webhook calls from Actions
wrangler secret put WEBHOOK_SECRET
```

#### `src/index.js`

```javascript
export default {
  async fetch(request, env) {
    const url = new URL(request.url);
    const { pathname } = url;

    // CORS headers for all responses
    const cors = {
      'Access-Control-Allow-Origin': 'https://yallcode.github.io',
      'Access-Control-Allow-Methods': 'GET, POST, OPTIONS',
      'Access-Control-Allow-Headers': 'Content-Type',
    };

    if (request.method === 'OPTIONS') {
      return new Response(null, { headers: cors });
    }

    // ── POST /build ───────────────────────────────────────────
    // Receives: multipart/form-data with zip + config
    // Returns:  { build_id }
    if (pathname === '/build' && request.method === 'POST') {
      try {
        const form = await request.formData();
        const zip = form.get('zip');           // File object
        const platforms = form.get('platforms').split(','); // e.g. "apk,exe"
        const appName = form.get('app_name') || 'YallApp';
        const appVersion = form.get('app_version') || '1.0.0';
        const bundleId = form.get('bundle_id') || 'com.yallexe.app';

        // Generate a unique build ID
        const buildId = crypto.randomUUID();

        // Store ZIP in R2
        const zipKey = `builds/${buildId}/project.zip`;
        await env.BUILDS.put(zipKey, zip.stream(), {
          httpMetadata: { contentType: 'application/zip' },
        });

        // Public URL for the ZIP (Workers R2 public bucket or presigned URL)
        // For simplicity, we serve it via this worker at /file/:buildId
        const zipUrl = `https://yallexe-api.yallcc-info.workers.dev/file/${buildId}`;
        const webhookUrl = `https://yallexe-api.yallcc-info.workers.dev/webhook`;

        // Initialise build status in KV (or use a D1 DB for more features)
        const buildStatus = {};
        platforms.forEach(p => { buildStatus[p] = 'queued'; });

        await env.BUILDS.put(`status/${buildId}`, JSON.stringify({
          build_id: buildId,
          platforms,
          status: buildStatus,
          created_at: new Date().toISOString(),
          run_ids: {},
        }));

        // Trigger one GitHub Actions workflow per platform
        const workflowMap = {
          apk: 'build-android.yml',
          exe: 'build-windows.yml',
          mac: 'build-mac.yml',
          ipa: 'build-ios.yml',
        };

        await Promise.all(platforms.map(platform => {
          const workflow = workflowMap[platform];
          if (!workflow) return;
          return triggerWorkflow(env, workflow, {
            build_id: buildId,
            zip_url: zipUrl,
            app_name: appName,
            app_version: appVersion,
            webhook_url: webhookUrl,
            bundle_id: bundleId,
          });
        }));

        return new Response(JSON.stringify({ build_id: buildId }), {
          headers: { 'Content-Type': 'application/json', ...cors },
        });

      } catch (err) {
        return new Response(JSON.stringify({ error: err.message }), {
          status: 500,
          headers: { 'Content-Type': 'application/json', ...cors },
        });
      }
    }

    // ── GET /status/:buildId ──────────────────────────────────
    // Polled by the frontend every 3 seconds
    if (pathname.startsWith('/status/') && request.method === 'GET') {
      const buildId = pathname.split('/')[2];
      const raw = await env.BUILDS.get(`status/${buildId}`);
      if (!raw) {
        return new Response(JSON.stringify({ error: 'Not found' }), {
          status: 404,
          headers: { 'Content-Type': 'application/json', ...cors },
        });
      }
      const data = JSON.parse(raw.text());
      return new Response(raw.text(), {
        headers: { 'Content-Type': 'application/json', ...cors },
      });
    }

    // ── GET /file/:buildId ────────────────────────────────────
    // Serves the stored ZIP back to GitHub Actions
    if (pathname.startsWith('/file/') && request.method === 'GET') {
      const buildId = pathname.split('/')[2];
      const obj = await env.BUILDS.get(`builds/${buildId}/project.zip`);
      if (!obj) return new Response('Not found', { status: 404 });
      return new Response(obj.body, {
        headers: { 'Content-Type': 'application/zip' },
      });
    }

    // ── POST /webhook ─────────────────────────────────────────
    // Called by GitHub Actions when a build finishes
    if (pathname === '/webhook' && request.method === 'POST') {
      const payload = await request.json();
      const { build_id, platform, status, run_id } = payload;

      const raw = await env.BUILDS.get(`status/${build_id}`);
      if (!raw) return new Response('Not found', { status: 404 });

      const data = JSON.parse(raw.text());
      data.status[platform] = status;
      data.run_ids[platform] = run_id;

      // Check if all platforms are done
      const allDone = Object.values(data.status).every(s => s === 'done' || s === 'error');
      if (allDone) data.finished_at = new Date().toISOString();

      await env.BUILDS.put(`status/${build_id}`, JSON.stringify(data));
      return new Response('OK');
    }

    return new Response('YallEXE API v2', { headers: cors });
  },
};

// ── Helper: trigger a GitHub Actions workflow ─────────────────
async function triggerWorkflow(env, workflow, inputs) {
  const res = await fetch(
    `https://api.github.com/repos/${env.GITHUB_OWNER}/${env.GITHUB_REPO}/actions/workflows/${workflow}/dispatches`,
    {
      method: 'POST',
      headers: {
        Authorization: `Bearer ${env.GITHUB_TOKEN}`,
        Accept: 'application/vnd.github+json',
        'Content-Type': 'application/json',
        'User-Agent': 'YallEXE-Worker/2.0',
      },
      body: JSON.stringify({
        ref: 'main',
        inputs,
      }),
    }
  );

  if (!res.ok) {
    const text = await res.text();
    throw new Error(`GitHub API error (${res.status}): ${text}`);
  }
}
```

#### Deploy the Worker

```bash
wrangler deploy
```

Your API will be live at:
```
https://yallexe-api.YOUR_SUBDOMAIN.workers.dev
```

---

### Step 3 — Connect the Frontend

Replace the mock `startBuild()` function in `index.html` with this real implementation:

```javascript
const WORKER_URL = 'https://yallexe-api.yallcc-info.workers.dev';
let pollInterval = null;

async function startBuild() {
  if (state.buildRunning || !state.file || state.platforms.size === 0) return;

  state.buildRunning = true;
  buildBtn.disabled = true;
  buildPanel.classList.remove('hidden');
  setStatus('running');
  buildBtn.innerHTML = `<div class="spinner"></div> Uploading...`;

  try {
    // Build form data
    const form = new FormData();
    form.append('zip', state.file);
    form.append('platforms', [...state.platforms].join(','));
    form.append('app_name', document.getElementById('global-app-name').value || 'YallApp');
    form.append('app_version', document.getElementById('global-version').value || '1.0.0');
    form.append('bundle_id', document.getElementById('bundle-id')?.value || 'com.yallexe.app');

    log('› Uploading project to YallEXE servers...', 'log-info');

    const res = await fetch(`${WORKER_URL}/build`, {
      method: 'POST',
      body: form,
    });

    if (!res.ok) throw new Error(`Server error: ${res.status}`);

    const { build_id } = await res.json();
    log(`✓ Build queued — ID: ${build_id}`, 'log-success');
    log('› Waiting for GitHub Actions runners...', 'log-info');

    buildBtn.innerHTML = `<div class="spinner"></div> Building…`;

    // Poll for status
    pollInterval = setInterval(async () => {
      const statusRes = await fetch(`${WORKER_URL}/status/${build_id}`);
      if (!statusRes.ok) return;

      const data = await statusRes.json();

      // Update per-platform progress
      Object.entries(data.status).forEach(([platform, status]) => {
        const fill = document.getElementById(`plat-fill-${platform}`);
        const pct  = document.getElementById(`plat-pct-${platform}`);
        const lbl  = document.getElementById(`plat-status-${platform}`);

        if (status === 'done') {
          if (fill) fill.style.width = '100%';
          if (pct)  pct.textContent = '100%';
          if (lbl)  { lbl.textContent = '✓ done'; lbl.style.color = '#00d4aa'; }
          log(`✓ ${platform.toUpperCase()} build complete`, 'log-success');
        } else if (status === 'error') {
          if (lbl) { lbl.textContent = '✗ error'; lbl.style.color = '#ff4d4d'; }
          log(`✗ ${platform.toUpperCase()} build failed`, 'log-error');
        }
      });

      // Check if all done
      const allFinished = Object.values(data.status).every(
        s => s === 'done' || s === 'error'
      );

      if (allFinished) {
        clearInterval(pollInterval);
        progressFill.style.width = '100%';
        setStatus('done');
        log('✓ All builds finished!', 'log-success');
        state.buildRunning = false;
        buildBtn.disabled = false;
        buildBtn.innerHTML = `↻ Build Again`;

        // Show real GitHub Actions download links
        showDownloadLinks(data);
      }
    }, 3000);

  } catch (err) {
    log(`✗ Error: ${err.message}`, 'log-error');
    setStatus('error');
    state.buildRunning = false;
    buildBtn.disabled = false;
    buildBtn.innerHTML = `↻ Retry`;
  }
}

function showDownloadLinks(data) {
  downloadsSection.classList.remove('hidden');
  const labels = { apk: 'Android APK', exe: 'Windows EXE', mac: 'macOS DMG', ipa: 'iOS IPA' };
  const exts   = { apk: '.apk', exe: '.exe', mac: '.dmg', ipa: '.ipa' };

  Object.entries(data.run_ids).forEach(([platform, runId]) => {
    const name = document.getElementById('global-app-name').value || 'YallApp';
    const card = document.createElement('div');
    card.className = 'dl-card flex items-center justify-between px-4 py-3';
    card.innerHTML = `
      <div>
        <p class="text-sm font-display font-600 text-gray-200">${name}${exts[platform]}</p>
        <p class="text-xs font-mono text-gray-600">${labels[platform]}</p>
      </div>
      <a href="https://github.com/${GITHUB_OWNER}/${GITHUB_REPO}/actions/runs/${runId}"
         target="_blank"
         class="text-xs font-mono text-brand-teal border border-brand-teal/30 px-3 py-1.5 rounded-md hover:bg-brand-teal/10 transition-colors">
        View on Actions ↗
      </a>
    `;
    downloadsGrid.appendChild(card);
  });
}
```

---

## iOS Code Signing Guide

iOS is the hardest platform. Here's what each field in YallEXE's iOS Advanced Settings means:

| Field | What it is | Where to get it |
|-------|-----------|-----------------|
| Provisioning Profile | Links your app to allowed devices/distribution method | Apple Developer Portal → Certificates, IDs & Profiles |
| Certificate (.p12) | Your signing identity (exported from Keychain Access) | Keychain Access → Export as .p12 |
| Certificate Password | The password you set when exporting | You set this on export |
| Bundle ID | Your app's unique identifier | Apple Developer Portal → Identifiers |
| Team ID | Your Apple Developer Team | developer.apple.com → Account → Membership |

**For testing (Ad-Hoc distribution):**
1. Register your test device UDIDs in the Developer Portal
2. Create an Ad-Hoc provisioning profile
3. Export your certificate as `.p12` from Keychain Access
4. Upload both into YallEXE → iOS Advanced Settings

**For App Store distribution**, you need a Distribution certificate and an App Store provisioning profile. This requires an active Apple Developer Program membership ($99/year).

---

## Deploying to GitHub Pages

The frontend auto-deploys via GitHub Pages since `index.html` is in the repo root.

To enable it:
1. Go to your repo → **Settings → Pages**
2. Source: **Deploy from a branch**
3. Branch: **main** / root
4. Save → your site is live at `https://yallcode.github.io/YallEXE/`

To add a custom domain later, create a `CNAME` file in the repo root:
```
yallexe.yallcc-info.com
```

---

## Roadmap

- [x] Frontend UI (drag & drop, platform selector, console log)
- [x] iOS Advanced Settings (certificates, provisioning, Bundle ID)
- [x] Live on GitHub Pages
- [ ] Cloudflare Worker API
- [ ] GitHub Actions — Android (Capacitor + Gradle)
- [ ] GitHub Actions — Windows (Electron + NSIS)
- [ ] GitHub Actions — macOS (Electron + DMG)
- [ ] GitHub Actions — iOS (Capacitor + Xcode)
- [ ] Real-time WebSocket build log streaming
- [ ] Build history page
- [ ] Custom app icon injection
- [ ] Release signing (production certificates)

---

## Credits

Built by **[YallCode](https://yallcode.github.io/YallaYCode/)**

- 🌐 Site: [yallcode.github.io/YallaYCode](https://yallcode.github.io/YallaYCode/)
- 🐙 GitHub: [@yallcode](https://github.com/yallcode)
- 🐦 X/Twitter: [@YallCode](https://twitter.com/YallCode)
- 📺 YouTube: [@YallaYCode](https://youtube.com/@YallaYCode)
- 💬 Discord: [discord.gg/yUe8kE5fRF](https://discord.gg/yUe8kE5fRF)

---

<div align="center">
<sub>YallEXE v2.0.0-beta · HTML → Native · MIT License</sub>
</div>
