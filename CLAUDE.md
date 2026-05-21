# wceu-2026 — project instructions

Single-file static Kraków field-guide hosted on GitHub Pages. Leaflet maps, editorial newsprint style.

## Architecture at a glance

- `index.src.html` — the only editable source. **Gitignored, local-only.** Contains itinerary, day timelines, place-cards grid, leaflet map setup, modal JS.
- `index.html` — the **staticrypt-encrypted** build artifact. This is what ships to GitHub Pages. Never edit it directly.
- `menus/<slug>.json` — modal payloads (restaurant menus *and* event details). Loaded by the page's menu-modal JS over `fetch()`. Public, unencrypted.
- `images/<slug>/…` — image assets for cards and gallery, referenced from both `index.src.html` and `menus/*.json`. Public.
- `places.json` — historical / unused in current build. Don't add new data here; use TRIP_PINS in `index.src.html` instead.
- `zabka.svg` — Żabka logo asset.

## Encryption (important)

`index.html` is **staticrypt-encrypted** on `origin/main`. Do not edit it directly — it's the build artifact.

- **Edit `index.src.html`** (gitignored, local-only).
- **Public assets stay unencrypted**: `menus/*.json`, `places.json`, `images/`, `zabka.svg`. Only `index.html` (itinerary, locations, timings) is gated.
- **Never commit `index.src.html`** or the `encrypted/` directory.
- **`.staticrypt.json` (salt) is committed** so the same salt is reused across re-encrypts — keeps viewers' "Remember me" cookies valid.
- **Ask the user for the password each session.** Never store it in memory, code, or this file.

### Re-encrypt workflow

After every edit to `index.src.html`:

```sh
cd /home/bartech/projects/wceu   # IMPORTANT: must run from project root
STATICRYPT_PASSWORD='<pass>' npx staticrypt index.src.html --short \
  --template-title 'WCEU 2026 — Kraków' \
  --template-instructions 'Password required to view this guide.' \
  --template-color-primary '#8c2818' \
  --template-color-secondary '#faf3e1' \
  --template-button 'Open guide'
mv encrypted/index.src.html index.html && rmdir encrypted
```

Common failure: running `npx staticrypt` from a subdirectory like `images/<slug>/` after a previous `cd` — fails with `ENOENT: index.src.html`. Always re-cd to project root.

Re-encrypt **after every edit** so the local server (`python3 -m http.server 8000`) reflects changes through the encrypted `index.html`. For unencrypted preview while iterating, open `http://localhost:8000/index.src.html` directly.

## Editing `index.src.html` — the moving parts

### TRIP_PINS (around line 1238)

The master list of map pins. Each entry:

```js
{ slug: "u-babci-maliny", lat: 50.0635, lng: 19.9376, kind: "food", label: "9", name: "U Babci Maliny", note: "Thu Jun 4 breakfast · home-style pierogi" },
```

- `slug` matches the `#place-<slug>` anchor on the place-card and the chip `href`.
- `kind` is one of: `home`, `venue`, `food`, `sight`, `event`, `party` — drives marker colour via `PIN_COLOR`.
- `label` shows on the marker (1–2 chars). Use a digit for ordered food stops, a letter for ad-hoc, `★ ♦ P C ?` for special markers.

When you add a new venue, **also**:
- Add a place-card with matching `id="place-<slug>"`.
- If it's day-relevant, add the slug to the right `DAY_PINS` array.

### DAY_PINS (around line 1480)

Per-day filtered pin list rendered into each day's small `<div class="day-map">`. Keys are `map-day-may31` … `map-day-jun07`. Add the slug here when a venue belongs to that day's narrative.

### Day timeline entries (per `<article class="day-detail">`)

Each timeline `<li>` follows:

```html
<li class="ev"><time>HH:MM</time><h4>Title</h4><p>Description.</p><div class="ev-pin"><a class="day-chip" href="#place-slug"><span class="chip-dot" style="background:#8c2818"></span>Label</a></div></li>
```

Variants:
- `class="ev"` — normal step.
- `class="ev anchor"` — the day's anchor event (visually emphasized).
- `class="ev optional"` — optional / clash-warning (faded). Pair with `<span class="opt-tag">optional</span>` or `<span class="opt-tag">TBD</span>` etc.

**Multiple chip options for one timeline slot** are encouraged — chain `<a class="day-chip">` elements inside one `ev-pin`. Used for "pick one":
- Breakfast TBD → multiple breakfast options
- Lunch in centre → multiple walk-in spots
- Dinner → multiple sit-down restaurants

### Place-cards (food + side-events + visit grids)

```html
<div class="place-card" id="place-<slug>">
  <div class="place-img" style="background-image: url('images/<slug>/place-01.jpg');">
    <span class="place-kind">Polish · Pierogi</span>
    <span class="place-rating">★ 4.4</span>
  </div>
  <div class="place-body">
    <h3 class="place-name">Restaurant name</h3>
    <div class="place-addr">Street · neighbourhood · walk time</div>
    <p class="place-note">Short description with <strong>booking note</strong> if needed.</p>
    <div class="place-meta">
      <span>opening hours</span>
      <span>+48 phone</span>
      <a href="#" data-menu="<slug>">Menu ↗</a>
      <a href="https://restaurant-site/" target="_blank">Site ↗</a>
      <a href="https://maps.app.goo.gl/..." target="_blank">Map ↗</a>
    </div>
  </div>
</div>
```

Conventions:
- **`place-kind` is cuisine type only**, never the day/role. Good: "Polish · Pierogi", "Italian · Pizza", "Japanese", "Burgers". Bad: "Tue · Progressus dinner", "Optional · Old Town".
- Event cards (parties, side events) keep day/role labels — only food/restaurant cards use cuisine.
- `place-rating` ★ X.Y is the Google rating. Add via `WebSearch` if missing.
- Cards without an image use a CSS gradient fallback: `style="background: linear-gradient(135deg, #c9a86a 0%, #6b4a18 100%);"`. User often replaces these later with a pasted `images/<slug>/image.png`.
- `data-menu="<slug>"` opens the modal — JSON must exist at `menus/<slug>.json`.

### Modal JSON schema (`menus/<slug>.json`)

Used for both restaurant menus and events.

```json
{
  "id": "<slug>",
  "name": "Display name",
  "eyebrow": "Menu",            // optional; defaults to "Menu". Use "Event" for after-parties etc.
  "description": "1–3 sentence venue blurb.",
  "gallery": ["images/<slug>/place-01.jpg", "..."],
  "currency": "PLN",            // omit for events
  "source": "https://source-url",
  "last_updated_from_source": "YYYY-MM-DD",
  "price_caveat": "Menu rotates seasonally. Verify on-site.",  // optional
  "notes": "Free-form notes — doors, dress code, etc.",         // optional
  "sections": [
    {
      "name": "Przystawki — Starters",
      "notes": "optional section note",
      "items": [
        { "name": "Dish name", "name_en": "English translation", "description": "ingredients / details", "price": 39 },
        { "n": 1, "name": "Numbered item", "price": 48 },
        { "name": "Approx-price item", "price": 19, "price_approx": true }
      ]
    },
    {
      "name": "Section with subsections",
      "subsections": [ { "name": "Subhead", "items": [...] } ]
    }
  ]
}
```

- Use **Polish — English bilingual section names** when the source is Polish (e.g. "Pierogi — Dumplings", "Zupy — Soups"). For English-source menus, just English is fine.
- `item.name` is the primary display string; `item.name_en` shows after an em-dash; `item.name_pl` is the Polish form when the primary is English.
- `price` can be numeric (preferred) or a string for ranges like `"24 / 79"`.
- Events use the same schema — set `eyebrow: "Event"`, populate `description`, `gallery`, `notes`, and use `sections` for "At the venue" / "Practical" bullets without prices.

### Map links — IMPORTANT user preference

- **Use single-place links, never directions.** The user has explicitly rejected `https://www.google.com/maps/dir/?api=1&origin=…&destination=…` route URLs. Use plain place-pin maps instead.
- Acceptable forms (in order of preference):
  1. `https://maps.app.goo.gl/<short>` — when the user provides one, use it verbatim.
  2. `https://maps.google.com/?cid=<decimal>` — CID from the Google Maps URL (`!1s0x…:0xHEX` — convert the hex after the colon to decimal).
  3. `https://www.google.com/maps/search/?api=1&query=<name+address>` — fallback.

### Action items checklist (footer)

Located at the bottom of `index.src.html` (~line 2210). Update bookings / RSVPs / venue confirmations here when itinerary changes. Examples already present:
- `Book <restaurant> — <day> · ~HH:MM · team size`
- `RSVP <party> — <day>`
- `Decide <slot> venue (option A / option B / option C)`

## Image asset rules

- **Hero per card**: `images/<slug>/place-01.jpg` (or `image.png` if the user pasted one).
- **Gallery**: download 5–10 photos from the official site into `images/<slug>/` — interior shots, food shots, exterior. Reference in the `gallery` array of `menus/<slug>.json`.
- **Source preference**:
  - Restaurant's own website (sliders, hero galleries) — almost always public and explicitly published.
  - WCEU event pages — public event promo.
  - Google Maps photos: **avoid the carousel scrape.** The viewer shows "Images may be subject to copyright" and headless browsers can't navigate the carousel. The single photo whose ID is embedded in the maps URL (`!1s<id>` or `6s<full-url>`) is the only one practically grabbable — fetch `https://lh3.googleusercontent.com/p/<id>=s1600` or use the embedded `lh3.googleusercontent.com/gps-cs-s/...` URL with a swapped `w<size>-h<size>` suffix.
- For sites that block hot-linking, use `curl -A "Mozilla/5.0"` and check `file` to confirm you got a JPEG/PNG, not an HTML error page.
- For .pl sites with expired SSL certs (some Polish small businesses), the WebFetch tool will fail — use Google Maps photos or skip the gallery.

## What the user keeps coming back to

These have surfaced repeatedly across sessions — internalize them:

1. **Don't pre-set Google Maps travel mode** in trip links. Walk under ~2 km by default.
2. **No directions URLs** — single-place CID/short links only (see Map links above).
3. **`place-kind` = cuisine type for food cards.** Not the day, not "Optional".
4. **The user pastes hero images** into `images/<slug>/image.png` after the card is added with a gradient block. Re-edit the card to swap once they confirm.
5. **Encryption is mandatory before commit.** No raw `index.src.html` ever goes to git.
6. **Don't commit `image.png` at the repo root** — those are screenshots the user pastes for context (e.g. email screenshots, maps screenshots). They're ephemeral.
7. **Re-encrypt after every edit** even if you're not committing — the local server reads `index.html`.

## Commit conventions

- Title prefix: `Site:` for index/page edits, no prefix for asset-only commits.
- Style: short imperative, e.g. `Site: add Yoast Glow Party and Kinsta dinner side events.`
- **No Co-Authored-By** trailers. **No ticket IDs** in titles.
- Stage explicit files; avoid `git add -A`. Common bundle for a content edit: `index.html` + `images/<slug>/<file>` + `menus/<slug>.json`.
- Don't commit `image.png` at root (paste artifact). Don't commit `encrypted/` or `.playwright-cli/`.

## History note

The repo was rewritten to a single root commit (`8a3d14a`) to remove the pre-encryption history from the public remote. Local safety nets: tag `pre-encrypt-backup` and the `wip` branch still hold the old state.
