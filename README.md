# Workouts

A static, read-only dashboard for my StrongLifts 5×5 progress.
Connects directly to Supabase from the browser.
No build step, no server, no framework — one HTML file, one Supabase project, GitHub Pages.

Live: <https://jcontini.github.io/workouts/>

---

## What this is

- **Single static page.** Plain HTML + CSS + vanilla JS, ESM imports from CDN. Open `index.html` and it works.
- **Read-only by architecture.** The browser holds the Supabase publishable key. RLS policies allow `SELECT` for the anon role and block writes. There is no write path in the UI.
- **Writes happen elsewhere.** I add new sessions/sets through Claude in chat, which uses the Supabase MCP (service role) to insert rows. The site re-reads on page load.

## Sections

```
Up Next        → next workout (A or B), prescribed weights, plate math
Recent         → last 10 sessions, tap to expand into set-by-set detail
Progression    → per-lift line charts of top working set over time
Stats          → total sessions, total volume, days since last, week streak
```

## How it connects to Supabase

`index.html` imports `@supabase/supabase-js@2` from `esm.sh` and calls `createClient(url, publishableKey)`. The key and URL live in a `CONFIG` object at the top of the file — there are no env vars at runtime.

```js
const CONFIG = {
  supabaseUrl: 'https://vwejipgsculrwustpooe.supabase.co',
  supabaseAnonKey: 'sb_publishable_…',
  barWeightLbs: 45,
  availablePlates: [45, 25, 10, 5, 2.5],
};
```

The publishable key is **safe to expose** — it has no privileges beyond what RLS policies grant to the anon role. Any privacy guarantee comes from RLS, not from secrecy.

## Schema (already exists, do not recreate)

| Table          | Purpose                                              |
| -------------- | ---------------------------------------------------- |
| `programs`     | Training methodologies (StrongLifts, etc.)           |
| `lifts`        | Exercise catalog                                      |
| `sessions`     | One row per training session                          |
| `sets`         | One row per set                                       |
| `working_sets` | View — non-warmup sets joined with lift name + date  |

## Enable RLS — run this once in the Supabase SQL editor

This is the **first thing to run** before going live. Without it, `anon` could write through the publishable key.

```sql
-- 1. Turn RLS on for every table.
alter table public.programs enable row level security;
alter table public.lifts    enable row level security;
alter table public.sessions enable row level security;
alter table public.sets     enable row level security;

-- 2. Drop any pre-existing public policies (idempotent, safe to re-run).
drop policy if exists "anon read programs" on public.programs;
drop policy if exists "anon read lifts"    on public.lifts;
drop policy if exists "anon read sessions" on public.sessions;
drop policy if exists "anon read sets"     on public.sets;

-- 3. Allow SELECT for anon (and authenticated, in case you log in later).
create policy "anon read programs" on public.programs
  for select to anon, authenticated using (true);

create policy "anon read lifts" on public.lifts
  for select to anon, authenticated using (true);

create policy "anon read sessions" on public.sessions
  for select to anon, authenticated using (true);

create policy "anon read sets" on public.sets
  for select to anon, authenticated using (true);

-- 4. No INSERT / UPDATE / DELETE policies are created for anon, so all
--    write attempts from the browser will fail. Writes still work via
--    the service_role key (used by Claude through the Supabase MCP),
--    which bypasses RLS by design.

-- 5. The `working_sets` view inherits the policies of its underlying tables
--    when accessed via PostgREST. No extra policy needed — but if you ever
--    create the view with `security definer`, replace it with the default
--    `security invoker` so it respects the caller's RLS.
```

### Verify it's locked down

After running the SQL, paste this into the SQL editor — every line should return zero rows or an error:

```sql
-- This should succeed (read).
select count(*) from public.sessions;

-- These should all FAIL when run as anon. The SQL editor runs as an admin
-- role, so to actually verify, hit the REST API from the browser console:
--
--   await fetch('https://vwejipgsculrwustpooe.supabase.co/rest/v1/sessions', {
--     method: 'POST',
--     headers: {
--       apikey: 'sb_publishable_...',
--       authorization: 'Bearer sb_publishable_...',
--       'content-type': 'application/json',
--       prefer: 'return=representation'
--     },
--     body: JSON.stringify({ date: '2099-01-01', workout_label: 'X' })
--   }).then(r => r.json())
--
-- Expected response: { code: '42501', message: 'new row violates row-level security policy' }
```

## Deploy to GitHub Pages

```bash
git init
git add .
git commit -m "Initial: lifting dashboard"
gh repo create workouts --public --source=. --push
gh api -X POST repos/jcontini/workouts/pages -f build_type=workflow -f source[branch]=main -f source[path]=/
```

Then in the repo on github.com:

1. **Settings → Pages**
2. Source: **Deploy from a branch**
3. Branch: `main` / root
4. Save. Wait ~30 seconds. Visit <https://jcontini.github.io/workouts/>.

To update: edit `index.html`, `git push`. Pages redeploys automatically.

## How I add new data

Never edit the site. Always tell Claude in chat what I lifted, e.g.:

> Logged session today, Workout A. Squat 3×5 @ 70 all sets cleared. Bench 5×5 @ 50, missed last set (got 4). Row 5×5 @ 50 cleared. Felt 8/10 pre, 6/10 post.

Claude inserts rows into `sessions` + `sets` via the Supabase MCP (service role). Reload the page — new data shows up.

## Tweaking

| Want to change…                | Edit                                                     |
| ------------------------------ | -------------------------------------------------------- |
| Available plates / bar weight  | `CONFIG` object at the top of `index.html`               |
| Accent color                   | `--accent` in the `:root` CSS block                      |
| Lifts shown in progression     | `MAIN_LIFTS` and `MAIN_LIFT_ALIASES` arrays              |
| Number of recent sessions      | `.slice(0, 10)` in `renderRecent`                        |

## Files

```
index.html   — the entire app
README.md    — this file
LICENSE      — MIT
.gitignore
```
