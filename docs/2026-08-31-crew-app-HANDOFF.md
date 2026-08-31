# Handoff — the allocation board, and what a crew app inherits

**Date:** 31 August 2026
**Repo:** `youcef-stack/spmelb-allocation-page`
**Scope of this doc:** Part 1 documents the board exactly as it is built today.
Part 2 is analysis: what a crew-facing app can reuse from it, and what is
genuinely missing.

Everything in Part 1 is verifiable from `index.html` at the commit this doc was
written against (`60c9c21`). Part 2 is a reading of that code, not a spec anyone
has signed off.

---

## Part 1 — the board as built

### The shape of it

One file. `index.html`, 2,388 lines: a `<style>` block (lines 6–629), the markup
(lines 631–648), and one `<script>` (lines 650–2388). No build step, no bundler,
no dependencies, no tests, no linter. You edit the file and it is deployed.

It is a static page. All state lives in the Supabase Edge Function it talks to:

```
https://rkdfjhpjwaoqgsgadwem.supabase.co/functions/v1/allocation
```

**That function is not in this repo.** It is the entire backend — auth, the
database, the storage signing, the work-type list — and you cannot understand a
change to this page without it. Finding it is step one for whoever picks this up.

### Who the board is for

The office. Rows are contractors, columns are days, a card is a booking. The
top row is **Unallocated** — the work nobody is on yet — and it always draws its
cards rather than a count, deliberately:

> The cards, ALWAYS — never a bare count you have to know to tap. This row is the
> to-do list of work nobody is on yet, and hiding it behind a gesture nobody
> discovers is the same as not having it at all.

The vocabulary matters and the code is strict about it: **a job is a site, a
booking is a visit to it.** One address gets undergrounds now and hook-ups months
later, so the work type is asked on the visit and never on the site.

### Auth

Credentials arrive in the URL fragment and are read once at boot:

```js
const auth = new URLSearchParams(location.hash.slice(1));
```

Three opaque keys — `u`, `e`, `s` — are copied verbatim into the query string of
every request by `authQuery()`, along with `p`, the week-page offset. The page
never inspects or validates them; only the function does.

The reason for the fragment is written into the file:

> The token never reaches THIS host: it arrives in the fragment, which browsers do
> not send to the server, so GitHub never sees it. It is passed on only to
> Supabase.

Consequence worth stating plainly: **the link is the credential.** Anyone holding
the full URL has full write access to the board. There is no sign-in, no session,
no logout, and no per-user scoping in the client.

### The API contract

Reads are `GET ?<auth>&<one selector>`:

| Selector | Returns |
|---|---|
| *(none)* | the grid for page `p` |
| `find=<q>` | `{ sites: [{ id, client, address, lot_number }] }` |
| `job=<jobId>` | `{ job }` — see below |
| `supervisors=<clientId>` | `{ supervisors: [{ id, name }] }` |
| `builders=<q>` | `{ builders: [{ id, name, short_name }] }` |
| `files=<jobId>` | `{ files: [{ id, name, plan_type, size_bytes }] }` |
| `descriptions=1` | `{ descriptions: [string] }` — the description autocomplete |

Writes are `POST ?<auth>` with a JSON body. Every write is dispatched on an
`action` field **except the move**, which is the one un-namespaced body in the
app and posts `{ bookingId, from, to }` bare. That asymmetry is a wart; it is the
oldest write in the file and nothing else looks like it.

| `action` | Body | Notes |
|---|---|---|
| *(none)* | `bookingId, from, to` | move a card |
| `dayOff` | `resourceId, day` | toggles |
| `reorder` | `order: [resourceId]` | full order, not a delta |
| `createSite` | `client, clientId, supervisorId, address, lot` | → `{ jobId, reused }` |
| `createBooking` | `jobId, workType, day, startTime, resourceId` | |
| `editBooking` | `bookingId` + one of `day`/`startTime`/`workType`/`notes` | `startTime` always sends `day` too |
| `cancelBooking` | `bookingId` | **toggles** — also un-cancels |
| `repeatBooking` | `bookingId` | same again, next working day |
| `crew` | `bookingId, what: add\|remove\|lead, resourceId` | |
| `createBuilder` | `name` | → `{ clientId, name }` |
| `createSupervisor` | `clientId, name, email, phone` | reused on name match |
| `updateSupervisor` | `supervisorId` + `email`/`phone` | |
| `setJobSupervisor` | `jobId, supervisorId` | |
| `uploadUrl` | `jobId, filename, mime` | → `{ signedUrl, path }` |
| `recordFile` | `jobId, path, name, sha256, size, mime` | → `{ duplicate }` |
| `describeFile` | `fileId, description` | → `{ description }` |
| `deleteFile` | `fileId` | |
| `fileUrl` | `fileId` | → `{ url }`, signed |
| `reportUrl` | `reportId` | → `{ url }`, signed |

**Two error channels, and they are not the same.** A non-2xx response carries
`body.error` — the board replaces itself with the message. A 200 carries
`body.problem` — a business complaint ("that crew is already booked"), shown in
the sheet's `#prob` or as a toast. Handle both or you will swallow one.

**Writes return the grid.** Most mutations answer with a fresh `grid`, which the
client renders immediately:

> The grid always rides back with the write, so the board cannot drift from the
> table it is drawing.

There is no local mutation of `grid` anywhere. The server's grid is the only
truth, and `render()` is a full innerHTML rebuild of the table.

### Payload shapes

```
grid = {
  days:        [{ iso, label, isWeekend, isToday }],       // server decides how many; 21 today
  rows:        [{ resourceId, name, count,
                  off: [iso],                              // crossed-out days
                  cells: { [iso]: [card] } }],
  unallocated: { [iso]: [card] },
  workTypes:   [{ value, label }]                          // the whole picker
}

card = { bookingId, jobId, label, client, address, lot,
         workType, startLabel, isLead,
         crew: [{ resourceId, name, isLead }] }

job = { client, lot_number, address, client_id,
        supervisor, supervisorId, supervisorEmail, supervisorPhone,
        visits:  [{ id, day, startValue, workType, status, notes,
                    crew: [{ resourceId, name, isLead }] }],
        reports: [{ id, bookingId, day, workType,
                    state, stateLabel, lead, photos, hasPdf }] }
```

`grid.workTypes` is the only source of the work-type list — the client has no
copy. `COLOURS` (line 658) maps work type to a colour echoing BaseApp's palette,
with `other` as the fallback, so an unknown type renders grey rather than broken.

### Gestures, and why they are the way they are

This board is opened on phones, so every interaction has a touch path. The rules
are subtle and were each paid for by a bug — read this section before touching
the event handlers.

- **Drag a card** sideways to change the day, up/down to change who is on it.
- **Tap a card, then tap a square** — the touch fallback. HTML5 drag-and-drop
  does not fire on touch at all. Tapping the card again puts it down.
- **Drag a name** (`th[data-row]`) to reorder people. This is a *different* drag
  from the card drag, tracked in its own `draggingRow` state, and it bails the
  moment a card drag starts. Both register document-level `dragstart`/`drop`
  listeners; the order they are attached matters.
- **Tap an empty square** to add work there — but on a **260 ms timer**, because
  a double-tap crosses the day out and the browser fires the single tap first.
  The timer re-checks its guards *at firing time*, not when it was set:

  > 260 ms is long enough for a card to have been picked up or a sheet to have
  > opened in between.

  Getting this wrong is commit `bbdce72`: a stray tap opened a sheet over the
  next gesture, which reads to the user as the drag simply not working.
- **Double-tap an empty square** to cross somebody out for that day. Only on
  empty squares — "there is work here; that is not a free day".
- **Tap the address** on a card to open the job file. It sits *inside* a card, so
  it must `stopPropagation()` before the card branch picks the job up. Same for
  the `⋯` crew button. Both are handled first in the click listener.
- **Escape** closes sheets in priority order: job → crew → add → drop the picked
  card.

There is no per-cell add button on purpose: "29 rows across 21 days is 609 of
them."

### Sheets

Three modal sheets, one `.veil` at a time, each guarded so it cannot open over
another (`if (crewSheet || sheet) return;`).

- **Job sheet** (`openJobSheet`) — supervisor, bookings, reports, files. Saves
  repaint *in place* via `refreshBookings()` rather than closing and reopening;
  tearing the veil down made the screen flash the board for a beat on every save
  (`fb56abd`).
- **Crew sheet** (`openCrewSheet`) — reachable from any copy of a card, and
  always shows the whole crew, not just the row you tapped. Unlike the job sheet
  it *does* close and reopen itself after a write, deliberately, so the list is
  rebuilt from the fresh grid.
- **Add / quick-add** (`openSheet` / `openQuickJob`) — share `renderSiteBlock()`,
  branching on `sheet.quick`.

`stashSite()` exists because the site block is re-rendered whenever a builder is
picked, and losing a half-typed address to that is maddening. Any new field in
that block needs a line in `stashSite()` or it will evaporate.

### Files

Plans go **straight to Storage on a signed URL** — `uploadUrl` → `PUT` →
`recordFile` — so the bytes never pass through the function and a 20 MB plan set
is not squeezed through a JSON body. `recordFile` sends a SHA-256 the client
computes, and answers `{ duplicate }`. They land in `job_plans` and the private
`job-plans` bucket that the Groundplan ingest already writes to.

Two different file panels: the quick-add sheet holds files in memory and renames
them *before* upload (so what lands is not `IMG_4821`); the live panel on an
existing job uploads on drop. Descriptions autocomplete from `descriptions=1` and
are cached across job opens in `liveFiles.descriptions`.

### Reports

Read-only here. `📝` draft, `⏳` submitted, `✅` approved, with the photo count
and the lead. Every report on the job is listed in the order the work happened,
**including one whose visit was later called off** — "the visit went away and the
work did not" — flagged as such. `Open` mints a signed PDF URL via `reportUrl`.

The supervisor block deserves a note. The report pack is emailed to the
supervisor, and a missing address silently sends the builder's report to the
owner instead. So the gap is drawn in red, inline, where it is noticed and can be
filled in on the spot. That is not decoration — it is the fix for a real failure.

### Conventions in this code

- String-concatenated HTML, escaped through `esc()`, wired up by
  `querySelectorAll` afterwards. Follow it; don't introduce a framework for one
  panel.
- `esc()` escapes `& < > "`. **Every generated attribute is double-quoted** — a
  single-quoted attribute would be an XSS hole. Verified clean at `60c9c21`.
- Comments explain *why*, in the owner's voice, usually naming the failure the
  code prevents. Match that. A comment restating the code is noise here.
- `onchange`, not per-keystroke, for anything that writes — one save per
  description, not one per letter.
- IDs like `#prob`, `#save`, `#lfileBox` are global and reused across sheets. It
  works only because one sheet is open at a time. Commit `788a9cd` is what
  happens when that assumption breaks: two elements called `q` and the sheet
  grabbed the wrong one.
- `dayNameOf()` parses `iso + "T00:00:00Z"` as UTC on purpose. Local parsing
  shifts the day. The header says "Times are Melbourne time."

### Rough edges

Real, small, and safe to leave alone if you are here for something else:

1. `openSheet()` declares `const free = false` and stores `sheet.free`, which
   nothing reads. Its comment still describes a `cell === null` case from when
   the top button opened this sheet — it now opens `openQuickJob()` instead.
2. The move is the only action-less POST (above).
3. `cancelBooking` toggling both ways is undocumented anywhere but the button
   label ("Put it back").
4. No tests, no CI, no linter. The only verification is opening the page.

---

## Part 2 — what a crew app inherits

*This half is analysis. Nothing below has been built or agreed.*

The board is the office's side: who is working where. The counterpart is the
contractors' side — a crew member seeing their own visits, opening the plans, and
filing the report. Here is what that app would get for free and what it would
have to build.

### Already there, and reusable as-is

- **The auth pattern.** Fragment-carried credentials to a Supabase Edge Function,
  with the host never seeing the token. It works and it is cheap.
- **Signed URLs for private files.** `fileUrl` and `reportUrl` already mint them.
  A crew app can open plans and report PDFs on day one with no backend work.
- **Direct-to-Storage upload.** `uploadUrl` → `PUT` → `recordFile`, with SHA-256
  dedupe. This is exactly the path site photos from a phone would want, and it is
  already proven on 20 MB plan sets over site signal.
- **The job payload.** `?job=` already carries the address, the lot, the
  supervisor's name, email and phone, every visit with its crew, and every report
  with its state and photo count. Most of a crew-facing job screen is in that one
  response.
- **`booking.notes`.** The field is labelled "Note to contractor" and the office
  is already filling it in. **Nothing reads it back to the contractor.** That is
  the single clearest piece of value sitting unused in the data model.
- **The work-type list and palette.** `grid.workTypes` plus `COLOURS`, so both
  apps can look like the same system.
- **`resourceId`.** The contractor identity already exists and is already the key
  the whole board is organised by.

### What is missing, in rough order of difficulty

1. **Per-resource auth.** This is the blocker. The current credentials are the
   office's, and the grid endpoint returns *the entire board*. A crew app cannot
   reuse `load()` without handing every contractor every other contractor's work.
   It needs its own credential scoped to one `resourceId`, and a read endpoint
   that returns only that person's cells. That is a function change, not a client
   change.
2. **A read endpoint shaped for one person.** The 21-day × 29-row grid is the
   wrong payload for a phone showing "today and tomorrow". Same data, different
   cut.
3. **Report submission.** The state machine (draft → submitted → approved) is
   visible in the board but only ever read. Moving draft → submitted is precisely
   what the crew app is for, and none of that write path exists here.
4. **Writing `off`.** The office crosses out days. Crews are the ones who know
   they are unavailable. `dayOff` already exists as an action; it would need to
   accept a request from the crew side, which is a permissions question more than
   a code one.
5. **Notification.** Nothing pushes. The board is polled by opening it. A crew
   app that only shows changes when you happen to open it will get blamed for
   missed jobs.

### Advice for whoever builds it

- **Find and read the Edge Function first.** It is most of the system and none of
  it is in this repo. Do not start by writing client code.
- **Resist sharing a codebase with this page.** They share an API, not a UI. This
  file's string-concatenation style is right for a 2,000-line internal board and
  wrong for an app with real routing and offline behaviour.
- **Keep the vocabulary.** Job = site, booking = visit. The moment the two apps
  disagree on that, every conversation about a bug costs ten minutes.
- **Copy the touch discipline, not the code.** The 260 ms tap timer, the
  re-checked guards, the `stopPropagation` on nested controls — those are the
  scars of a board used one-handed in a ute. A crew app will earn the same ones
  faster if it starts from those rules.
