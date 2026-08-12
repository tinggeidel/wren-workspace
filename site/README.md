# wren.expo.app — marketing site + privacy policy

Static site served by EAS Hosting under the wren Expo project.

- `dist/index.html` — landing page (waitlist embed via Tally form lbZN8B)
- `dist/privacy/index.html` — privacy policy (public URL is /privacy/ — TRAILING SLASH required; bare /privacy falls back to the landing page)
- `dist/privacy.html` — flat copy (not reliably served; the /privacy/ directory version is canonical)

## Deploy (after editing)
From `wren/` (the app dir, where eas.json lives):

    npx eas-cli deploy --prod --export-dir ../site/dist --non-interactive

Notes:
- --export-dir must be a project-relative path (absolute /tmp paths fail).
- Edge cache is max-age=3600 — production can take up to 1 HOUR to show changes. Verify against the per-deploy URL printed by the command, not wren.expo.app.
- Privacy policy content commitments must track the app: if the app ever adds a new network call/SDK, update /privacy in the same change.
