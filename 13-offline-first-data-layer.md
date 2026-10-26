# Phase 13 — Offline-First Data Layer

**Tags:** `[Frontend]` `[Backend]`
**Depends on:** Phase 12 (service worker registered), Phase 7/8 (things worth entering offline)
**You'll finish this phase with:** the ability to create a journal entry or invoice with zero
connection, see it locally immediately, and have it sync to the server automatically (with
conflict handling) once the connection returns.

---

## 1. Learning objectives

- Understand **IndexedDB** as a real, transactional, in-browser database — not just a bigger
  localStorage.
- Build an **outbox pattern**: writes go to a local queue first, a background process replays
  them against the real API.
- Understand why conflict resolution is a genuine design decision, not an edge case to ignore.

## 2. Why this phase now, and why it's harder than Phase 12

Phase 12 made *reading* cached data tolerate a bad connection. This phase makes *writing* new
data possible with **no** connection at all — which means the data has to live somewhere durable
in the browser before the server has ever seen it, and the app needs a plan for "what if this
same record changed on the server in the meantime."

---

## 3. Install Dexie

```powershell
cd frontend
npm install dexie
```

`frontend/src/offline/db.js`:
```javascript
import Dexie from 'dexie'

// A local, in-browser database — survives closing the tab, closing the app,
// even losing the network entirely. Structurally: one table per thing you can
// create offline, plus one "outbox" table of pending server writes.
export const localDb = new Dexie('eaccounting-offline')

localDb.version(1).stores({
  // '++id' = auto-incrementing LOCAL id, separate from the server's real id
  // until sync assigns one — this is the crux of the whole offline design.
  draftJournalEntries: '++localId, entry_date, syncStatus',
  draftInvoices: '++localId, issue_date, syncStatus',
  outbox: '++id, entityType, localId, createdAt',
})
```

*Why a separate `localId` instead of reusing the server's `id`:* when created offline, there IS
no server id yet. Every offline-capable record needs its own identity until sync assigns the real
one — trying to force server IDs onto offline-created records is the single most common design
mistake in "add offline mode" projects, and untangling it later is painful. Design for it from the
start, as we're doing here.

---

## 4. Writing offline: the outbox pattern

`frontend/src/offline/offlineJournalEntryService.js`:
```javascript
import { localDb } from './db'
import api from '../services/api'

export async function createJournalEntry(payload) {
  // 1. Always write locally FIRST — this must never fail due to network.
  const localId = await localDb.draftJournalEntries.add({
    ...payload,
    syncStatus: 'pending',
    createdAt: Date.now(),
  })

  // 2. Queue the "real" server write for whenever we're online.
  await localDb.outbox.add({
    entityType: 'journalEntry',
    localId,
    payload,
    createdAt: Date.now(),
  })

  // 3. Try immediately, in case we're actually online right now —
  //    but the UI never waits on this; the record already "exists" locally.
  if (navigator.onLine) syncOutbox()

  return { localId, ...payload, syncStatus: 'pending' }
}

export async function syncOutbox() {
  const pending = await localDb.outbox.orderBy('createdAt').toArray()

  for (const item of pending) {
    try {
      if (item.entityType === 'journalEntry') {
        const { data } = await api.post('/journal-entries', item.payload)
        await localDb.draftJournalEntries.update(item.localId, {
          syncStatus: 'synced',
          serverId: data.data.id,
        })
      }
      // Only remove from the outbox once the server confirmed success —
      // if this throws, the item stays queued and we retry next time.
      await localDb.outbox.delete(item.id)
    } catch (error) {
      if (error.status === 422) {
        // The server rejected it outright (e.g. an unbalanced entry that
        // somehow got past client checks) — retrying won't help. Flag it
        // for the user instead of retrying forever.
        await localDb.draftJournalEntries.update(item.localId, { syncStatus: 'error' })
        await localDb.outbox.delete(item.id)
      }
      // Any other error (network dropped mid-sync) — leave it queued, try again later.
    }
  }
}

// Fire automatically whenever the browser regains connectivity.
window.addEventListener('online', syncOutbox)
```

---

## 5. Conflict handling — the part most tutorials skip

Two real conflict scenarios exist in this app, and they need **different** answers:

**Scenario A — a brand-new record created offline.** There's no conflict possible; nobody else
could have touched something that didn't exist on the server yet. The `syncOutbox()` flow above
handles this correctly as-is.

**Scenario B — editing something that existed before going offline** (a later feature, e.g.
editing a draft invoice). Here, someone else (or you, on another device) might have changed the
same record on the server while you were offline. Our answer, deliberately simple and safe rather
than clever:

> **Never silently overwrite.** If the server's `updated_at` for a record is newer than the
> timestamp this device last saw for it, do NOT push the local change automatically. Mark it
> `syncStatus: 'conflict'` and surface it to the user to review both versions and choose — for
> accounting data, a silent "last write wins" merge can quietly lose a transaction, which is
> unacceptable. A little manual friction here is the correct trade-off.

```javascript
// Inside syncOutbox(), before pushing an EDIT (not a create):
const server = await api.get(`/journal-entries/${item.serverId}`)
if (new Date(server.data.data.updated_at) > new Date(item.lastSeenServerUpdate)) {
  await localDb.draftJournalEntries.update(item.localId, { syncStatus: 'conflict' })
  continue // skip pushing; wait for the user
}
```

---

## 6. Surfacing sync status in the UI

`frontend/src/components/SyncStatusBadge.vue` — a small badge next to any record showing
`pending` (local only, waiting), `synced` (confirmed on server), `error` (server rejected it), or
`conflict` (needs the user's decision). Reuses the exact `ConnectionStatus.vue` pattern from
Phase 12, extended to also show "3 pending sync" counts by querying
`localDb.outbox.count()` reactively.

---

## 7. What you built, and why it matters later

- A genuine **offline-first** write path — the second of the two depths you asked for, now fully
  distinct from Phase 12's read-only caching.
- A conflict policy that's **conservative on purpose** — the same "never silently overwrite
  financial history" instinct as Phase 4's immutable-once-posted rule, applied to the sync layer.
- This exact outbox pattern is what Phase 19's AI agent should also respect when acting on a
  device that might be offline — actions queue and confirm, they don't assume instant success.

## 8. Next: Phase 14

With both PWA depths built, Phase 14 adds real localization — multiple languages and genuine
Arabic RTL layout, not just translated strings.
