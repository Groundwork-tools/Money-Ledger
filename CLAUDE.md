# Money Ledger

Money Ledger is a monthly money tracker: you log what actually happened, by
hand, and review where it went by category, week, and month.
It deliberately is not a bank-linked auto-importer — typing every entry in
yourself is the point, not a limitation, the same way Hours Ledger makes you
log the week you actually had rather than infer it.

This is the second tool in a family with [Hours Ledger](https://groundwork-tools.github.io/Hours-Ledger)
— same method (log reality, judge it after), applied to money instead of time.
It should look like it belongs next to Hours Ledger: same colours, same type,
same ledger-paper, square-cornered plainness — not a reskin, since money has
different shapes than time (no time-of-day grid, a signed ledger instead).

Built and maintained by Sebastian Poulsen, who is learning to work with Claude Code
through these projects. **Explain your reasoning as you go.** When there is more
than one sensible way to do something, say what the options were and why you picked
one. Teaching matters here as much as shipping. If I accept a change without
understanding it, that is a failure of the session even if the code is correct.

Push back on me. If a request is a bad idea, say so before doing it. If I ask for
something that contradicts a rule below, stop and say which rule.

---

## Hard rules — do not break these

1. **Entry schema and storage format must never silently change shape.** An
   `Entry` is `{id, date, amountMinor, direction, categoryId, note, recurringId}`
   (see data model below). If the schema has to change — a new field, a
   renamed key, a new storage format — write a migration that upgrades old
   data in place before any new code reads it, and keep the old reader
   working until the migration has been verified against a real exported
   ledger, not a synthetic one. This applies to both the localStorage copy
   and the Google Drive file format once sync exists. *Real user data is the
   one thing you cannot apologise your way out of.*
2. **No framework, no bundler, no npm dependency shipped to the user.**
   Vanilla HTML/CSS/JS, loaded straight by the browser, same as Hours Ledger.
   The one exception is Google's Identity Services script, loaded via a
   `<script src>` tag for Drive OAuth — not installed as a package, nothing
   to bundle. Dev-only tooling (a formatter, a linter) never ships to the
   site.
3. **No build step.** `index.html` + `style.css` + `app.js`, served directly
   by GitHub Pages. Don't introduce a transpiler or bundler to "solve" a
   problem — solve it in plain JS instead.
4. **Nothing leaves the device unless the user explicitly connects Drive
   sync.** By default, entries live only in `localStorage`. The opt-in
   "Connect Google Drive" flow syncs directly from the browser to the user's
   own Drive via the narrow `drive.file` scope (only files this app
   created) — no server under my control ever receives, stores, or can read
   a user's ledger data, because there is no server. If any future change
   would route user data through infrastructure I control, stop and flag it
   before writing the code — that promise is the whole reason sync is safe
   to offer to strangers.
5. **Every entry must stay editable and deletable after the fact.**
   Disconnecting Drive must never delete local data. Connecting Drive must
   never silently overwrite one device's data with another's — if two
   devices disagree, surface the conflict, don't auto-resolve it silently.
6. **No sync operation may remove a record the other side hasn't
   acknowledged.** Neither direction — pushing to Drive nor pulling from
   it — may cause a record present on one side to disappear unless that
   side has actually seen it. A whole-file timestamp comparison is not
   enough to satisfy this: "pushed most recently" is not the same claim
   as "has seen everything the other side has," and treating them as
   equivalent is exactly the bug this rule exists to prevent (2026-08-03:
   a stale device's push silently erased a newer entry from another
   device, twice — once on pull via a "newer data?" prompt the user
   correctly didn't trust, once on push with no prompt at all). Sync must
   merge record-by-record before writing, in both directions, with no
   exception for "just this once, it's probably fine."
   This rule governs whether a *record* (an id) survives — it does not
   require every conflicting *edit* to survive too. See "Sync design
   decisions" below: when a record survives on both sides but its content
   genuinely disagrees, the losing edit's specific field values may be
   discarded outright. That's a deliberate, narrower exception to this
   rule, not a violation of it.
7. **Ask before restructuring.** Propose and wait. Do not refactor broadly in
   a session that was asked for a small fix.
8. **Do not claim something works if you have not verified it.** "This
   should work" and "I ran this and it worked" are different sentences. Use
   the honest one.
9. **The self-test suite must run and pass before any push touching sync,
   merge, or storage.** From the repo root: `python3 -m http.server`, then
   open `money-ledger-selftest.html` and confirm "0 failing" in the summary
   line. Not optional verification — the gate itself. See "Testing before
   you claim it works" for what TEST_MODE isolates and why the suite exists
   (2026-08-03/04 — it exists specifically to catch the class of bug that
   shipped that night).
10. **Anything touching money, totals, dates, or budgets ships with tests
    in the same commit — not after, not as manual checks run once and
    thrown away.** This is automatic; Sebastian does not need to ask for
    "with tests" — a feature request in one of these areas already implies
    it, the same way "no build step" doesn't need to be repeated per
    request. Applies to both kinds of correctness: the *math* (does the
    number come out right — extract a pure function and assert on it
    directly, the way `mergeRecords` already is) and *rendering* (does the
    right value/state actually reach the DOM — read `d`, the iframe's
    contentDocument, already threaded through `money-ledger-selftest.html`
    but unused for years; see "Testing before you claim it works" for why
    that gap existed and how it's closed now). A scratch/disposable
    verification page is fine for figuring out *whether* something works
    during a session, exactly as before — but once it works, the assertion
    that proved it belongs in the permanent suite before the commit, not
    deleted with the scratch file (2026-08-11 — the Budgets feature's own
    25 checks and three rounds of circle-redesign screenshots shipped this
    way and left zero permanent coverage behind; see "Testing before you
    claim it works" for the full diagnosis).

11. **Every app on this origin shares ONE localStorage quota, and a new tool on
    this domain inherits it.** An origin is scheme + host + port; the path is
    not part of it, so `/Money-Ledger/`, `/Hours-Ledger/` and any tool added
    later under `groundwork-tools.github.io` read and write the same ~5 MiB
    store. (Measured in headless Chrome: 5,242,880 characters, key + value. The
    phone that reported the bug read 5,169 KB when full in DevTools units of 2
    bytes per character, roughly half as many characters as Chrome's limit;
    that is the phone's report, not re-measured in WebKit.) Why this is a rule:
    one app filling the store breaks the others. On 2026-10-10 Hours Ledger's
    undo history (4,340 KB of that 5,169 KB) left this app unable to save, and
    the screen said "Expense logged" while it was not stored (General fixes
    2026-10-10 below). What follows:
    - **Size the worst case before persisting anything new**, and say who else
      pays for it. Anything that grows with use needs a cap or pruning rule
      *when it is written*. This app's own spend is ~73,000 characters (the
      phone: 143 KB in DevTools units); it keeps no undo stack, so it is the
      smaller tenant - it is starved by the other app far more than it starves
      it.
    - **A write can fail, so failure is a code path**: handle
      `QuotaExceededError` on EVERY localStorage write, including the ones at
      load (the device-id write threw out of the whole script on a full origin
      until 2026-10-10), and never show a success message for something that did
      not store (`toastIfSaved()`).
    - **A new tool is not isolated by its own path or repo.** To get its own
      quota it needs its own origin (another host or a custom domain). Until
      then it shares this budget. Keep Hours Ledger's matching rule (its rule
      10, which carries the current numbers and the undo-cap reasoning) in step.

## Sync design decisions

**Same-record conflicts: last-write-wins, not keep-both (2026-08-04).**
When the same record has been edited differently on two devices, both
still live (not a delete-vs-edit case — that's the separate
never-delete-on-ambiguous bias, which stays), the later edit wins
outright and the earlier one is discarded. This was originally built as
keep-both-flagged-for-review; deliberately changed after live testing,
for a reason worth preserving so it isn't re-litigated the next time
someone eyes the "what if we lose an edit" question:

Editing the same record on two devices while one hasn't synced yet is
rare for a personal ledger, and when it happens both edits are usually
real, deliberate decisions rather than noise — there's no principled way
to guess which one the user actually meant to keep. Discarding the
older one and moving on costs one edit, occasionally; a conflict-review
UI for something that happens a couple of times a year isn't worth the
complexity it would add everywhere else.

If this ever needs revisiting — e.g. multi-user ledgers, or edits
frequent enough that losing one actually stings — reopen it as a new
decision, don't just quietly add complexity back.

**No user-facing notice when last-write-wins discards an edit
(2026-08-04).** The first version of the decision above required a
toast whenever a device's own edit got superseded, on the theory that
"silent" was the unacceptable part, not the resolution. Reconsidered
after actually testing it: this is a personal tool where the point is
to think as little as possible while using it, and a notification that
fires maybe once a year is noise Sebastian will have forgotten the
meaning of by the time it appears. The resolution is correct either
way — there's no decision here he'd need to be told about, because
last-write-wins is what he'd have chosen himself. `mergeRecords` still
counts superseded edits and `syncNow` logs the count to the console,
but nothing reaches the UI. If this changes — e.g. sync becomes
frequent enough that losing edits silently actually causes confusion —
reopen it rather than quietly adding the toast back.

**Seed category resurrection — fixed 2026-08-30 (two parts, one
principle).** A deleted seed category could come back from the dead and
propagate the resurrection to every device via Drive. Reported by
Sebastian after opening the app on `localhost` with a wiped
`localStorage`; diagnosed and reproduced in the self-test harness before
any fix. Mechanism: a device with no stored ledger re-runs
`defaultState()`, which recreated all 12 `SEED_CATEGORIES` as **live**
records. The bug was in what timestamp they got:

- **Part 1 — seeds were stamped `updatedAt: nowIso()`.** "Now" always
  beats a months-old tombstone, so when the re-seeded device synced
  against a Drive file where a seed category had been deleted long ago,
  `mergeRecords`' delete-vs-live rule (`newerSide` → the fresh side wins)
  picked the live re-seed over the real deletion — then `syncNow` pushed
  that un-deletion back to Drive. Fix: seed categories are stamped
  `updatedAt: SEED_UPDATED_AT` (`new Date(0).toISOString()`, matching the
  "treat pre-sync data as very old" convention already in `loadState`),
  so a pristine re-seed can never out-rank a real tombstone *or* a real
  edit. `sameContent` ignores `updatedAt`, so two genuinely-fresh devices
  still merge to one copy each (no duplication — the existing test 10 /
  15 / 16 behaviour is unchanged); the epoch only ever matters when
  content genuinely conflicts, and there a default seed *should* lose.

- **Part 2 — `DEVICE_ID` survived a state wipe, so `remoteIsMine` fired.**
  When `localStorage` is *corrupt* rather than absent (`loadState`'s
  `catch`), or the state key alone is cleared, the separate
  `money-ledger-device-id` key can survive. The re-seed's records then
  carry this device's own id, `mergeRecords`' `remoteIsMine`
  short-circuit trusts them outright, and the timestamp check (Part 1's
  fix) is bypassed entirely. Fix: `loadState()` calls a new
  `regenerateDeviceId()` on **both** fallback paths (`!raw` and `catch`).
  Rationale — one coherent principle: *a device that has lost its entire
  ledger has genuinely lost its identity*, and a fresh `DEVICE_ID` there
  is exactly what that field exists for (telling "my own stale push"
  from "another device"). On a normal first install the id generated
  moments earlier has no meaning yet, so replacing it costs nothing.

Rejected alternatives: (A) Part 1 only — leaves the corrupt-storage path
open on data already proven risky. (C) an explicit "a pristine seed never
wins a merge conflict" guard inside `mergeRecords` — adds a special case
to a deliberately generic core. Both parts have fail-first regression
tests (16b: cross-device re-seed vs. old tombstone; 16c: corrupt-storage
re-seed vs. own-id tombstone), each watched to fail against pre-fix code
and to fail with only one of the two fixes applied. **Not addressed
here:** whether Sebastian's real Drive file still holds resurrected
categories from the incident — that's a separate cleanup question,
pending confirmation of remaining exposure.

## Budget design decisions

**Income is a plain counter, never measured against anything (2026-08-11).**
The three circles on the Budget screen (income, net, expenses) don't share
one "measure everything against the total budget" rule, even though that
was the first design on the table. Income specifically doesn't get a
budget field, doesn't get a predicted/actual split, doesn't turn red — it
just shows what's been logged. Reasoning, reconstructed from the actual
conversation: budgeting is optional everywhere else in the app (a category
with no budget just shows a plain total), and forcing income through the
same "predicted vs actual" frame would mean either inventing a new
"income target" setting nobody asked for, or making the circle empty and
alarming-looking early in the month before payday, for no good reason.
Expenses and Net *do* gain a second "predicted" line, but only once at
least one expense category has a budget set — the same graceful
degradation as the category bars, not a separate mode to turn on.

**Predicted net = actual income so far − total expense budget, not
predicted income − predicted expenses (2026-08-11).** A real alternative
was on the table: give income categories budgets too, and define predicted
net as planned income minus planned expenses — a pure, static plan number.
Rejected in favor of a live number grounded in what's actually been
logged, because that's this app's whole ethos (log reality, don't just
plan) — and because it was resolved by a *better* idea that came up in the
same conversation: predictable income (a monthly paycheck) isn't a
budgeting problem at all, it's a "don't make me retype this" problem,
which is exactly what recurring subscriptions (backlog #3) already exists
to solve. Income will get *automated*, not *budgeted* — see that backlog
item. Once recurring income ships, this formula doesn't need to change;
the income circle already just reads whatever's been logged, automatically
or by hand.

**Category bars fill up as you spend, not drain down (2026-08-11, revised
same day).** Originally built as a depleting gauge — full/bright at zero
spend, draining down as spend accrues, pinning back to a full red bar once
over budget rather than shrinking to invisible. Reconsidered after seeing
it on screen: the fill-up direction is the more familiar "progress toward
a limit" pattern (it's what the Summary panel's spent-vs-income bar and
the budget circles' rings both already do), and running two opposite
metaphors for the same underlying concept — spent vs. a limit — on one
screen read as inconsistent rather than deliberate. Now: empty at zero
spend, filling as you spend, pinning at 100% width in `--flag` red once
spend exceeds budget (strictly `>`, not `>=` — hitting the budget exactly
still reads as normal, not over). The number stays honest and uncapped
(`2,340 / 2,000`), so the bar never has to encode *how far* over — the
text already says that precisely.

**Budgets live on their own screen, not inline in the category rail
(2026-08-11).** First idea was reusing the existing colour-picker popout
in the rail to also hold a budget field. Rejected by Sebastian: a swatch
button reads as "colour," and budgeting is too central to the feature to
hide behind an affordance advertising something else. Setting a budget is
described as "a monthly sit-down, not something I do mid-logging" — it
earns a dedicated screen. The main register only *reads* budgets (the
depletion bars); it never lets you set one, and the Budget screen never
lets you rename/recolour/delete a category — that stays on the register's
Categories panel, one job per surface.

**Budget circles are SVG progress rings, not text-in-a-circle
(2026-08-11).** The original build put a full sentence ("of 9,100.00 kr.
planned") inside a plain bordered circle — a circle is a bad container for
text, the words crowd the INCOME/NET/EXPENSES label out once a number
appears, and the border was too thin to read as a deliberate shape.
Rewritten so the ring itself carries the data: an 8–12px arc on a pale
track, filling clockwise from the top. Inside the ring: one compact number
only, no currency symbol, no decimals (`2,704`, via a dedicated
`formatCompact`, distinct from the app-wide `formatMoney` rule below — a
headline figure is allowed to be less precise than a ledger figure). The
context moved to an uppercase mono caption *below* the ring, not fighting
for space inside it.
- **Colour, corrected same day.** First pass coloured income green and
  expenses red-when-under-budget-too, matching entry-row conventions.
  Sebastian's own reaction: seeing the expenses circle in warning-adjacent
  red while comfortably under budget read as "I've overspent" even when he
  hadn't. Changed to: all three circles neutral ink by default; Expenses
  turns `--flag` red only once spend exceeds its total budget; Net turns
  `--flag` red only when negative; Income never turns red and is no longer
  green either — colour on this screen now means exactly one thing
  (over/under, or positive/negative), not "which direction is this money."
- **Income and Net rings have no natural fill percentage.** Income isn't
  measured against anything (see above), and Net's only spec was "reads
  from its sign" — neither has a real denominator to fill proportionally
  against. Both rings are therefore a presence indicator, not a
  proportional gauge: full ring once there's any relevant activity this
  month, ghosted (dimmed, dashed "—" instead of a number) before that.
  Income ghosts on its own zero; Net ghosts when there's been no income
  *and* no expenses at all. This is an interpretation of "fills as income
  lands" from the original request, not a literal spec — worth re-raising
  if it stops feeling right in daily use.
- **Expenses ghosts on its own zero too, same as Income (2026-08-11,
  corrected same day).** Originally built as "never ghost Expenses — a
  known spent amount is real data, and 'no budget set' shouldn't look the
  same as 'nothing logged.'" That reasoning is still right for the case it
  was written for (spend something, no budget → show the real number,
  don't ghost) — but it accidentally also covered the *other* case, spend
  *nothing* at all, where a solid "0" is exactly as ambiguous
  ("deliberately zero" vs. "haven't opened the app") as it would be for
  Income, and Income already ghosts for exactly that reason. Caught
  because it left the three circles visibly inconsistent — Income and Net
  faded with a dash, Expenses solid with a "0" — when nothing had been
  logged. Fixed to mirror Income exactly: Expenses ghosts when
  `expenses === 0`, full/unghosted the moment anything's been spent,
  independent of whether a budget is set. The "no budget ≠ nothing
  logged" distinction still holds — it just now only matters once
  `expenses > 0` is already true.
- The empty-state CSS class was almost named `.ghost`, which collides with
  the app's existing `.ghost` button utility class (`border:1px solid
  var(--rule)`, used on "Budget," "Connect Google Drive," etc.) — caught
  by screenshotting the empty state and seeing a stray box around the
  ring. Named `.ring-empty` instead. Worth remembering if a future class
  name reuses a common word like "ghost," "active," or "empty" — check
  for an existing global utility of the same name before reusing it.
- **Phone-width layout is a triangle, not a shrunk copy of the desktop row
  (2026-08-12).** Below 700px, Net (the larger, derived circle) moves
  above Income and Expenses rather than staying in a horizontal line with
  them — `.budgetcircles` switches from flex to a two-row CSS grid
  (`"net net" / "income expenses"`) purely inside that one media query.
  Desktop's flex row is untouched — the grid rules only exist below the
  breakpoint, so there's no risk of the phone layout's structure leaking
  upward. Verified by rendering the iframe at both a 390px and a 1000px
  width and screenshotting each, rather than trusting the media query
  alone — OS-level window resizing in this dev environment doesn't
  reliably reflect in a screenshot, but an iframe's own width always
  drives its internal viewport correctly, which is why that's the
  technique to reach for first when a real device isn't at hand.
- **The triangle grid's rings were misaligned on phone width, fixed
  2026-08-12.** Sebastian noticed Income's ring sitting lower than
  Expenses' ring. Cause: `.budgetcircles` inherits `align-items:center`
  from the desktop rule (never overridden in the mobile grid), and
  Expenses' box is taller than Income's whenever a budget is set — it has
  an extra `.bcircle-capsub` caption line ("· Budget X") that Income never
  gets. Centering the shorter Income box within a grid row tall enough
  for Expenses' extra line pushed Income's ring down relative to
  Expenses'. Fixed with `align-items:start` inside the mobile media query
  only (top-align both rings regardless of caption height difference) —
  desktop still isn't touched, same as the rest of this entry. Also
  tightened the grid's row-gap (22px → 14px) per Sebastian's ask to bring
  Income/Expenses closer to Net. Verified on screen with a budget actually
  set on an expense category, since the misalignment only shows up once
  Expenses' extra caption line exists — an empty-budget screenshot
  wouldn't have caught it.

**Net's predicted line was removed entirely (2026-08-11, revised same
day).** The circle rebuild above shipped with "NET · PREDICTED 2,700" as a
caption, mirroring Expenses' "· OF 9,100." After using it on screen,
Sebastian asked for it gone — it read as clutter under the ring, not
useful context. The formula and reasoning documented above ("Predicted net
= actual income so far...") still holds and the number is still computed
(it feeds the category-list total, and could resurface elsewhere later),
it's just no longer shown on the Net circle. Net's caption is now always
just "Net," full stop — no conditional, no second line.

**Expenses caption is two lines, "Expenses" then "Budget 9,100," not one
line with "· of" (2026-08-11).** Small wording/layout change after seeing
the single-line "EXPENSES · OF 9,100" on screen — splitting the label from
its context onto two lines reads more clearly than one long uppercase
mono line, and "Budget" is more explicit than "of." Implemented as a
`.bcircle-caption` (main label) plus a separate `.bcircle-capsub` (second
line, only populated when a budget exists) — Income and Net don't get a
sub-line at all.

**Category-budget list gets a "Total budgeted" footer row
(2026-08-11).** A plain sum of every expense category's `budgetMinor`
(categories with no budget contribute 0), shown once at least one expense
category exists — same `totalBudget` figure the Expenses circle already
computes, just surfaced as its own row so it doesn't require mental
addition down the list. Bold, top-bordered, styled as the list's natural
last row rather than a separate panel.

**Budget circles' arc colour is user-customizable, same hue-picker as
categories (2026-08-11).** Originally scoped out same-session ("skip it
for now") on the reasoning that the neutral colour scheme removed most of
the motivation — reopened minutes later once Sebastian saw the neutral
version on screen and wanted to try it anyway. Each of the three circles
gets its own swatch button (next to its caption) opening the exact same
picker component the category rail uses (`.picker`/`.presets`/`.hue`,
literally the same CSS, wired once in `initBudgetView()` since the three
circles are static DOM rather than per-category rebuilt rows). Stored as
`Settings.budgetRingColors: {income, net, expenses}`, each either `null`
(default) or `{hue, color}`, mirroring how `Category.hue`/`color` are
stored together. **The one hard rule a custom colour can't override:**
Expenses still forces `--flag` red when over budget, and Net still forces
it when negative, regardless of any custom colour set — red keeps meaning
exactly one thing on this screen (the neutral-colour decision above), even
after letting the base colour be customized. A custom colour replaces the
default *ink*, not the warning state. **Colour lives on the arc only, never
the number (revised same day).** The first version also recoloured the
number text inside the ring — Sebastian asked for the number to always
stay plain ink, with colour (custom or flag-red) carried by the arc alone.
`applyRingColor()` no longer touches the `.bcircle-num` element at all,
only `.bcring-arc`; the now-unused `.bcircle-num.flag` CSS rule was
removed. This field is additive/optional on
`Settings`, read defensively (`state.settings.budgetRingColors || {}`)
rather than via a formal migration script — same lighter-weight approach
already used for `Category.budgetMinor` when that field was added to
already-live category records, since `undefined` and `null` are handled
identically by the `!= null` checks throughout.

**Amounts hide decimals when they're whole, everywhere, including editable
fields (2026-08-11, revised 2026-08-12).** `formatMoney` passes
`minimumFractionDigits: 0` when the amount has no minor-unit remainder, so
`4,250 kr.` displays instead of `4,250.00 kr.` — applies to the register,
the summary panel, the category rail, and the budget screen. Editable
input fields (the edit-entry amount field, the per-category budget input,
the recurring amount field) originally kept a fixed two-decimal string via
`.toFixed(2)` on purpose, reasoning that a field you're about to type into
benefits from a stable, predictable format more than a display-only
figure does. Reversed the next day after direct feedback: showing `.00`
on an amount that's never had decimals just reads as wrong, not stable —
the "predictability" argument didn't hold up against actually looking at
it. Now uses a shared `amountInputValue(minor)` helper (hide decimals
when whole, keep them when genuinely fractional — e.g. `4000` but
`1.99`), used by all three editable amount fields instead of each calling
`.toFixed(2)` separately.

**Budget input selects its whole value on focus (2026-08-12).** Tapping
into the per-category budget field used to drop the cursor wherever the
tap landed within the (right-aligned) value, so clearing it to type a new
number meant first manually navigating to the far side. Now the field's
`focus` handler calls `.select()`, so any tap/click selects the entire
value — typing immediately overwrites it, and Backspace/Delete clears it
in one press, no manual cursor navigation needed. Scoped to the budget
field specifically, since that's the one this was reported against — the
edit-entry and recurring amount fields weren't asked for and weren't
changed; revisit if the same friction shows up there too.

## Recurring design decisions

**Covers income as well as expenses, on one screen, split by the same
Spent/Income toggle used everywhere else (2026-08-11).** The backlog
originally described this as "recurring subscriptions" — expense-only by
implication. Scoped wider before building anything: "Budget design
decisions" had already committed to recurring entries being the answer to
predictable income (a paycheck), specifically as the alternative to
budgeting income categories (rejected — see "Out of scope"), so building
expense-only now would have meant reopening this feature almost
immediately. One screen, not two competing header buttons, because it's
one underlying record type either way — but never one mixed list: every
other direction-aware surface in this app (quick-add, the category rail,
`Category.direction` itself) splits expense/income into two separate
views because a mixed list reads as confusing, and "Netflix −89 kr." next
to "Paycheck +28,500 kr." in one list would be exactly that. Named
"Recurring," not "Subscriptions" — the latter doesn't linguistically cover
a paycheck, and "Recurring" already matched `Entry.recurringId`, a field
name that (in hindsight) had the right instinct before the screen's scope
was even discussed out loud.

**Auto-insert is silent plus a toast, not a manual confirm step
(2026-08-11).** A real alternative was on the table: surface "Rent, 6,250
kr., due today — log it?" and wait for a tap, closer to the app's "log
reality, don't infer it" ethos (see the bank-auto-import rejection in "Out
of scope"). Rejected because it reintroduces the retyping-adjacent
friction this feature exists to remove — if every recurring entry still
needs a tap, declaring it once saved nothing. The distinction from
bank-linked auto-import: a bank feed pulls from an external, untrusted
source that might not reflect what you actually intended; a Recurring
record is something *you* explicitly declared once ("yes, Spotify is 89
kr. on the 5th, every month") — auto-inserting from your own prior
declaration isn't inference, it's not making you retype something you
already told the app was going to happen. The toast keeps it from being a
silent surprise without requiring a tap either.

**Auto-insert only ever checks the current real-world month — it does not
backfill months missed while the app was closed (2026-08-11).** If you
don't open the app for two months, you get the most recent month
backfilled next time you open it, not both. Considered checking every
month between "last opened" and "now" instead, but rejected: silently
bulk-inserting several months of history the first time a long-standing
subscription gets declared felt like more surprise than this feature
should introduce, not less. Revisit if missed months turn out to be a
real annoyance in practice — the pure `dueRecurring(recurringList,
entries, today)` function only checks one month by design, so extending
it would mean changing that function's contract, not just its caller.

**Editing or deleting a Recurring record never touches entries already
auto-inserted from it.** Same principle as editing a Category never
rewriting an existing Entry's stored fields — once an entry exists, it's
independent history reflecting what was true at the time it was created,
not a live reference to the Recurring record's current values. Deleting a
Recurring record is a tombstone (same as Entry/Category), stops future
auto-insertion, and leaves every entry it already produced untouched — no
exception carved into hard rule 6 for this record type, it just already
satisfies it by construction.

**Recurring records sync via the same per-record merge as Entry/Category,
not local-only.** Consistent with the phone-to-log/laptop-to-review
workflow this whole app is built around — a subscription declared on one
device should apply on the other without re-declaring it. Uses the same
tombstone shape (`updatedAt`/`updatedBy`/`deleted`/`deletedAt`) and the
same `mergeRecords` algorithm, keyed by `RECURRING_CONTENT_FIELDS`
(`name`, `amountMinor`, `direction`, `categoryId`, `dayOfMonth`, `deleted`,
`deletedAt`) — no new merge logic needed, `mergeRecords` was already
generic over "a collection of tombstoned records," proven by Entry and
Category both already using it.

**The Recurring screen reuses the register's actual components, not just
its "feel" (2026-08-11, revised same day).** First build was a standalone
layout: toggle above a flat list, "Add recurring" form below the list in
a bespoke `.addrecurring` box. Reworked after Sebastian asked for it to
read as the same screen family as the front page specifically: the add
form now literally reuses `.quickadd`/`.qa-field` — the same striped
green box, the same labeled-field layout, direction toggle included as a
`.qa-type` field exactly like quick-add's own — moved *above* the list to
match quick-add sitting above the register. The list reuses `.register`
and `.dayhead` directly: recurring items group by day-of-month under a
day header showing that day's total, the same rhythm as the register
grouping entries by day. A `.summary`-styled box (reusing that component
too) shows total recurring income and total recurring expenses — both
figures always, regardless of which side the Spent/Income toggle is on,
since the ask was "a total of recurring income and expenses somewhere,"
not a per-view number. Deliberately did **not** merge the Spent/Income
toggle away even though the register itself doesn't split by direction —
that's a separate, already-settled decision (see the "one screen, split
by toggle" entry above); borrowing the register's visual/interaction
patterns doesn't mean re-litigating what those patterns are applied to.

**The list rows themselves went further — reusing `.entryrow` outright,
not just the surrounding `.register`/`.dayhead` shell (2026-08-12).** The
list-level reuse above still left each *row* as its own bespoke design
(`.recrow`: a dot, an inline-editable name input, an inline-editable
amount input, a category select, a day select, and a delete "×", all in
one flex row) — fine at desktop width, but on a phone it wrapped onto two
lines, which Sebastian flagged as broken-looking. He asked to copy the
register's row design specifically: category on top, the item's own name
as a muted note line underneath, one line tall, editing moved into a
popup triggered by clicking the row. Built exactly that: each row is now
a plain `.entryrow` button (the literal class the register's entries use,
with `r.name` filling the `.note` slot a register row uses for its own
optional note), and clicking it opens `#editRecurringScrim`, a new sheet
that reuses `#editScrim`'s markup and CSS wholesale — a Name field
standing in for Note, a Day `<select>` standing in for Date. No new CSS
for either the row or the sheet; both fully inherit their look from
classes the register/entry-edit sheet already defined. The old
`.recrow`/`.rec-*` CSS was removed entirely, not left dormant, since
nothing renders that markup anymore. See "General fixes (2026-08-12)" for
the testing side of this change.

**Start month and pause (2026-08-22).** The gap: a newly-declared
Recurring item had no way to say "don't start logging this until next
month" (e.g. signed up but not charged yet), and no way to temporarily
stop auto-insert without deleting the record and losing its history/day/
category setup. Scoped through a batch of clarifying questions before any
code — see the conversation this was decided in for the full set — landing
on:
- `Recurring.startMonth: "YYYY-MM"` and `Recurring.paused: boolean` added
  to the record, both real content fields in `RECURRING_CONTENT_FIELDS` so
  they sync via the same per-record merge as every other field — no
  special-case handling, confirmed deliberately rather than skipped.
- **Default start month is the current month**, same as how the day field
  already defaults to "today" — you flip it forward yourself for a
  not-yet-charged subscription, rather than every ordinary item needing a
  manual step back to "now."
- **Pause is a single indefinite toggle** (not a lighter "skip next month
  only" action) — stops future auto-insert without deleting the record,
  resumable with one action.
- **Paused/not-yet-started items are excluded from the Recurring screen's
  summary totals and from their day-group total** — both describe what's
  currently actually going to auto-insert, not everything ever declared.
  The two totals were deliberately kept consistent with each other (a day
  total disagreeing with the summary about what "active" means would be
  confusing) rather than only fixing one.
- **They still appear in the list itself**, marked inactive (dimmed via a
  new `.entryrow.inactive{opacity:.55}`, same "still real, just not
  currently doing anything" language as `.bcircle-ring.ring-empty`) with a
  status appended to the note line — "· Paused" or "· Starts Aug 2026" —
  rather than a second line, keeping the row the one-line-tall shape from
  the redesign above. Hiding them entirely was considered and rejected —
  Sebastian's call was that a declared-but-inactive item is still
  information worth seeing, not noise to sweep away.
- **Start month is editable anytime** through the edit sheet, not
  creation-only — same as every other field on the record.
- **Resuming a paused item triggers the due-check synchronously in the
  same click**, not deferred to the next app open — a deliberate small
  divergence from how a brand-new item behaves (creation doesn't
  synchronously insert even if its day has already passed this month; it
  waits for the next `applyDueRecurring` run). Sebastian's explicit
  reasoning: a silent multi-day gap before a resumed item's entry shows up
  has the same "did this actually work?" problem the original "auto-insert
  is silent plus a toast, not a manual confirm" decision above was
  designed to avoid. Pausing mid-month, by contrast, leaves that month's
  already-auto-inserted entry completely untouched — consistent with
  editing/deleting a Recurring record never touching entries already
  produced from it (see above); pausing is just another field edit.
- **No new risk to the deterministic-entry-id fix**
  (`recurring:<id>:<year>-<month>`, 2026-08-21): pause/startMonth only
  narrow `dueRecurring`'s eligibility filter — they never touch
  `recurringInsertId` or the entries table, so the one-entry-per-
  recurring-per-month invariant that id relies on holds by construction,
  not by new code defending it.
- **No formal migration** for the two new fields — same lighter-weight
  defensive-read precedent as `Category.budgetMinor` and
  `Settings.budgetRingColors`: a record with no `paused`/`startMonth` key
  at all (every record that existed before this shipped) reads as "not
  paused, no restriction," which is exactly the correct default for data
  that was already running before this feature existed. Verified as a
  standing regression case in the self-test suite, not just asserted in
  this doc.

## Category filter design decisions (2026-08-28)

**An opt-in category filter on the register and Recurring screen, replacing
an earlier "auto-group same-category entries" idea.** The real problem was
scanning the register for everything under one category when entries aren't
adjacent. Auto-reordering was rejected — it fights the app's "log reality,
don't infer" ethos. Instead: a filter layered on top, changing what's
shown, never what's logged or its order. Design settled through a batch of
clarifying questions before any code — see the conversation for the full
set. What landed:
- **UI is an inline chip row** (`.catfilter`), a wrapping row of
  pressed/unpressed pills above the register list (and the Recurring
  list), reusing `.dirtoggle`'s pressed language as individually-bordered
  chips with a category-colour dot. Chosen over a popout panel (less
  discoverable) and a rail-embedded control (rail rows are already fully
  allocated to rename/recolour/delete). A leading "All" pill clears back
  to baseline.
- **View-state only, zero schema/sync change.** Two module-level `Set`s
  (`registerCatFilter`, `recurringCatFilter`), reset on reload, never in
  `State`/`localStorage`/`mergeRecords` — exact same discipline as
  `viewDir`/`recurringDir`. Confirmed explicitly: no `Entry`/`Category`/
  `Recurring`/`Settings` field, no migration. `resetState()` in the test
  hook clears them too, for test isolation.
- **Switching the Spent/Income toggle clears the active filter** — expense
  and income categories are disjoint, so a carried-over selection could
  only ever resolve to nothing. Not per-direction memory; just cleared.
- **The chip row offers only categories with ≥1 entry this month in the
  current direction**, recomputed on every render — so month-nav always
  re-derives the offered set from the new month (never stale), while the
  filter `Set` itself is untouched by month-nav (navigate away and back,
  the selection is still there). A still-selected id whose category has
  nothing in the current month simply has no pill and contributes to the
  zero-match empty state.
- **Day/week headers collapse when a filter is active.** Unfiltered, they
  keep the existing `IN … · OUT …` both-direction totals (the 2026-08-26
  decision stands). Filtered, they show a single figure of the visible
  rows, labelled with the existing IN/OUT vocabulary for whichever side is
  being viewed — `OUT <sum>` in Expense view, `IN <sum>` in Income view
  (2026-08-30; first shipped as a generic `Selected <sum>` prefix, changed
  because the filter's categories are already direction-scoped, so this is
  just the same concept the header already had a word for — the other side
  of a full `IN … · OUT …` would only ever be zero). Recurring's day-group
  total gets the same IN/OUT prefix.
- **Summary panel and the category rail's own totals stay unfiltered** —
  same rule as the monthly category table being unaffected. The rail
  *does* lightly mark selected categories (`.cat.selected`, an inset ink
  bar + bold name) as a passive echo of the chip row — no second control,
  the rail never *sets* the filter.
- **Recurring's summary box is unaffected**; its list and day-group totals
  follow the filter. Categories used only by paused / not-yet-started
  Recurring items are still offered as chips — they're visible in the
  list, so they should be filterable.
- **Zero-match empty state:** "No entries in the selected categories this
  month." with the chip row still visible and a `Show all categories`
  clear button. Same for Recurring.
- **Verification:** 14 self-test cases (register direction+category
  compose, multi-select, chip eligibility, header collapse vs. the
  unchanged IN/OUT baseline, direction-clears-filter, zero-match empty
  state, rail highlight with rail totals staying unfiltered, Recurring
  parity, paused-item category still filterable, month-nav re-derivation,
  view-state not in `State`). Suite: 265 tests / 0 failing. Screenshotted
  on-screen (headless Chrome via CDP, iframe/emulation widths) at 390px
  and 1280px, both screens, filtered and unfiltered. **Not yet
  real-device-verified by Sebastian** — built on a branch
  (`category-filter`), pending a phone check at 390px before merge.

## Quick-add design decisions

**The post-log amount-field auto-focus is suppressed on phones only
(2026-09-01).** After a successful quick-add submit, the handler clears
the amount/note fields and calls `document.getElementById("qaAmount")
.focus()` so a run of entries can be typed one after another without
reaching for the mouse (`initQuickAdd`, the `form` submit listener). On a
laptop this is exactly right and stays untouched. On a phone it's a small
irritation: it pops the soft keyboard back up the instant you've logged
something, right when the on-screen "Log it" button is the close, natural
thing to tap for the next entry — the keyboard is in the way, not helping.

- **Detection is `matchMedia("(max-width: 1000px) and (pointer: coarse)")`
  — both conditions, AND.** Extracted as `isTouchPhone()` (the codebase's
  first `matchMedia` call — every breakpoint before this lived only in
  `style.css`), consumed via a one-line `maybeRefocusAmount()` wrapper so
  the guard has a single home and the self-test has a single seam. Only
  the post-submit call at the end of the submit handler is guarded.
- **Why AND, not width-only or touch-only.** The two failure directions
  are not equally costly. A false positive — a laptop mistaken for a phone
  — silently kills the rapid keyboard-entry workflow Sebastian actually
  relies on, and the hard requirement on this change was "keep laptop
  exactly as-is." A false negative — a phone mistaken for a laptop — just
  restores today's mild annoyance. AND-ing width and pointer biases every
  uncertain device toward the laptop behavior, which is the safe
  direction. Width-only would suppress auto-focus on a narrow or
  split-screen desktop window (a real laptop case, wrong direction).
  Touch-only would misfire on touchscreen laptops and 2-in-1s in tablet
  mode (also a real laptop case). The combined test only "misfires" on a
  narrow touch device with no mouse that is genuinely a laptop — i.e. a
  tablet, where tapping the button really is the natural gesture anyway.
- **Why 1000px.** Reuses the app's existing primary breakpoint (`.cols`
  collapses to one column there). Because it's AND-ed with `pointer:
  coarse`, being generous on width is safe — a coarse-pointer device up
  to 1000px wide is a phone or small tablet, both of which want the
  no-keyboard-pop behavior, and this width still catches a phone held in
  landscape. The tighter 700px breakpoint would miss landscape phones for
  no benefit.
- **The "Today" button and the FAB keep focusing the amount field
  unconditionally, phone included** — deliberately not changed. Both are
  "I'm about to type an amount" gestures: you tap "Today" mid-logging to
  reset the sticky date for a backdated batch, and the FAB exists solely
  to jump you into logging. In both, the keyboard appearing is the next
  thing you wanted anyway. The post-submit case is the only one where
  "maybe I'm done now" is plausible, which is what singles it out.
- **No schema / storage / sync / migration change** — a pure runtime
  behavior gate, nothing persisted.
- **Verification:** self-test cases asserting `.focus()` *is* called with
  a mocked non-phone `matchMedia` and is *not* called with a mocked phone
  one (stubbing the iframe's `matchMedia`, spying on `qaAmount.focus` — no
  `activeElement` assertion, since the hidden test iframe can't hold real
  browser focus so that check couldn't tell a pass from a failure), plus
  a truth-table test of `isTouchPhone()` across the four width/pointer
  combinations. Watched to fail against pre-change code two ways: the new
  tests on the old `app.js` trip the exposed-function guard (`1 failing`,
  same pattern as tests 94–96), and deleting just the `isTouchPhone()`
  guard line while leaving the hook exposed fails the two behavioral
  assertions ("phone → focus() NOT called"). Suite 325/0.
  **Confirmed on the real device (2026-09-01):** merged to `main` and
  deployed. Phone — logging an entry no longer pops the soft keyboard,
  the on-screen "Log it" button is right there for the next tap. Laptop —
  rapid keyboard entry unchanged.

## General fixes (2026-08-12)

**Recurring items now render as one-line `.entryrow` rows with editing
moved into a popup sheet** — see "Recurring design decisions" above for
the full reasoning; this note is just the verification record. Verified
via the self-test suite (rewrote tests 49 and 54, which asserted against
the old `.recrow` markup and would have false-failed otherwise, now 162
tests/0 failing) and a DOM-structure dump confirming the new rows carry
`flex-wrap: nowrap` (no wrap rule exists for `.entryrow` at any width)
and that zero `.recrow` elements remain in the rendered output. **Not
confirmed by an actual on-screen screenshot this session** — repeated
attempts to screenshot a real Chrome window hit the same OS-level
flakiness noted elsewhere in this file (`screencapture` intermittently
captured this coding surface instead of the Chrome window, even after
`activate`), so this one should still be eyeballed on a real phone before
considering it fully done.

**Testing gotcha: the browser's disk cache can serve a stale `app.js`
even after a hard-reload of `index.html` (2026-08-12).** Hit while
verifying the recurring-row redesign above: `index.html`'s
`<script src="app.js">` has no cache-busting query string (only the
self-test harness's *own* iframe `src` is cache-busted, via
`?mltest=1&cb=Date.now()`), so a normal Chrome profile can keep serving a
disk-cached `app.js` from an earlier page load in the same profile even
though the HTML around it is demonstrably fresh — confirmed via `curl`
that the server was returning the new file while the browser tab's
behavior (and a debug DOM dump) still showed old code running. A
Cmd+Shift+R hard-reload of the tab did not fix it either. What did:
running the same test/verification page in a fresh **incognito** window
(`open -na "Google Chrome" --args --incognito "<url>"`), since incognito
uses its own cache separate from the regular profile's. Worth remembering
any time a fix "isn't taking effect" in this local Chrome despite the
served file being correct — check incognito before assuming the app code
is still wrong.

**Testing gotcha: a "fresh incognito window" isn't actually fresh if
another incognito window from the same Chrome launch is still open
(2026-08-22).** While verifying the Budget-list over-budget flag (see
"General fixes" below), the self-test suite showed 6 tests failing —
including two from already-shipped, already-verified fixes, with no
code change that could explain their breaking — across three separate
attempts, each via a newly opened `open -na "Google Chrome" --args
--incognito "<url>"` window. All three actually shared one underlying
incognito session/cache with every other incognito window already open
from earlier in the same conversation, because repeated `--incognito`
launches attach to the *existing* browser process's one incognito
profile instead of each starting an independent one — the opposite of
what "open a fresh incognito window" sounds like it should do, and
exactly the isolation the previous entry's fix depends on. Confirmed
this was a stale-verification-window artifact, not a real app
regression, via Chrome's DevTools Protocol (`--remote-debugging-port`,
driven over a raw websocket with Python) against a Chrome instance
launched with its own `--user-data-dir` — an *actually* separate
profile, unlike another `--incognito` window — which showed 0 failing
against the same code. CDP is also worth reaching for on its own
merits: it reads results straight from the page's DOM/console over the
socket, so it sidesteps the `screencapture`/Space-switching flakiness
noted throughout this file entirely, rather than fighting it. The
general lesson: if a *previously verified* behavior appears to have
broken with nothing in the code to explain it, suspect the verification
window's accumulated state before re-debugging the app from scratch —
close every incognito window and start over (or use `--user-data-dir`)
before trusting a failing result.

**The FAB stays visible on every screen and always does something
useful, rather than hiding on Budget/Recurring.** Sebastian asked which
was the right call: hide the FAB where quick-add isn't shown, or keep it
everywhere and have it take you back to logging. Recommended and built
the latter — the FAB is the one thumb-reachable "log something" button
on a phone (bottom-corner-fixed, unlike the logo which sits at the top of
the page and needs a scroll-up to reach), so making it disappear on two
of the app's three screens would remove the more mobile-ergonomic path
back to logging, not just an alternate one. It now calls the same
`showBudgetView(false)`/`showRecurringView(false)` pair the logo does
before its existing scroll-into-view-and-focus behaviour, so from any
screen it either lands directly in the amount field (already on the
register) or gets you there first (from Budget/Recurring) — it was
previously wired only for the first case, silently doing nothing useful
on the other two since quick-add isn't in the DOM's visible flow there.

**Budget ring colours now actually sync across devices.** Reported by
Sebastian: colours set on his laptop still showed as plain ink on his
phone. Investigated and confirmed real, not a loading issue —
`syncNow()` merged `entries`/`categories`/`recurring` but never touched
`state.settings` at all; nothing ever read `remote.settings` back into
local state, so a device's own settings simply never changed via sync,
regardless of what any other device had pushed. Fixed by adding a
`budgetRingColorsUpdatedAt` timestamp (bumped only when
`budgetRingColors` itself changes) and comparing it in `syncNow()` —
whichever side is newer wins, same last-write-wins rule as same-record
content conflicts elsewhere. **Deliberately scoped to `budgetRingColors`
only** — `currency` was left untouched (still device-local, unsynced,
matching existing behaviour) since silently changing a device's currency
via sync is a bigger, unrequested behavioural change Sebastian didn't
ask for; `driveConnected`/`driveFileId`/`introSeen` must never come from
remote at all, since they describe *this device's* own connection state,
not a shared preference — syncing those would be actively wrong (e.g.
one device's Drive file id overwriting another's independently-resolved
one). See the data model's `Settings` entry for the full field-by-field
sync/no-sync breakdown.

**The toast's Undo button now gets hidden again when the toast dismisses.**
Found by Sebastian as "a weird very small black box sticking" under the
browser chrome. Root cause: `#toastUndo` gets `hidden = false` whenever an
undoable toast fires (delete entry, delete category, clear month) but
nothing ever set it back to `hidden = true` — not on the undo-click path,
not on the natural timeout. So after the *first* undoable action anywhere
in the app, that button stayed permanently un-hidden, just translated
off-screen along with the rest of `.toast`. `.toast`'s hidden position is
`translateY(-140%)` — a percentage of the box's *own* height, not a fixed
distance — so a permanently-taller box (now always including the Undo
button) wasn't pushed quite far enough off-screen, leaving a sliver of
the ink-black background visible. Fixed both ends: `btn.hidden = true` is
now set on both dismiss paths (timeout and undo-click), and `.toast`'s
resting state also gets `opacity:0` + `pointer-events:none` so any future
content-size growth can't reproduce a visible sliver even if the
transform math is ever off again.

**The "Money Ledger" logo is now a way back to the register from
Budget/Recurring.** Wrapped in a `<button>` inside the existing `<h1>`
(valid nesting — button is phrasing content) with a `.wordmark-btn` reset
so it's visually identical to the plain heading it replaces. Calls both
`showBudgetView(false)` and `showRecurringView(false)`, which is safe
regardless of which screen (if either) is currently open.

**Two categories can no longer share a name within the same direction.**
Reported as duplicate categories showing up on Sebastian's phone.
Investigated rather than assumed: `createCategory`/the rename handler had
*no* duplicate-name check at all, and the add-category form had no
double-submit protection — a double-tap on "Add" (easy on mobile with no
loading state on the button) would create two distinct-id records with
the same name before the first submission's re-render even happened, and
since JS event handlers run synchronously, this reliably reproduces with
just two fast taps. Added a case-insensitive, same-direction duplicate
check to both the add-category form and the rename handler — blocked
attempts show a toast and (for rename) revert the input, they don't fail
silently. **This does not retroactively clean up any duplicates already
sitting in Sebastian's real synced data** — old duplicates predating this
fix (or predating the 2026-08-03/04 sync-merge fixes, if that's actually
where these came from — hard rule 6 was written *because* of exactly this
class of bug) are still there and weren't touched, since deleting/merging
real category data isn't something to do silently without Sebastian
confirming which records are the actual duplicates first. **Confirmed
resolved (2026-08-12):** Sebastian deleted the existing duplicates
himself on his phone and they stayed deleted through subsequent syncs,
rather than reappearing — real-device confirmation that both halves hold
now: the prevention (no new ones form) and the underlying delete/tombstone
sync path (a real device's delete sticks, doesn't get resurrected by a
merge). No separate cleanup action was needed beyond the fix itself.

**The budget field shows a pretty formatted value at rest (thousands
separator + currency, e.g. "3.333 kr."), and the plain editable number on
focus.** `type="number"` can't hold formatted text — a number input
rejects any value containing non-digit characters like "." as a thousands
separator or "kr." — so this required switching `.bcat-budget` to
`type="text"`. Editing always happens on the plain number (via the same
`amountInputValue` used for the pre-fill); only the resting/blurred
display uses `formatMoney`. Implementation note for future editable-field
formatting like this: `blur` (not `change`) is what restores the pretty
display, and it runs unconditionally on every blur — `change` already
triggers `renderAll()`, which tears down and rebuilds this exact row when
the value actually changed, so by the time `blur` fires afterward it's
acting on an already-detached node (harmless no-op); when nothing
changed, `change` never fires at all, and `blur` on the still-live node
is the *only* thing that restores the formatting. Relying on `change`
alone would leave the field stuck showing the plain number after a
focus-then-blur-without-editing. Reused verbatim for the recurring
amount field (`.rec-amount`) the same day, once Sebastian asked for the
same treatment there — same `type="text"` swap, same
focus/blur/change trio, same reasoning throughout.

## General fixes (2026-08-26)

**The main register gets a Spent/Income toggle, matching the pattern
already used on the Recurring screen — but consolidated with the existing
Categories-panel toggle rather than added alongside it.** Before this, the
register mixed both directions in one list, and a *separate* Spent/Income
toggle (`railDir`) already existed on the Categories panel, filtering only
that panel's rail. Sebastian confirmed a batch of design questions before
any code:
- **The toggle is unified, not duplicated.** One `.dirtoggle` (renamed
  internally from `railDir` to `viewDir`, since it now scopes more than
  the rail) drives both the register list and the category rail beside
  it — moved to the Register panel-label, removed from the Categories
  panel-label. Two independent toggles on one screen risked showing
  contradictory filters (register on Expenses, rail on Income) with no
  way to tell at a glance; a single control can't disagree with itself.
- **State doesn't persist** — same as `railDir` and Recurring's own
  `recurringDir` before it: a plain in-memory `let`, resets to `"expense"`
  on every page load, never touches `Settings` or sync. Matching existing
  precedent rather than introducing a third, differently-behaved toggle.
- **Totals stay unfiltered.** The Summary panel, and the register's own
  week/day IN/OUT headers, keep summing *both* directions regardless of
  which side the toggle is showing — same rule Recurring's summary box
  already follows (see "Recurring design decisions"). Concretely:
  `renderRegister()` now computes two totals per week/day from the full
  unfiltered month (`weekAllTotals`/`dayAllTotals`), separate from the
  filtered `entries` array that decides which rows actually render — so a
  day header can read "IN $3,000 · OUT $50" even while the list below it,
  filtered to Expenses, shows only the $50 row.
- **Budget UI is untouched** — out of scope by design, confirmed
  explicitly: the category rail's own budget bars, the Total budgeted
  footer, and the budget circles all live on separate screens/panels this
  toggle was never wired to touch.
- Empty-state copy became direction-aware ("No income logged yet this
  month." / "No expenses logged yet this month.") instead of the old
  always-mixed "Nothing logged yet this month.", matching the pattern
  Recurring's own empty list message already used.
- **Verification:** confirmed via the self-test suite (4 new regression
  tests: direction filtering, totals staying unfiltered, the new empty-state
  copy, and the rail following the same toggle — plus one pre-existing test
  that assumed a mixed register and had to be updated to toggle direction
  per assertion, now 226 tests/0 failing) and on-screen screenshots at both
  390px and 1280px widths confirming no layout squeeze from moving the
  toggle onto the (longer) "Register" label — the move actually gains room
  on phone width, since `.cols` collapses to a single full-width column
  below 1000px where the old 320px-capped aside toggle used to live.
  **Confirmed working live on `main` (2026-08-26):** merged and deployed,
  confirmed by Sebastian on the real site — not just verified via the
  self-test suite and screenshots pre-merge.

## General fixes (2026-08-30)

**Google sign-in stops opening its popup in the wrong browser tab on
mobile.** Reported by Sebastian on Chrome for iOS: disconnect→reconnect
Drive and the account picker consistently lands in a *different*, unrelated
pinned tab — the same one every time, regardless of tab order — and picking
an account there often leaves that tab on a Google page instead of
returning. Diagnosed rather than assumed to be the same class as the old
mobile-Safari popup issue (Backlog #1 / SYNC-LESSONS #8) — it isn't. Two
independent causes:

- **Page-load token refresh was firing an un-activated `window.open`.**
  `initDriveSilentReconnect()` (now renamed `resumeDriveSyncIfTokenCached`)
  called `requestAccessToken({ prompt: "" })` on every `DOMContentLoaded`
  where the cached token had expired, in the belief that an empty prompt
  does a silent, popup-free token refresh. It does not — confirmed by
  probing the real `gsi/client`: GIS's token client only skips the popup
  for `prompt: "none"` *and* an in-page refresh session a fresh load
  doesn't have; `""` always calls `window.open`. An un-activated
  `window.open` on load is handled erratically by mobile browsers
  (background tab, wrong focus, severed opener) and, because mobile
  discards and reloads backgrounded tabs, could fire from a tab the user
  wasn't even looking at — the "picker in a tab that wasn't doing
  anything" symptom. **Fix:** page load now *never* triggers an OAuth
  popup. The decision is extracted as `pageLoadSyncPlan()` →
  `"not-connected"` | `"resume-from-cache"` | `"await-user-reconnect"`. A
  backendless static site cannot refresh a Google token without a user
  gesture (hard rule 4 rules out the server-side flow that could), so with
  no valid cached token the app shows the already-existing "Drive
  connected — needs reconnect" state and waits for a tap. Sync still
  resumes with zero friction whenever the cached token (its own
  `localStorage` key, ~1hr life) is still good.

- **GIS reuses one frozen popup window name for a page's whole lifetime.**
  Verified empirically (stub `window.open`, load real `gsi/client`, call
  `requestAccessToken` 3×): every call opens `window.open(url,
  "g_auth_token_window_<hex>", …)` with the **same** `<hex>` — it's
  randomised once at script-eval and never again — and it regenerates only
  on a full page reload. Sebastian's Money Ledger tab is *pinned* and
  never reloads, so that name is fixed forever; each connect/reconnect
  `window.open`s the same name, and the browser retargets whatever stale
  popup tab still holds it (mobile Chrome frequently ignores GIS's
  `window.close()`). This half is **not fixable from the app** — GIS owns
  the name; the token client has no `ux_mode: "redirect"`. Mitigations
  shipped instead: (1) an `error_callback` on every token request —
  without one, GIS's *own* abandoned/closed-popup detection never arms and
  every failure is swallowed silently (this was the "I tap an account and
  nothing happens" half); (2) a 120s watchdog that toasts "Google sign-in
  didn't come back…" if neither callback fires, so a misrouted flow is
  visible and recoverable instead of a silent dead end.

**Verification:** self-test suite (6 new cases — `pageLoadSyncPlan`'s four
outcomes incl. the safety-margin edge, and the connected-but-no-token
state rendering the Reconnect affordance; watched to fail against pre-fix
code; suite 282/0). The "never a page-load popup" and "frozen window name"
claims are backed by the two headless `gsi/client` probes above plus the
minified-source reading, not by the self-test (TEST_MODE never loads GIS).
**Device status:** first attempt "failed" because the branch was never
deployed — the phone was still running `main`; traced and confirmed via
the live `app.js` (see the merge-and-deploy discipline note below). After
merging to `main` (`544d987`) and deploying: (b) reconnect completes and
(c) back-out surfaces a toast were **confirmed on the real device
(2026-08-31)** as part of the 2026-08-31 disconnect→reconnect run; (a) no
unprompted picker on an idle-tab/page-load is verified by the live
headless probe (0 `requestAccessToken` calls on `DOMContentLoaded` with
connected + no cached token), not a separate on-device run. The
frozen-window-name reuse this fix *left in place* for the deliberate-tap
path was then fixed outright — see "General fixes (2026-08-31)".

**Merge-and-deploy discipline (learned here the hard way):** a "checkpoint,
not merge" branch commit does **nothing** for on-device testing — GitHub
Pages serves `main`. Before asking for a real-device check, the fix must
be on `main` and deployed; verify with
`curl https://groundwork-tools.github.io/Money-Ledger/app.js | grep <new-symbol>`,
not by assuming.

**Update (2026-08-31): the deliberate-tap wrong-window bug turned out to
be fixable from the app after all — see "General fixes (2026-08-31)"
below.** The `544d987` fix above shipped and was confirmed on the real
device to remove the *unprompted* picker, but disconnect→reconnect still
reliably opened the picker in one specific stale tab every time — and that
tab was **not** on `accounts.google.com` at the time, contradicting the
"close the dormant Google tab" mitigation. Traced to `window.open`'s
name-based retargeting surviving cross-origin navigation.

## General fixes (2026-08-31)

**The OAuth picker no longer retargets a stale tab on the deliberate
connect/reconnect path.** Follow-up to "General fixes (2026-08-30)". After
that fix deployed, `Disconnect → Connect` still opened the Google picker in
the *same specific tab every time* — a tab that was **not** showing
`accounts.google.com` at the time (whatever the user had last navigated it
to). The "close the dangling Google tab" mitigation was the wrong model:

- **Root cause, confirmed by direct test (headed Chrome + real handles):**
  `window.open(url, name, features)` retargets any existing browsing
  context with that `name` that the caller opened — *even after that tab
  has navigated to a completely different origin* — as long as the opener
  link isn't severed (`h2 === h1` after a cross-origin nav; a unique name
  gives a fresh window; the opener can `.close()` a cross-origin popup it
  opened). GIS builds exactly **one** popup window name per page load
  (`g_auth_token_window_<random>`, random half fixed at `gsi/client`
  eval), so on a tab that never reloads (a pinned phone tab) the very
  first OAuth tab it ever opened is retargeted forever, wherever it now
  sits and whatever it now shows. The persistent thing is *the tab*, bound
  to that name for its whole life — not its contents. Scope note: this is
  confined to one un-reloaded page-instance; a reload regenerates `Id` and
  old orphan tabs become un-retargetable.

- **Fix (`app.js`, near `driveIsConfigured`):** wrap `window.open` so any
  name starting `g_auth_token_window` gets a unique `.<suffix>` appended
  (`freshOAuthWindowName` — pure, everything else passed through
  untouched), forcing GIS to open a fresh window every call.
  `openOAuthAwareWindow` also captures the returned handle as
  `lastOAuthWindow`; `settleDriveAuth()` (= `clearDriveAuthWatchdog()` +
  `closeLastOAuthWindow()`) runs on every settle path — the shared
  `handleDriveAuthResult`, `driveAuthErrorCallback`, and
  `onDriveAuthWatchdogFired` — so fresh windows don't pile up. Money
  Ledger never calls `window.open` itself, so the wrapper only ever
  affects GIS's popup.

- **KNOWN FRAGILITY — watch for this.** The wrapper is safe *only* because
  GIS never reads the popup window name back: auth-result routing is by
  `origin` + a per-request nonce in the `redirect_uri` + `client_id`, with
  the name used *solely* in the `window.open()` call. That was established
  by **reading the current `gsi/client` source (2026-08-31)** — it is
  **not** a public API guarantee. If a future GIS update starts
  correlating on the window name, this wrapper would **silently stop
  preventing the retarget**: no error, no test failure (TEST_MODE never
  loads real GIS). Mitigation baked in: the prefix is preserved (only a
  suffix is appended), so a *prefix*-based GIS check still matches; only an
  exact-name check would break. If the wrong-window bug ever resurfaces
  after a GIS update, this wrapper is the first thing to re-verify against
  the then-current `gsi/client` source.

- **Verification:** 19 new self-test cases (`freshOAuthWindowName` rewrite
  + uniqueness + prefix preservation + non-OAuth/non-string pass-through;
  `openOAuthAwareWindow` rewriting only GIS's name and tracking only GIS's
  popup, driven by a spy native-open; `closeLastOAuthWindow` closing +
  clearing + tolerating a throwing `close()`; and `closeLastOAuthWindow`
  firing on all three settle paths). Watched to fail against pre-fix code.
  Suite 301/0. The cross-origin retarget behaviour, the "unique name →
  fresh window", and "opener can close a cross-origin popup" facts are
  from a headed-Chrome test (Blink; WebKit/Chrome-iOS not directly
  tested — but the user already demonstrated the retarget there, and
  "fresh name → fresh window" is engine-independent).
  **Confirmed on the real device, live site (2026-08-31):** merged
  (`b7ab1cc`) and deployed; Sebastian ran the disconnect→reconnect repro
  five times in a row — every time the picker opened correctly, completed,
  and returned to "Synced to Drive", with no stale-tab retargeting and no
  dead tabs accumulating. This also stands as a real-device confirmation
  of the `544d987` fix's auth-flow half (checks (b) and (c) from "General
  fixes (2026-08-30)" — reconnect completes cleanly, back-out is
  recoverable); the idle-tab/page-load half (check (a)) remains verified
  by the live headless probe rather than a separate on-device run.

## General fixes (2026-09-01)

**"Tap anywhere to reconnect" an expired Drive token.** Follow-up to the
"General fixes (2026-08-30)" page-load-popup removal. That fix was correct
— page load now never fires an OAuth popup, it lands in the "needs
reconnect" state and waits for a tap — but the *only* signal of that state
is the sync-status label buried in the settings panel (Money Ledger has no
persistent status line, unlike Hours Ledger). So a device whose ~1hr token
lapsed would sit there silently not syncing until the user happened to
scroll down and find the button. Ported Hours Ledger's forgiving pattern
(its `pointerdown`-delegated reconnect, CLAUDE.md there under "Interactive
OAuth without a click"): the *next ordinary tap anywhere* resolves it.

- **`resolveGestureReconnect(target)` (`app.js`, after
  `handleDriveButtonClick`)** — the extracted gating decision, its own
  function purely so the self-test can assert every branch without loading
  GIS (same rationale as `pageLoadSyncPlan`). Returns one of six outcome
  strings; only `"reconnecting"` has an effect. Checks, in order:
  `driveIsConfigured()` → tap not on `#driveConnectBtn` (its own click
  handler owns that) → `state.settings.driveConnected && !driveAccessToken`
  (Money Ledger has no `driveNeedsReconnect` flag — this *is* the
  needs-reconnect condition, the same one `renderSettings` and
  `handleDriveButtonClick` already branch on) → `!driveReconnectAttempted`
  → `driveAuthWatchdog === null` (no auth flow already pending). All clear
  → claims the episode's one attempt and returns `"reconnecting"`.
- **The listener (`initSettings`)** — a capture-phase `pointerdown` on
  `document`, `if (resolveGestureReconnect(ev.target) === "reconnecting")
  reconnectDrive();`. `pointerdown` not `click` (fires earliest, always
  carries user-gesture weight, can't be suppressed by a `preventDefault`
  elsewhere); capture phase; **no `preventDefault`** — the triggering tap
  still does its normal job, the reconnect just rides along (Sebastian's
  explicit call: "ride along", not "swallow the first tap"). **Not wired in
  `TEST_MODE`** — the suite drives `resolveGestureReconnect` directly, and
  a stray real `pointerdown` must never reach `reconnectDrive()`'s real
  OAuth.
- **`driveReconnectAttempted`** — module-level `let`, in-memory only, never
  persisted, never synced, cleared on reload (same discipline as the
  category filter's `Set`s and `viewDir`). Set `true` on the
  `"reconnecting"` outcome; **stays** `true` through a dismissed / failed /
  timed-out attempt (so one dismissal doesn't re-prompt on the very next
  tap — the button and a reload are the way back); reset to `false` only
  when a token is actually obtained (`handleDriveAuthResult` success and
  the `resume-from-cache` branch), so a *later, separate* expiry the same
  session still resolves itself on a tap.
- **No new OAuth path.** The trigger routes strictly through the existing
  `reconnectDrive()`, so it inherits the fresh-OAuth-window-name wrapper
  (`openOAuthAwareWindow`, "General fixes 2026-08-31"), the watchdog, and
  every settle path with zero new code. `client.requestAccessToken(` still
  has exactly two call sites (`connectDrive` + `reconnectDrive`) — there's
  a self-test asserting that count, so a future gesture-less caller can't
  sneak in. The page-load path is untouched: still `pageLoadSyncPlan()` →
  `"await-user-reconnect"` → `renderSettings()`, no popup.
- **No schema / storage / sync / migration change** — view-state only.
- **Known boundary, deliberately out of scope:** this resolves the
  *page-load* needs-reconnect state. If the cached token expires while the
  app stays open, `driveAccessToken` still holds the stale string, so
  `syncNow` fails quietly and `resolveGestureReconnect` returns
  `"skip-not-needed"` — the user still has to reload to enter the
  reconnect state. Pre-existing behaviour (nothing nulls the token on a
  401), not a regression. Fixable later by having a sync failure clear
  `driveAccessToken`; that's a separate change.

**Verification:** 13 new self-test cases (block 97 — every gating outcome,
one-shot-per-episode, flag surviving a dismissed attempt, real-token
success re-arming, the in-flight guard, connect-button + descendant
deferral, and the `client.requestAccessToken(` call-site-count guard).
Watched to fail against pre-change code two ways: new tests on the old
`app.js` trip the exposed-function guard (`1 failing`, same pattern as
tests 94–96), and deleting just the `driveReconnectAttempted` guard line
from the shipped code fails the two one-shot assertions. Suite 314/0.
**Confirmed on the real device, live site (2026-09-01):** merged
(`b01e300`) and deployed (verified with `curl … | grep
resolveGestureReconnect` before the device check). Sebastian opened the
app with an already-expired token — no unprompted popup on load, Sync
panel correctly showed needs-reconnect; the first ordinary interaction (a
scroll) triggered the picker, which completed cleanly and returned to
"Synced to Drive". Also confirmed the negative case: after a manual
disconnect + reload (`driveConnected` false, not merely token-expired), a
tap does nothing — correct, `resolveGestureReconnect` returns
`"skip-not-needed"`. The dismiss-once path (checklist step 3 — close the
picker without choosing an account, confirm the next tap doesn't
re-prompt) was **not** force-tested; it'll be exercised naturally on a
future real expiry. Its logic is covered by the self-test (`flag survives
a dismissed attempt` → still `"skip-already-attempted"`).

## General fixes (2026-09-05)

**Org transfer, SebPoulsen → Groundwork-tools.** Repo transferred and all
hardcoded links (OG images, README, cross-links, sync scripts) updated
accordingly. Google OAuth authorized origins updated to
`groundwork-tools.github.io` (old `sebpoulsen.github.io` origin removed
after real-device sync confirmed working on both apps). As reported by
Sebastian — the OAuth console change and the real-device confirmation
happened outside this tool session, so they're recorded here on his word,
not independently verified from here.

## General fixes (2026-10-04)

**Sync failures are now diagnosable and classified (phone Chrome "The quota
has been exceeded" report).** Reported as a Drive quota error on phone only.
Diagnosed rather than assumed:

- **The text did not come from Drive.** Every Drive failure used to be thrown
  as a bare `drive-<op>-<status>` and the response body was never read, so no
  Drive `reason` could ever have reached a toast. "The quota has been
  exceeded." is, as far as I recall, WebKit's stock message for a full
  `localStorage` (Chrome on iPhone is WebKit). Chrome desktop's message is
  different (checked: "Failed to execute 'setItem' on 'Storage'..."). The
  old catch-all toast then labelled it "Couldn't reach Google Drive".
  **Not yet confirmed on the phone**: it is the leading explanation, not a
  proven cause. The new toast names the biggest localStorage keys so the
  next occurrence identifies the consumer. GitHub Pages project sites share
  ONE origin (see SYNC-LESSONS #5), so Hours Ledger's keys count against the
  same quota; Hours Ledger stores up to 12 full-state undo snapshots in
  localStorage, which is the first thing to check.
- **No request loop exists.** Measured with the real `app.js` against a
  counting `fetch` stub: page load 3 requests, connect/reconnect 3, each
  debounced edit 2 (three edits inside 1.2s still 2). No render-to-save
  path, no retry timers. Before this change two overlapping `syncNow` calls
  interleaved (read, read, update, update).
- **New behavior.** `driveHttpError` reads status + `error.errors[0].reason`
  + message. `classifySyncError`: 401 is "auth" (drop token, existing
  needs-reconnect state, tap-anywhere reconnect picks it up); 429 / 5xx /
  rate-limit reasons are "backoff" (token kept, exponential back-off from
  30s capped at 10min, one retry timer, zero requests while backing off);
  local `QuotaExceededError` is "storage" (never blamed on Drive, and a
  failed local write no longer stops the push or lets "Synced" overwrite
  the warning). `syncNow` checks the token's known expiry before sending
  (the Hours Ledger pattern), holds an in-flight guard with a queued rerun,
  and a `visibilitychange` check surfaces a lapse on tab refocus.
  Deviations from Hours Ledger, on purpose: it never reacts to a 401 (a
  revoked token has a valid-looking expiry), and it has no rate-limit class.
- **Decision: `saveState` catches a full-origin `QuotaExceededError`
  instead of throwing (not originally requested).** Before: the exception
  escaped every UI handler that saves, so on a full origin an action just
  failed with nothing on screen. Now: the entry stays in memory (the app
  keeps working this session), it is NOT in `localStorage`, and the toast
  says "Not saved on this device: ... storage ... is full ... in memory only
  and will be lost if you close or reload the app". If Drive is connected
  the debounced push still runs, so the entry reaches Drive and Drive can
  hold what this device cannot. Reasoning: silently failing is the worst
  option for a ledger, and an honest message plus a Drive copy is strictly
  safer than a crash; the alternative of refusing the edit outright was
  rejected because the person may have just typed an entry they would lose
  either way. Only `QuotaExceededError` is caught; any other storage error
  still throws. Pinned by self-test block 100.
- **Hard rule 6 under a failed local write.** A failed local write never
  removes a record on either side: the merge happens in memory, the push
  still goes out, so Drive receives a superset (including local tombstones
  as tombstones, not removals). Tested with records present only locally,
  only on Drive, and a local delete of a record Drive still holds live.
  Limit of the test: it simulates the failure by making `setItem` throw; it
  assumes (per the Web Storage spec) that a failed `setItem` leaves the
  previous stored value intact, which only a real browser can confirm.
- **Cache-busting.** `index.html` loads `style.css?v=TOKEN` and
  `app.js?v=TOKEN` so a phone cannot serve a stale cached script beside a
  fresh page (the 2026-08-12 gotcha). TOKEN is a hand-bumped string (current:
  `20261010a`), not a commit hash, because a commit cannot contain its own
  hash and there is no build step to inject one. **On every deploy that
  changes `app.js` or `style.css`, change TOKEN in BOTH tags** (date plus a
  letter, e.g. `20261015a`). A self-test fails if the two tokens differ or
  are missing, but nothing can detect a forgotten bump. Limit: it only
  helps once the phone has a fresh `index.html`, which GitHub Pages lets
  browsers cache for about 10 minutes. `privacy.html` also links
  `style.css` unversioned; left alone.
- **Verification:** self-test blocks 98 to 101 (75 new checks, suite
  400/0), written first and watched fail against old behavior (3, 16 and 6
  failing in the three rounds). Nine single-behavior mutations each flip
  the intended tests. Two vacuous passes were caught and fixed in the tests themselves
  (a stub keyed on the wrong storage key; a toast element that keeps stale
  text). **Not real-device-verified.**

**Real-device results for 2026-10-04 (reported by Sebastian, recorded on his
word, not independently verified from here).**
- **Quota error: not reproduced after `58819b6` deployed. Cause
  unconfirmed. NOT marked fixed.** The error did not recur on the phone with
  the new code live, but it was intermittent before, so one clean run proves
  little. The "full localStorage" explanation above remains a leading
  hypothesis only: nothing observed on the device has confirmed or refuted
  it. If it recurs, the toast now names the cause (storage full with the
  biggest key sizes, or a Drive reason), which is the evidence to act on.
- **Session expiry: not yet verified on a real device.** The 401 / expired
  token handling, the "session expired" toast and the tap-anywhere reconnect
  that follows it have only been exercised by the self-test suite (fake
  Drive) and mutation checks. Needs an hour-idle tab on the phone.
- **Post-login landing on the wrong screen (investigated, no code).**
  Hypothesis was that the page reloads during the OAuth round trip (for
  example a discarded background tab on mobile) and in-memory view state
  (screen, month, Spent/Income) resets. Mechanism check: in headless Chrome a
  reload does reset all three (Recurring to register, September to October,
  Income to Spent), and nothing else in the connect path changes screens.
  Phone test: with Income selected and the previous month showing, Sebastian
  tapped Connect and completed the picker. Income and the previous month
  survived, and no extra tab opened. So the page did not reload on that
  run, and the reload theory is not supported by it. The original
  intermittent report was **not reproduced** and its cause is unknown.
  A one-off clean run does not rule out an intermittent cause.
  **Decision: the proposed `sessionStorage` "restore pre-connect view" fix
  is deliberately NOT built.** Reasoning: no observed failure to fix, it
  would add persisted view state to an app that deliberately resets view
  state on load (`viewDir`, the category filters), and it would not address
  a wrong-tab cause if that is what happened. Reopen only if the wrong
  landing recurs, and capture first whether the page reloaded (the Income /
  previous-month check above) before building anything.

## General fixes (2026-10-04, Trends)

**Trends screen built (backlog #4).** Third header button, own screen, per
the decisions recorded under backlog #4. View-state only (`trendsDir`,
`trendsCatId`, never in State, storage or sync). Data layer is pure
(`categoryHistory`, `categorySeries`, `historyCategoryOptions`), built on the
existing `sumByCategory` and cross-checked against it, `monthTotals` and
`entriesInMonth` in the self-test.

- **Header.** Budget, Recurring and Trends sit in a `.screennav` wrapper:
  `display:contents` on desktop (button geometry measured identical to
  before), own full-width row of three equal buttons below 700px. Measured at
  320/360/390/1000/1280px: with default `.ghost` padding "Recurring" overran
  its content box at 360px (text 78.9px, box 75px) and was clipped at 320px,
  so the buttons get `padding:8px 4px; letter-spacing:.06em` inside the 700px
  block only (slack at 360px: 18px). Labels unchanged, no icons.
- **How the current month is marked as partial, and why.** The window always
  ends at the real current month, so the last row is the in-progress one. It
  carries the text "so far" on a second line under the month label, plus a
  hatched (striped) bar fill. Text, because it is unambiguous and testable;
  stacked under the label rather than beside it, because beside it does not
  fit next to the bar and an exact figure at 390px; hatching as a second cue
  that works at a glance. The bar width stays honest (scaled to the largest
  month like every other row), only the fill says "not finished". A one-line
  hint under the list says the same in words. The marker follows the real
  clock, not the month being viewed elsewhere in the app.
- **Bug found by a screenshot, not by the DOM tests.** The figure column was
  `auto`, so a short figure ("999 kr.") left a longer bar track than a long
  one ("3.050,25 kr."): bars were scaled to the same maximum but drawn on
  tracks of different lengths, so lengths could not be compared across rows.
  Fixed with one shared figure-column width for the list (`--tr-numw`, the
  longest exact label in `ch`, set from JS), tighter month column and figure
  size under 700px. Pinned by a geometry test (block 105) that runs the real
  page in an offscreen frame at 320 and 360px with a seven-figure amount.
  Lesson: DOM and class checks cannot see layout; the main sandbox frame has
  no layout at all (`iframe{display:none}`), so layout checks need their own
  offscreen frame (and must force `display:block`, or every size is 0 and the
  checks pass vacuously, which happened once while writing block 102).
- **Exact numbers, no red, square bars.** Figures use `formatMoney` (exact),
  never `formatCompact`. Bars use the category's own colour, a deleted
  category's tombstone colour included. Zero months show an exact 0 and no
  bar at all. A selected category with no spend shows six zero rows.
- **Default selection** was the category with the most spend in the window.
  **Superseded the same day by "All" (see "Trends: All and the desktop cap"
  below).** A direction switch clears the selection so it re-picks.
  Deleted categories appear only with spend in the window, as "Name (deleted)".
- **Navigation.** The logo, the FAB and Back all return to the register;
  opening Budget or Recurring from Trends (the header is visible on every
  screen) hides Trends, and vice versa, so two screens are never open at once.
- **Verification.** Blocks 102 to 105. Fail-first runs: 3 failing, then 0.
  Eleven single-behavior mutants of the Trends code each flip tests; three
  needed test work first (one crashed the suite instead of failing cleanly,
  one exposed that the "clear selection on direction switch" line is only
  observable for an unknown category id that has spend in both directions,
  one showed nothing checked a deleted category's bar colour). Pure-layer
  mutants: eight of nine flip; the ninth (the `<= endKey` future filter) is an
  equivalent mutant, because a future-only ledger already yields a negative
  month count. Screenshots at 390px and 1280px checked by eye in headless
  Chrome. **Not real-device-verified.** Desktop note: at 1280px the bars
  stretch very wide (about 1600px); left as is, not requested.

## General fixes (2026-10-04, Trends: All and the desktop cap)

**"All" is the first option in the Trends switcher and the default.** Added
after the first release, on Sebastian's request. Decisions, each with its
reasoning:

- **All follows the Spent/Income toggle; it never mixes income and expense.**
  Spent shows total expenses per month, Income shows total income. Mixing
  would break the app's no-mixing rule (every direction-aware surface splits
  the two), and a combined net per month would be misleading for the
  in-progress month: income often lands once (payday) while spend accrues
  daily, so a partial-month net swings for reasons that say nothing about the
  month.
- **The default changed from "biggest category" to All. This REVERSES the
  earlier decision recorded in the first Trends entry above.** Reason: the
  overview comes before the detail. Opening on one arbitrary category (the
  largest) answered a narrower question than the one a person usually has
  first ("where is my month going overall"), and the biggest category is
  one tap away in the switcher. Consequences: a direction switch with a
  single category selected now re-picks All (not the biggest income
  category), and with All selected it simply stays All. A stale or missing
  selection falls back to All. `pickTrendsCategory` no longer looks at
  totals at all; a regression test pins that All wins even when a single
  category is bigger.
- **All must equal `monthTotals` for every month in the window, both
  directions.** It is built as the sum of every category total
  `categoryHistory` kept for the month (`totalSeries`), not as a sum over
  known categories, so it includes spend under deleted categories and under
  ids with no category record (what the register shows as "Uncategorized"),
  exactly like `monthTotals`. If it excluded either it would silently
  disagree with the Summary panel. The self-test cross-checks the rendered
  figures and the pure function against `monthTotals(entriesInMonth(...))`
  for every month, for expenses and income.
- **Same window rules, marker and scale as a single category:** at most 6
  months, never before the earliest entry in that direction, zero months an
  exact 0 with no bar, the current month "so far" and hatched, bars scaled to
  the largest month. A direction with no entries shows the existing plain
  empty state.
- **Neutral ink fill for All, never a category colour, never red.** All is
  not one category, so no category's colour should speak for it.
- **Desktop cap: added, then REVERSED the same day.** At 1280px the bar track
  stretched to about 990px (more on a wider screen), which I flagged and
  Sebastian asked to cap; the list was capped at 720px (track about 470px).
  Seeing it, he said the Trends line should expand the whole screen: the
  capped box ended well short of the full-width switcher box and header rule
  above it, which read as unfinished. Asked which of three layouts he wanted
  (full width with bars filling; full-width box with capped bars and the
  figure beside the bar; same with the figure at the far right) and he chose
  **full width, bars fill the row**. So the cap was removed entirely and the
  list spans the page like the Budget and Recurring screens. Accepted
  trade-off, stated plainly: on a very wide monitor a bar can be 1600px or
  more, which is the thing the cap was meant to prevent. If that ever
  bothers in daily use, reopen it as a new decision (the capped-bar,
  figure-beside-the-bar layout was the runner-up). Geometry is now identical
  to before the cap at every width measured (track 60.8 / 100.8 / 130.8 /
  708.9 / 988.9px at 320 / 360 / 390 / 1000 / 1280). Block 107 pins it: the
  list is exactly as wide as the switcher box and the tracks fill it, and
  reintroducing a cap (720px or a looser 1100px) fails it.
- **Verification.** Blocks 104 (updated for the new default), 105 (also 390px),
  106 (All) and 107 (row width, plus phone widths pinned). Fail-first run: 4
  failing (two guards, two cap checks with real numbers), then 1 failing
  (my own ink-colour assertion compared an inline `var(--ink)` to an rgb
  value; fixed to compare the computed colour), then 0. Twelve single-
  behavior mutants all flip tests: All excluding deleted categories, All
  dropping unknown-id spend, wrong default, a category colour in All, All not
  following the toggle, direction switch not clearing the selection, no
  partial marker in All, max instead of sum, All listed last, All renamed,
  cap removed, cap too wide. One mutant run crashed on a Chrome launch
  failure (harness, not a result) and was rerun alone. Screenshots checked at
  390px and 1280px. **Not real-device-verified.**

## Testing before you claim it works

There is an automated self-test suite — `money-ledger-selftest.html`,
modeled directly on Hours Ledger's own `hours-ledger-selftest-reference.html`.
It started as sync/merge-only (built 2026-08-03/04, see below) and stayed
that way for its first ~23 tests even as the app grew well past sync —
Budgets shipped entirely on 25 manual checks run once by hand, never kept,
and the three rounds of circle-redesign work that followed shipped the
same way. Backfilled 2026-08-11 to close that gap; see "Why the suite
didn't grow" below before assuming the old framing (sync-only) still
describes it.

**Why the suite didn't grow, diagnosed 2026-08-11 — two compounding
causes, not one:**
1. **The harness had a real hole, not just an unused habit.** `runTests`
   was passed `d` (the iframe's `contentDocument`) from the very first
   commit — `this.contentDocument` was threaded into the call specifically
   so a test could read what the app actually rendered — and then never
   used it, not once, across all 23 original tests. There was no
   established pattern for "assert on the DOM," and no pure functions for
   the hook to expose for the money math either (`formatMoney`,
   `monthTotals`-equivalent logic, `sumByCategory`-equivalent logic all
   lived inline inside render functions, unreachable except by reading
   rendered output — which nothing did). Writing a money or rendering test
   meant inventing the pattern from scratch every time, so nobody did.
2. **The file's own framing discouraged it.** It was named, scoped, and
   described (including in this very doc) as "the sync/merge self-test
   suite" — a title and self-description that made a five-line arithmetic
   assertion feel like it belonged somewhere else, even though nowhere
   else existed. Combined with cause 1, the path of least resistance for
   verifying a new feature was always a disposable scratch HTML page
   driving the hook once, not a permanent addition to a file that
   described itself as being about something else.

**The fix, not just more tests:** `parseAmountToMinor`, `monthTotals`, and
`sumByCategory` were extracted from the render functions they were
duplicated inside (`renderSummary`/`renderBudgetView` both computed
month totals identically; `renderCategoryRail`/`renderBudgetView` both
built the same per-category totals Map; amount-string-to-minor-units
parsing was duplicated three ways across quick-add, edit, and the budget
field) into named, hook-exposed, directly-testable functions — the same
status `mergeRecords` already had. `d` finally gets used: rendering tests
read real DOM (`.tot-bar i` width/colour, the circles' text and
`ring-empty` class, the summary panel's figures, the edit sheet's
pre-filled fields) after driving state through the hook and calling
`H.renderAll()`. `setView(y, m)` was added to the hook so date-boundary
tests don't depend on which month the suite happens to run in.

`mergeRecords` behaving correctly in isolation was never the whole
story for sync, either — 2026-08-03/04's actual bugs were in how the app
*called* merge in sequence on one device, which only a full sync-path
test could catch.

**How to run it:** serve the folder locally (`python3 -m http.server`,
same as the app itself — opening as `file://` hits a cross-origin error on
the iframe) and open `money-ledger-selftest.html` by hand. It's never
linked from the real app. It loads `index.html?mltest=1` in a hidden
iframe, drives the real functions via `window.__ML_TEST__` (not
reimplementations of them), and renders PASS/FAIL rows with a
failing/total summary.

**Isolation — a test run cannot reach real data:**
- `?mltest=1` sets `TEST_MODE`, which suffixes `STORAGE_KEY` and
  `DEVICE_ID_KEY` with `-TESTMODE`, so a test run's `localStorage` never
  overlaps the real ledger's key.
- The four Drive network functions (`driveFindFiles`, `driveCreateFile`,
  `driveUpdateFile`, `driveReadFile`) each branch on `TEST_MODE` *before*
  constructing any `fetch()` call, and operate on an in-memory `FAKE_DRIVE`
  object instead. This isn't "tests avoid calling the real network
  function" — the real function itself cannot reach `googleapis.com` in
  this mode, structurally, not by convention.
- No real OAuth popup ever fires in TEST_MODE (`resumeDriveSyncIfTokenCached`
  no-ops); tests that need a connected state call `testConnectDrive()`,
  which sets the same fields a real successful connect would leave behind.
- `window.confirm` is stubbed to auto-accept in TEST_MODE only, so
  delete/import/clear-month flows don't hang a headless run waiting for a
  dialog no one will click.

**Testing gotcha: `useDevice(id)`'s first-ever call to a new device id
wipes `FAKE_DRIVE`, not just local state (2026-08-12).** `resetState()`
(what `useDevice` calls internally the first time it sees a given id)
resets `FAKE_DRIVE = {}` alongside the local state — correct for the
*very first* device in a test, since the drive should start empty, but a
trap the second time: if device A has already pushed something and
device B then appears for the *first* time, B's own `resetState()`
silently erases A's pushed data before B ever reads it, since
`useDevice` can't tell "first appearance, drive should be empty" apart
from "first appearance, but another device already has real data on it."
Existing multi-device tests (11–16) don't hit this because both devices'
first appearances happen before *either* has pushed anything meaningful,
with any real data-carrying push always coming from a *second*
appearance (restored via `setState`, which never touches `FAKE_DRIVE`).
Bit the `budgetRingColorsUpdatedAt` sync test the first time it was
written, for exactly this reason — traced with a standalone debug page
logging state at each step (see "Verification technique" below) after
staring at the sync code itself found nothing wrong there. **The safe
pattern: call `useDevice(id).save()` once for every device involved,
establishing each one's "born" snapshot, before any of them touch Drive
for real** — mirrors what tests 11–16 already do, just made explicit
here since it's easy to get this exact ordering wrong by accident.

**What it covers (314 tests as of 2026-09-01, up from 23):**
- **Sync/merge** (the original 23): the `mergeRecords` algorithm directly
  (only-local, only-remote, both-edited, delete-vs-edit,
  identical/skewed/ambiguous timestamps, empty sides, seed-category id
  collisions) and full sequences through the real
  `createEntry`/`editEntry`/`deleteEntry`/`syncNow` path, simulating two
  devices in one iframe by snapshotting and restoring state between them.
  Includes a regression test for each of 2026-08-03/04's five bugs (silent
  push overwrite, silent pull overwrite, undelete, edit duplication,
  category duplication), one longer test walking the full manual repro
  end to end, and a regression test for `budgetRingColors` actually
  syncing (2026-08-12) — a colour set on one device reaching another, and
  device-local Settings fields (`driveConnected`/`driveFileId`) staying
  untouched by that same sync. **Seed category resurrection (2026-08-30,
  tests 16b/16c):** a fresh re-seed vs. another device's months-old
  tombstone (must not resurrect, on the device or back on Drive), and the
  same via corrupt-`localStorage` recovery where `DEVICE_ID` survives —
  asserts the epoch `updatedAt` on recovered seeds and that
  `loadState()`'s fallback regenerates `DEVICE_ID`. See "Seed category
  resurrection" under Sync design decisions. **Page-load sync trigger
  (2026-08-30, tests 92/93):** `pageLoadSyncPlan()` returns
  `"await-user-reconnect"` (never anything that reaches an OAuth popup)
  when connected with no usable cached token — incl. the safety-margin
  edge where a near-expiry token is treated as gone — `"resume-from-cache"`
  only on a still-valid token, `"not-connected"` for a local-only device;
  and the connected-but-no-token state renders the "Reconnect Google
  Drive" button + "needs reconnect" status rather than firing a popup. See
  "General fixes (2026-08-30)". The two claims the self-test *can't* reach
  (page load never popups; GIS freezes one popup window name per page
  load) are covered by headless `gsi/client` probes, not TEST_MODE.
  **OAuth popup window-name wrapper (2026-08-31, tests 94–96):**
  `freshOAuthWindowName` rewrites only GIS's `g_auth_token_window*` name
  (unique suffix per call, prefix preserved) and passes every other name /
  non-string through untouched; `openOAuthAwareWindow` (driven by a spy
  native-open) rewrites only that name and tracks only that popup as
  `lastOAuthWindow`; `closeLastOAuthWindow` closes + clears the ref +
  tolerates a `close()` that throws; and it fires on all three settle
  paths (`driveAuthErrorCallback`, `onDriveAuthWatchdogFired`,
  `handleDriveAuthResult` for both connect and reconnect). See "General
  fixes (2026-08-31)"; the `window.open` retarget-across-origins behaviour
  itself is from a headed-Chrome test, not TEST_MODE.
  **Tap-anywhere reconnect (2026-09-01, test 97):** `resolveGestureReconnect`
  returns the right outcome for every state — `"skip-not-configured"` (GIS
  absent), `"skip-not-needed"` (local-only, or connected with a live
  token), `"reconnecting"` (connected + token gone) claiming the episode's
  one attempt, `"skip-already-attempted"` on the next tap, the flag
  surviving a dismissed attempt (`driveAuthErrorCallback`) but reset by a
  real `handleDriveAuthResult` success, `"skip-in-flight"` while
  `driveAuthWatchdog` is armed, and `"skip-on-connect-button"` for a tap on
  `#driveConnectBtn` or a descendant. Plus a structural guard:
  `client.requestAccessToken(` appears at exactly two call sites in
  `app.js` (read from the served source), so no gesture-less caller can
  slip in. See "General fixes (2026-09-01)".
- **Money math:** `parseAmountToMinor` (rounding, rejection of
  zero/negative/blank/non-numeric input, no float drift across repeated
  parses), `formatMoney`/`formatCompact` (decimal-hiding on whole amounts,
  no currency symbol on the compact form, rounds rather than truncates),
  the currency setting itself, `monthTotals` (income/expense split, net,
  a negative-net month, an all-zero month), `sumByCategory`, income-vs-
  expense classification, and month/year boundaries — an entry dated the
  1st, the 31st, the day before, and the day after, plus the Dec 31→Jan 1
  year rollover, plus a direct test of `parseIso`'s local-Y-M-D
  construction (the actual mechanism that prevents a timezone-driven
  off-by-one-day bug — see "Why the suite didn't grow" above for why this
  wasn't simulated across real system timezones instead: not practical to
  do honestly without mocking `Date`/`Intl` internals, and the mechanism
  test covers the real risk directly).
- **Budgets:** depletion percentage, overspend rendering the real number
  uncapped while the bar pins at 100%, a regression test for the exact-
  budget off-by-one (2026-08-11), a category with no budget rendering
  exactly as it did before budgets existed, the Total budgeted footer row
  (including the all-zero case), the three circles' values, editing and
  clearing a budget through the real input field (folding in the original
  25 manual checks as permanent assertions), and regressions for the
  ring-colour work (colour lives on the arc only, never the number; a
  custom colour applies when not flagged but never overrides the
  over-budget/negative flag).
- **Rendering, generally:** the Summary panel's actual figures and
  positive/negative styling, register rows' sign and colour, the edit
  sheet opening pre-filled (amount, date, note, category — the Money
  Ledger equivalent of Hours Ledger's "does a field open pre-filled"
  coverage), the week header's IN/OUT totals, and the quick-add category
  dropdown refreshing live on category create/delete without a reload
  (2026-08-11 regression). Empty states are asserted as empty (a dash,
  `ring-empty`) rather than as a rendered zero, for every circle that has
  an empty state. **The register's own Spent/Income toggle (2026-08-26):**
  the list filters by direction, week/day IN/OUT headers keep summing both
  directions regardless of the filter, the empty-state copy is
  direction-aware, and the category rail switches in lockstep since one
  toggle now drives both — see "General fixes (2026-08-26)".
  **The category filter (2026-08-28):** direction + selected categories
  compose (AND), multi-select, chip eligibility (this-month/this-direction
  only), day/week headers collapsing to `OUT <sum>` / `IN <sum>` when
  filtered while the unfiltered `IN … · OUT …` baseline is untouched, the direction
  toggle clearing the filter, the zero-match empty state, the rail's
  passive `.selected` highlight with rail totals staying unfiltered,
  month-nav re-deriving chip eligibility without touching the filter set,
  and the filter never appearing in a serialized `State`. Recurring
  parity: list + day-group totals follow the filter, summary box doesn't,
  a paused item's category is still offered — see "Category filter design
  decisions".
- **Recurring:** day-of-month clamping (`clampDay`/`daysInMonth`, incl.
  leap-year February), `dueRecurring`'s pure due-detection logic (not yet
  due, due with nothing inserted, already inserted this month, an entry
  from a previous month not counting, a deleted auto-inserted entry not
  counting, a different recurring id not counting, a deleted recurring
  record never due), `applyDueRecurring` creating a real entry with the
  right fields and not double-inserting on a second call the same month,
  editing/deleting a Recurring record never touching entries already
  produced from it, sync merge via `RECURRING_CONTENT_FIELDS`, and
  rendering (the screen split by direction, adding one through the real
  DOM form, editing a name inline, deleting through the DOM). **Start
  month and pause (2026-08-22):** `dueRecurring`/`isRecurringActive`
  respecting `startMonth` and `paused` in every combination, including a
  regression for a record with no `startMonth`/`paused` key at all
  behaving exactly as it did before the fields existed;
  `pauseRecurring`/`resumeRecurring` bumping `updatedAt`/`updatedBy`;
  `startMonth`/`paused` actually reaching `RECURRING_CONTENT_FIELDS` and
  merging like any other field; `applyDueRecurring` skipping a paused or
  not-yet-started item even when its day has passed; the list still
  rendering a paused/future item marked `.inactive` with the right status
  suffix rather than hiding it; the summary and day-group totals agreeing
  with each other about excluding those items; the add form defaulting
  its start-month field to the real current month and the chosen option
  flowing into the created record; the edit sheet pre-filling start-month
  and flipping its Pause/Resume label from the record's own state; and a
  regression for the deliberate Resume divergence — clicking Resume
  triggers the due-check synchronously in the same click rather than
  waiting for the next app open.

**The standing rule, going forward, same as Hours Ledger: every bug gets a
test that would have caught it, written before the fix, watched to fail
against the old code first.** Verified this actually works before trusting
it (2026-08-04): extracted `9eeabdb` and `11f58ff` — the commits before
each fix — into scratch copies with a network-fetch shim standing in for
Drive, ran the same scenarios against them directly, and confirmed each
failed exactly as reported (Drive silently missing an entry; a deleted
entry reappearing; an edit producing a second live record; seed categories
exactly doubling) before confirming they pass on current code.

Beyond the self-test suite, still verify manually and say which of these
you actually did, not which apply in theory:

- Log an expense and an income entry; confirm direction, sign, and colour
  are correct (plain ink for expense, green for income, red only when a
  category or month goes over budget). Now also covered by the self-test
  suite (register rows, the budget circles' colour rules) — this manual
  pass is about how it actually looks, not just whether the right class
  landed on the right element.
- Reload the page and confirm entries persisted from `localStorage`.
- Edit and delete an existing entry; confirm the change survives a reload.
- If Drive sync is involved: connect on one browser profile, confirm the
  file appears in the user's actual Drive, then read it from a second
  profile and confirm entries match, including after an edit on either side.
- Mobile viewport width: confirm logging an entry is still reachable
  one-thumb, matching Hours Ledger's FAB pattern.

If you couldn't test something, say so plainly and tell me what to click.

## Deploy

Push to `main` → GitHub Pages serves it directly, no build step, live within
a minute or two. Pages isn't enabled on this repo yet (checked: currently
404s) — it activates the first time `index.html` lands on `main`.

---

## The data model

```
Entry {
  id: string            // uuid
  date: "YYYY-MM-DD"
  amountMinor: integer   // cents / øre — never a float, avoids rounding drift
  direction: "expense" | "income"
  categoryId: string
  note: string           // optional, "" if empty
  recurringId: string | null   // set only if auto-inserted from a Recurring record
  updatedAt: string       // ISO 8601, bumped on every change to THIS record —
                          // what sync merges by, never the whole-file updatedAt
  updatedBy: string | null   // DEVICE_ID of whoever last touched this record;
                              // null means "unknown" (pre-sync data), treated
                              // conservatively as "not this device"
  deleted: boolean        // tombstone, not physical removal — see hard rule 6
  deletedAt: string | null
}

Category {
  id: string
  name: string
  color: string           // hex, chosen via the same hue-slider as Hours Ledger
  direction: "expense" | "income"   // separate lists per direction, not one shared pool
  budgetMinor: integer | null       // recurring monthly amount, expense
                                     // categories only — set from the Budget
                                     // screen, never from the main register
  updatedAt: string
  updatedBy: string | null
  deleted: boolean
  deletedAt: string | null
}

Recurring {                // backlog #3, named to match Entry.recurringId
                           // below — covers both subscriptions (expense)
                           // and predictable income (e.g. a paycheck), not
                           // expense-only, see "Recurring design decisions"
  id: string
  name: string
  amountMinor: integer
  direction: "expense" | "income"   // same split as Category/Entry — a
                                     // Recurring screen mixing "Netflix"
                                     // and "Paycheck" in one list would be
                                     // exactly the confusion the app avoids
                                     // everywhere else
  categoryId: string
  dayOfMonth: integer     // 1-31, clamped to the real last day of short months
  updatedAt: string        // syncs like Entry/Category — same tombstone shape
  updatedBy: string | null
  deleted: boolean
  deletedAt: string | null
}

Settings {
  currency: string         // ISO 4217, e.g. "DKK", "USD" — chosen on first launch, changeable after
  introSeen: boolean       // this device's own — never synced, a new device should see its own intro
  driveConnected: boolean  // this device's own connection state — never synced
  driveFileId: string | null   // the Drive file this device is synced to — never synced
  budgetRingColors: { income, net, expenses: {hue, color} | null }  // the one Settings
                            // field that IS a cross-device preference — see budgetRingColorsUpdatedAt
  budgetRingColorsUpdatedAt: string | null   // bumped only when budgetRingColors changes;
                            // syncNow() last-write-wins on this single field using it,
                            // since Settings has no per-record tombstone shape to merge by
}

State (top-level, the whole localStorage/Drive-file blob) {
  version: integer
  updatedAt: string        // ISO 8601, bumped on every save — bookkeeping
                            // only; sync decisions are per-record now, never
                            // by comparing this field (that was the bug)
  settings: Settings
  categories: Category[]
  entries: Entry[]
  recurring: Recurring[]
}
```

Seed categories use fixed ids (`"seed-groceries"` etc., see `SEED_CATEGORIES` in
`app.js`), not generated ones — two freshly-installed devices need to agree
that their starter "Groceries" is the same record, or merging duplicates
every seed category on first connect.

`DEVICE_ID` is a separate `localStorage` key (`money-ledger-device-id`),
generated once per browser profile — deliberately **not** part of `State`,
so it's never overwritten by a pull from Drive. It exists purely so merge
can tell "a stale copy of my own last push" (no real ambiguity — one
device's clock ordering of its own actions is never in question) apart
from "another device's edit" (where clock skew is a genuine concern).

- Amounts are always integer minor units. Format for display using
  `Settings.currency`'s locale rules; never do arithmetic on formatted
  strings.
- `Category.direction` splits expense and income into separate pickers —
  follows from the ledger being a single signed stream that defaults to
  expense with a manual flip to income; a mixed "Salary / Groceries"
  dropdown would be confusing.
- `note` and `recurringId` may legitimately be empty/null — not a bug.
- Dates are local calendar dates (`YYYY-MM-DD`), no time-of-day component —
  money doesn't have an hour, unlike Hours Ledger's entries.

## How the code is organised

Same shape as Hours Ledger, as planned: `index.html` (markup for every
screen — register, Budget, Recurring, edit sheet, intro — toggled via
`hidden`, not separate pages), `style.css`, and `app.js` (one file,
organized in commented sections: storage/schema, utils, mutations
[create/edit/delete for Entry/Category/Recurring], quick-add, edit sheet,
month nav, rendering per screen, Drive sync, the `__ML_TEST__` hook, init).
No framework, no bundler, no build step — all three files loaded directly
by the browser, same as Hours Ledger.

## Design constraints

Should look and feel like the same family as Hours Ledger, applied to a
different resource. It must never drift toward looking like a generic
budgeting SaaS product (soft shadows, rounded cards, upsell energy) — the
ledger-paper plainness *is* the trust signal.

- **Colours**, reused from Hours Ledger's palette (`hours-ledger-reference.css`):
  `--paper`/`--ink`/`--rule`/`--muted` for structure and text, same as
  Hours Ledger. `--flag` (red) stays reserved for warnings and destructive
  actions — here specifically for over-budget states — never for ordinary
  expense amounts. Income entries get a new green, not in the original
  palette, reserved solely for marking a row as income so direction is
  never ambiguous at a glance.
- **Type**: `--display` (Archivo) for headings and category names,
  `--mono` (Spline Sans Mono) for all amounts, dates, and uppercase labels
  — same split as Hours Ledger, where numbers and data are always mono.
- **Shape language**: square corners, 1px rules, no shadows, no border-radius
  — carried over exactly as-is from Hours Ledger. **One deliberate
  exception**: the three circles on the Budget screen (income/net/expenses).
  Requested explicitly and repeatedly by Sebastian as that screen's
  signature element (2026-08-10/11) — not an oversight. Everything else
  about them stays in the family's material language (1px ink border, no
  shadow, mono numbers) even though the shape doesn't.
- **The signature element**: the chronological register (running list
  grouped by week, then by day) paired with a category-totals panel is
  what shipped for the main screen — built and confirmed. The Budget
  screen's three circles are its own, separate signature element; see the
  shape-language exception above.

---

## Backlog, roughly in order

1. **V1 — logging, review, and sync.** Frictionless entry logging (defaults
   to expense, one press flips to income), category CRUD with the
   hue-picker/dot system, seeded starter categories, first-launch
   currency-choice popup with a "Start Logging" CTA (mirrors Hours Ledger's
   intro), week and month review views, and opt-in Google Drive sync via
   `drive.file`. Sync is in v1, not deferred, because it changes the
   storage layer's shape and is easier to design in from the start than to
   retrofit. **Status:** code is written (`GOOGLE_CLIENT_ID` constant near
   the top of `app.js`, currently empty — paste in the real OAuth Client ID
   from Google Cloud Console). Sync has now been exercised extensively
   against a real Google account across multiple real devices/browser
   profiles (2026-08-03/04) — connect, reconnect, disconnect, concurrent
   edits, deletes, and conflicts all verified live, not just via the
   self-test suite. `driveTokenClientFor()`'s reused-token-client pattern
   held up fine under that real load, no stale-callback issues observed.
   **Status: v1 is fully done (confirmed 2026-08-12).** The mobile Safari
   OAuth popup issue tracked here — Safari's ITP was believed to break the
   popup-to-main-window handoff after "Connect Google Drive," leaving a
   white/blank screen instead of a completed connection — no longer
   reproduces. Sebastian confirmed it works on a real phone, both Safari
   and Chrome. Nothing in the code changed the connect flow itself between
   when this was first flagged and now (only token caching and error
   messaging were touched, not `connectDrive()`'s popup mechanics), so
   this was most likely fixed upstream (an iOS/Safari or Google Identity
   Services update) rather than by anything in this repo — worth knowing
   if it ever resurfaces, since there's no local fix to point back to.
2. **Budgets.** Per-category monthly budget amounts and the budget-vs-actual
   visual — this is what finally gives `--flag` red something real to
   trigger on. The category bar and the month's Net figure should read as a
   *depletion* against the budget, not a plain proportion: full/bright at
   the start of the month, draining down as spending accrues against that
   category's (or the month's) budget, turning to `--flag` red once it runs
   out. Confirmed with Sebastian on 2026-08-03. **Status: built (2026-08-11)**
   — see "Budget design decisions" below for the shape it actually took,
   which moved a fair way from the one-line description above over a longer
   design conversation. Verified via on-screen screenshots (the circles,
   the fill-up bars, the ring colour picker) and the permanent self-test
   suite, which now covers this area directly rather than via disposable
   manual checks — see "Testing before you claim it works." Not yet
   real-device-verified by Sebastian.
3. **Recurring.** Declare once (name, amount, direction, category,
   day-of-month), auto-inserted as a real entry every month without
   retyping — covers both expense subscriptions and predictable income (a
   paycheck), not expense-only; see "Recurring design decisions."
   **Status: built (2026-08-11).** Own screen (button next to "Budget" in
   the header), split by a Spent/Income toggle like everywhere else
   direction-aware in this app. Syncs via the same per-record merge as
   Entry/Category. Auto-insert only ever checks the current real-world
   month — deliberately does not backfill months missed while the app
   wasn't opened; see "Recurring design decisions" for why. Verified via
   the self-test suite (clamping, due-detection, auto-insert, sync merge,
   rendering) and an on-screen screenshot showing the actual auto-insert
   split correctly across due/not-yet-due items in the register. Not yet
   real-device-verified by Sebastian. **Start month + pause added
   (2026-08-22)** — see "Recurring design decisions" for the full shape;
   verified via the self-test suite and an on-screen screenshot showing a
   paused and a future-dated item both marked inactive in the list with
   the summary/day totals correctly excluding them. Not yet
   real-device-verified by Sebastian.

4. **Per-category spending history (2026-10-04).** See how much was spent
   per category across previous months, with numbers, and a graph that can
   switch between categories (e.g. groceries over the last 6 months). Plain
   SVG or CSS, no chart library, must respect the design constraints
   (ledger paper, square corners, mono numbers). **Status: built
   (2026-10-04), see "General fixes (2026-10-04, Trends)"; design below.** Any total-by-category-by-month function must be
   checked against the existing `sumByCategory`/`monthTotals` first, and
   ships with tests in the same commit (hard rule 10). View-state only: no
   schema, storage or sync change.

   **Design decisions (Sebastian, 2026-10-04, final):**
   - **Its own screen, not a section of Budget.** Budget is about limits
     (what you plan to spend) and history is about the past (what you
     did). Mixing them would break the one-job-per-surface rule already
     used for the Budget screen and the category rail.
   - **Window: fixed 6 months ending at the current real-world month, and
     never starting before the earliest entry in that direction. No
     selector.** The window cannot usefully exceed the data that exists
     (the real ledger only goes back to when sync work began in August), so
     a longer or selectable range would mostly render empty months. A
     selector is more UI for no current benefit. Months with no spend
     inside the window still render as an explicit zero, not omitted.
   - **Layout: a category switcher on top, then one row per month for the
     selected category (month, bar, number).** Not a category-by-month
     grid: five or more mono number columns do not fit at 390px, and one
     category at a time reads cleanly on a phone and matches the
     one-row-per-item rhythm of the register.
   - **A deleted category with past spend in the window is shown as
     "Name (deleted)".** The tombstone keeps the name, so history stays
     truthful. Folding it into "Uncategorized" (what the register does for
     live entries) would hide where past money actually went.
   - Expense and income stay separate, using the same Spent/Income toggle
     as the rest of the app.
   - **Added 2026-10-04, after first use: an "All" option, first in the
     switcher and the DEFAULT.** A desktop cap on the bar track was added and
     then reversed the same day (rows span the full width). Full reasoning
     under "General fixes (2026-10-04, Trends: All and the desktop cap)".

   **Cut, deliberately:** tooltips; gridlines and axis ticks; a budget
   overlay line (`budgetMinor` is today's value, not a per-month value, so
   an overlay would misstate past months); multi-category comparison; a
   year view; a window selector.

5. **Success toasts that don't check the save (opened 2026-10-10).** The
   2026-10-10 fix routed only the *entry* toasts (quick-add, edit, delete and
   its Undo, automatic recurring entries) through `toastIfSaved()`. Still
   plain `toast()` straight after `saveState()`: `"<name>" deleted` (category,
   with Undo), `Backup imported.`, `<label> cleared` (a whole month, with
   Undo), and the Recurring item toasts (`added to Recurring`, `Recurring item
   updated / resumed / paused`, `removed from Recurring`). Same defect: on a
   full origin the success text replaces the "Not saved" warning in the same
   tick and the person sees only the success text. It was left out because the
   report was about entries and the fix was kept to what was asked (hard rule
   7); it is not harmless, since an import or a month-clear that did not store
   reads as done. Each needs the same fail-first test as the entry toasts (a
   failing `setItem`, then the toast text), and the Undo variants must not
   offer an Undo for a change that never persisted. Trigger: do it with
   Hours Ledger's backlog 27 re-measure on 2026-11-10 at the latest, since both
   depend on how soon the shared origin fills again.
6. **Other localStorage writes that are not guarded (opened 2026-10-10, found
   by reading, NOT run).** `saveDriveTokenCache()` writes the cached Drive
   token with a bare `localStorage.setItem`; it runs on the connect and
   reconnect paths, not at load, so it was not part of the load-time fix. On a
   full origin it would throw out of the connect flow. Audit every
   `localStorage.setItem` against hard rule 11 and add a test per site, as was
   done for the device id. Also open: the "Not saved" toast disappears after
   five seconds while the data stays unsaved; Hours Ledger uses a persistent
   banner for the same condition.

Do not add features that are not on this list without discussing them first.

## Out of scope

- **Bank-linked auto-import.** Deliberately rejected — manual entry is the
  point, the same way Hours Ledger doesn't infer your week from a calendar.
- **A Sebastian-controlled backend storing user financial data.** Rejected
  because this is a public tool for real strangers' real money data — see
  hard rule 4. If sync ever needs more than Drive can offer, stop and raise
  it as a new decision, don't quietly add a database.
- **Per-category keep/compress/cut verdict**, Hours Ledger's reflective
  judgment ritual — not carried over for now. Revisit once budgets (backlog
  #2) exist, since a verdict without a budget number behind it has less to
  anchor on for money than it does for time. (Budgets now exist as of
  2026-08-11 — this is worth actually revisiting, not just noting.)
- **Budgeting income categories.** Considered and rejected in favor of
  Recurring (backlog #3, built 2026-08-11) covering predictable income
  instead — see "Budget design decisions" and "Recurring design
  decisions." Income stays a plain logged counter, no budget field, no
  target.

## Open questions

- The main view's actual layout (register vs. grid vs. table-only) is a
  direction, not a decision — settle it once there's a working version to
  look at, not in the abstract.
- Google's OAuth app verification (required to remove the "unverified app"
  warning for the public) needs a privacy policy page and a review that can
  take days to weeks. Until approved, real strangers will see a click-through
  warning screen on "Connect Google Drive" — decide together whether that's
  acceptable for an early public v1 or whether to gate the sync button
  behind something until verification clears.
  - **Research finding (2026-08-26):** checked `drive.file`
    (`https://www.googleapis.com/auth/drive.file`, the exact scope this app
    requests — see `DRIVE_SCOPE` in `app.js`) directly in Google Auth
    Platform's Data Access page and confirmed it's classified
    **non-sensitive** — not sensitive, not restricted. That means the
    "days to weeks" review above is the standard brand-verification path
    (privacy policy page, consistent branding, Search Console domain
    verification, a written scope justification, a demo video) — **not**
    the much heavier restricted-scope path, which requires an annual
    third-party security assessment and would be a real barrier for a
    free hobby project. This is a finding, not a completed action —
    nothing has actually been submitted for verification yet. Hours Ledger
    requests the identical `drive.file` scope for its own Drive sync
    (confirmed by reading its `app.js`), so this same finding applies
    there too — mirrored in Hours Ledger's own CLAUDE.md rather than only
    living here, since neither app's verification has actually been
    submitted as of this date.
  - **Privacy policy page shipped (2026-09-01):** `privacy.html` — plain
    language, no legalese, reuses `style.css` wholesale (no separate
    stylesheet). Covers what Drive sync touches (`drive.file` scope only),
    what it's for (syncing the user's own ledger between their own
    devices), where data lives (their `localStorage` + their own Drive, no
    server ever in the loop), no sharing/selling, disconnect-anytime
    without local data loss, and a contact address. Linked from three
    places (the intro popup's note, the Sync &amp; settings hint, and the
    colophon) — three because the intro popup is skipped once `introSeen`
    is set, so it can't be the only discovery path. Real-device verified
    on a phone over LAN (not localhost) before merge: all three links
    resolve, and the page itself reads well at phone width. Merged to
    `main` (`1899707`) and confirmed live on GitHub Pages by hashing the
    served files against local. This satisfies the privacy-policy-page
    dependency for the brand-verification path above — still not submitted,
    but no longer blocked on this piece.
- GitHub Gist was considered and rejected as the sync target over Google
  Drive: a "secret" gist is unlisted, not access-controlled — anyone with
  the URL can read it. Worth revisiting only if a stronger case for it shows
  up later.

## General fixes (2026-10-10, a full localStorage)

Hours Ledger and Money Ledger share one origin and therefore one ~5 MB
localStorage quota on GitHub Pages; a real phone filled it (mostly Hours
Ledger's undo history), and Money Ledger could no longer save. Branch
`investigate/quota-full`, not merged.

- **A failed save never shows a success message for an entry.** Found by
  recording every text the toast was set to: with Drive NOT connected,
  `saveState()` sets "Not saved on this device..." and the handler sets
  "Expense logged" in the same tick, so only the success text is ever
  painted. With Drive connected the order is "Expense logged", then
  "Not saved..." 1.2 s later (the debounced sync's own failed write), which
  is what a phone with sync on shows. `saveState()` now returns whether the
  write landed (`lastSaveOk`) and the entry toasts (quick-add, edit, delete
  with its Undo, automatic recurring entries) go through `toastIfSaved()`.
  Not changed: success toasts for categories, recurring items, import and
  clear - same shape, not part of this fix (Backlog 5).
- **A new install on a full origin still starts.** `money-ledger-device-id`
  was written unguarded at load (and again by `loadState()` on a first
  install); a full origin threw out of the script before anything rendered.
  `persistDeviceId()` swallows only `QuotaExceededError`; the id then lasts
  one page load (the next load makes another, as for any device with no
  stored identity). Limit: an existing install already has the key, so this
  protects new installs and installs whose key was evicted, not the phone
  that reported the bug.
- Tests: `money-ledger-selftest.html` (611), new frame
  `money-ledger-quota-frame.html` for the load-time case. Fail-first: 7 and
  2. Not tested: the rethrow of a non-quota error in `persistDeviceId()`
  (read only).
- **Export backup on a full origin works (ran, headless Chrome, 2026-10-10).**
  With the origin exactly full and nothing left to free, "Export backup
  (.json)" downloaded `money-ledger-backup-<today>.json` (~100 KB) with all 33
  categories, 22 recurring items and every original entry unaltered, plus the
  entries the app logs for due recurring items at load and one that failed to
  save - those exist only in memory, and the file includes them. Not tested:
  how a phone presents the downloaded file.
