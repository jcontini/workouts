# Set up your own lifting tracker

This is the source for [jcontini.github.io/workouts](https://jcontini.github.io/workouts/) — a private dashboard for a barbell-lifting program. It connects to your own database, deploys as a free public web page, and lets you log new workouts by chatting with Claude.

You don't need to know how to code or use a terminal. An AI walks you through everything.

## How it works

1. **Get Claude Code on your computer** — it's a chat app with a "hands" — it can run things on your machine for you.
   - macOS / Windows / Linux: download from <https://claude.com/claude-code>
2. Open it. You'll see a chat box.
3. **Optional but recommended:** start Claude Code with `--dangerously-skip-permissions` so it doesn't pause every few seconds asking for approval. (You can also approve as you go — your choice.)
4. **Copy the prompt below** (use the Copy button) and paste it into Claude Code. Press Enter.

```
I want to set up my own lifting tracker like https://jcontini.github.io/workouts/.

Please fetch the full instructions at https://jcontini.github.io/workouts/content/setup.md and walk me through it. The instructions are written for you — follow them top to bottom. Ask me questions as you need them. If we run out of context partway through, remind me to save the URL above to your memory so we can resume in a new session.

I am non-technical. Don't ask me to do anything in a terminal — you do it. The only things I'll do myself are: sign up for a couple of websites in my browser when you tell me to, and confirm/approve actions when you ask. That's it.
```

5. Answer Claude's questions as it goes. Approve actions when it asks. The whole setup takes about 15-30 minutes, depending on how fast the websites load.

That's it. The rest of this page is technical instructions written *for the AI*. You don't need to read them.

---

## Instructions for Claude (the AI reading this)

You're helping a **non-technical user** clone a personal lifting tracker. The end state is:

- Their own GitHub repo (scaffolded from this one) deployed to their own GitHub Pages URL.
- Their own Supabase project, with the same schema, RLS configured for read-only anon access.
- The user can log new sessions through you in chat (writes go through the Supabase MCP using the service-role key, never through the deployed site).

### How to behave

- The user is non-technical. They almost certainly haven't used a terminal before. Don't assume they know what "command line", "Homebrew", "git", "shell", "package manager" mean. If you need to use those words, define them inline. ("Homebrew is a tool that installs other tools — like an app store for the terminal.")
- **Never ask them to type into a terminal**. You have shell access; you do it. They watch.
- **Never ask them to edit a code file**. You have file-editing tools; you do it.
- **Do ask them to do things in the browser** when there's no other way: sign up for accounts, copy access tokens, click "Authorize" on OAuth flows. Make those steps very explicit — tell them exactly what URL to open, what to click, what to copy back to you.
- Be friendly and concrete. After each phase, tell them what you just did and what's about to happen next, in plain English. ("Just installed the GitHub command-line tool. Next we're going to log you into your GitHub account — your browser will pop up.")
- Treat any token or password as a secret. Once they paste it, don't echo it back. Don't write it to disk in plaintext anywhere except the MCP config you control.
- If something errors, don't dump a stack trace at them. Diagnose silently, then tell them what's happening in one sentence and what you're going to try.

### Probing the environment

Don't assume anything about their machine. Run quick non-destructive checks first to see what's there:

- `uname -a` — what OS
- `which git`, `which gh`, `which brew`, `which python3`
- `gh auth status` (will fail noisily if not signed in — that's fine, you're just probing)

Likely state of a non-technical user's macOS machine:

- `git` — present (Apple ships it; first run may trigger an Xcode Command Line Tools prompt)
- `gh` — missing
- `brew` — missing
- A GitHub account — maybe; they may not have used it on this machine
- A Supabase account — probably not yet
- Claude Code MCP servers — Supabase MCP is not installed by default

### Phase 1 — Install Homebrew (only if missing, only on macOS / Linux)

If `which brew` returns nothing on macOS or Linux, install it. The official one-liner is at <https://brew.sh>. As of late 2025 it's:

```
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

Important nuances:

- The Homebrew installer is **interactive** — it pauses for the user to press Return and may prompt for their macOS password (the one they log into the computer with). Tell them this is happening and that the password prompt is real and from their computer, not from you.
- After install on Apple Silicon Macs, the installer prints a "Next steps" block with two `eval` commands to add Homebrew to the shell PATH. Run those in a way that takes effect for your subsequent commands.
- On Windows, there is no Homebrew. Use winget instead: `winget install --id GitHub.cli` for `gh`. Confirm `winget` exists first.

If they're on macOS and `git --version` triggers the Xcode Command Line Tools prompt, walk them through clicking "Install" in the dialog that pops up; this also gets them `git` if it wasn't there.

### Phase 2 — Install the GitHub CLI

```
brew install gh
```

(Or `winget install --id GitHub.cli` on Windows.)

### Phase 3 — GitHub account + auth

1. Run `gh auth status`. If logged in, skip.
2. If not, run `gh auth login`. This is interactive — it will:
   - Ask which host (choose `GitHub.com`)
   - Ask which protocol (choose `HTTPS` for non-technical users — easier than SSH)
   - Ask whether to authenticate with a web browser (yes)
   - Print a one-time code and open the user's browser to <https://github.com/login/device>
3. Tell the user: "Your browser is going to open. Paste this code: `XXXX-XXXX`. Click 'Authorize'. Come back here when done."
4. If the user doesn't have a GitHub account yet, the same flow lets them sign up. Walk them through it.

### Phase 4 — Supabase account + access token

Supabase is the database service the dashboard talks to. This is the one step that genuinely requires the user's hands — they need to sign up in their browser. Open the browser for them with `open https://supabase.com` (macOS) / `xdg-open https://supabase.com` (Linux) / `start https://supabase.com` (Windows) so they don't have to copy-paste the URL.

Walk them through it like this:

1. **Sign up.** "Your browser is opening to supabase.com. Click 'Start your project' and sign up — you can use Google or GitHub for one-click signup. The free tier is what we want — no credit card needed." Wait for them to confirm they're logged in.
2. **Get an access token** so you can manage their projects on their behalf. Open a second tab for them: `open https://supabase.com/dashboard/account/tokens`. Tell them: "On this page, click 'Generate new token'. Name it 'Claude Code'. Copy the token that appears — you'll only see it once. Paste it back into this chat."
3. When they paste the token, **treat it as a secret**. Don't echo it back. Don't write it to disk anywhere except the MCP config you'll write next.

### Phase 5 — Install the Supabase MCP

The Supabase MCP is a small program that lets you (the AI) talk to the user's Supabase database directly. Install it for them:

```
claude mcp add supabase \
  --env SUPABASE_ACCESS_TOKEN=<their-token> \
  -- npx -y @supabase/mcp-server-supabase@latest
```

After installing, the user needs to **restart Claude Code** for the MCP to load — you can't restart your own host process. Tell them: "I just connected your database. Quit Claude Code and reopen it, then say 'continue setup' and I'll pick up from here."

When they're back, confirm a Supabase tool is available (e.g. `mcp__supabase__list_organizations`). If not, troubleshoot before proceeding.

### Phase 6 — Create the Supabase project

Using the MCP:

1. List their organizations.
2. Create a new project named e.g. `lifting`. Pick a region close to them. Use the free tier.
3. Wait for it to finish provisioning (status `ACTIVE_HEALTHY`).
4. Get the project URL and the **publishable** anon key (sometimes shown as "anon key" or "publishable key" — it's the public one, safe to expose). The MCP has `get_project_url` and `get_publishable_keys` for this.

### Phase 7 — Schema

Apply this migration to their project. Use the Supabase MCP's `apply_migration` (DDL).

```sql
-- Programs (training methodologies)
create table public.programs (
  id bigint generated by default as identity primary key,
  name text not null,
  description text,
  progression_compound numeric not null default 5,
  progression_ohp numeric not null default 2.5,
  deload_threshold int not null default 3,
  deload_percent numeric not null default 0.10,
  schedule_days int[] not null default '{1,3,5}',
  created_at timestamptz not null default now(),
  constraint programs_schedule_days_valid check (
    schedule_days <@ '{1,2,3,4,5,6,7}'::int[] and array_length(schedule_days, 1) >= 1
  )
);

-- Lift catalog
create table public.lifts (
  id bigint generated by default as identity primary key,
  name text not null unique,
  category text,
  equipment text,
  primary_muscles text[] default '{}'::text[],
  notes text,
  video_url text,
  created_at timestamptz not null default now()
);

-- Structured workout definitions per program
create table public.program_workouts (
  id bigint generated by default as identity primary key,
  program_id bigint not null references public.programs(id) on delete cascade,
  label text not null,
  position int not null,
  lift_id bigint not null references public.lifts(id),
  set_count int not null check (set_count > 0),
  target_reps int not null check (target_reps > 0),
  created_at timestamptz not null default now(),
  unique (program_id, label, position),
  check (length(label) > 0)
);
create index program_workouts_program_label_idx on public.program_workouts (program_id, label, position);

-- One row per training session
create table public.sessions (
  id bigint generated by default as identity primary key,
  date date not null,
  program_id bigint references public.programs(id),
  workout_label text,
  location text,
  duration_min int,
  energy_pre int,
  energy_post int,
  mood text,
  notes text,
  created_at timestamptz not null default now()
);

-- One row per set performed
create table public.sets (
  id bigint generated by default as identity primary key,
  session_id bigint not null references public.sessions(id) on delete cascade,
  lift_id bigint not null references public.lifts(id),
  set_number int not null,
  is_warmup boolean not null default false,
  weight numeric,
  target_reps int,
  actual_reps int,
  rpe int,
  rest_seconds int,
  notes text,
  created_at timestamptz not null default now()
);
```

### Phase 8 — Lock down with RLS (critical)

The browser will hold the publishable anon key. Without RLS, anyone can write to the database. Apply this migration:

```sql
alter table public.programs         enable row level security;
alter table public.lifts            enable row level security;
alter table public.sessions         enable row level security;
alter table public.sets             enable row level security;
alter table public.program_workouts enable row level security;

create policy "anon read programs"          on public.programs          for select to anon, authenticated using (true);
create policy "anon read lifts"             on public.lifts             for select to anon, authenticated using (true);
create policy "anon read sessions"          on public.sessions          for select to anon, authenticated using (true);
create policy "anon read sets"              on public.sets              for select to anon, authenticated using (true);
create policy "anon read program_workouts"  on public.program_workouts  for select to anon, authenticated using (true);
```

No INSERT/UPDATE/DELETE policies for `anon`. Writes will fail from the browser. Writes from you (via the service-role MCP) bypass RLS.

**Verify before proceeding** — make a request as anon and confirm it's blocked:

```
curl -s "$SUPABASE_URL/rest/v1/sessions" \
  -X POST \
  -H "apikey: $PUBLISHABLE_KEY" \
  -H "authorization: Bearer $PUBLISHABLE_KEY" \
  -H "content-type: application/json" \
  -d '{"date":"2099-01-01","workout_label":"X"}'
```

Expected response includes `"code":"42501"` or similar RLS-violation message. If the insert succeeds, **stop** and fix the policies before continuing — you've just confirmed they have an open database.

### Phase 9 — Seed data (optional but recommended)

Most users following this guide want StrongLifts 5×5 as a starting program. Insert it via `execute_sql`:

```sql
insert into public.programs (name, description, progression_compound, progression_ohp, deload_threshold, deload_percent, schedule_days)
values (
  'StrongLifts 5x5',
  'Linear progression beginner program. 3x/week, alternating Workout A (Squat/Bench/Row) and Workout B (Squat/OHP/Deadlift).',
  5, 2.5, 3, 0.10, '{1,3,5}'
);

insert into public.lifts (name, category, equipment, primary_muscles, notes, video_url) values
  ('Squat',          'compound', 'barbell', '{"quads","glutes","hamstrings","core"}',                    'Low bar back squat',     'https://www.youtube.com/watch?v=bs_Ej32IYgo'),
  ('Bench Press',    'compound', 'barbell', '{"chest","triceps","front delts"}',                        'Flat barbell bench',     'https://www.youtube.com/watch?v=BYKScL2sgCs'),
  ('Barbell Row',    'compound', 'barbell', '{"lats","upper back","rear delts","biceps"}',              'Bent over Pendlay-style','https://www.youtube.com/watch?v=RQU8wZPbioA'),
  ('Overhead Press', 'compound', 'barbell', '{"shoulders","triceps","upper chest","core"}',             'Standing strict press',  'https://www.youtube.com/watch?v=wol7Hko8RhY'),
  ('Deadlift',       'compound', 'barbell', '{"hamstrings","glutes","lower back","lats","traps","core"}','Conventional stance',   'https://www.youtube.com/watch?v=Y1IGeJEXpF4');

-- Workout A: Squat / Bench / Row, 5x5
-- Workout B: Squat / OHP / Deadlift (1x5)
insert into public.program_workouts (program_id, label, position, lift_id, set_count, target_reps)
select
  (select id from public.programs where name = 'StrongLifts 5x5'),
  v.label, v.position,
  (select id from public.lifts where name = v.lift),
  v.set_count, v.target_reps
from (values
  ('A', 1, 'Squat',          5, 5),
  ('A', 2, 'Bench Press',    5, 5),
  ('A', 3, 'Barbell Row',    5, 5),
  ('B', 1, 'Squat',          5, 5),
  ('B', 2, 'Overhead Press', 5, 5),
  ('B', 3, 'Deadlift',       1, 5)
) as v(label, position, lift, set_count, target_reps);
```

Ask the user before running this — some users may want a blank slate or a different program.

### Phase 10 — Clone the dashboard repo

Scaffold a fresh copy of the dashboard into the user's home folder. Don't make them choose a path — just put it at `~/workouts` (a folder named "workouts" inside their home directory). They never need to `cd` to it themselves; you'll do everything for them.

```
git clone https://github.com/jcontini/workouts ~/workouts
cd ~/workouts
rm -rf .git
git init -b main
gh repo create workouts --public --source=. --description "My lifting tracker"
```

If `~/workouts` already exists, ask the user whether they want to overwrite it or pick a different name (e.g. `~/workouts-mine`).

### Phase 11 — Wire the user's Supabase project into index.html

Edit the `CONFIG` block at the top of `index.html`. Replace the two values:

```js
const CONFIG = {
  supabaseUrl: 'https://<their-project-ref>.supabase.co',
  supabaseAnonKey: '<their-publishable-key>',
  ...
};
```

Use `Read` + `Edit`. Don't have the user touch the file by hand.

### Phase 12 — Commit and deploy

```
git add -A
git commit -m "Initial: my lifting tracker"
git push -u origin main
gh api -X POST repos/<user>/workouts/pages -f 'source[branch]=main' -f 'source[path]=/'
```

Wait ~30 seconds for Pages to build, then `curl -I https://<user>.github.io/workouts/` until it returns 200.

### Phase 13 — Verify

Open the live URL in their browser for them: `open https://<user>.github.io/workouts/`. The "Sessions" tab should load with empty progression charts and a "Next session" card showing the program's first workout.

Tell the user: "Your dashboard is live at https://<user>.github.io/workouts/. To log a workout, just tell me about it — for example: 'today I did Workout A, Squat 5×5 at 65 lb all sets cleared, Bench 5×5 at 50 missed last set'. I'll add it to your database. Refresh the page in your browser and you'll see the data."

### Important notes

- **The publishable anon key is safe to commit** to a public repo — its only privileges are what RLS grants. Phase 8 is what makes the key safe.
- **The personal access token is NOT safe to commit.** It lives in the user's local Claude MCP config only.
- **The service role key is NEVER part of the deployed app.** Writes happen from the user's machine through the MCP, using a key only Claude has access to.
- If the user's session ends and they want to resume in a new conversation, suggest they save a memory entry with this URL so the next agent can re-fetch the instructions.

### What lives in the repo

```
index.html           the dashboard, single file SPA
content/
  bodymind.json      reference graph for the Body & Mind tab
  setup.md           this file
README.md            short description
```

The user can edit `content/bodymind.json` to make the Body & Mind tab their own — it's just data.

### Troubleshooting

- **"new row violates row-level security policy"** when logging through Claude → you're using the publishable key by mistake. The Supabase MCP should be using the service role under the hood; check the MCP env config has `SUPABASE_ACCESS_TOKEN`, not the anon key.
- **The site shows "Failed to fetch"** for `bodymind.json` → the user is opening `index.html` via `file://`. Tell them to either deploy to Pages (the actual goal) or run `python3 -m http.server` in the repo and open `http://127.0.0.1:8000`.
- **Pages 404 after enabling** → wait a minute, GitHub takes a moment. Check the Actions tab on the repo to see the deploy status.
