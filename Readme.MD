<div align="center">

<img src="icon.png" width="80" alt="YallEXE" />

# YallEXE

turn any html project into a real native app. android, windows, mac, ios.

[![Live Site](https://img.shields.io/badge/live-yallcode.github.io/YallEXE-ff6b35?style=flat-square)](https://yallcode.github.io/YallEXE/)
[![Version](https://img.shields.io/badge/version-2.0.0--beta-00d4aa?style=flat-square)](https://github.com/yallcode/YallEXE/releases)
[![By YallCode](https://img.shields.io/badge/by-YallCode-white?style=flat-square)](https://yallcode.github.io/YallaYCode/)

</div>

---

## what is this

YallEXE takes a zipped HTML project (your index.html, js, css, assets, all of it) and compiles it into installable native apps. you pick which platforms you want and it handles the rest.

the frontend is live at [yallcode.github.io/YallEXE](https://yallcode.github.io/YallEXE/). the backend is still being wired up so this readme covers everything including all the code you need to make builds actually work.

platforms it supports:

| platform | output | how it builds |
|----------|--------|---------------|
| Android | .apk | Capacitor + Gradle |
| Windows | .exe | Electron + NSIS |
| macOS | .dmg | Electron + DMG |
| iOS | .ipa | Capacitor + Xcode |

---

## table of contents

- [project structure](#project-structure)
- [how it works](#how-it-works)
- [backend setup](#backend-setup)
  - [github actions workflows](#github-actions-workflows)
  - [cloudflare worker](#cloudflare-worker)
  - [connecting the frontend](#connecting-the-frontend)
- [ios signing](#ios-signing)
- [github pages](#github-pages)
- [roadmap](#roadmap)

---

## project structure

```
YallEXE/
├── index.html
├── icon.png
├── README.md
└── .github/
    └── workflows/
        ├── build-android.yml
        ├── build-windows.yml
        ├── build-mac.yml
        └── build-ios.yml
```

nothing complicated. single html file for the frontend, github actions for the actual building, cloudflare worker as the api in between.

---

## how it works

```
user uploads zip on the site
       |
cloudflare worker receives it
stores zip in R2, fires off github actions, returns a build_id
       |
github actions runner picks it up
runs gradle or electron or xcode depending on platform
uploads the artifact, pings the webhook back
       |
frontend polls /status/:build_id every 3 seconds
once all platforms report done it shows the download links
```

i went with Cloudflare Workers for the api because i already use it for my YallAnime streaming site and the setup is exactly the same. R2 handles storing the uploaded zips for free at the scale this needs. no reason to learn a new platform when this stack already works.

---

## backend setup

### github actions workflows

create `.github/workflows/` in your repo and drop these four files in. each one gets triggered remotely and runs on github's machines so nothing compiles locally, which matters if you are on a chromebook.

before anything else go to repo Settings, then Actions, then General, and set workflow permissions to **Read and write**.

---

**build-android.yml**

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
      - name: setup java
        uses: actions/setup-java@v3
        with:
          java-version: '17'
          distribution: 'temurin'

      - name: setup node
        uses: actions/setup-node@v3
        with:
          node-version: '18'

      - name: install capacitor
        run: npm install -g @capacitor/cli

      - name: download zip
        run: |
          curl -L "${{ github.event.inputs.zip_url }}" -o project.zip
          mkdir project && cd project && unzip ../project.zip

      - name: setup capacitor android
        run: |
          cd project
          npm init -y
          npm install @capacitor/core @capacitor/android
          npx cap init "${{ github.event.inputs.app_name }}" \
            "com.yallexe.$(echo ${{ github.event.inputs.app_name }} | tr '[:upper:]' '[:lower:]' | tr -d ' ')" \
            --web-dir .
          npx cap add android
          npx cap sync android

      - name: build apk
        run: |
          cd project/android
          chmod +x gradlew
          ./gradlew assembleDebug

      - name: upload artifact
        uses: actions/upload-artifact@v3
        with:
          name: android-apk-${{ github.event.inputs.build_id }}
          path: project/android/app/build/outputs/apk/debug/app-debug.apk

      - name: notify webhook on success
        if: success()
        run: |
          curl -X POST "${{ github.event.inputs.webhook_url }}" \
            -H "Content-Type: application/json" \
            -d "{\"build_id\":\"${{ github.event.inputs.build_id }}\",\"platform\":\"apk\",\"status\":\"done\",\"run_id\":\"${{ github.run_id }}\"}"

      - name: notify webhook on failure
        if: failure()
        run: |
          curl -X POST "${{ github.event.inputs.webhook_url }}" \
            -H "Content-Type: application/json" \
            -d "{\"build_id\":\"${{ github.event.inputs.build_id }}\",\"platform\":\"apk\",\"status\":\"error\"}"
```

---

**build-windows.yml**

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
      - name: setup node
        uses: actions/setup-node@v3
        with:
          node-version: '18'

      - name: download and setup project
        shell: bash
        run: |
          curl -L "${{ github.event.inputs.zip_url }}" -o project.zip
          mkdir project && cd project && unzip ../project.zip

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
          function createWindow() {
            const win = new BrowserWindow({ width: 1200, height: 800 })
            win.loadFile('index.html')
          }
          app.whenReady().then(createWindow)
          EOF

          npm install electron electron-builder --save-dev

      - name: build exe
        shell: bash
        run: cd project && npm run build

      - name: upload artifact
        uses: actions/upload-artifact@v3
        with:
          name: windows-exe-${{ github.event.inputs.build_id }}
          path: project/dist/*.exe

      - name: notify webhook on success
        if: success()
        shell: bash
        run: |
          curl -X POST "${{ github.event.inputs.webhook_url }}" \
            -H "Content-Type: application/json" \
            -d "{\"build_id\":\"${{ github.event.inputs.build_id }}\",\"platform\":\"exe\",\"status\":\"done\",\"run_id\":\"${{ github.run_id }}\"}"

      - name: notify webhook on failure
        if: failure()
        shell: bash
        run: |
          curl -X POST "${{ github.event.inputs.webhook_url }}" \
            -H "Content-Type: application/json" \
            -d "{\"build_id\":\"${{ github.event.inputs.build_id }}\",\"platform\":\"exe\",\"status\":\"error\"}"
```

---

**build-mac.yml**

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
      - name: setup node
        uses: actions/setup-node@v3
        with:
          node-version: '18'

      - name: download and setup project
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

      - name: build dmg
        run: cd project && npm run build

      - name: upload artifact
        uses: actions/upload-artifact@v3
        with:
          name: mac-dmg-${{ github.event.inputs.build_id }}
          path: project/dist/*.dmg

      - name: notify webhook on success
        if: success()
        run: |
          curl -X POST "${{ github.event.inputs.webhook_url }}" \
            -H "Content-Type: application/json" \
            -d "{\"build_id\":\"${{ github.event.inputs.build_id }}\",\"platform\":\"mac\",\"status\":\"done\",\"run_id\":\"${{ github.run_id }}\"}"

      - name: notify webhook on failure
        if: failure()
        run: |
          curl -X POST "${{ github.event.inputs.webhook_url }}" \
            -H "Content-Type: application/json" \
            -d "{\"build_id\":\"${{ github.event.inputs.build_id }}\",\"platform\":\"mac\",\"status\":\"error\"}"
```

---

**build-ios.yml**

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
      - name: setup node
        uses: actions/setup-node@v3
        with:
          node-version: '18'

      - name: install capacitor
        run: npm install -g @capacitor/cli

      - name: download project
        run: |
          curl -L "${{ github.event.inputs.zip_url }}" -o project.zip
          mkdir project && cd project && unzip ../project.zip

      - name: setup capacitor ios
        run: |
          cd project
          npm init -y
          npm install @capacitor/core @capacitor/ios
          npx cap init "${{ github.event.inputs.app_name }}" "${{ github.event.inputs.bundle_id }}" --web-dir .
          npx cap add ios
          npx cap sync ios

      - name: build ipa unsigned
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

      - name: upload artifact
        uses: actions/upload-artifact@v3
        with:
          name: ios-ipa-${{ github.event.inputs.build_id }}
          path: project/ios/App/build/**/*.ipa

      - name: notify webhook
        if: always()
        run: |
          STATUS="${{ job.status == 'success' && 'done' || 'error' }}"
          curl -X POST "${{ github.event.inputs.webhook_url }}" \
            -H "Content-Type: application/json" \
            -d "{\"build_id\":\"${{ github.event.inputs.build_id }}\",\"platform\":\"ipa\",\"status\":\"${STATUS}\",\"run_id\":\"${{ github.run_id }}\"}"
```

---

### cloudflare worker

Same as what i used for YallAnime. One Worker does everything: receives the zip, stores it in R2, fires off the right github actions workflow, handles the webhook callback, and serves the status to the frontend when it polls.

**create the R2 bucket**

```bash
wrangler login
wrangler r2 bucket create yallexe-builds
```

**create the worker**

```bash
mkdir yallexe-worker && cd yallexe-worker
wrangler init --type javascript
```

**wrangler.toml**

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

**add secrets**

go to github, make a fine-grained personal access token with Actions (write) and Contents (read) permissions, then:

```bash
wrangler secret put GITHUB_TOKEN
wrangler secret put WEBHOOK_SECRET
```

**src/index.js**

```javascript
export default {
  async fetch(request, env) {
    const url = new URL(request.url)
    const path = url.pathname

    const cors = {
      'Access-Control-Allow-Origin': 'https://yallcode.github.io',
      'Access-Control-Allow-Methods': 'GET, POST, OPTIONS',
      'Access-Control-Allow-Headers': 'Content-Type',
    }

    if (request.method === 'OPTIONS') {
      return new Response(null, { headers: cors })
    }

    if (path === '/build' && request.method === 'POST') {
      try {
        const form = await request.formData()
        const zip = form.get('zip')
        const platforms = form.get('platforms').split(',')
        const appName = form.get('app_name') || 'YallApp'
        const appVersion = form.get('app_version') || '1.0.0'
        const bundleId = form.get('bundle_id') || 'com.yallexe.app'

        const buildId = crypto.randomUUID()

        await env.BUILDS.put(`builds/${buildId}/project.zip`, zip.stream(), {
          httpMetadata: { contentType: 'application/zip' },
        })

        const statusObj = {}
        platforms.forEach(p => { statusObj[p] = 'queued' })
        await env.BUILDS.put(`status/${buildId}`, JSON.stringify({
          build_id: buildId,
          platforms,
          status: statusObj,
          run_ids: {},
          created_at: new Date().toISOString(),
        }))

        const zipUrl = `https://yallexe-api.yallcc-info.workers.dev/file/${buildId}`
        const webhookUrl = `https://yallexe-api.yallcc-info.workers.dev/webhook`

        const workflowMap = {
          apk: 'build-android.yml',
          exe: 'build-windows.yml',
          mac: 'build-mac.yml',
          ipa: 'build-ios.yml',
        }

        await Promise.all(platforms.map(p => {
          if (!workflowMap[p]) return
          return triggerWorkflow(env, workflowMap[p], {
            build_id: buildId,
            zip_url: zipUrl,
            app_name: appName,
            app_version: appVersion,
            webhook_url: webhookUrl,
            bundle_id: bundleId,
          })
        }))

        return new Response(JSON.stringify({ build_id: buildId }), {
          headers: { 'Content-Type': 'application/json', ...cors },
        })

      } catch (err) {
        return new Response(JSON.stringify({ error: err.message }), {
          status: 500,
          headers: { 'Content-Type': 'application/json', ...cors },
        })
      }
    }

    if (path.startsWith('/file/') && request.method === 'GET') {
      const buildId = path.split('/')[2]
      const obj = await env.BUILDS.get(`builds/${buildId}/project.zip`)
      if (!obj) return new Response('not found', { status: 404 })
      return new Response(obj.body, { headers: { 'Content-Type': 'application/zip' } })
    }

    if (path.startsWith('/status/') && request.method === 'GET') {
      const buildId = path.split('/')[2]
      const obj = await env.BUILDS.get(`status/${buildId}`)
      if (!obj) return new Response(JSON.stringify({ error: 'not found' }), {
        status: 404,
        headers: { 'Content-Type': 'application/json', ...cors },
      })
      return new Response(await obj.text(), {
        headers: { 'Content-Type': 'application/json', ...cors },
      })
    }

    if (path === '/webhook' && request.method === 'POST') {
      const { build_id, platform, status, run_id } = await request.json()
      const obj = await env.BUILDS.get(`status/${build_id}`)
      if (!obj) return new Response('not found', { status: 404 })

      const data = JSON.parse(await obj.text())
      data.status[platform] = status
      data.run_ids[platform] = run_id

      const allDone = Object.values(data.status).every(s => s === 'done' || s === 'error')
      if (allDone) data.finished_at = new Date().toISOString()

      await env.BUILDS.put(`status/${build_id}`, JSON.stringify(data))
      return new Response('ok')
    }

    return new Response('YallEXE API', { headers: cors })
  },
}

async function triggerWorkflow(env, workflow, inputs) {
  const res = await fetch(
    `https://api.github.com/repos/${env.GITHUB_OWNER}/${env.GITHUB_REPO}/actions/workflows/${workflow}/dispatches`,
    {
      method: 'POST',
      headers: {
        Authorization: `Bearer ${env.GITHUB_TOKEN}`,
        Accept: 'application/vnd.github+json',
        'Content-Type': 'application/json',
        'User-Agent': 'YallEXE/2.0',
      },
      body: JSON.stringify({ ref: 'main', inputs }),
    }
  )
  if (!res.ok) throw new Error(`github api ${res.status}: ${await res.text()}`)
}
```

**deploy it**

```bash
wrangler deploy
```

---

### connecting the frontend

replace the fake `startBuild()` in index.html with this real version. it uploads to the worker, gets a build_id back, then polls every 3 seconds.

```javascript
const WORKER_URL = 'https://yallexe-api.yallcc-info.workers.dev'
let pollInterval = null

async function startBuild() {
  if (state.buildRunning || !state.file || state.platforms.size === 0) return

  state.buildRunning = true
  buildBtn.disabled = true
  buildPanel.classList.remove('hidden')
  setStatus('running')
  buildBtn.innerHTML = `<div class="spinner"></div> Uploading...`

  try {
    const form = new FormData()
    form.append('zip', state.file)
    form.append('platforms', [...state.platforms].join(','))
    form.append('app_name', document.getElementById('global-app-name').value || 'YallApp')
    form.append('app_version', document.getElementById('global-version').value || '1.0.0')
    form.append('bundle_id', document.getElementById('bundle-id')?.value || 'com.yallexe.app')

    log('uploading project...', 'log-info')

    const res = await fetch(`${WORKER_URL}/build`, { method: 'POST', body: form })
    if (!res.ok) throw new Error(`server error ${res.status}`)

    const { build_id } = await res.json()
    log(`build queued, id: ${build_id}`, 'log-success')
    log('waiting for github actions to pick it up...', 'log-info')
    buildBtn.innerHTML = `<div class="spinner"></div> Building...`

    pollInterval = setInterval(async () => {
      try {
        const statusRes = await fetch(`${WORKER_URL}/status/${build_id}`)
        if (!statusRes.ok) return
        const data = await statusRes.json()

        Object.entries(data.status).forEach(([platform, status]) => {
          const fill = document.getElementById(`plat-fill-${platform}`)
          const pct = document.getElementById(`plat-pct-${platform}`)
          const lbl = document.getElementById(`plat-status-${platform}`)
          if (status === 'done') {
            if (fill) fill.style.width = '100%'
            if (pct) pct.textContent = '100%'
            if (lbl) { lbl.textContent = 'done'; lbl.style.color = '#00d4aa' }
            log(`${platform} build done`, 'log-success')
          } else if (status === 'error') {
            if (lbl) { lbl.textContent = 'error'; lbl.style.color = '#ff4d4d' }
            log(`${platform} build failed`, 'log-error')
          }
        })

        const allDone = Object.values(data.status).every(s => s === 'done' || s === 'error')
        if (allDone) {
          clearInterval(pollInterval)
          progressFill.style.width = '100%'
          setStatus('done')
          log('all done', 'log-success')
          state.buildRunning = false
          buildBtn.disabled = false
          buildBtn.innerHTML = 'Build Again'
          showDownloadLinks(data)
        }
      } catch (e) {
        // network blip, try again next tick
      }
    }, 3000)

  } catch (err) {
    log(`error: ${err.message}`, 'log-error')
    setStatus('error')
    state.buildRunning = false
    buildBtn.disabled = false
    buildBtn.innerHTML = 'Retry'
  }
}

function showDownloadLinks(data) {
  downloadsSection.classList.remove('hidden')
  const labels = { apk: 'Android APK', exe: 'Windows EXE', mac: 'macOS DMG', ipa: 'iOS IPA' }
  const exts = { apk: '.apk', exe: '.exe', mac: '.dmg', ipa: '.ipa' }
  const name = document.getElementById('global-app-name').value || 'YallApp'

  Object.entries(data.run_ids).forEach(([platform, runId]) => {
    const card = document.createElement('div')
    card.className = 'dl-card flex items-center justify-between px-4 py-3'
    card.innerHTML = `
      <div>
        <p class="text-sm font-display font-600 text-gray-200">${name}${exts[platform]}</p>
        <p class="text-xs font-mono text-gray-600">${labels[platform]}</p>
      </div>
      <a href="https://github.com/yallcode/YallEXE/actions/runs/${runId}"
         target="_blank"
         class="text-xs font-mono text-brand-teal border border-brand-teal/30 px-3 py-1.5 rounded-md hover:bg-brand-teal/10 transition-colors">
        view artifact
      </a>
    `
    downloadsGrid.appendChild(card)
  })
}
```

---

## ios signing

ios is the painful one. here is what each field in the iOS Advanced Settings panel actually means and where to get it.

**provisioning profile (.mobileprovision)** — links your app to allowed devices and a distribution method. get it from the apple developer portal under Certificates, IDs and Profiles. for device testing pick Ad-Hoc, for app store pick App Store Distribution.

**certificate (.p12)** — your signing identity. on a mac open Keychain Access, find your Apple Distribution certificate, right click it and export as .p12. set a password when it asks.

**bundle identifier** — a unique ID like com.yallcode.myapp. has to match what you registered in the developer portal under Identifiers.

**team ID** — find it at developer.apple.com under Account then Membership Details.

the build-ios.yml right now produces an unsigned IPA which you can sideload with [Sideloadly](https://sideloadly.io) or [AltStore](https://altstore.io) for personal testing without needing a paid account. if you want proper distribution you need the $99/year Apple Developer membership.

---

## github pages

the site deploys from the main branch automatically. if pages is not set up yet go to repo Settings, click Pages, set source to Deploy from a branch, pick main and root, save. it goes live at `https://yallcode.github.io/YallEXE/` in about a minute.

for a custom domain create a file called `CNAME` in the repo root:

```
yallexe.com
```

---

## roadmap

- [x] frontend ui
- [x] ios advanced settings
- [x] live on github pages
- [ ] cloudflare worker api
- [ ] android build (capacitor + gradle)
- [ ] windows build (electron + nsis)
- [ ] macos build (electron + dmg)
- [ ] ios build (capacitor + xcode)
- [ ] websocket log streaming
- [ ] build history page
- [ ] custom icon injection
- [ ] production signing support

---

built by [YallCode](https://yallcode.github.io/YallaYCode/)

[github](https://github.com/yallcode) · [twitter](https://twitter.com/YallCode) · [youtube](https://youtube.com/@YallaYCode) · [discord](https://discord.gg/yUe8kE5fRF)
