# Tennis team results & standings — multi-club

A simple, branded web page that collects a club's LTA tennis results into one
mobile-friendly view — league standings, a head-to-head grid, recent and upcoming
matches, knockout cups, and a club-wide player leaderboard — so members don't have to
navigate the LTA competition site. **One shared codebase serves many clubs**, each with
its own branded page.

**Live:** https://pclutton.github.io/lta-club/ (Paddington Sports Club; Cumberland LTC at `/cltc/`)

---

## How it works

```
LTA site ──(scheduled scraper)──▶ clubs/<slug>/data/results.js ─┐
   daily full + frequent                                        ├─(build)──▶ _site/<slug>/ ──▶ GitHub Pages
   results-only refresh                                         │
app/index.html + clubs/<slug>/{club.json, teams.yaml} ──────────┘
   shared shell        per-club branding + scrape + team names
```

- **`app/index.html`** — the entire app (no framework). It carries *no* club identity:
  branding (name, logo, colours, nav, social, the home-club matcher, `ltaUrl`) comes from
  `window.__CLUB__`, the data from `window.__RESULTS__`. Tabs: one per competition
  (standings · head-to-head matrix · team players), **Upcoming Matches**, **Recent
  Matches**, grouped **Knockout Competitions**, and a **Players** leaderboard.
- **`clubs/<slug>/`** — everything specific to one club:
  - `club.json` — branding **and** the scraper `scrape` block (plus knobs like `upcomingDays`).
  - `teams.yaml` *(optional)* — rename LTA events to your own team labels (below).
  - `assets/logo.png` + `icon-180/192/512.png` (home-screen icons).
  - `data/results.js` — that club's scraped data (`window.__RESULTS__ = { … }`), committed by the scraper.
- **`scraper/scrape.mjs`** — Node + Playwright. Loops every `clubs/*/club.json`. Two modes
  (`SCRAPE_MODE`): **full** (discovery + rosters + everything) and **results** (a light
  refresh of live league standings/matches only — no discovery or player pages — reusing
  the draw URLs already in the data; a fraction of the load). **Fails safe per club**:
  retries, and on a bad read it keeps that club's last-good data (see *Robustness*).
- **`scripts/build-site.mjs`** — assembles `_site/<slug>/` from `app/index.html` + each
  club folder (writes `club.js`, copies data/assets, generates the manifest), plus a root
  redirect to the default club. Validates each club's data before publishing.
- **`scripts/make_icon.py`** — home-screen icons from the crest + brand colours:
  `uv run --with pillow python scripts/make_icon.py <slug>`.
- **`.github/workflows/update-results.yml`** — full scrape at **06:00 UTC**, results-only
  at **12:00/16:00/19:00/21:00**, plus manual dispatch (with a `mode` input). Commits
  changed data → builds `_site` → deploys. A plain push just rebuilds & deploys (no scrape).

### `club.json` scrape sources

`scrape.sources` is a list, so a club can pull from several places:

- `{ "type": "group", "url": "…/association/group/<GUID>" }` — crawl a county/association
  group page and auto-find every league the club is in (no GUIDs to maintain).
- `{ "type": "leagues", "leagues": [{ "id": "<GUID>", "name": "…" }] }` — explicit league GUIDs.
- `{ "type": "link", "name": "NPL …", "url": "…", "knockout": true }` — a link-only entry
  (no table scraped); with `knockout: true` it joins the grouped Knockout tab.

**NPL note:** the NPL publishes through LTA's *legacy* `/sport/draw.aspx` interface (a
different page structure), so it's shown as a **link** inside the Knockout tab rather than
table-scraped.

### Team-name overrides — `clubs/<slug>/teams.yaml` (optional)

LTA labels a team by its *event* + *division* ("Mens Doubles", "West Intermediate"). To
show your own labels instead ("Mens Team 1"), add a YAML file keyed by LTA **League**;
each row matches on `LTAEvent`/`LTADivision` (and optional `LTATeam`) and renames via
`DisplayEvent` (+ optional `DisplayDivision`). The division is still shown. Unmatched
teams keep the LTA name, and the label also drives sort order within a division tier.

```yaml
Middlesex Summer League 2026:
  - { LTAEvent: Mens Doubles, LTADivision: West Intermediate, DisplayEvent: Mens Team 1 }
  - { LTAEvent: Mens Doubles, LTADivision: West Division 3,   DisplayEvent: Mens Team 2 }
```

The scraper bakes these in, so **update it each season** (the LTA reassigns divisions) —
it applies on the next scrape, no code change.

## Robustness & freshness

Built to be safe enough for a club's official site:

- **Never shows wrong/blank data.** A scrape only overwrites good data when the new read
  is at least as complete; a degraded competition keeps its **last-good** version, flagged
  `stale` with an "as of &lt;date&gt;" note. A full failure keeps the whole last-good file.
- **Tells you when something's wrong.** A separate CI **`health` job** turns the run red
  (→ GitHub emails the owner) when the daily full scrape fails or a club degrades, *without*
  blocking the deploy. Frequent results-only runs are silent (the daily run is canonical).
- **On-site transparency.** A subtle banner if data is &gt;2 days old, a per-competition
  "as of" note when showing retained data, and a **link to the club's LTA page** whenever
  data is missing/stale so members can still find live results.
- **Freshness.** Results-only runs every few hours mean a result usually appears within a
  few hours of a captain uploading it, at roughly the same daily LTA load as one full scrape.

## Add a club

1. `clubs/<slug>/club.json` — name, shortName, `matcher` (regex matching the club's
   standings rows, e.g. `"cumberland"`), colours, nav, social, address, `upcomingDays`,
   and the `scrape` block (clubSearch + sources).
2. `clubs/<slug>/assets/logo.png` — the crest; then
   `uv run --with pillow python scripts/make_icon.py <slug>`.
3. *(Optional)* `clubs/<slug>/teams.yaml` for team-name labels.
4. Push. A scheduled or manually-dispatched (`mode: full`) run scrapes it → `/<slug>/`.

Each club folder is self-contained, so extracting one to its own repo later is a
copy-paste relocation, not a rewrite.

## Local preview

```bash
cd scraper && npm install && cd ..   # Playwright + js-yaml (only needed to scrape)
node scripts/build-site.mjs          # assemble _site/
python -m http.server 8000 --directory _site   # visit http://localhost:8000/psc/
```

Scrape locally: `cd scraper && npm run scrape` (full) or `SCRAPE_MODE=results npm run scrape`
(light). `DEBUG=1` dumps page HTML to `scraper/debug/` when adjusting selectors.

### Login: not normally needed

LTA league/standings pages are public; the scraper reads them anonymously. An optional
login via `LTA_USERNAME`/`LTA_PASSWORD` (env vars or repo secrets) is supported — never
commit credentials.

## Roadmap / TODO

- Render the full **knockout bracket** (all rounds) per cup, not just the last match.
- Table-scrape **NPL** (league + finals) from LTA's legacy `/sport/draw.aspx` interface.
- Optional: a precise external trigger (e.g. cron-job.org → `workflow_dispatch`) if GitHub's
  scheduled-run timing (often delayed on the hour) proves too loose.

## Status

In use as a proposal/prototype for PSC and Cumberland LTC on the live LTA feed. Per-club
branding, team names, and colours are all config; robustness and alerting are in place.
