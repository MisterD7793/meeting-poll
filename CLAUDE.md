# meeting-poll

A group availability polling tool. Static HTML files backed by Supabase.

## What it is

- **index.html** — the respondent-facing form. People mark each time window as Always / Sometimes / Never available for a recurring weekly meeting. Unanswered slots are allowed.
- **results.html** — a password-protected dashboard showing aggregated scores, a clickable availability grid, a respondents list, and a poll picker for switching between the active poll and past (closed) ones.
- **match.html** — a password-protected drag-and-drop UI linking Groups.io mailing list members to poll respondents by name, so the dashboard can show who hasn't responded yet. Backed by the `member_matches` table and a `groups-members` Supabase edge function that lists the mailing list roster. Matches are not poll-scoped — they persist across polls.

All three files are self-contained (no build step, no dependencies beyond Google Fonts).

## Deployment

- **Live URL:** https://misterd7793.github.io/meeting-poll/
- **Host:** GitHub Pages, deploying from `main` branch root
- **Results password:** `Schedule!` (hardcoded in results.html)

## Backend

- **Supabase project:** https://eqpqtswfkfmukzuhenjt.supabase.co
- **Table:** `polls` — columns: `id` (uuid pk), `name` (text), `created_at` (timestamptz), `closed_at` (timestamptz, null = active poll). Exactly one poll should have `closed_at IS NULL` at a time.
- **Table:** `responses` — columns: `name` (text), `email` (text — identity key; nullable at the DB level for pre-existing rows, but required on the form), `availability` (jsonb), `top_picks` (text[] of slot keys like `wed_early_aftn`, 1–3 required on the form; null on rows from before 2026-09-30), `submitted_at` (timestamptz), `poll_id` (uuid fk → `polls.id`). Unique on `(poll_id, email)`. index.html looks up the active poll's id on load, tags submissions with it, and upserts on `(poll_id, email)` — resubmitting with the same email edits your existing answer instead of creating a duplicate.
- **Table:** `member_matches` — columns: `groups_io_email`, `groups_io_name`, `respondent_name`. Not poll-scoped.
- **Auth:** anon key only; RLS allows public insert/select/update on `responses` (update is required for the email-upsert edit flow), public insert/select on `polls`, public update on `polls` (for closing a poll), and public insert/select on `member_matches`
- **Direct DB access:** `db.eqpqtswfkfmukzuhenjt.supabase.co:5432` is IPv6-only and unreachable from dev-mac's network. For running SQL directly (vs. the dashboard SQL editor), use the pooler: `postgresql://postgres.eqpqtswfkfmukzuhenjt@aws-1-us-west-2.pooler.supabase.com:6543/postgres` with the DB password via `PGPASSWORD`. No GitHub-linked migrations workflow exists yet — schema changes are ad hoc (dashboard SQL editor or direct psql). Considered setting one up 2026-08-25, deferred in favor of just running things directly.

## Poll lifecycle

- New responses always attach to the poll with `closed_at IS NULL` (the "active" poll).
- "Save & start new poll" (results.html) never deletes data: it PATCHes the active poll's `name` and sets `closed_at = now()`, then POSTs a brand-new poll row with `closed_at = null`. Old responses stay in the DB, tagged to their original poll — that's the "save."
- results.html's poll picker lets you view any past poll's grid/respondents read-only; invite/mail actions in "Still needed" are hidden unless viewing the active poll.
- If no poll exists yet (fresh DB), index.html shows "This poll is not currently open" and blocks submission — a `polls` row must exist first.

## Time slots

Defined in both files. Base timezone is **US Eastern (ET)**. The form converts to the respondent's local timezone at page load using `Intl.DateTimeFormat`.

| Slot ID      | Name                    | ET hours   |
|--------------|-------------------------|------------|
| morning      | Morning                 | 9–11 am    |
| early_aftn   | Early Afternoon         | 12–3 pm    |
| afternoon    | Afternoon               | 3–6 pm     |
| evening      | Evening / After Dinner  | 8–10 pm    |

Days: Sun–Thu (all 4 slots), Fri (morning + early_aftn only).

## Scoring (results dashboard)

`score = (always×2 + sometimes) / (total×2) × 100`

- ≥70% → green (high)
- 40–69% → yellow (mid)
- <40% → red (low)
- No responses → gray

**Winner (outlined cell):** among slots within 10 points (`TIE_BAND`) of the top score, the one with the most ★ top picks wins; score breaks star ties. Top picks exist because most respondents answer "Sometimes" to nearly everything, which clustered weekday slots within a few points and no clear winner emerged. Stars can only go on Always/Sometimes slots. Entering an email that already responded to the active poll reloads that person's previous answers into the form.

## What's been done

- Initial form + success screen
- Results dashboard with grid, popup detail view, respondents list
- Fixed popup key parsing (slot IDs with underscores, e.g. `early_aftn`)
- Timezone conversion: slot times shown in respondent's local TZ with abbreviation (e.g. "6–8 am PDT"); handles half-hour offsets and am/pm crossings
- Mobile responsive: on screens ≤540px, slot label stacks above the Always/Sometimes/Never buttons
- Groups.io member matching (match.html) to track who on the mailing list hasn't responded yet
- Poll lifecycle: `polls` table, active-poll lookup on submit, "Save & start new poll" flow, poll picker on results.html for browsing past rounds
- Email as respondent identity: `responses.email`, unique per `(poll_id, email)`, submit upserts on that key — duplicate prevention and edit/update-response solved together by resubmitting with the same email
- "← Back to poll" link on results.html, next to the poll picker
- ★ Top picks (1–3 per respondent) as the tie-breaker for the winning slot; form reloads previous answers by email so earlier respondents can add stars

## Possible next steps

- **Runoff poll:** if ★ top picks still don't separate the leading slots, start a follow-up poll limited to the top 3–4 slots (would need a per-poll slot list; the `polls` table has none today)
- **results.html timezone:** the dashboard shows raw ET times; consider converting to viewer's local TZ for consistency
- **Admin controls:** delete a response from the dashboard
- **Multi-tenant / SaaS direction:** the poll-scoped schema (`polls` + `poll_id`) was chosen with this in mind — a future `account_id`/owner concept could scope polls to a customer
- **Reminder emails:** now that respondents supply their own email, send nudges to people who haven't answered the active poll
- **"Email the group" tool:** compose/send to all current respondents (or all matched members) directly from results.html
- **match.html could auto-match by email:** respondents now supply email directly, which could replace (or supplement) the manual drag-and-drop name-to-Groups.io-member matching with a direct email match
- **GitHub-linked Supabase migrations:** deferred 2026-08-25 in favor of running SQL directly; worth revisiting if the SaaS direction firms up and schema changes get more frequent
