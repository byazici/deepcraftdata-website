# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm run dev       # Start local development server
npm run build     # Production build (outputs to dist/)
npm run preview   # Preview the production build
```

No lint or test commands are configured.

## Architecture

**Astro 4** static site deployed to **Cloudflare Pages**. Uses Tailwind CSS and MDX for content.

### Dual-Entry Pattern

The site has two distinct parts that work independently:

- **`/public/index.html`** — The landing/home page. This is plain HTML, not part of Astro's build pipeline. Blog post previews in it are **hardcoded** and must be updated manually when new posts are published.
- **`/src/pages/blog/`** — The blog section, fully Astro-generated with dynamic routes.

### Staging-Aware Links

`Base.astro` contains inline JavaScript that rewrites specific link `href`s at runtime when `hostname` includes `staging` or `.pages.dev`. The affected elements are identified by ID:

- `nav-logo-link` / `footer-site-link` → `https://staging.deepcraftdata.com`
- `nav-audit-link` / `post-cta-audit-link` → `https://app-staging.deepcraftdata.com`

When adding new links that should switch between prod/staging, assign one of these IDs or follow the same inline script pattern.

### Content Collections

Blog posts live in `src/content/blog/` as Markdown or MDX files. The schema (defined in `src/content/config.ts`) enforces:

```ts
title: z.string()
date: z.date()
summary: z.string()
tags: z.array(z.string())
draft: z.boolean().default(false)
```

Draft posts (`draft: true`) are excluded from all output. Tags power both the `/blog/tags/[tag]` filtered views and the RSS feed.

Prev/next navigation in `[...slug].astro` sorts posts **ascending** (oldest→newest) to determine neighbors, while the blog index sorts **descending** (newest first).

### Layout Hierarchy

```
Base.astro      ← fixed nav, sticky footer, dark theme CSS vars, scroll shadow script
  └─ Post.astro ← article header (date, reading time, tags), prev/next nav, CTA box
```

Reading time is calculated as `Math.ceil(wordCount / 200)` inline in `Post.astro`.

### Styling

- **Tailwind utilities** for layout/spacing
- **CSS custom properties** for the dark color scheme (defined in `src/styles/global.css`): `--bg`, `--bg2`, `--blue`, `--purple`, `--green`, `--text`, `--muted`, `--dim`, etc.
- **`@tailwindcss/typography`** (`.prose`) for rendered Markdown in blog posts — the dark-mode overrides are in `tailwind.config.mjs`
- Breakpoints: 900px, 640px, 400px (media queries in Astro template `<style>` blocks)

### No Components Directory

There is no `src/components/` directory. All UI is rendered directly in layouts and page templates. Navigation and footer markup exist only in `Base.astro`.

### Site URL

Canonical site URL is `https://deepcraftdata.com`, set in `astro.config.mjs`. Code blocks use `github-dark` syntax highlighting via Shiki.

## Local Planning Files

The `local/` directory is gitignored and contains private planning docs. Maintain these files as the project evolves:

- **`project-calendar.txt`** — Phase-level timeline (milestones, launch dates, social media schedule).
- **`project-plan.txt`** — Full task list. Mark completed items with `[x]` and append `— REPO commit:hash (date)`. Pending items use `[ ]`. REPO prefix: `site` = deepcraft-site, `app` = deepcraft-audit.
- **`release-notes.txt`** — Changelog of every meaningful release. Add a new entry at the top when shipping a significant change. Format: `[LABEL — YYYY-MM-DD] Title` followed by bullet points and commit ref.
- **`sharing-plan.txt`** — Social media post copy and calendar. Update when posts are published.

When a task is completed, always update `project-plan.txt` and `release-notes.txt` in the same session.
