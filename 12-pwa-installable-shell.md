# Phase 12 — PWA Basics: Installable Shell

**Tags:** `[Frontend]`
**Depends on:** Phase 6 (frontend shell)
**You'll finish this phase with:** the Vue app installable on the desktop ("Add to Home
Screen"/desktop icon), with a service worker caching the app shell so it opens instantly and
tolerates a flaky connection — still assuming you're online for real data.

---

## 1. Learning objectives

- Understand what a **service worker** actually is: JavaScript the browser runs in the
  background, able to intercept network requests, independent of any open tab.
- Understand the difference between caching the **app shell** (HTML/CSS/JS — safe to cache
  aggressively) and caching **API data** (must stay fresh — cache cautiously, if at all, until
  Phase 13).
- Learn the manifest fields that make "Add to desktop" actually appear.

## 2. Why this phase now (and why it's simpler than Phase 13)

You asked for both the simple installable-shell version and the full offline-entry version. This
phase is deliberately the simple one: the app **looks** installed and native, and static assets
load instantly from cache, but journal entries and invoices still require a live connection.
Phase 13 is where entering data offline becomes possible — don't skip ahead; this phase's
service-worker registration is what Phase 13 builds its sync logic on top of.

---

## 3. Install the PWA plugin

```powershell
cd frontend
npm install -D vite-plugin-pwa
```

* [ ] frontend/vite.config.js`:

```javascript
import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'
import { VitePWA } from 'vite-plugin-pwa'

export default defineConfig({
  plugins: [
    vue(),
    VitePWA({
      registerType: 'autoUpdate',
      manifest: {
        name: 'E-Accounting',
        short_name: 'E-Accounting',
        description: 'Cloud & offline accounting for MENA businesses',
        theme_color: '#0f172a',
        background_color: '#ffffff',
        display: 'standalone', // no browser chrome — looks like a native app once installed
        icons: [
          { src: 'icon-192.png', sizes: '192x192', type: 'image/png' },
          { src: 'icon-512.png', sizes: '512x512', type: 'image/png' },
        ],
      },
      workbox: {
        // Cache-first for the app's own built files — they only change on a new
        // deploy, and Workbox handles busting the cache when they do.
        runtimeCaching: [
          {
            urlPattern: /^https:\/\/fonts\.googleapis\.com\/.*/i,
            handler: 'CacheFirst',
            options: { cacheName: 'google-fonts-cache' },
          },
          {
            // API calls: network-first. Try the real server; only fall back to
            // whatever was last cached if the network genuinely fails. This is
            // NOT the same as Phase 13's offline writes — this only helps
            // re-showing already-seen data when briefly offline.
            urlPattern: /^http:\/\/127\.0\.0\.1:8000\/api\/.*/i,
            handler: 'NetworkFirst',
            options: { cacheName: 'api-cache', networkTimeoutSeconds: 3 },
          },
        ],
      },
    }),
  ],
})
```

Add real `icon-192.png` and `icon-512.png` files to `frontend/public/` — any square PNG at those
sizes; a placeholder logo is fine for now.

---

## 4. Build and test as a real PWA

Service workers only activate on a **production build** (not `npm run dev`):

```powershell
npm run build
npm run preview
```

Open the preview URL in Chrome/Edge. You should see an install icon (⊕) in the address bar. Click
it — the app opens in its own window, no browser tabs/address bar, with its own taskbar icon.
Open DevTools → Application tab → Service Workers, to confirm one is registered and "activated."

---

## 5. A visible online/offline indicator

Small, but important for trust — a user should always know which mode they're in, even before
Phase 13 makes offline mode do anything useful.

`frontend/src/components/ConnectionStatus.vue`:

```vue
<script setup>
import { ref, onMounted, onUnmounted } from 'vue'
const isOnline = ref(navigator.onLine)
function update() { isOnline.value = navigator.onLine }
onMounted(() => {
  window.addEventListener('online', update)
  window.addEventListener('offline', update)
})
onUnmounted(() => {
  window.removeEventListener('online', update)
  window.removeEventListener('offline', update)
})
</script>

<template>
  <span :class="isOnline ? 'status-online' : 'status-offline'">
    {{ isOnline ? 'Online' : 'Offline' }}
  </span>
</template>
```

Drop it into `AppLayout.vue`'s nav bar (Phase 6).

---

## 6. What you built, and why it matters later

- The app is genuinely **installable** — a real, testable deliverable you asked for as one of the
  two offline "methods."
- The service worker registration and `online`/`offline` browser events are the exact
  infrastructure Phase 13 builds real offline writes on top of — nothing here gets thrown away.

## 7. Next: Phase 13

Now the deeper version: full offline data entry. A user creates a journal entry or invoice with no
connection at all, it's stored locally, and syncs automatically once back online.
