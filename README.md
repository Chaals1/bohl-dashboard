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

---

## Files

| File | Where it lives | Purpose |
|---|---|---|
| `index.html` | GitHub repo (this is the whole site) | The dashboard itself |
| `bohl-sheets-proxy-worker.js` | Local only, deployed to Cloudflare | Worker source — Google Sheets proxy |
| `bohl-live-fetch-setup-checklist.md` | Local only | One-time setup steps for the live-fetch backend (completed) |
| `levi_brochure_notes.md`, `sean_brochure_notes.md` | Local only | Research notes from reading each site's brochure, feeding the manual criteria overrides in `index.html` |
