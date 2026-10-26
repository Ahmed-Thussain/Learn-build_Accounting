# Phase 6 — Frontend Foundation

**Tags:** `[Frontend]`
**Depends on:** Phase 5 (a real API to talk to)
**You'll finish this phase with:** an app shell — layout, navigation, router guards, a Pinia
store pattern — that Phase 7's actual accounting screens slot into.

---

## 1. Learning objectives

- Structure a Vue app so that **adding a new page never means touching five unrelated files**.
- Understand why route guards and a persistent layout matter for a business app (vs. a marketing
  site).

## 2. Why this phase now

We have working auth (Phase 3) and a working API (Phase 5) but no real app shell — everything so
far lived in one `App.vue`. Before writing the Chart of Accounts screen, invoice screen, and
report screens in the phases that follow, they need somewhere consistent to live.

---

## 3. Routes and layout

`frontend/src/router/index.js`:
```javascript
import { createRouter, createWebHistory } from 'vue-router'
import { useAuthStore } from '../stores/authStore'

const routes = [
  { path: '/login', name: 'login', component: () => import('../views/LoginView.vue') },
  {
    path: '/',
    component: () => import('../layouts/AppLayout.vue'),
    meta: { requiresAuth: true },
    children: [
      { path: '', name: 'dashboard', component: () => import('../views/DashboardView.vue') },
      { path: 'accounts', name: 'accounts', component: () => import('../views/AccountsView.vue') },
      { path: 'journal-entries', name: 'journal-entries', component: () => import('../views/JournalEntriesView.vue') },
    ],
  },
]

const router = createRouter({ history: createWebHistory(), routes })

router.beforeEach((to) => {
  const auth = useAuthStore()
  if (to.meta.requiresAuth && !auth.token) return { name: 'login' }
  if (to.name === 'login' && auth.token) return { name: 'dashboard' }
})

export default router
```

*Why lazy-loaded components (`() => import(...)`)*: each page's code only downloads when a user
navigates to it, instead of the whole app loading in one bundle — this matters more, not less,
once the app is a PWA users expect to open instantly (Phase 12).

`frontend/src/layouts/AppLayout.vue`:
```vue
<script setup>
import { useAuthStore } from '../stores/authStore'
const auth = useAuthStore()
</script>

<template>
  <div class="app-shell">
    <nav>
      <router-link :to="{ name: 'dashboard' }">Dashboard</router-link>
      <router-link :to="{ name: 'accounts' }">Chart of Accounts</router-link>
      <router-link :to="{ name: 'journal-entries' }">Journal Entries</router-link>
      <button @click="auth.logout()">Log out</button>
    </nav>
    <main>
      <!-- Every child route renders here — this IS the "one shell, many pages" pattern -->
      <router-view />
    </main>
  </div>
</template>
```

---

## 4. The store pattern, generalized

Phase 3 wrote `authStore.js` directly against the API. From here on, every domain store follows
this shape: state → an `api`-backed action per verb → nothing else. Example,
`frontend/src/stores/accountStore.js` (Phase 7 fills this in fully; this is the skeleton):
```javascript
import { defineStore } from 'pinia'
import api from '../services/api'

export const useAccountStore = defineStore('accounts', {
  state: () => ({
    items: [],
    loading: false,
  }),
  actions: {
    async fetchAll() {
      this.loading = true
      try {
        const { data } = await api.get('/accounts')
        this.items = data.data // Laravel Resource collections wrap in { data: [...] }
      } finally {
        this.loading = false
      }
    },
  },
})
```

*Why every store looks the same:* once you (or the AI assistant, generating admin screens later)
know this one pattern, every new domain object — invoices, contacts, cheque templates — is a
copy-paste-and-rename away from a working store, not a fresh design decision each time.

---

## 5. A minimal login view (wires Phase 3's store to a real form)

`frontend/src/views/LoginView.vue`:
```vue
<script setup>
import { ref } from 'vue'
import { useRouter } from 'vue-router'
import { useAuthStore } from '../stores/authStore'

const email = ref('')
const password = ref('')
const error = ref(null)
const auth = useAuthStore()
const router = useRouter()

async function submit() {
  error.value = null
  try {
    await auth.login(email.value, password.value)
    router.push({ name: 'dashboard' })
  } catch (e) {
    error.value = e.message
  }
}
</script>

<template>
  <form @submit.prevent="submit">
    <input v-model="email" type="email" placeholder="Email" required />
    <input v-model="password" type="password" placeholder="Password" required />
    <button type="submit">Log in</button>
    <p v-if="error" role="alert">{{ error }}</p>
  </form>
</template>
```

---

## 6. What you built, and why it matters later

- A **shell + child routes** structure — Phase 7 through Phase 10 each just add one more child
  route and one more store, never touch the router's guard logic or the layout again.
- A **standard store shape** that Phase 13's offline queue plugs into consistently (it wraps this
  same `fetchAll`/action pattern rather than replacing it).

## 7. Next: Phase 7

With the shell in place, Phase 7 builds the first genuinely useful screens: Chart of Accounts
management and the journal entry form, including live client-side balance checking before submit.
