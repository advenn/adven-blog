# Deploying to Cloudflare Pages

Plain Hugo site at the repo root. PaperMod is **vendored** under `themes/` as
normal files (no git submodule, no upstream pointer) — edit it freely.

## One-time setup (Git integration — recommended)

1. Create a new GitHub repo and push this project to it.
2. Cloudflare dashboard → **Workers & Pages** → **Create** → **Pages** →
   **Connect to Git** → pick the repo.
3. Build settings:

   | Setting                | Value                |
   | ---------------------- | -------------------- |
   | Production branch      | `master` (or `main`) |
   | Framework preset       | Hugo                 |
   | Build command          | `hugo --gc --minify` |
   | Build output directory | `public`             |

4. **Environment variables** (Settings → Variables and Secrets → Build):

   | Name           | Value           |
   | -------------- | --------------- |
   | `HUGO_VERSION` | `0.148.0`       |
   | `TZ`           | `Asia/Tashkent` |

   `HUGO_VERSION` makes Cloudflare install the correct **extended** Hugo.

5. Deploy. No submodules, no Dart Sass — the build is just `hugo`.

6. **Custom domain**: project → **Custom domains** → add `bek-kurbonov.com`
   (and `www.bek-kurbonov.com` if you want it). `baseURL` in `hugo.yaml` is
   already `https://bek-kurbonov.com/`. If the domain's nameservers are on
   Cloudflare, the DNS record is added for you automatically.

## Manual / local deploy (fallback)

```bash
hugo --gc --minify
npx wrangler pages deploy public --project-name bek-kurbonov
```

## Local preview

```bash
hugo server
```

