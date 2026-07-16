# pubsigzt — Zeotap email signature generator + campaign banner

Two things live in this repo:

1. **`banner.png`** — the campaign banner image used in the Zeotap email signature tool. It's served publicly via jsDelivr and referenced by URL from the signature generator, so replacing this file updates the banner everywhere the signature is used (subject to the caching notes below).
2. **`logo.png`** — the Zeotap wordmark used in the signature (sourced from the `zeotap-design` brand skill's assets, brand blue on transparent). Referenced the same way as the banner, via `LOGO_URL`.
3. **`app/`** — the source for the signature generator web app, deployed to Google Cloud Run.

Live app: **https://signature-generator-1016745695114.europe-west1.run.app**

## How it works

The signature generator (`app/index.html`) is a single self-contained HTML page. Employees fill in their name, title, mobile, email, and LinkedIn URL, and it builds an email-client-safe HTML signature (table-based markup, inline styles, Arial fallback font) that they copy into Gmail/Outlook.

The optional campaign banner is a fixed block in that generated signature:

```js
const BANNER_VERSION = 2;
const BANNER_URL = `https://cdn.jsdelivr.net/gh/mackiiops/pubsigzt@main/banner.png?v=${BANNER_VERSION}`;
const BANNER_LINK_URL = "https://zeotap.com/...";
```

- `BANNER_URL` points at `banner.png` in *this* repo, served through jsDelivr's CDN (`cdn.jsdelivr.net/gh/OWNER/REPO@BRANCH/PATH`) rather than `raw.githubusercontent.com`, because jsDelivr responds to purge requests and is built for this kind of "swap the file, everyone sees it" use case.
- Because the `<img>` tag references a live external URL rather than an embedded image, **most email clients (Outlook, Apple Mail, webmail) refetch the banner each time an email is opened** — so replacing `banner.png` here updates the banner retroactively in every email that uses the signature, past and future, without anyone re-copying anything.
- **Gmail's Settings → Signature editor is the one exception.** Gmail proxies and caches signature images *at the moment the signature is saved*, not on every email open. Swapping `banner.png` won't change what's already saved in someone's Gmail signature until they revisit Settings → Signature and hit Save again (which forces Gmail to refetch). This is a known Gmail platform quirk, not a bug in this app.

## How to update the banner

1. Replace `banner.png` in this repo with the new image (same filename, ideally the same ~600×220px-or-narrower aspect ratio so it doesn't resize awkwardly in people's signatures).
2. Commit and push.
3. **Bump `BANNER_VERSION` by 1** in `app/index.html` (find the `const BANNER_VERSION = N;` line near the top of the `<script>` block). This is required — jsDelivr sets a 7-day browser cache-control on the image, so without a version bump, anyone who already loaded the signature generator (or an email with the banner) keeps seeing the *old* image for up to a week even after the file changes on GitHub.
4. Redeploy to Cloud Run (see below).
5. Optionally force jsDelivr's edge cache to drop the old file immediately:
   ```
   curl https://purge.jsdelivr.net/gh/mackiiops/pubsigzt@main/banner.png
   ```
6. To change where the banner links to, edit `BANNER_LINK_URL` in the same file.

## How to redeploy the app

The live Cloud Run service serves whatever is in `app/` at deploy time — it does not read live from GitHub. After editing `app/index.html` (e.g. for a `BANNER_VERSION` bump or any other change):

```bash
cd app
gcloud run deploy signature-generator \
  --source . \
  --project zeotap-internal-warehouse \
  --region europe-west1 \
  --platform managed \
  --allow-unauthenticated
```

This project uses `CLOUDSDK_PYTHON` pinned to Python 3.11 for `gcloud` — if you hit a Python-version error, run `export CLOUDSDK_PYTHON=$(which python3.11)` first.

## Access

- The Cloud Run service is public/unauthenticated by design — no data is stored or transmitted server-side, the whole tool runs client-side in the browser.
- The GitHub repo is public — required so `raw`/jsDelivr URLs can be fetched by email clients without authentication. Don't put anything sensitive in here.

## Known limitations

- **Gmail signature caching** — see above. There's no automatic fix for this without pushing signature updates directly via the Gmail API with Google Workspace domain-wide delegation (a bigger, separate project requiring Workspace admin sign-off). For now this is accepted as manual: Gmail users occasionally need to re-save their signature to pick up a new banner.
