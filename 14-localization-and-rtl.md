# Phase 14 — Localization & RTL

**Tags:** `[Backend]` `[Frontend]`
**Depends on:** Phase 6 (frontend shell)
**You'll finish this phase with:** the app fully usable in English and Arabic, with genuine
right-to-left layout (not mirrored text in a left-to-right layout), and backend validation
messages/error text translated too.

---

## 1. Learning objectives

- Understand that **RTL is a layout direction, not a font setting** — `margin-left` and
  `margin-right` are wrong concepts in RTL; "logical" CSS properties fix this permanently.
- Set up translation files on both backend and frontend that stay in sync conceptually (same
  keys, two systems).

## 2. Why this phase now

The UI (Phases 6–10) and reports exist now, so there's real content to translate. Doing this
before the tax/cheque phases matters because Egyptian cheques and ETA documents have Arabic-first
requirements — better to have the i18n plumbing solid before those phases need it.

---

## 3. Backend: Laravel lang files

```powershell
php artisan lang:publish
```
`backend/lang/en/validation.php` and `backend/lang/ar/validation.php` — Laravel ships English by
default; add an Arabic file with the same keys translated. For custom messages (like Phase 4's
`UnbalancedJournalEntryException`), use `__()`:
```php
throw new UnbalancedJournalEntryException(__('accounting.unbalanced_entry', [
    'debit' => $totalDebit, 'credit' => $totalCredit,
]));
```
`backend/lang/en/accounting.php`:
```php
return ['unbalanced_entry' => 'Journal entry does not balance: debits :debit != credits :credit'];
```
`backend/lang/ar/accounting.php`:
```php
return ['unbalanced_entry' => 'القيد غير متوازن: المدين :debit لا يساوي الدائن :credit'];
```
Set the locale per-request from a header the frontend sends:
```php
// A middleware, registered on the v1 group
class SetLocale
{
    public function handle(Request $request, Closure $next)
    {
        App::setLocale($request->header('X-Locale', 'en'));
        return $next($request);
    }
}
```

---

## 4. Frontend: vue-i18n

```powershell
cd frontend
npm install vue-i18n
```

`frontend/src/i18n/index.js`:
```javascript
import { createI18n } from 'vue-i18n'
import en from './locales/en.json'
import ar from './locales/ar.json'

export const i18n = createI18n({
  legacy: false, // required to use the Composition API's useI18n()
  locale: localStorage.getItem('locale') || 'en',
  fallbackLocale: 'en',
  messages: { en, ar },
})
```
`frontend/src/i18n/locales/en.json`:
```json
{ "nav": { "dashboard": "Dashboard", "accounts": "Chart of Accounts" }, "journal_entry": { "debit": "Debit", "credit": "Credit" } }
```
`frontend/src/i18n/locales/ar.json`:
```json
{ "nav": { "dashboard": "لوحة التحكم", "accounts": "دليل الحسابات" }, "journal_entry": { "debit": "مدين", "credit": "دائن" } }
```
Register in `main.js`: `app.use(i18n)`.

Use in components:
```vue
<script setup>
import { useI18n } from 'vue-i18n'
const { t } = useI18n()
</script>
<template>
  <h1>{{ t('nav.accounts') }}</h1>
</template>
```

Send the chosen locale to the backend, updating Phase 3's `api.js` interceptor:
```javascript
api.interceptors.request.use((config) => {
  config.headers['X-Locale'] = localStorage.getItem('locale') || 'en'
  // ...existing Authorization header logic
  return config
})
```

---

## 5. RTL: the part that actually breaks layouts

**Wrong approach:** leaving CSS as `margin-left: 16px` and just flipping `<html dir="rtl">` — text
direction flips, but padding/margins/icons stay on the visually wrong side.

**Correct approach:** use CSS **logical properties**, which mean "start of reading direction" and
"end of reading direction" instead of physical left/right:
```css
/* Instead of: */
.card { margin-left: 16px; padding-right: 8px; text-align: left; }

/* Use: */
.card { margin-inline-start: 16px; padding-inline-end: 8px; text-align: start; }
```
With logical properties, the *same* CSS automatically renders correctly in both directions — no
`[dir="rtl"] .card { margin-right: 16px; }` override rules needed anywhere.

Toggle direction based on locale, in `App.vue` or a layout root:
```vue
<script setup>
import { watch } from 'vue'
import { useI18n } from 'vue-i18n'
const { locale } = useI18n()
watch(locale, (newLocale) => {
  document.documentElement.dir = newLocale === 'ar' ? 'rtl' : 'ltr'
  document.documentElement.lang = newLocale
  localStorage.setItem('locale', newLocale)
}, { immediate: true })
</script>
```

**One accounting-specific RTL gotcha:** numbers and monetary amounts should generally stay
left-to-right even in an RTL layout (an Arabic reader still expects "48,500.00 ج.م" with digits in
their normal order). Wrap numeric output in `<bdi>` (bidirectional isolate) or apply
`direction: ltr; unicode-bidi: isolate;` specifically to amount cells in tables — otherwise
numbers can visually reorder inside RTL text in a way that's actively wrong for anything
financial.

---

## 6. What you built, and why it matters later

- **Logical CSS properties as a standing rule** for every component written from here on —
  cheaper to apply as you go than to retrofit across 15 phases' worth of components later.
- A locale-aware API (`X-Locale` header) so that Phase 15's tax-authority error messages and
  Phase 19's AI assistant responses can also be returned in the user's language, using the same
  mechanism.

## 7. Next: Phase 15

With multi-language support in place, Phase 15 builds the actual reason this app needs to be
correct and auditable: submitting real e-Invoice and e-Receipt documents to Egypt's tax authority.
