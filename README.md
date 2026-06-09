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
- **Drag-and-drop ordering**, collapsible month/week grouping, and dual sort modes (by date logged vs. by planned date).
- **Summary dashboards** — breakdowns by instance, category, and status, with bar and pie charts.
- **Searchable log** — a global search box on top of the instance/category/status filters, with a one-tap **Reset filters**. Filters decide what's shown; search only drills down within them.
- **Slash commands & autofill** — type `/gas`, `/claude`, `/icloud`, etc. to prefill recurring entries.
- **Cross-device sync** via Supabase with email/password auth and row-level security; falls back to `localStorage` when offline.

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

## The planning & journaling system

This is the part that makes the app unusual. It's designed to pair a digital ledger with a physical planner.

### Status lifecycle

Planned expenses and bounties move through statuses: **In progress → Greenlit → Executed** (or **Canceled**). When an entry is marked Executed, you record the `actualDate` it actually happened.

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
- **Open bounties** → `⏳ MON 1` — bounties roll forward to the 1st of the current month for a monthly check-in.

When the checkbox appears:

- Non-planning instances: always.
- Open bounties: always (rolling forward to the 1st each month).
- Other planning instances: when a planned date or actual date exists.

Checking the box dims the card and persists across sessions; open bounties reset their recorded state at month boundaries.

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
- Per-device UI state (collapsed sections, sort mode, manual drag order) is stored locally and not synced.

## Archiving & clearing

The two "Clear…" buttons in the Expense log **delete** entries (from Supabase and the cache) — there is no built-in trash or undo. Before clearing, use **Summary → Data → Export** to save a JSON snapshot; that file is your archive. **Import** later merges entries back in, keeping any existing entry with the same `id` as-is, so re-importing an archive is safe and non-destructive.

Caveats to be aware of with the current model:

- Export/import only moves the *expense objects*. Anything derived from them disappears when those entries are cleared and only returns if you re-import.
- Import is keyed on `id` (a creation-time `Date.now()`), so it dedupes but never updates; an edited entry won't overwrite an older copy on import.

A low-friction monthly workflow: export at month end → clear the month → keep the dated JSON files as your archive, re-importing only when you need an old entry back.

### Location / URL memory

Location suggestions are backed by a **persistent `knownLocations` list** that is accumulated on every save and survives month clears independently of the expense entries. Even after wiping all entries, the datalist in the Location field still shows every vendor/URL you've ever typed. New locations are appended on save; the list is sorted alphabetically.

**Data shape note:** the Supabase row now stores `{ entries: [...], locations: [...] }` instead of a bare array. The code auto-migrates old bare-array data on first load. If you ever need to revert to code that predates this change, update `sbLoad` to handle the new shape (or export/reimport via the Data panel first).

---

## Project history

This tracker was built iteratively across several sessions. Notable milestones:

- Started as a local-only tracker (`localStorage`), then migrated to Supabase for real cross-device sync and email/password auth with row-level security.
- Grew the entry taxonomy to include planned expenses, indulgence slots, and three bounty tiers with price floors.
- Added the Hobonichi Weeks journaling layer: transcription checkboxes, ⏳/☑️ day hints, monthly roll-forward for open bounties, and the physical sticker-placement workflow.
- Added drag-and-drop ordering, dual sort modes, collapsible month/week grouping, and summary charts.
- Added the `Transcribe to Memo` label beside the transcription checkbox on non-planning expense cards.
- **Latest change:** the Expense log gained a **global search box** (matches description, location, category, instance/status labels, savings, recipient, split, and amounts) layered under the existing filters, plus a **Reset filters** button. The `/delta` shortcut in Savings details gained a **backward direction** (`/delta $Y` → `$X /delta $Y`, symmetric inverse: `X = Y + paid`). Location/URL suggestions are now backed by a **persistent `knownLocations` list** that survives month clears; the Supabase row format changed from a bare array to `{entries, locations}` with auto-migration.

---

## License

Personal project — add a license of your choice (e.g. MIT) before publishing if you want others to reuse it.
