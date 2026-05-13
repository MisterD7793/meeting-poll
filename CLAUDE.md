# meeting-poll

A group availability polling tool. Two static HTML files backed by Supabase.

## What it is

- **index.html** — the respondent-facing form. People mark each time window as Always / Sometimes / Never available for a recurring weekly meeting. Unanswered slots are allowed.
- **results.html** — a password-protected dashboard showing aggregated scores, a clickable availability grid, and a respondents list.

Both files are self-contained (no build step, no dependencies beyond Google Fonts).

## Deployment

- **Live URL:** https://misterd7793.github.io/meeting-poll/
- **Host:** GitHub Pages, deploying from `main` branch root
- **Results password:** `Schedule!` (hardcoded in results.html)

## Backend

- **Supabase project:** https://eqpqtswfkfmukzuhenjt.supabase.co
- **Table:** `responses` — columns: `name` (text), `availability` (jsonb), `submitted_at` (timestamptz)
- **Auth:** anon key only; RLS should allow public insert and authenticated/password-gated read

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

## What's been done

- Initial form + success screen
- Results dashboard with grid, popup detail view, respondents list
- Fixed popup key parsing (slot IDs with underscores, e.g. `early_aftn`)
- Timezone conversion: slot times shown in respondent's local TZ with abbreviation (e.g. "6–8 am PDT"); handles half-hour offsets and am/pm crossings
- Mobile responsive: on screens ≤540px, slot label stacks above the Always/Sometimes/Never buttons

## Possible next steps

- **results.html timezone:** the dashboard shows raw ET times; consider converting to viewer's local TZ for consistency
- **Duplicate submission prevention:** currently nothing stops the same name from submitting twice
- **Edit/update response:** let respondents return and change their answers
- **Admin controls:** delete a response from the dashboard
- **Close the poll:** a way to mark it as finalized and stop accepting submissions
