# Deploying to Cloudflare (Workers Builds)

Plain Hugo site at the repo root. PaperMod is **vendored** under `themes/` as
normal files (no git submodule). Hosted on **Cloudflare Workers** (static
assets) via Git — the project deploys as a Worker named `bek-kurbonov`.

## Build configuration (Cloudflare dashboard)

Project → **Settings** → **Build configuration**:

| Setting                | Value                 |
| ---------------------- | --------------------- |
| Build command          | `hugo --gc --minify`  |
| Build output directory | `public`              |
| **Deploy command**     | `npx wrangler deploy` |

> ⚠️ This deploy uses Cloudflare **Workers static assets**, which requires a
> committed [`wrangler.jsonc`](wrangler.jsonc) at the repo root naming the
> Worker (`bek-kurbonov`) and its assets directory (`public`). Without it,
> `npx wrangler deploy` refuses to overwrite the existing Worker
> (*"A Worker named 'bek-kurbonov' already exists … could not confirm that it
> should update that Worker"*) and the deploy fails. Do **not** delete
> `wrangler.jsonc`.

## Environment variables (optional but recommended)

Settings → **Variables and Secrets** (Build):

| Name           | Value           | Why                                        |
| -------------- | --------------- | ------------------------------------------ |
| `HUGO_VERSION` | `0.148.0`       | Pin the build; don't drift with CF default |
| `TZ`           | `Asia/Tashkent` | Consistent post-date rendering             |

The theme needs Hugo ≥ 0.146 extended. Cloudflare's current default (0.147.x)
works, but pinning keeps future builds reproducible.

## Custom domain

Project → **Custom domains** → add `bek-kurbonov.com` (and `www.` if wanted).

Until the custom domain is active, `hugo.yaml` uses `baseURL: /` with
`relativeURLs: true` so links follow whatever domain serves the site
(`*.pages.dev`, preview, or the custom domain). **Once `bek-kurbonov.com` is
live**, switch back to absolute URLs for correct canonical/sitemap/RSS:

```yaml
baseURL: https://bek-kurbonov.com/
# remove relativeURLs and canonifyURLs lines
```

Keep the build command as plain `hugo --gc --minify` — do **not** add
`-b $CF_PAGES_URL`, which would re-inject an absolute domain and break the
relative links.

## Manual / local deploy (fallback)

```bash
hugo --gc --minify
npx wrangler deploy      # reads wrangler.jsonc; updates the bek-kurbonov Worker
```

## Local preview

```bash
hugo server
```
