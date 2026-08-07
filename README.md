# Bohl Phoenix Market — Site Selection Dashboard

**Live:** https://chaals1.github.io/bohl-dashboard/
**Repo:** Chaals1/bohl-dashboard (single file: `index.html`)

A shared site-selection dashboard for Bohl's Phoenix-metro franchise expansion, used jointly by
four franchisees — Charles Swindler, Sean Provost, Levi Ayriss, and Kyle Garrett. It scores every
candidate site each broker is tracking against Bohl's tenant profile (size, patio, visibility,
access, parking, center age, population, income), plots them on a map, and tracks each deal's
real progress from first tenant interest through a signed lease.

---

## What it does

- **Interactive map** (Leaflet + OpenStreetMap) — every candidate site plotted, pin color/score
  reflecting fit against Bohl's criteria.
- **Franchisee filter** — All / Charles / Sean / Levi / Kyle, filtering the map and ranked list
  together. Kyle's region isn't set up with a broker sheet yet, so his tile stays at zero until it is.
- **Stat tiles** — market total plus a per-franchisee breakdown (strong fit / possible fit /
  needs data) instead of a single number.
- **Sites ranked by fit**, with a second tab for **sites in progress**, ranked by how far along
  the deal actually is.
- **Real deal-stage pipeline** (Tenant Interested → Broker Contacted → Details Received → LOI
  Drafting → LOI Submitted → Lease Negotiation → Signed, or Passed) — read directly from each
  broker's own "Site Status Updates" log, not a manual tracker we maintain separately.
- **Live data, no manual uploads.** The dashboard fetches straight from Charles's, Sean's, and
  Levi's Google Sheets on every page load — see **Live-fetch architecture** below. When a broker
  moves a site to their sheet's "Non-Active" tab, it simply disappears from the dashboard on the
  next load; the full history stays on their sheet, nothing to archive on our side.
- **Brochure links** come straight from each broker's "Marketing Material" hyperlink in their
  sheet — no manual downloading/re-uploading of PDFs.
- **Chipotle sales-volume overlay** — a toggleable reference layer (top-left of the map, next to
  "Show all sites") plotting 52 existing Chipotle locations across the Phoenix metro, each
  labeled with its estimated annual sales volume, for competitive/market-strength context.
- **Automated brochure checks** — a scheduled task checks all three sheets several times a day
  for a brand-new or replaced brochure, reads it, scores it against Bohl's criteria, and
  republishes the dashboard on its own. No one has to notice a new brochure got attached and ask
  for it to be read — see **Automated brochure scoring** below for how it works and what it
  depends on.
- **Password gate** — client-side deterrent plus a server-side check in the Worker (see below);
  keeps the link out of search engines and casual hands. Not meant to withstand a determined
  attacker — the underlying broker data is only ever served after the correct password reaches
  the Worker.
- **Dark mode** toggle, count-up stat animations, competitive-proximity notes per site.

---

## Live-fetch architecture

```
Charles's Sheet  ─┐
Sean's Sheet     ─┼─▶  Cloudflare Worker  ─▶  index.html (GitHub Pages)  ─▶  browser
Levi's Sheet     ─┘     b-dashboard-proxy       (fetches on every load,
 (read-only,             .workers.dev            password required)
  service account)
```

- **Cloudflare Worker** (`b-dashboard-proxy`, source kept locally as
  `bohl-sheets-proxy-worker.js` — not committed to GitHub) authenticates to Google as a service
  account each of the three sheets has been explicitly shared with (Viewer access), fetches the
  "Active" and "Status Updates" tabs, checks the dashboard password, and returns clean JSON.
- **`index.html`** calls the Worker for Charles, Sean, and Levi in parallel on load. Each
  franchisee fails independently — if one broker's sheet is briefly unreachable, that franchisee
  falls back to a last-known snapshot embedded in the file while the other two stay live.
- If the Worker is fully unreachable, the whole dashboard falls back to the embedded snapshot
  (dated) rather than breaking.
- Nothing sensitive is ever committed to GitHub: the service account key, dashboard password, and
  spreadsheet IDs live only as encrypted secrets in the Cloudflare Worker's settings.

**To update site data:** nothing to do — brokers update their own Google Sheet, and it shows up
on the dashboard the next time anyone opens it.

**To update the Worker:** edit `bohl-sheets-proxy-worker.js` locally, then paste the full file into
Cloudflare Dashboard → Workers & Pages → `b-dashboard-proxy` → Edit code → Deploy.

**To update the dashboard itself:** edit `index.html`, then GitHub → this repo → Add file → Upload
files → commit to `main`. GitHub Pages rebuilds automatically (~30–60s); check the Actions tab for
the `pages-build-deployment` run to confirm.

---

## Automated brochure scoring

A scheduled task (`bohl-brochure-check`, defined at
`/Users/charles.swindler/Claude/Scheduled/bohl-brochure-check/SKILL.md`) runs at 8am, 11am, 2pm,
5pm, and 8pm daily. Each run: fetches all three live sheets, compares every site's current
brochure link against the one last read for it (tracked via a `sourceUrl` field on each scoring
entry in `index.html`), and for anything new or replaced, reads the PDF, scores it against Bohl's
criteria, updates `index.html`, and republishes — all without anyone asking. Most runs find
nothing and stay silent; that's expected.

**It also flags a different, softer problem it can't fix on its own:** a broker sometimes attaches
a brochure by email, Drive, or plain-text paste without actually using Google Sheets' "Insert
link" on the Marketing Material cell — so nothing about the sheet actually changes from this
system's point of view, even though the broker thinks it's done (this happened for real, Aug 2026 —
Sean's "The Groves"). Each run scans the notes on any site with no Marketing Material hyperlink for
language suggesting material was sent but never linked, and reports it once (tracked in a local
`missing_link_flags.json` so the same note doesn't get re-flagged every few hours) — framed as
"worth checking with the broker," since fixing the sheet itself isn't something this system can do.

**What it depends on:** the Claude desktop app running and Chrome open with the extension
connected, on Charles's machine, at each check time. If the laptop's closed or the app isn't
running, that check simply doesn't fire — it isn't a background server process. In practice it
behaves like "catches up whenever the laptop's next open" rather than hitting all five slots
precisely. The task checks its own browser connection first and will tell you directly if a run
couldn't proceed, rather than silently doing nothing and leaving it unclear whether it's working.

**Dream-state idea, not yet built:** true instant-on-upload scoring (dashboard reflects a new
brochure within moments, with nothing running on anyone's computer) would mean moving the scoring
step off this scheduled task entirely — a Google Apps Script trigger on each sheet firing the
moment a brochure link changes, calling a new Cloudflare Worker endpoint that hits the Anthropic
API directly. That needs a real (billed) Anthropic API key and moving scoring data out of the
static HTML into something the Worker can write to live. Pinned for later.

---

## Known limitations / lessons learned

- **Address-matching is the fragile point.** Coordinates, brochure scoring, and competitive-proximity
  data are all keyed by a normalized version of each site's address. Most broker addresses include
  a city/state suffix, but intersection-only addresses sometimes don't — and a broker can edit a
  sheet's formatting at any time, even for a site that's already been scored, silently breaking an
  exact-match lookup that used to work. This actually happened (Aug 2026): three of Levi's East
  Valley sites started showing up on the map in Charles's West Valley territory because their
  addresses lost their city suffix after being keyed with one. The lookup logic now tolerates a
  city/state suffix appearing on one side but not the other, and any site that still can't be
  matched falls back to a point within its *own* franchisee's territory (never someone else's) and
  renders with a dashed pin outline so a future mismatch is visible on the map instead of blending
  in silently. Worth a periodic glance at the map for any dashed pins.
- **Single shared password today.** Everyone uses the same password; revoking one person's access
  means rotating it for everyone. Pinned idea: an "Access" tab in a sheet mapping password → person
  → allowed markets/franchisees, read live by the Worker the same way broker data is — so adding a
  broker, cutting someone off, or scoping a future Utah/Idaho team to their own region is a
  spreadsheet edit, not a code change.
- **One market today (Phoenix).** Expanding to Utah, Idaho, etc. is mostly data entry on the
  backend (new owner + spreadsheet ID), but the UI currently assumes one region (map default
  center, Chipotle overlay, hero text). Pinned idea: a market selector above the franchisee filter
  that scopes the map, stats, and competitive overlay to whichever market is selected.

---

## How we got here

1. Compiled initial site data from the broker spreadsheet, built the Bohl fit-scoring model
   (size, patio, visibility, access, parking, center age, population, income), and geocoded every
   address.
2. Built the first live dashboard artifact with a real Leaflet/OpenStreetMap map, Bohl brand
   colors, animated stat tiles, dark mode, and competitive-proximity notes per site.
3. Published a password-gated, shareable version on GitHub Pages.
4. Expanded from a single-franchisee tool into a shared 4-person Phoenix market dashboard —
   compiled and read every brochure for Sean's and Levi's sites, built manual criteria overrides
   for each.
5. Redesigned the UI: 4-metric stat tiles per franchisee, a combined franchisee filter driving
   both map and ranked list, matched panel heights, and a "sites in progress" tab.
6. Built the real deal-stage pipeline, sourced directly from each broker's existing status logs
   instead of a simple keyword scan.
7. Built the locked-down live-fetch backend (Cloudflare Worker + Google service account),
   replacing manually-exported CSV snapshots with a live read straight from each broker's sheet —
   including brochure links, which now come from the broker's actual hyperlink instead of a
   manually maintained list.
8. Simplified: dropped unused "Non-Active" tracking once we confirmed passed sites already just
   disappear from the dashboard on their own when a broker moves them off their Active tab.
9. Added the toggleable Chipotle sales-volume map layer, sourced from a Google My Maps export.
10. Built the automated brochure-checking scheduled task — closing the last manual gap, where a
    new brochure's link updated live but its actual fit score still needed someone to notice and
    ask for it to be read. Verified with a live test run that found and correctly scored four real
    new brochures unprompted, then refined the task's instructions based on what that test
    actually required (a validated PDF-reading method, and reporting clearly instead of failing
    silently when the browser connection is down).
11. Fixed a real bug where three of Levi's East Valley sites were showing up in Charles's West
    Valley territory on the map (and one had a stale, unapplied score) due to an address-key
    mismatch, and hardened the address-matching and coordinate-fallback logic so the same class of
    bug degrades safely (and visibly) instead of silently again — see **Known limitations** above.

---

## Files

| File | Where it lives | Purpose |
|---|---|---|
| `index.html` | GitHub repo (this is the whole site) | The dashboard itself |
| `bohl-sheets-proxy-worker.js` | Local only, deployed to Cloudflare | Worker source — Google Sheets proxy |
| `bohl-live-fetch-setup-checklist.md` | Local only | One-time setup steps for the live-fetch backend (completed) |
| `levi_brochure_notes.md`, `sean_brochure_notes.md` | Local only | Research notes from reading each site's brochure, feeding the manual criteria overrides in `index.html` |
| `bohl-brochure-check/SKILL.md` | `~/Claude/Scheduled/bohl-brochure-check/` | The scheduled task's instructions — read this to see exactly what the automated check does each run |
