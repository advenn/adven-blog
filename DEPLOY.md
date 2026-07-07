# Deploying to Cloudflare Pages

Plain Hugo site at the repo root. PaperMod is **vendored** under `themes/` as
normal files (no git submodule). Hosted on **Cloudflare Pages** via Git.

## Build configuration (Cloudflare dashboard)

Project → **Settings** → **Build configuration**:

| Setting                | Value                |
| ---------------------- | -------------------- |
| Build command          | `hugo --gc --minify` |
| Build output directory | `public`             |
| **Deploy command**     | **(leave EMPTY)**    |

> ⚠️ The deploy command **must be empty**. Pages auto-uploads the output
> directory after the build. If it's set to `npx wrangler deploy`, the deploy
> fails with *"Missing entry-point to Worker script or to assets directory"* —
> that's a Workers command, not a Pages one. Do **not** commit a `wrangler.toml`
> either; Cloudflare will re-suggest that broken deploy command if it sees one.

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
`baseURL` in `hugo.yaml` is already `https://bek-kurbonov.com/`.

## Manual / local deploy (fallback)

```bash
hugo --gc --minify
npx wrangler pages deploy public --project-name <your-pages-project-name>
```

## Local preview

```bash
hugo server
```
