# ⚡ react-prerender-worker

**Zero-cost SEO for React SPAs — serve pre-rendered HTML to bots, pure SPA to humans.**

> Make any React app fully crawlable by Google, ChatGPT, Claude, and 100+ bots for **$0/month**.

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Cloudflare Workers](https://img.shields.io/badge/Cloudflare-Workers-F38020?logo=cloudflare)](https://workers.cloudflare.com/)
[![Supabase](https://img.shields.io/badge/Supabase-Edge_Functions-3ECF8E?logo=supabase)](https://supabase.com/)

---

## The Problem

React SPAs ship an empty `<div id="root"></div>`. Search engines and AI crawlers see **nothing**. Next.js solves this but locks you into a framework, a Node.js server, and hosting costs.

## The Solution

A **Cloudflare Worker** sits in front of your app and makes a split-second decision:

```
                    yoursite.com
                        │
                        ▼
              ┌─────────────────────┐
              │  Cloudflare Worker  │
              │   (Bot Detector)    │
              └─────────────────────┘
                   │           │
            Bot? ──┘           └── Human?
              │                     │
              ▼                     ▼
     ┌────────────────┐    ┌────────────────┐
     │   Supabase DB  │    │ Cloudflare Pages│
     │ (Cached HTML)  │    │   (React SPA)   │
     └────────────────┘    └────────────────┘
```

- **Bots** get content-rich, SEO-optimized HTML from a database cache (~50ms TTFB)
- **Humans** get your blazing-fast React SPA from the CDN edge
- **Cost**: $0/month on free tiers

---

## How It Compares to Next.js

| Metric | This Solution | Next.js (Vercel) |
|--------|:------------:|:----------------:|
| **Monthly cost at scale** | $0 | $20–$100+ |
| **Bot TTFB** | ~50ms (DB lookup) | ~200–500ms (SSR) |
| **Human experience** | Pure SPA (instant nav) | SSR + hydration |
| **Framework lock-in** | None (React, Vue, anything) | Next.js only |
| **Pages supported** | 10,000+ tested | Depends on plan |
| **Infrastructure** | Edge (global) | Regional servers |
| **Setup complexity** | Medium (one-time) | Low (built-in) |

**Trade-off**: Next.js gives humans SSR on first load. This solution gives humans a loading spinner until the SPA boots — but after that, navigation is instant.

---

## Quick Start

### Prerequisites

- A React app (CRA, Vite, anything)
- A [Supabase](https://supabase.com) project (free tier)
- A [Cloudflare](https://cloudflare.com) account (free tier)
- Your domain's DNS on Cloudflare

### 1. Create the Cache Table

Run this SQL in your Supabase SQL Editor:

```sql
-- Store pre-rendered HTML for bot consumption
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

-- Fast lookups by path
CREATE INDEX IF NOT EXISTS idx_prerendered_pages_path ON prerendered_pages(path);

-- Allow public read access (bots need this)
ALTER TABLE prerendered_pages ENABLE ROW LEVEL SECURITY;
CREATE POLICY "Public read access" ON prerendered_pages FOR SELECT USING (true);
```

### 2. Deploy the Prerender Edge Function

Create `supabase/functions/prerender/index.ts`:

```typescript
import { createClient } from "https://esm.sh/@supabase/supabase-js@2";

const supabaseUrl = Deno.env.get("SUPABASE_URL")!;
const supabaseKey = Deno.env.get("SUPABASE_SERVICE_ROLE_KEY")!;
const supabase = createClient(supabaseUrl, supabaseKey);

const corsHeaders = {
  "Access-Control-Allow-Origin": "*",
  "Access-Control-Allow-Methods": "GET, OPTIONS",
  "Access-Control-Allow-Headers": "*",
};

Deno.serve(async (req) => {
  if (req.method === "OPTIONS") {
    return new Response(null, { headers: corsHeaders });
  }

  const url = new URL(req.url);
  let path = url.searchParams.get("path") || "/";
  
  // Normalize: strip trailing slash (except root)
  if (path !== "/" && path.endsWith("/")) {
    path = path.slice(0, -1);
  }

  // Look up cached HTML
  const { data, error } = await supabase
    .from("prerendered_pages")
    .select("html, title")
    .eq("path", path)
    .maybeSingle();

  if (data?.html) {
    // Increment hit counter (fire-and-forget)
    supabase.rpc("increment_hit_count", { page_path: path }).then(() => {});

    return new Response(data.html, {
      headers: {
        ...corsHeaders,
        "Content-Type": "text/html; charset=utf-8",
        "X-Cache": "hit",
        "X-Prerendered": "true",
        "Cache-Control": "public, max-age=3600",
      },
    });
  }

  // Cache miss — return 404 so the Worker falls back to SPA
  return new Response("Not found in cache", {
    status: 404,
    headers: { ...corsHeaders, "X-Cache": "miss" },
  });
});
```

Add to `supabase/config.toml`:

```toml
[functions.prerender]
verify_jwt = false
```

### 3. Deploy the Cloudflare Worker

Copy `worker.js` from this package and deploy:

```bash
# Install Wrangler
npm install -g wrangler

# Login
wrangler login

# Deploy
wrangler deploy
```

### 4. Configure Environment Variables

In the Cloudflare Dashboard → Your Worker → Settings → Variables:

| Variable | Value |
|----------|-------|
| `SUPABASE_URL` | `https://YOUR-PROJECT.supabase.co` |
| `SUPABASE_ANON_KEY` | Your Supabase anon key |
| `PAGES_ORIGIN` | `https://your-project.pages.dev` (**must include `https://`!**) |

### 5. Set Up Worker Routes

In Cloudflare → Websites → Your Domain → Workers Routes:

| Route | Worker |
|-------|--------|
| `yourdomain.com/*` | `your-worker-name` |
| `www.yourdomain.com/*` | `your-worker-name` |

⚠️ **Do NOT** add your domain as a Custom Domain on the Pages project — use Worker Routes.

---

## Populating the Cache

You need to generate HTML for your pages. Here's the pattern:

```typescript
// supabase/functions/generate-cache/index.ts
import { createClient } from "https://esm.sh/@supabase/supabase-js@2";

const supabase = createClient(
  Deno.env.get("SUPABASE_URL")!,
  Deno.env.get("SUPABASE_SERVICE_ROLE_KEY")!
);

function generatePageHtml(page: { path: string; title: string; content: string; description: string }) {
  return `<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>${page.title}</title>
  <meta name="description" content="${page.description}">
  <meta property="og:title" content="${page.title}">
  <meta property="og:description" content="${page.description}">
  <meta property="og:url" content="https://yourdomain.com${page.path}">
  <link rel="canonical" href="https://yourdomain.com${page.path}">
</head>
<body>
  <main>
    <h1>${page.title}</h1>
    ${page.content}
  </main>
  <script type="application/ld+json">
  ${JSON.stringify({
    "@context": "https://schema.org",
    "@type": "WebPage",
    "name": page.title,
    "description": page.description,
    "url": `https://yourdomain.com${page.path}`
  })}
  </script>
</body>
</html>`;
}

Deno.serve(async () => {
  // Fetch your content from whatever tables you have
  const { data: pages } = await supabase.from("your_content_table").select("*");

  let count = 0;
  const batch = [];

  for (const page of pages || []) {
    const html = generatePageHtml({
      path: page.slug,
      title: page.title,
      content: page.body,
      description: page.meta_description || page.title,
    });

    batch.push({ path: page.slug, html, title: page.title, updated_at: new Date().toISOString() });

    // Upsert in batches of 50
    if (batch.length >= 50) {
      await supabase.from("prerendered_pages").upsert(batch, { onConflict: "path" });
      count += batch.length;
      batch.length = 0;
    }
  }

  // Final batch
  if (batch.length > 0) {
    await supabase.from("prerendered_pages").upsert(batch, { onConflict: "path" });
    count += batch.length;
  }

  return new Response(JSON.stringify({ success: true, pages_cached: count }), {
    headers: { "Content-Type": "application/json" },
  });
});
```

### Auto-Refresh with pg_cron

```sql
-- Refresh cache every 6 hours
SELECT cron.schedule(
  'refresh-prerender-cache',
  '0 */6 * * *',
  $$
  SELECT net.http_post(
    'https://YOUR-PROJECT.supabase.co/functions/v1/generate-cache',
    '{}',
    'application/json',
    ARRAY[
      net.http_header('Authorization', 'Bearer YOUR-ANON-KEY')
    ]
  );
  $$
);
```

---

## The Worker (Explained)

The `worker.js` file does three things:

### 1. Bot Detection
Checks the `User-Agent` against 100+ known patterns:
- Search engines: Google, Bing, Baidu, Yandex, DuckDuckGo
- AI crawlers: GPTBot, ClaudeBot, PerplexityBot, Google-Extended
- Social media: Facebook, Twitter, LinkedIn, WhatsApp, Discord
- SEO tools: Ahrefs, SEMrush, Screaming Frog

### 2. Smart Routing
```
Static asset (.js, .css, .png)?  → Cloudflare Pages (always)
Excluded path (/cms, /api)?      → Cloudflare Pages (always)
Bot detected?                    → Supabase prerender cache
Human visitor?                   → Cloudflare Pages (SPA)
```

### 3. Script Injection (Optional)
The Worker can inject analytics/CMP scripts into every HTML response at the edge — no build step needed. Remove the `INJECT_SCRIPT` constant and `injectScript()` function if you don't need this.

---

## Testing

```bash
# Test as a regular user (should get SPA)
curl -I https://yourdomain.com/
# → No X-Prerendered header

# Test as Googlebot (should get cached HTML)
curl -I -H "User-Agent: Googlebot/2.1" https://yourdomain.com/
# → X-Prerendered: true
# → X-Cache: hit

# Test as ChatGPT (should get full HTML content)
curl -H "User-Agent: ChatGPT-User" https://yourdomain.com/about
# → Full HTML with <h1>, <p>, Schema.org markup

# Test social media previews
curl -H "User-Agent: facebookexternalhit/1.1" https://yourdomain.com/
# → HTML with Open Graph tags
```

---

## Customization

### Adding Your Own Script Injection

Replace the `INJECT_SCRIPT` constant in `worker.js`:

```javascript
const INJECT_SCRIPT = `<script>
  // Your analytics, CMP, or tracking script here
</script>`;
```

### Adding More Bot Patterns

Add to the `BOT_AGENTS` array:

```javascript
const BOT_AGENTS = [
  // ... existing bots
  'my-custom-bot',
];
```

### Excluding Paths

Add to `isExcludedPath()`:

```javascript
function isExcludedPath(path) {
  const excludedPatterns = [
    '/cms',
    '/api/',
    '/admin/',     // Add your own
    '/dashboard/', // Add your own
  ];
  return excludedPatterns.some(pattern => path.startsWith(pattern));
}
```

---

## Cost Breakdown

| Service | Free Tier Limit | Typical Usage |
|---------|----------------|---------------|
| Cloudflare Pages | Unlimited bandwidth | ✅ Well within |
| Cloudflare Worker | 100,000 requests/day | ✅ Well within |
| Supabase Database | 500MB storage | ✅ ~50MB for 10K pages |
| Supabase Edge Functions | 500K invocations/month | ✅ Well within |
| **Total** | — | **$0/month** |

---

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| `PAGES_ORIGIN` missing `https://` | Always include the protocol: `https://abc.pages.dev` |
| Bot Fight Mode enabled | Cloudflare → Security → Bots → Turn OFF |
| Domain added as Pages Custom Domain | Use Worker Routes instead |
| Empty `prerendered_pages` table | Run your cache generator first |
| `robots.txt` blocking crawlers | Ensure `Allow: /` for all bots |

---

## Project Structure

```
your-project/
├── cloudflare-worker/
│   └── worker.js          ← The bot-detection router
├── supabase/
│   └── functions/
│       ├── prerender/      ← Serves cached HTML to bots
│       └── generate-cache/ ← Populates the HTML cache
├── wrangler.toml           ← Worker deployment config
└── public/
    └── _redirects           ← SPA fallback for Cloudflare Pages
```

---

## FAQ

**Q: Does this work with Vite/CRA/any React setup?**
Yes. The Worker doesn't care what builds your SPA — it only routes traffic.

**Q: What about Vue/Svelte/Angular?**
Works with any SPA framework. The bot detection and routing logic is framework-agnostic.

**Q: How many pages can I cache?**
Tested with 10,000+ pages. Supabase free tier gives you 500MB — enough for ~50,000 pages.

**Q: What if a page isn't in the cache?**
The Worker falls back to serving the SPA shell, so the bot gets the standard React app.

**Q: How fresh is the cached HTML?**
As fresh as your `pg_cron` schedule. Default is every 6 hours.

**Q: Can I use this without Supabase?**
Yes — swap the prerender function for any API/database that returns HTML by path. The Worker is the core.

---

## License

MIT — use it, fork it, ship it.

---

## Credits

Built with [Cloudflare Workers](https://workers.cloudflare.com/), [Supabase](https://supabase.com), and stubbornness to avoid paying for SSR.

*Serving 9,000+ SEO-optimized pages for $0/month since 2025.*
