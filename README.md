# Game Site Development & Deployment

A single-file methodology for building and deploying SEO-friendly game / content sites, packaged as a WorkBuddy Skill. It covers the build-and-deploy side of the job: choosing a stack, keeping facts traceable to their sources, generating pages, deploying to Cloudflare Pages, binding a custom domain, wiring analytics, and avoiding the pitfalls found in real projects.

This repository contains one methodology document and this English guide. `SKILL.md` is kept in its original Chinese.

## Background

This Skill was distilled from a complete game-keyword site project, in which the work splits into three stages. An upstream material factory (a cloud agent) selects keywords, researches them and produces one source document per page — Google guides, YouTube transcripts and official pages — without building anything. The middle stage turns those materials into pages and an engineering setup. The downstream stage deploys to Cloudflare Pages, binds a domain, connects GSC / GA4, adds pages and handles on-page SEO. This Skill owns the middle and downstream stages only; topic selection, keyword strategy and monetisation decisions are explicitly out of scope.

The most useful part of the document is a mistake, recorded so it is not repeated. The source-material red lines were originally written to constrain the upstream collection step: keep the original text, label the source, do not invent facts, cross-check at least two sources. They were then over-applied to page writing as "if the material library has no hit, do not write the fact". That overreach left demonstrably wrong answers live on a site — a demo that had been removed, a platform that was PC-only — while waiting for a source that was never going to arrive. The correction is the A/B/C fact classification now in the document: hard first-party facts (price, release date, platform, system requirements, demo availability) get verified directly and written immediately with a source and a check date; operational and community content keeps the two-source rule; unannounced speculation is not written at all. Verification means actively going to find a source, not waiting for one to appear in a folder.

Conflicts are also pre-resolved: launch ordering belongs to a separate Skill, and any SEO judgement is ultimately governed by the numbered rules in `seo-dev-spec`. This document is the "how" and the field notes, not the authority.

## What it does

1. Makes the stack decision explicit: Astro static output by default (zero runtime JavaScript, plain HTML, `dist/` directly hostable), Cloudflare Pages via direct upload, and a static API-token file instead of an OAuth MCP flow. It also states where this choice does not fit — heavily interactive SPAs.
2. Defines source-material red lines and the A/B/C fact classification, requires a Source note on each page distinguishing official from community-sourced facts, and treats external SEO reports as claims to verify rather than instructions to obey.
3. Names the public Steam `appdetails` API as the preferred live check for time-sensitive A-class facts such as price, platform and demo availability.
4. Covers page-building standards, on-page SEO, GSC / GA4 wiring, AdSense notes and custom-domain binding.
5. Records a pitfall checklist accumulated from real builds, including eight additions from 2026-10 covering images, external links, h1 hierarchy and preview environments.

## When to use it

Read it before and during a build that turns a per-page source library into a static content site, and again before deployment. Use it as a checklist for the build-and-deploy stage, and as the reference for how to handle a fact that conflicts with an external report.

Do not use it to plan the order of a launch or to make SEO acceptance decisions; those belong to `站点上线总纲` and `seo-dev-spec` respectively.

## Repository contents

- `SKILL.md` — the full methodology (Chinese): scope boundaries, stack decision, source red lines and fact classification, live verification, build standards, deployment, analytics, and the pitfall checklist.
- `README.md` — this English guide.

## How to use

1. Read `SKILL.md` before starting the build to fix the division of labour: upstream materials are the fact source, the middle stage builds pages, the downstream stage deploys and verifies.
2. While writing pages, classify each fact as A, B or C. Verify A-class facts directly (for example the Steam `appdetails` endpoint), require two sources for B-class content, and refuse C-class speculation. Add a Source note to every page.
3. Generate pages with the main copy, h1, breadcrumbs and internal links present in the static HTML output, then deploy the `dist/` directory to Cloudflare Pages and bind the custom domain.
4. Work through the pitfall checklist before release and before adding pages after release.

There are no runnable scripts in this package. It is prose guidance; the commands it quotes are examples from the described workflow, not bundled tooling.

## Requirements

- No build step, no scripts, no credentials, no private buckets and no build artifacts are included.
- External services appear only as part of the described workflow: Cloudflare Pages (API / wrangler), the public Steam `appdetails` API, a page-extraction API, GA4 / GSC, AdSense and a GitHub MCP connector. Nothing here bundles an account, token or key for any of them.

## Scope and limitations

- `SKILL.md` is in Chinese. This README is the English entry point, not a translation.
- Topic selection, keyword strategy and monetisation decisions are deliberately excluded.
- Where this document and `seo-dev-spec` disagree on an SEO rule, the numbered rules win. The two are companions, not alternatives.
- Availability of third-party endpoints and platform behaviour changes over time; the dates recorded in the document are the dates those observations were made.

## Origin

Extracted from a personal WorkBuddy skill library and cleaned for public release: machine-specific absolute paths were replaced with portable ones, backup and build artifacts were excluded, and no credentials are included.

## Companion site

A short account of how this deployment method was assembled is at [WorkBuddy Central: Game Site Development](https://workbuddycentral.com/game-site-development/). The methodology itself lives in `SKILL.md`; the site is a plain-language profile for readers deciding whether the method fits their project.

WorkBuddy Central is an independent third-party documentation site; it is not affiliated with WorkBuddy or Tencent.
