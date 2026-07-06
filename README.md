# Expense Tracker

A single-file, browser-based expense tracker built for a deliberate, planner-driven approach to personal spending. It goes well beyond logging purchases: it models *planned* spending, gamified "bounties," indulgence allowances, and generates journaling hints designed to be transcribed into a paper [Hobonichi Weeks](https://www.1101.com/store/techo/en/) planner.

The entire app is one `index.html` file — vanilla HTML/CSS/JavaScript with no build step — backed by [Supabase](https://supabase.com) for cross-device sync and authentication, with a `localStorage` fallback for offline use.

---

## Highlights

- **Four-tab workspace** — Add expense, Expense log, Summary, and an in-app Reference.
- **Rich entry taxonomy** — ordinary expenses, recurring subscriptions, gifts, *planned* expenses, five indulgence slots, and three tiers of "bounty."
- **Planning workflow** — planned and bounty entries carry a status lifecycle (in progress → greenlit → executed) plus planned/actual dates.
- **Hobonichi journaling hints** — every entry shows a transcription checkbox; planning entries also show a day-of-week + day-number page hint so you know which planner page to write on.
- **Sticker placement logic** — built-in rules for placing physical bounty and indulgence stickers in the planner.
- **Drag-and-drop ordering** (synced across devices), collapsible month/week grouping, and dual sort modes (by date logged vs. by planned date). In *By date logged*, weeks stay chronological but each week internally surfaces ☑️ executed planning entries first, then regular expenses (newest first), then ⏳ open planning entries at the bottom; priority outranks dragging (drags reorder within a priority group).
- **Summary dashboards** — breakdowns by instance, category, and status, with bar and pie charts.
- **Searchable log** — a global search box on top of the instance/category/status filters, with a one-tap **Reset filters**. Filters decide what's shown; search only drills down within them.
- **Duplicate as new** — a ⧉ button on each log card opens a pre-filled Add draft (description, full price, category, location) as a fresh **Expense**, so repeat purchases take seconds. Metadata and discounts deliberately reset; the date defaults to today.
- **Slash commands & autofill** — type `/gas`, `/claude`, `/icloud`, etc. to prefill recurring entries.
- **Arithmetic in $ fields** — Full price, Amount paid, and split amount accept quick calculations (`+ - * /`, parentheses), evaluated on blur/submit. e.g. `3 * 4.99` or `12.50 + 3.99`.
- **Discover 🟠 cashback tracker** — a Reference-tab table for the quarterly 5% categories, synced across devices and auto-cleared each new year.
- **🚩 Flagging** — type `/flag` (or the emoji) in a description to mark an entry as incomplete. Flagged cards turn light red in the log and their journal/transcription checkbox is blocked until you clear the flag via the per-card **Reviewed?** control.
- **Manage saved locations** — a panel in the Reference tab to rename or remove remembered locations. Renaming/removing also propagates to the entries that use the location (with a confirmation count), so it never silently reappears.
- **Unsaved draft recovery** — a half-filled Add-expense form is saved continuously and restored on reload or app switch (new entries only), cleared on Add or Clear. Works on mobile and desktop.
- **Cross-device sync** via Supabase with email/password auth and row-level security; falls back to `localStorage` when offline. A rolling 72-hour sign-in window and an offline banner cover iOS PWA token purges (see [Sign-in & sync](#sign-in--sync)).

---

## Tech stack

| Layer | Choice |
| --- | --- |
| Frontend | Plain HTML + CSS + JavaScript (no framework, no bundler) |
| Persistence | Supabase (Postgres + Auth + RLS) |
| Offline cache | Browser `localStorage` (`expenses_v9`) |
| Charts | Hand-rolled SVG / CSS bars and pies (no chart library) |
| Hosting | Any static host — open the file locally, or serve it from GitHub Pages, Netlify, etc. |

The Supabase client is loaded from a CDN, so the only external dependency at runtime is `@supabase/supabase-js`.

---

## Data model

Each expense is a plain JavaScript object. Common fields:

| Field | Meaning |
| --- | --- |
| `id` | Unique id (`Date.now()` at creation) |
| `desc` | Description |
| `full` | Full price (0 allowed for not-yet-priced planning entries) |
| `instance` | Entry type (see taxonomy below) |
| `category` | Spending category |
| `location` | Optional vendor / place |
| `date` | Primary date (date logged for planning entries, purchase date otherwise) |

Planning entries (planned expenses and bounties) add:

| Field | Meaning |
| --- | --- |
| `dateLogged` | When you committed to the plan |
| `plannedDate` | Target execution date |
| `plannedTime` | Optional time of day |
| `status` | `inprogress`, `greenlit`, `executed`, or `canceled` |
| `actualDate` | Set only when status is `executed` |
| `indulgenceTag` | Optional secondary tag (planned entries support all indulgence tags; bounties support only `gift`) |

Optional extras on any entry: `recipientNote` (for gifts) and `split` (`{ direction, amount, with, via }`) for shared costs.

### Entry taxonomy (`instance`)

| Instance | Label | Emoji |
| --- | --- | --- |
| `expense` | Expense | 💵 |
| `subscription` | Subscription / Recurring | 🌙 |
| `gift` | Gift | 🎁 |
| `planned` | Planned expense | 🪙 |
| `indulgence1`–`indulgence5` | Indulgence 1–5 | 🪶 🪷 🫧 🍎 ✨ |
| `bounty` | Bounty (≥ $50) | 🍲 |
| `bigbounty` | Big bounty (≥ $100) | 🎆 |
| `megabounty` | Mega bounty (≥ $250) | 🏯 |

Bounty tiers enforce a price floor: a bounty must have a full price ≥ $50, big bounty ≥ $100, and mega bounty ≥ $250.

### Categories

Food · Vehicle and Transportation · Miscellaneous · Home and Housing · Health, Medical, and Personal Care · Clothing · Utilities. Each has a fixed accent color used throughout the badges and charts.

---

## Bill splitting & net spend

Any entry can be flagged as split, recording a `direction`, `amount`, `with`, and `via`. Splits feed into the Summary's spend rollups through a single `netPaid(e)` helper, so the headline numbers reflect what you actually bore rather than the gross bill:

- **No split** → `paid` is used as-is.
- **"Others paid me"** → `paid − amount`, floored at 0. You fronted the gross bill and a friend reimbursed their share, so the reimbursement is subtracted. (Sushi lunch: `128.72 − 65.00 = 63.72`.)
- **"I paid others"** → your share `amount` when recorded, otherwise `paid`. This covers both logging styles: entering only your portion as the main amount, *or* entering the gross bill and putting your portion in the share field. Either way the total reflects your share.

Which numbers this affects: **Total spent** (the donut centers) and the **category** and **instance** breakdowns all use `netPaid`. **Total saved** and **Full price value** deliberately stay gross — they measure discounts, a separate axis from reimbursement, and netting them would conflate the two. One consequence: if you log the *gross* bill in the main amount on a split entry, Full price value will show that gross figure for the entry, not your share.

---

## The planning & journaling system

This is the part that makes the app unusual. It's designed to pair a digital ledger with a physical planner.

### Status lifecycle

Planned expenses and bounties move through statuses: **In progress → Greenlit → Executed** (or **Canceled**). When an entry is marked Executed, you record the `actualDate` it actually happened.

### Returning to a planned entry

When you're adding an entry with a planning instance (Planned, Bounty, Big/Mega Bounty) and the description exactly matches an existing **pending** planning entry, a small picker appears under the Description field listing the match(es). Click a row to load that entry into the form and continue it (e.g. to mark it Executed). Nothing loads automatically — the form is only overwritten when you click a row, so typing a description that happens to collide with a past entry never hijacks a new one. Executed and canceled entries are deliberately excluded from the picker, since there's nothing left to act on.

### Date restrictions

To keep the ledger honest, new entries are date-restricted:

- **New non-planning entries:** current calendar month, plus a 7-day grace window back into the previous month at the start of each month.
- **New planned/bounty "date logged":** current month + 7-day grace; the planned date itself can be anywhere from today into the future.
- **Editing existing entries:** the picker relaxes back to the first day of the month containing the entry's earliest date. A bounty's "date logged" stays locked once set.
- Logging anything more than 7 days in the past raises an overridable retroactive warning.

### Journal page hints

Everything is transcribed onto the **memo side** of the Hobonichi Weeks spread, and a checkbox on each card tracks whether you've done that transcription yet. The label beside the checkbox depends on the entry type:

- **Normal (non-planning) expenses** → a checkbox labeled **`Transcribe to Memo`**. Tick it once it's written into the memo.
- **Open / in-progress planning entry** → `⏳ FRI 22` — the hourglass plus day-of-week and day number tell you which day's page the entry maps to.
- **Executed planning entry** → `☑️ FRI 22` — same day-of-week + day-number format, hourglass swapped for a checkmark, using the `actualDate`.
- **Open bounties** → `⏳ MON 1` — bounties roll forward to the 1st of the current month for a monthly check-in. On the first open of a new month, an open bounty whose planned execution date now sits in a past month has that date (and time) cleared and a 🚩 auto-appended (`applyBountyMonthReset`), forcing a review: the red card and blocked transcription checkbox persist until the entry is revisited, and re-saving it requires choosing a new planned date. The reset is self-limiting (a cleared date can't re-trigger; an already-flagged entry isn't double-flagged) and, like the Discover year clear, runs on open rather than at midnight.

When the checkbox appears:

- Non-planning instances: always.
- Open bounties: always (rolling forward to the 1st each month).
- Other planning instances: when a planned date or actual date exists.

Checking the box dims the card and persists across sessions; open bounties reset their recorded state at month boundaries.

### Flagging incomplete entries 🚩

Logging fast means sometimes leaving an entry half-finished. Type `/flag` anywhere in the description (it expands to 🚩 inline, while creating *or* editing), or just type the emoji. Any entry whose description contains a 🚩:

- turns **light red** in the Expense log (`#FDEEEB` fill, soft `#FA9F8C` border) as a "fix me" signal;
- shows a **Reviewed?** checkbox at the bottom-left of the card — ticking it strips the 🚩, restores the normal background, and removes the control;
- has its **journal/transcription checkbox disabled** until the flag is cleared, so incomplete entries can't be marked transcribed into the planner.

A flagged entry never counts as recorded. The 🚩 is just text in the description, so it has no effect on amounts, categories, or any Summary figure.

### Sticker placement

Physical sticker inventory is one bounty sticker per tier per month (3 total) plus 5 indulgence stickers per month. Placement follows the page hint:

- **Bounty stickers** go on whichever day the hint indicates *at transcription time*. While open that's the 1st (⏳); once executed the hint switches to ☑️ on the `actualDate`.
- **Indulgence stickers** go on the planned date while open (⏳), then on the `actualDate` once executed (☑️).
- The symbol is the cue: ⏳ means "still open," ☑️ means "executed — transcribe the outcome."

---

## Productivity shortcuts

### Slash commands

Type one of these in the Description field to prefill an entry (auxiliary fields only fill if currently empty, so manual input is never overwritten):

`/gas` · `/sonic` · `/trak` · `/narwhal` · `/allstate` · `/icloud` · `/claude`

### Autofill rules

Typing certain keywords (e.g. "gasoline", "sonic fiber", "fastrak") auto-suggests the matching instance and category.

### Savings-field shortcuts

In the **Savings details** field:

- `/dsc`, `/chs`, `/ven` expand to labeled icons (Discover Cashback, Chase Ultimate Reward, Venmo Balance).
- `/delta` does balance arithmetic in two directions, each triggered by a trailing space:
  - **Forward** — `$X.XX /delta ` → `$X.XX Δ $Y.YY `, where `Y.YY = X.XX − (amount paid, or full price if paid is 0/empty)`. Reads as "balance went from X to Y" (e.g. remaining gift-card balance after a purchase).
  - **Backward** — `/delta $Y.YY ` → `$X.XX /delta $Y.YY `, where `X.XX = Y.YY + amount paid` (fallback: full price). True inverse of forward — since forward subtracts paid to find the remaining balance, backward adds paid back to recover the original. Only fires when amount paid (or full price) is set.

  Note the two use different operands by design — forward subtracts *amount paid*, backward adds *you saved* — so they are true inverses only when the credit/stored value applied equals the savings (i.e. you pay the remainder out of pocket).

---

## Setup

### 1. Supabase project

1. Create a Supabase project.
2. Create a table named `expenses` with at least:
   - `id` — primary key identifying the user/row (text or uuid)
   - `data` — `jsonb`, holds the full array of expense objects
3. Enable **Row Level Security** and add policies so each authenticated user can only read/write their own row.
4. Create the user account(s) you'll sign in with (email + password) under Authentication.

The app stores all expenses as a single JSONB array under one row (`id = SB_ROW_ID`), loaded with `select('data')` and saved with `upsert({ id, data })`.

### 2. Configure the app

Credentials are **not** hardcoded in `index.html`. Instead:

1. Copy `config.example.js` to `config.js`.
2. Fill in your project URL and public anon key:

```js
window.SUPABASE_URL = 'https://YOUR-PROJECT-REF.supabase.co';
window.SUPABASE_ANON_KEY = 'YOUR-PUBLIC-ANON-KEY';
```

`config.js` is listed in `.gitignore`, so it never gets committed. The committed
`config.example.js` is a placeholder template.

The **anon key is safe to expose** in client-side code — it's meant to be public,
and access is gated by your RLS policies and the signed-in session. Keeping it out
of the repo is for tidiness and to avoid public-repo scanners, not because exposure
alone would compromise your data. Do **not** put the service-role key anywhere in
client code.

### 3. Lock down auth (do this once)

In the Supabase dashboard:

- **Authentication → Sign In / Providers → Email** (or the general configuration's
  "Allow new users to sign up" toggle): turn **off** new sign-ups so the project is
  effectively invite-only and randoms can't create accounts against it. Existing
  users can still sign in.
- Confirm **RLS is enabled on every table** (see step 1). This is the control that
  actually protects your data; the anon key relies on it.

### 4. Run it

Open `index.html` in a browser (or serve it from any static host). Sign in with your Supabase email/password at the auth gate. Once signed in, every read/write carries your session token automatically; if Supabase is unreachable, the app transparently uses the `localStorage` cache.

---

## How sync works

- On load, the app tries Supabase first. A `null` result means the connection failed (and the local cache is used); an empty array means "connected but no data yet."
- Every save writes to Supabase and mirrors to `localStorage`.
- Per-device UI state (collapsed sections, sort mode) is stored locally and not synced. **Manual drag order is synced** in the JSONB row (with a device-local fallback that migrates into the row on the next save), so drags survive device switches and iOS storage purges.

### Sign-in & sync

- **Rolling 72-hour window.** After sign-in you stay signed in for 72 hours, and the window *slides* — every app open (whether on a live Supabase session or the cached fallback) resets the clock via the `auth_last_ok` timestamp. The gate only returns after ~72h of not opening the app, rather than 72h after the last live session. This is the change that keeps the login screen from reappearing every couple of days under normal use.
- **The window is checked synchronously, first.** On load, the 72h timestamp is read *before* (and independently of) the async Supabase `getSession()` call — including a tiny inline script that runs during page parse to hide the gate before it can paint. This matters because a slow, hung, or throwing `getSession()` (e.g. a corrupted/expired stored token, or the Supabase CDN being unreachable) must never strand you on the login screen while you're still inside the window. The live session is then confirmed in the background only to drive the offline banner.
- **iOS purge caveat — stated honestly.** The fallback timestamp lives in the *same* `localStorage` that iOS (WebKit/ITP) can wipe for home-screen web apps. The 72h fallback only papers over Supabase's own session/refresh failures; a genuine full-storage purge clears `auth_last_ok` too, and re-login is then unavoidable. There's no pure-`localStorage` way around that.
- **Offline banner.** When there's no live cloud session — or a read/write fails — `setSyncState(false)` shows an "⚠️ Offline" banner under the tabs. Edits still persist to the device cache (`expenses_v9`), but RLS rejects the cloud upsert silently, so they don't reach Supabase until you sign in again. Because the window now slides, you could otherwise sit in this not-syncing state for a while without noticing; the banner is the signal to re-auth.

## Archiving & clearing

The two "Clear…" buttons in the Expense log **delete** entries (from Supabase and the cache) — there is no built-in trash or undo. Before clearing, use **Summary → Data → Export** to save a JSON snapshot; that file is your archive. **Import** later merges entries back in, keeping any existing entry with the same `id` as-is, so re-importing an archive is safe and non-destructive.

Caveats to be aware of with the current model:

- Export/import only moves the *expense objects*. Anything derived from them disappears when those entries are cleared and only returns if you re-import.
- Import is keyed on `id` (a creation-time `Date.now()`), so it dedupes but never updates; an edited entry won't overwrite an older copy on import.

A low-friction monthly workflow: export at month end → clear the month → keep the dated JSON files as your archive, re-importing only when you need an old entry back.

### Location / URL memory

Location suggestions are backed by a **persistent `knownLocations` list** that is accumulated on every save and survives month clears independently of the expense entries. Even after wiping all entries, the datalist in the Location field still shows every vendor/URL you've ever typed. New locations are appended on save; the list is sorted alphabetically.

**Managing locations.** The Reference tab has a *Manage saved locations* panel. Because `saveData()` re-harvests locations from the entries themselves, edits there can't be suggestion-only — they also touch matching entries:

- **Rename** rewrites `knownLocations` *and* the `location` field on every entry using the old name (confirmed with a count). This is how you fix a typo everywhere at once.
- **Remove** drops the suggestion; if any entries still use it you confirm a count and it's cleared from those entries too — otherwise it would just reappear on the next save. A location used by zero entries is removed silently.

**Data shape note:** the Supabase row stores `{ entries: [...], locations: [...], discover: {...}, order: [...] }` instead of a bare array. The code auto-migrates old bare-array data on first load, and treats missing `discover`/`order` keys as empty. If you ever need to revert to code that predates this change, update `sbLoad` to handle the new shape (or export/reimport via the Data panel first).

### Amount arithmetic

The `$` fields in Add expense (Full price, Amount paid, split amount) are text inputs (`inputmode="decimal"`) that accept a quick calculation instead of a bare number: `+ - * /`, parentheses, and a leading minus, with standard operator precedence. The expression is evaluated by a small hand-rolled recursive-descent parser (`evalAmount`) — never `eval()` — on blur, on <kbd>Enter</kbd> (in-place, via `onAmountKey`), and again via `normalizeAmountFields()` at submit, which rejects anything that isn't a valid number or calculation. All four operators are supported: `+`/`-` for combining receipt lines or subtracting a credit, `*` for quantity × unit price, `/` for a quick split. Note the mobile trade-off below.

**Mobile keyboards:** `inputmode="decimal"` keeps the fast numeric keypad for the common single-number case, but iOS's decimal keypad does not expose `+ - * /`. So on iPhone the operators aren't directly typeable without switching to the symbol keyboard. If you want first-class arithmetic on mobile, the options are `inputmode="text"` (full keyboard everywhere, at the cost of the numeric pad) or a small operator button row on the Add tab.

### Discover cashback categories

The Reference tab has a *Discover 🟠 Cashback Categories* table with a row per quarter (Jan–Mar, Apr–Jun, Jul–Sep, Oct–Dec). Values live in `discoverCashback = { year, q1..q4 }`, synced in the JSONB row alongside entries and locations. Because a static PWA can't run code at midnight on Jan 1, the "clear every new year" behavior is implemented as a clear-on-first-open-of-the-new-year: on load, if the stored `year` doesn't match the current year, the four fields reset and the new year is stamped (mirroring how open bounties roll forward at month boundaries). Editing a field stamps the current year, so entries made late in the year aren't wiped until the following January.

### Unsaved Add-expense drafts

A half-filled Add form is snapshotted to `localStorage` (`expense_draft_v1`) on every input — debounced, plus a hard flush on `pagehide`/`visibilitychange` so iOS can't lose it when the tab is backgrounded. On the next load (after the gate clears) the draft is restored and a brief toast confirms it. Restore re-fires the form's conditional handlers (`onInstanceChange`, `onStatusChange`, `toggleSplit`, `toggleSavings`, etc.) so planning/split/savings sections come back in the right state, not just the raw values.

Scope and lifecycle: **new entries only** (suppressed while `editingId !== null`), cleared on a committed Add and on the explicit Clear button, and re-saved when you prefill a duplicate. A draft with no meaningful content is discarded rather than stored.

---

## Project history

This tracker was built iteratively across several sessions. Notable milestones:

- Started as a local-only tracker (`localStorage`), then migrated to Supabase for real cross-device sync and email/password auth with row-level security.
- Grew the entry taxonomy to include planned expenses, indulgence slots, and three bounty tiers with price floors.
- Added the Hobonichi Weeks journaling layer: transcription checkboxes, ⏳/☑️ day hints, monthly roll-forward for open bounties, and the physical sticker-placement workflow.
- Added drag-and-drop ordering, dual sort modes, collapsible month/week grouping, and summary charts.
- Added the `Transcribe to Memo` label beside the transcription checkbox on non-planning expense cards.
- **Latest change:** the log's price cell now reflects **execution state** — open planning entries show *Pending* even when a projected full price is logged (the projection stays in the savings sub-lines and Summary's Pending boxes), canceled entries show *Canceled*, and only executed entries show a dollar amount. Canceled planning entries were also removed from the Summary's Pending projections (they count nowhere: not realized, not pending). The Actual-date-of-execution modal's date is now centered (iOS needs its `-webkit-date-and-time-value` pseudo-element targeted). Just before: within-week tiers reordered (executed → regular → open planning) and drag order synced across devices.
- Before that: a **🚩 flag** for incomplete entries (`/flag` → 🚩, light-red card, a per-card *Reviewed?* control, transcription lock until cleared); a **Manage saved locations** panel (rename/remove that rewrites matching entries, with confirmation counts); **unsaved Add-expense draft recovery**; and a login-flow hardening where the 72-hour window slides on every open and is checked synchronously before the async Supabase session call, with an offline banner surfacing local-only state.
- Earlier in this line of work: each Expense log card gained a **⧉ Duplicate** button that opens a pre-filled Add draft as a new Expense — carrying description, full price, category, and location, while resetting instance, discounts, and all per-transaction metadata, with the date defaulting to today. Before that: the Summary's spend rollups switched to net out-of-pocket for split bills, the planning-match picker stopped auto-hijacking new expenses, a global search box and Reset filters landed in the Expense log, the `/delta` shortcut gained a backward direction, and location suggestions became a persistent `knownLocations` list surviving month clears.

---

## License

Personal project — add a license of your choice (e.g. MIT) before publishing if you want others to reuse it.
