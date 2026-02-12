# react-prerender-worker

Pre-rendering infrastructure for React SPAs to enable search engine and AI crawler indexing without framework migration.

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Cloudflare Workers](https://img.shields.io/badge/Cloudflare-Workers-F38020?logo=cloudflare)](https://workers.cloudflare.com/)
[![Supabase](https://img.shields.io/badge/Supabase-Edge_Functions-3ECF8E?logo=supabase)](https://supabase.com/)

## Overview

Client-side rendered applications (React, Vue, Svelte, etc.) serve minimal HTML to web crawlers, resulting in poor search engine indexing and broken social media previews. This project provides a lightweight edge-computing solution that serves pre-rendered HTML to bots while maintaining the SPA experience for users.

**Designed for:** Applications built with AI development platforms (Lovable, Bolt.new, v0, Cursor) and traditional SPAs requiring SEO without SSR migration.

## Architecture

```
                    yoursite.com
                        │
                        ▼
              ┌─────────────────────┐
              │  Cloudflare Worker  │
              │   (User-Agent       │
              │    Detection)       │
              └─────────────────────┘
                   │           │
            Bot? ──┘           └── User?
              │                     │
              ▼                     ▼
     ┌────────────────┐    ┌────────────────┐
     │   Supabase     │    │   Origin CDN   │
     │   (Cached      │    │   (React SPA)  │
     │    HTML)       │    │                │
     └────────────────┘    └────────────────┘
```

- **Bots**: Receive pre-rendered HTML with complete metadata from edge cache (~50ms response)
- **Users**: Access the standard SPA from origin CDN
- **Cache**: Automated refresh via PostgreSQL cron jobs

## Problem Statement

Single-page applications deliver empty HTML shells to crawlers:

```html
<html>
  <body>
    <div id="root"></div>
    <script src="/assets/index.js"></script>
  </body>
</html>
```

**Impact:**
- Search engines cannot index content
- Social media platforms display broken previews
- AI language models (ChatGPT, Claude, Perplexity) cannot access site content
- SEO performance requires complete framework migration

## Solution Comparison

| Feature | This Solution | Next.js SSR | Static SSG |
|---------|--------------|-------------|------------|
| Migration Required | None | Full rewrite | Partial rewrite |
| Monthly Cost | $0 (free tiers) | $0-20+ | $0+ |
| Bot Response Time | ~50ms (edge) | ~200-500ms | ~50ms |
| Framework Lock-in | None | High | Medium |
| Hosting Flexibility | Any provider | Limited | Any provider |
| Cache Management | Automated (pg_cron) | Manual ISR config | Build-time only |
| Setup Time | ~30 minutes | Days-weeks | Days |

## Supported Platforms

### Frameworks
- React (Vite, Create React App, Remix SPA mode)
- Vue.js
- Svelte / SvelteKit (static adapter)
- Angular
- Astro (client-rendered pages)
- Any framework outputting static HTML/JS/CSS

### Hosting Providers
- Cloudflare Pages
- Vercel
- Netlify
- GitHub Pages
- Firebase Hosting
- Custom origins (any URL)

Set `PAGES_ORIGIN` environment variable to your app's URL.

## Detected Crawlers

The Worker identifies 100+ bot user-agents across categories:

| Category | Examples |
|----------|----------|
| **Search Engines** | Googlebot, Bingbot, Yandex, Baidu, DuckDuckGo |
| **AI Crawlers** | GPTBot, ClaudeBot, PerplexityBot, Google-Extended, Gemini |
| **Social Media** | Facebook, Twitter, LinkedIn, WhatsApp, Discord, Reddit |
| **SEO Tools** | Ahrefs, SEMrush, Screaming Frog, Moz |
| **Platform Bots** | Applebot, AmazonBot, Slackbot |
| **Archives** | Internet Archive, Archive.org |

Full list: [worker.js](cloudflare-worker/worker.js#L45-L150)

## Installation

### Prerequisites
- Supabase account (free tier sufficient)
- Cloudflare account (free tier sufficient)
- Domain with DNS managed by Cloudflare
- Application hosted on any platform

### 1. Database Setup

Execute in Supabase SQL Editor:

```sql
CREATE TABLE IF NOT EXISTS prerendered_pages (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  path TEXT UNIQUE NOT NULL,
  html TEXT NOT NULL,
  title TEXT,
  hit_count INTEGER DEFAULT 0,
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now(),
  expires_at TIMESTAMPTZ
);

CREATE INDEX idx_prerendered_pages_path ON prerendered_pages(path);

ALTER TABLE prerendered_pages ENABLE ROW LEVEL SECURITY;
CREATE POLICY "Public read access" ON prerendered_pages FOR SELECT USING (true);
```

### 2. Deploy Supabase Edge Function

Create `supabase/functions/prerender/index.ts`:

```typescript
import { createClient } from "https://esm.sh/@supabase/supabase-js@2";

const supabase = createClient(
  Deno.env.get("SUPABASE_URL")!,
  Deno.env.get("SUPABASE_SERVICE_ROLE_KEY")!
);

Deno.serve(async (req) => {
  if (req.method === "OPTIONS") {
    return new Response(null, { 
      headers: { "Access-Control-Allow-Origin": "*" } 
    });
  }

  const url = new URL(req.url);
  let path = url.searchParams.get("path") || "/";
  if (path !== "/" && path.endsWith("/")) {
    path = path.slice(0, -1);
  }

  const { data } = await supabase
    .from("prerendered_pages")
    .select("html")
    .eq("path", path)
    .maybeSingle();

  if (data?.html) {
    return new Response(data.html, {
      headers: { 
        "Content-Type": "text/html; charset=utf-8",
        "X-Cache": "hit" 
      },
    });
  }

  return new Response("Not found", { 
    status: 404,
    headers: { "X-Cache": "miss" } 
  });
});
```

Deploy:
```bash
supabase functions deploy prerender
```

### 3. Deploy Cloudflare Worker

```bash
npm install -g wrangler
wrangler login
wrangler deploy
```

### 4. Configure Environment Variables

In Cloudflare Dashboard → Workers → Settings → Variables:

| Variable | Value | Example |
|----------|-------|---------|
| `SUPABASE_URL` | Your Supabase project URL | `https://xxx.supabase.co` |
| `SUPABASE_ANON_KEY` | Your Supabase anon key | `eyJhbGc...` |
| `PAGES_ORIGIN` | Your SPA origin URL | `https://app.pages.dev` |

**Important:** `PAGES_ORIGIN` must include protocol (`https://`)

### 5. Configure Worker Routes

Cloudflare Dashboard → Websites → [Your Domain] → Workers Routes:

| Route | Worker |
|-------|--------|
| `yourdomain.com/*` | `prerender-worker` |
| `www.yourdomain.com/*` | `prerender-worker` |

**Note:** Use Worker Routes, not Pages Custom Domains.

### 6. Automated Cache Refresh (Optional)

Configure PostgreSQL cron job for automatic cache updates:

```sql
SELECT cron.schedule(
  'refresh-prerender-cache',
  '0 */6 * * *',  -- Every 6 hours
  $$
  SELECT net.http_post(
    url := 'https://[PROJECT].supabase.co/functions/v1/generate-cache',
    headers := '{"Authorization": "Bearer [ANON_KEY]"}'::jsonb
  );
  $$
);
```

## Populating the Cache

Insert pre-rendered HTML for each route:

```typescript
await supabase.from("prerendered_pages").upsert({
  path: "/about",
  title: "About Us",
  html: `<!DOCTYPE html>
<html lang="en">
<head>
  <title>About Us | YourApp</title>
  <meta name="description" content="Learn about our mission">
  <meta property="og:title" content="About Us">
  <meta property="og:description" content="Learn about our mission">
  <link rel="canonical" href="https://yourdomain.com/about">
</head>
<body>
  <h1>About Us</h1>
  <p>Your content here</p>
</body>
</html>`
}, { onConflict: "path" });
```

Example generator script: [examples/generate-cache.ts](examples/generate-cache.ts)

## Verification

Test bot detection:

```bash
# Standard request (returns SPA)
curl -I https://yourdomain.com/

# Bot request (returns pre-rendered HTML)
curl -I -H "User-Agent: Googlebot/2.1" https://yourdomain.com/
# Expected: X-Prerendered: true, X-Cache: hit
```

## Common Issues

| Issue | Solution |
|-------|----------|
| 403 responses | Disable Bot Fight Mode in Cloudflare Security settings |
| PAGES_ORIGIN errors | Verify protocol (`https://`) is included |
| Empty responses to bots | Check cache table is populated |
| Worker not triggering | Verify Worker Routes configuration (not Custom Domain) |

## SEO Compliance

This implementation follows Google's [dynamic rendering guidelines](https://developers.google.com/search/docs/crawling-indexing/javascript/dynamic-rendering). The pre-rendered HTML contains identical content to what users receive after JavaScript execution, satisfying search engine requirements for equivalent content delivery.

## Performance

- **Edge response time:** ~50ms (Cloudflare global network)
- **Cache hit rate:** 99%+ after warm-up
- **Tested scale:** 9,000+ pages in production
- **Infrastructure cost:** $0 (within free tier limits)

## Production Deployments

This infrastructure serves pre-rendered content for applications with:
- 9,000+ indexed pages
- Mixed static and dynamic content
- Global user bases
- Sub-second bot response requirements

## Related Projects

For sites requiring server-side rendering for user traffic:
- [Next.js](https://nextjs.org) - Full-stack React framework
- [Remix](https://remix.run) - Full-stack React framework
- [Astro](https://astro.build) - Multi-framework static site generator

For AI search optimization (requires crawlable HTML as prerequisite):
- Schema.org markup implementation
- Open Graph protocol implementation
- Structured data generators

## Contributing

Contributions welcome. Please submit issues for bugs and pull requests for improvements.

## License

MIT License - see [LICENSE](LICENSE) for details.

---

**Note:** This is infrastructure, not a framework replacement. Evaluate your requirements:
- For new projects with server-side rendering needs → Consider Next.js/Remix
- For existing SPAs requiring crawler compatibility → Use this solution
- For static content → Consider static site generators
