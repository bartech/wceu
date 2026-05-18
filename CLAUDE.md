# wceu-2026 — project instructions

Single-file static Kraków field-guide hosted on GitHub Pages. Leaflet maps, editorial newsprint style.

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
STATICRYPT_PASSWORD='<pass>' npx staticrypt index.src.html --short \
  --template-title 'WCEU 2026 — Kraków' \
  --template-instructions 'Password required to view this guide.' \
  --template-color-primary '#8c2818' \
  --template-color-secondary '#faf3e1' \
  --template-button 'Open guide'
mv encrypted/index.src.html index.html && rmdir encrypted
```

Then commit + push as normal (no force-push needed — history is clean from `8a3d14a` onward).

## History note

The repo was rewritten to a single root commit (`8a3d14a`) to remove the pre-encryption history from the public remote. Local safety nets: tag `pre-encrypt-backup` and the `wip` branch still hold the old state.
