# 📣 Megaphone — get self-hosted Postiz publishing to Meta (Facebook + Instagram) from localhost

**The survival kit for connecting a self-hosted [Postiz](https://github.com/gitroomhq/postiz-app) to
Facebook and Instagram — including the parts the docs don't warn you about.**

If you self-host Postiz on `http://localhost` and try to wire up Meta, you hit a wall of cryptic errors:
`Invalid Scopes: read_insights`, *"This content isn't available right now"*, *"couldn't find any business
connected to the selected pages"*, `Invalid redirect_uri`, *"Insufficient Developer Role"*. Each one cost
us hours. This repo is the map out — a step-by-step playbook plus the one **code patch** Postiz actually
needs, so you can skip the pain.

> Scope: Postiz **v2.21.x**, Meta Graph **v20.0**, mid-2026. Meta moves these goalposts often — the
> *reasoning* here outlasts the exact button names.

---

## TL;DR — the five things that actually block you

1. **Use "Facebook Login for Business," NOT classic Facebook Login.** Meta retired classic login for new
   apps; Postiz's docs require for-business. (Old guides telling you to "switch to classic" are wrong now.)
2. **Postiz hard-codes Meta-deprecated scopes** (`read_insights`, `instagram_basic`,
   `instagram_manage_insights`). Meta rejects them → OAuth dies with *Invalid Scopes*. **You must patch
   Postiz to stop requesting them.** → [`patch/Dockerfile`](patch/Dockerfile).
3. **Facebook-Business Instagram is broken** against current Meta (it needs the dead `instagram_basic` to
   enumerate the page's IG account). Use **Instagram *Standalone* (Instagram Login)** instead.
4. **Meta requires HTTPS redirect URIs.** `http://localhost` is rejected. Postiz tries to bridge this via
   `redirectmeto.com`, which is unreliable/often down. The clean fix is a **local HTTPS front (mkcert +
   Caddy)** → [`Caddyfile.example`](Caddyfile.example). No public tunnel needed.
5. **Instagram fetches images from a public URL.** With `local` media storage on localhost, image posts
   fail (*"inbound image isn't publicly accessible"*). Use **object storage (Cloudflare R2)** for media.

---

## Step 1 — Patch Postiz's dead Meta scopes (the core fix)

Postiz's provider files request scopes Meta deprecated. They're hardcoded, so the only fix is to strip them
from the image — persistently, so it survives `docker compose down/up`.

[`patch/Dockerfile`](patch/Dockerfile) builds a thin image on top of Postiz and `sed`s the dead scopes out:

```dockerfile
FROM ghcr.io/gitroomhq/postiz-app:latest
RUN for f in $(grep -rEl 'read_insights|instagram_basic|instagram_manage_insights' /app); do \
      sed -i -E "s/['\"]read_insights['\"],?//g; \
                 s/['\"]instagram_basic['\"],?//g; \
                 s/['\"]instagram_manage_insights['\"],?//g" "$f"; \
    done
```

Make it stick with a compose override ([`docker-compose.override.yml.example`](docker-compose.override.yml.example)):

```yaml
services:
  postiz:
    build:
      context: ./patch
    image: postiz-app:meta-patched
```

```bash
docker compose build postiz && docker compose up -d postiz
docker exec postiz pm2 restart backend
# verify the dead scopes are gone (should print nothing):
docker exec postiz sh -c "grep -rEn 'read_insights|instagram_basic|instagram_manage_insights' /app/apps"
```

> Maintain this until Postiz fixes it upstream (tracking: Postiz issues #1346 `read_insights`,
> #1578 `instagram_basic`).

## Step 2 — The Meta app (Facebook + Instagram)

- App type: **Facebook Login for Business**. Permissions now live inside **use cases**, not a global
  "App Review → Permissions" list — open **"Manage everything on your Page"** and add, at *Standard
  Access* (no review needed for your own Page): `pages_show_list`, `pages_read_engagement`,
  `pages_manage_posts`, `pages_manage_engagement`, `business_management`.
- Redirect URIs go under **Facebook Login for Business → Settings → Valid OAuth Redirect URIs**.
- Newly-added scopes take a few minutes to propagate before OAuth honours them — if you just added one and
  still see *Invalid Scopes* for it, wait and retry with a **fresh** auth attempt (don't reuse a stale
  `dialog/oauth?...` URL — that throws `Invalid state`).

**Facebook page connect:** with the scope patch applied and the page perms added, Add Channel → Facebook
just works. (Dev mode is fine to connect; flip the app **Dev → Live** — needs a Privacy Policy URL — for
posts to be publicly visible.)

## Step 3 — Instagram via **Standalone** (skip Facebook-Business)

The Facebook-Business IG path can't resolve your IG account on current Meta. Use the standalone
**Instagram API with Instagram Login**:

1. In the Meta app → **Instagram → API setup with Instagram login**: note the **Instagram App ID + Secret**
   (these are *different* from your Facebook App ID/Secret).
2. Set them in Postiz: `INSTAGRAM_APP_ID` / `INSTAGRAM_APP_SECRET`.
   ⚠️ In `docker-compose.yaml`, **mind the YAML colon-space**: `KEY: 'value'` (with a space) — `KEY:'value'`
   silently fails to set the variable. This bites everyone once.
3. The IG account must be **Business/Creator**, and (in Dev mode) added as an **Instagram Tester** in the
   app's Roles **and accepted** from the IG account (Settings → Apps and websites → Tester Invites) — else
   you get *Insufficient Developer Role*.
4. Register the redirect under **Instagram → Business login settings → OAuth redirect URIs** (a *different*
   field from Facebook Login), pointing at `…/integrations/social/instagram-standalone`.

## Step 4 — Solve "Meta requires HTTPS" locally (mkcert, no public tunnel)

Meta rejects `http://localhost` redirects. Postiz wraps the redirect through `redirectmeto.com` when
`FRONTEND_URL` isn't https — but that public bouncer is flaky and was fully down for us
(`SSL_ERROR_ZERO_RETURN`). A **public tunnel** works but logs Postiz out (domain change + Next.js bakes the
backend URL) and exposes your whole instance. The clean, **on-box** answer is a locally-trusted HTTPS front:

```bash
brew install mkcert caddy
mkcert -install
mkcert localhost 127.0.0.1 ::1          # trusted cert, no browser warnings
caddy run --config Caddyfile            # see Caddyfile.example
```

Then set Postiz's `MAIN_URL` / `FRONTEND_URL` / `NEXT_PUBLIC_BACKEND_URL` to `https://localhost:8443`,
register `https://localhost:8443/integrations/social/instagram-standalone` in Meta, and connect through the
HTTPS URL. Because `FRONTEND_URL` is now https, Postiz sends the redirect **directly** (no redirectmeto).
See [`Caddyfile.example`](Caddyfile.example).

> Caddy tries to grab `:80` for auto HTTP→HTTPS redirects; if `:80` is taken, add
> `{ auto_https disable_redirects }` (it's in the example).

## Step 5 — Public media so Instagram can fetch images

Instagram's Content Publishing API **downloads the image from a public URL server-side**. With `local`
storage on localhost, IG can't reach it → image posts fail (text posts don't). Switch Postiz to object
storage:

```yaml
STORAGE_PROVIDER: 'cloudflare'      # Cloudflare R2 (S3-compatible, free tier)
CLOUDFLARE_ACCOUNT_ID: '...'
CLOUDFLARE_ACCESS_KEY: '...'
CLOUDFLARE_SECRET_ACCESS_KEY: '...'
CLOUDFLARE_BUCKETNAME: 'postiz-media'
CLOUDFLARE_BUCKET_URL: 'https://pub-xxxx.r2.dev'   # the bucket's public URL
CLOUDFLARE_REGION: 'auto'
```

Only your **outbound media** (which is about to be public on Instagram anyway) lives in the bucket — nothing
else. This is also the standard Postiz production setup; `local` is essentially dev-only.

---

## Error → cause cheat-sheet

| Error you see | Real cause | Fix |
|---|---|---|
| `Invalid Scopes: read_insights` (or `instagram_basic`, `instagram_manage_insights`) | Postiz requests Meta-deprecated scopes | **Step 1** patch |
| `Invalid Scopes: pages_manage_posts` (a *valid* scope) | Not added to the app's use case yet | Add it (Standard Access) + wait to propagate |
| `Invalid state` on connect | Reused a stale `dialog/oauth?...` URL | Start a fresh Add Channel |
| *"couldn't find any business connected to the selected pages"* | Facebook-Business IG path (needs dead `instagram_basic`) | Use **Standalone** (Step 3) |
| `Invalid redirect_uri` | Redirect not registered in the right place / not HTTPS | Register under the correct settings; **Step 4** |
| *Insufficient Developer Role* | IG account isn't an accepted Tester (Dev mode) | Add + **accept** the Instagram Tester invite |
| `ERR_SSL_PROTOCOL_ERROR` via redirectmeto.com | The public bouncer is down | **Step 4** mkcert front |
| *"inbound image isn't publicly accessible"* | `local` media on localhost; IG can't fetch | **Step 5** object storage |
| `TypeError: tweets.data.data is not iterable` (logs) | X *analytics* poll, not posting | Harmless — ignore |

## Built by

Built by **[KodeKing](https://www.kodeking.net)** · author **[Fazal Shah](https://www.fazalshah.com)**.
We build local, private, multi-agent AI systems for teams who can't send their data to the cloud.
Not affiliated with Postiz or Meta. Issues and PRs welcome — especially as Meta shifts the goalposts.

## License

MIT — see [LICENSE](LICENSE).

<!-- fazalshah-personal-site:start -->
## Author and engineering notes

Published by [Fazal Shah](https://fazalshah.com/). Explore my local AI experiments, Mac hardware test notes, agent engineering articles, and public tools on my [personal website](https://fazalshah.com/).

[All public repositories](https://github.com/fazalrshah?tab=repositories) · [YouTube builds and walkthroughs](https://www.youtube.com/@fazalrshah)
<!-- fazalshah-personal-site:end -->
