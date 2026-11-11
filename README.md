# mahinraza.com
## 1. Prereqs

- Node 20+, git, GitHub account (`gh` CLI optional)
- A domain or subdomain for the blog (e.g. `blog.yourdomain.com`), optional

## 2. Create the project (Astro blog template)

```bash
npm create astro@latest /tmp/blog -- --template blog --no-install --no-git --yes
# project name: blog, install deps: yes, git: yes
cd blog
npm run dev   # http://localhost:4321
```

Structure (leave it flat, don't restructure):

```
src/content/blog/*.md     # posts
src/content.config.ts     # front matter schema
src/pages/                # routes (template provides index, blog, about, rss)
public/                   # static assets, CNAME
astro.config.mjs
scripts/devto.mjs         # cross-post script (step 6)
.github/workflows/
```

## 3. Config

`astro.config.mjs`: set your real URL. Needed for sitemap, RSS, and canonical URLs.

```js
export default defineConfig({
  site: 'https://blog.yourdomain.com',
  // integrations stay as the template has them
});
```

If you deploy to `username.github.io/blog` without a custom domain, also add `base: '/blog'`.

Extend the schema in `src/content.config.ts` (add three fields inside the blog `z.object`):

```ts
tags: z.array(z.string()).optional(),
devto: z.boolean().optional(),
devtoId: z.number().optional(),
```

## 4. Post format

`src/content/blog/my-first-post.md`:

```md
---
title: 'My first post'
description: 'One-line summary for SEO and cards'
pubDate: 'Nov 11 2026'
tags: ['llm', 'devops']
devto: true
---

Content here. Use absolute image URLs if cross-posting.
```

Draft workflow: write on a branch, merge to `main` to publish.

## 5. GitHub repo + deploy (GitHub Pages)

```bash
git add -A && git commit -m "init blog"
gh repo create blog --public --source=. --push
# or: create the repo on github.com, then git remote add origin ... && git push -u origin main
```

In the repo: **Settings → Pages → Source: GitHub Actions**.

`.github/workflows/deploy.yml`:

```yaml
name: Deploy
on:
  push:
    branches: [main]
  workflow_dispatch:
permissions:
  contents: read
  pages: write
  id-token: write
concurrency:
  group: pages
  cancel-in-progress: true
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: withastro/action@v3
  deploy:
    needs: build
    runs-on: ubuntu-latest
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    steps:
      - id: deployment
        uses: actions/deploy-pages@v4
```

Push to `main`, and the site goes live in about 1 min.

**Custom domain:**
1. `public/CNAME` containing `blog.yourdomain.com`
2. DNS: `CNAME blog → <username>.github.io`
3. Settings → Pages → enter the domain, tick **Enforce HTTPS**

(Alternative: Cloudflare Pages or Vercel, connect the repo, build `npm run build`, output `dist`. Skip the workflow above.)

## 6. Auto cross-post to Dev.to

Get an API key: dev.to → Settings → Extensions → Generate API key. Add it to the repo as the secret `DEVTO_API_KEY`.

```bash
npm i -D gray-matter
```

`scripts/devto.mjs`:

```js
import fs from 'node:fs';
import path from 'node:path';
import matter from 'gray-matter';

const dir = 'src/content/blog';
const { SITE_URL, DEVTO_API_KEY } = process.env;

for (const f of fs.readdirSync(dir).filter((f) => /\.mdx?$/.test(f))) {
  const p = path.join(dir, f);
  const raw = fs.readFileSync(p, 'utf8');
  const { data, content } = matter(raw);
  if (!data.devto || data.devtoId) continue;

  const slug = f.replace(/\.mdx?$/, '');
  const r = await fetch('https://dev.to/api/articles', {
    method: 'POST',
    headers: { 'api-key': DEVTO_API_KEY, 'Content-Type': 'application/json' },
    body: JSON.stringify({
      article: {
        title: data.title,
        description: data.description,
        body_markdown: content,
        published: true,
        tags: (data.tags || []).slice(0, 4),
        canonical_url: `${SITE_URL}/blog/${slug}/`,
      },
    }),
  });
  if (!r.ok) { console.error(f, r.status, await r.text()); process.exit(1); }
  const j = await r.json();
  fs.writeFileSync(p, raw.replace(/^---\r?\n/, `---\ndevtoId: ${j.id}\n`));
  console.log('posted', f, j.url);
}
```

`.github/workflows/crosspost.yml` (runs only after a successful deploy, so the canonical URL exists):

```yaml
name: Cross-post
on:
  workflow_run:
    workflows: [Deploy]
    types: [completed]
    branches: [main]
permissions:
  contents: write
jobs:
  devto:
    if: github.event.workflow_run.conclusion == 'success'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 20 }
      - run: npm ci
      - run: node scripts/devto.mjs
        env:
          DEVTO_API_KEY: ${{ secrets.DEVTO_API_KEY }}
          SITE_URL: https://blog.yourdomain.com
      - run: |
          git config user.name github-actions
          git config user.email actions@users.noreply.github.com
          git add -A src/content
          git diff --cached --quiet || (git commit -m "chore: devto ids [skip ci]" && git push)
```

Behavior: any post with `devto: true` and no `devtoId` gets posted once; the returned ID is written back so it never double-posts. It only creates posts and does not sync later edits.

## 7. Medium and LinkedIn (manual, 1 min each)

- **Medium:** Profile → Stories → **Import a story** → paste your post URL. It sets the canonical link to your site automatically.
- **LinkedIn:** short post with a link, or paste as an article with the link to the original at the bottom.

## 8. Daily workflow

```bash
git checkout -b post/slug
# write src/content/blog/slug.md
npm run dev            # preview
git push -u origin post/slug   # open PR, check build
# merge to main → deploy → Dev.to cross-post
# then Medium import + LinkedIn post
```

## 9. Optional upgrades

- **Analytics:** Plausible or Umami script in the layout head
- **Search Console:** add the domain, submit `/sitemap-index.xml`
- **Spellcheck in CI:** `cspell` or `markdownlint` step before the build
- **OG images:** per-post cards via Satori
- **Scheduling:** `pubDate` in the future plus a daily cron on Deploy (`schedule: cron`) and filter future dates in `getCollection`

Ask if you want the Hugo or Next.js variant, or the Hashnode sync added.