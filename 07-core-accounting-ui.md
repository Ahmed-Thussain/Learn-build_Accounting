# Phase 7 — Core Accounting UI

**Tags:** `[Frontend]` `[API]`
**Depends on:** Phase 6 (shell), Phase 5 (API)
**You'll finish this phase with:** a working Chart of Accounts screen and a journal entry form
with live, client-side debit/credit balance checking — the first screens a real user could use.

---

## 1. Learning objectives

- Build a form that mirrors a **backend invariant** (debits = credits) on the client, for instant
  feedback — while remembering the server check (Phase 5) is the one that actually can't be
  bypassed.
- Practice a **dynamic list of form rows** (add/remove journal entry lines) in Vue 3.

## 2. Why this phase now

Phase 5/6 gave us API + shell but nothing a user would actually look at. This is the first phase
that produces a screen you'd show someone.

---

## 3. Chart of Accounts screen

Backend: add the missing CRUD route (Phase 5's pattern applied to `Account`):

```php
Route::apiResource('accounts', AccountController::class);
```

`AccountController` mirrors `JournalEntryController`'s shape (`index`, `store`, using a
`StoreAccountRequest` and `AccountResource`) — deliberately not repeated here in full, since it's
the exact same template from Phase 5.

Frontend, `frontend/src/views/AccountsView.vue`:

```vue
<script setup>
import { onMounted, computed } from 'vue'
import { useAccountStore } from '../stores/accountStore'

const store = useAccountStore()
onMounted(() => store.fetchAll())

// Group by type so the screen reads like a real chart of accounts
// (Assets, Liabilities, Equity, Revenue, Expenses), not a flat table.
const grouped = computed(() => {
  const groups = { asset: [], liability: [], equity: [], revenue: [], expense: [] }
  for (const account of store.items) groups[account.type].push(account)
  return groups
})
</script>

<template>
  <h1>Chart of Accounts</h1>
  <section v-for="(accounts, type) in grouped" :key="type">
    <h2>{{ type }}</h2>
    <ul>
      <li v-for="account in accounts" :key="account.id">
        {{ account.code }} — {{ account.name }}
      </li>
    </ul>
  </section>
</template>
```

---

## 4. The journal entry form — the interesting part

`frontend/src/stores/journalEntryStore.js`:

```javascript
import { defineStore } from 'pinia'
import api from '../services/api'

export const useJournalEntryStore = defineStore('journalEntries', {
  state: () => ({ items: [], loading: false }),
  actions: {
    async fetchAll() {
      const { data } = await api.get('/journal-entries')
      this.items = data.data
    },
    async create(payload) {
      const { data } = await api.post('/journal-entries', payload)
      this.items.unshift(data.data)
      return data.data
    },
    async post(id) {
      const { data } = await api.post(`/journal-entries/${id}/post`)
      const index = this.items.findIndex((e) => e.id === id)
      if (index !== -1) this.items[index] = data.data
    },
  },
})
```

`frontend/src/components/JournalEntryForm.vue`:

```vue
<script setup>
import { ref, computed } from 'vue'
import { useAccountStore } from '../stores/accountStore'
import { useJournalEntryStore } from '../stores/journalEntryStore'

const accountStore = useAccountStore()
const entryStore = useJournalEntryStore()

const entryDate = ref(new Date().toISOString().slice(0, 10))
const description = ref('')
// Start with the two lines every entry needs at minimum.
const lines = ref([
  { account_id: null, debit: 0, credit: 0 },
  { account_id: null, debit: 0, credit: 0 },
])
const error = ref(null)

function addLine() {
  lines.value.push({ account_id: null, debit: 0, credit: 0 })
}
function removeLine(index) {
  if (lines.value.length > 2) lines.value.splice(index, 1)
}

// The exact same rule as PostingService (Phase 4) and StoreJournalEntryRequest
// (Phase 5), computed here purely for INSTANT feedback — this client-side check
// is a UX convenience, never the security boundary.
const totalDebit = computed(() => lines.value.reduce((sum, l) => sum + Number(l.debit || 0), 0))
const totalCredit = computed(() => lines.value.reduce((sum, l) => sum + Number(l.credit || 0), 0))
const isBalanced = computed(() => Math.abs(totalDebit.value - totalCredit.value) < 0.005)

async function submit() {
  error.value = null
  try {
    await entryStore.create({
      entry_date: entryDate.value,
      description: description.value,
      lines: lines.value,
    })
  } catch (e) {
    error.value = e.message
  }
}
</script>

<template>
  <form @submit.prevent="submit">
    <input v-model="entryDate" type="date" required />
    <input v-model="description" placeholder="Description" />

    <table>
      <thead><tr><th>Account</th><th>Debit</th><th>Credit</th><th></th></tr></thead>
      <tbody>
        <tr v-for="(line, i) in lines" :key="i">
          <td>
            <select v-model="line.account_id" required>
              <option v-for="a in accountStore.items" :key="a.id" :value="a.id">
                {{ a.code }} — {{ a.name }}
              </option>
            </select>
          </td>
          <td><input v-model.number="line.debit" type="number" step="0.01" min="0" /></td>
          <td><input v-model.number="line.credit" type="number" step="0.01" min="0" /></td>
          <td><button type="button" @click="removeLine(i)">Remove</button></td>
        </tr>
      </tbody>
    </table>

    <button type="button" @click="addLine">+ Add line</button>

    <p :class="{ balanced: isBalanced, unbalanced: !isBalanced }">
      Debit: {{ totalDebit.toFixed(2) }} — Credit: {{ totalCredit.toFixed(2) }}
    </p>

    <button type="submit" :disabled="!isBalanced">Save as draft</button>
    <p v-if="error" role="alert">{{ error }}</p>
  </form>
</template>
```

---

## 5. What you built, and why it matters later

- A form where the **UI enforces the same invariant as the backend**, so users get instant
  feedback instead of a round-trip error — but the disabled-submit-button check is *convenience*,
  not security; someone could still POST an unbalanced payload directly, which is exactly why
  Phase 4/5's server-side checks exist regardless.
- A store/component split that Phase 8 (Invoicing) copies directly — an invoice form is, under
  the hood, "create a journal entry with specific accounts pre-filled."

## 6. Next: Phase 8

With manual journal entries working, Phase 8 builds Invoicing and AR/AP — the layer that lets a
non-accountant user "create an invoice" while the system generates the correct journal entry
underneath, using `PostingService` from Phase 4.
