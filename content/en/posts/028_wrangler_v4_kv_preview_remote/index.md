+++
date = '2025-12-07T11:52:29+09:00'
draft = false
title = 'Cloudflare Workers + KV Preview/Production Setup'
categories = ["cloudflare"]
+++


**Personal Development Log & Reproducible Setup Guide**

## Overview

This guide documents the process of building a minimal UI prototype using Cloudflare Workers (Wrangler v4) and Workers KV.

**Goal**: Fetch JSON articles from Cloudflare KV → Render minimal HTML UI, and ensure that the process is fully reproducible later, even after Cloudflare changes its tooling.


### Environment

- macOS: 13.7.2 (Intel)
- Node.js: 20.2.0
- npm: 9.6.6
- Wrangler: 4.53.0
- Cloudflare Workers Runtime: 2025-12-05

### What This Guide Documents

- The exact steps I performed
- The differences between Wrangler v3 and v4
- How preview KV was generated
- How remote KV access was enabled in dev mode
- How UI finally succeeded in reading the preview KV

It is intentionally verbose, because it also serves as a personal development log.

---

## 1. Project Initialization

```bash
mkdir your-app
cd your-app
wrangler init .
npm install --save-dev wrangler
```

Wrangler created:

- `src/index.ts`
- `wrangler.jsonc` (later replaced with `wrangler.toml`)
- TypeScript configs / test files

---

## 2. Replacing wrangler.jsonc → wrangler.toml

I replaced the JSONC config with TOML for clarity.

```bash
rm wrangler.jsonc
touch wrangler.toml
```

Initial contents:

```toml
name = "your-app"
main = "src/index.ts"
compatibility_date = "2025-12-05"

[observability]
enabled = true
```

---

## 3. Creating Workers KV Namespaces (Production + Preview)

Cloudflare KV requires:

- **id** — production namespace
- **preview_id** — dev-time namespace

**Important**: Wrangler does NOT automatically generate `preview_id` in v4 (unlike v3). Therefore I manually created both namespaces.

### 3.1 Production KV Namespace

```bash
npx wrangler kv namespace create test_kv
```

This returned:

```
id = "abcde12345"
```

### 3.2 Preview KV Namespace (Manual — Wrangler v4 Required)

Wrangler v4 does not auto-generate preview KV. I generated it explicitly:

```bash
npx wrangler kv namespace create test_kv --preview
```

This returned:

```
preview_id = "fghi67890"
```

> ⚠️ **Warning**: This step is missing in most online docs, because Wrangler v4 changed its dev behavior.

---

## 4. Configuring wrangler.toml for KV

Final KV configuration:

```toml
[[kv_namespaces]]
binding = "test_kv"
id = "abcde12345"        # Production KV
preview_id = "fghi67890" # Dev KV (Preview)
remote = true            # Force using remote KV in dev
```

### Why `remote = true`?

Because:

- Wrangler v4 defaults to **local Miniflare KV**
- Even when `preview_id` is provided
- Therefore Worker kept returning `RAW: null`
- **Only after setting `remote = true`, dev mode read Preview KV correctly**

This is one of the major v3 → v4 breaking differences.

---

## 5. Loading Sample Data into KV

### Local Version

```bash
npx wrangler kv key put latest_articles "$(cat ./data/latest_articles.json)" --binding=test_kv --local
```

### Preview (Remote) Version

```bash
npx wrangler kv key put latest_articles \
  "$(cat ./data/latest_articles.json)" \
  --binding=test_kv --remote
```

### Verification

```bash
npx wrangler kv key get latest_articles --binding=test_kv --remote
```

Result (correct):

```json
[
  {
    "title": "Example Article",
    "url": "https://example.com",
    "summary": "This is a test summary."
  }
]
```

---

## 6. Worker Code (src/index.ts)

A minimal HTML renderer that logs raw KV output:

```typescript
export default {
  async fetch(request, env) {
    const raw = await env.test_kv.get("latest_articles");
    console.log("RAW:", raw);

    let articles = [];
    try {
      articles = raw ? JSON.parse(raw) : [];
    } catch {}

    const html = `
      <html>
      <body>
        <h1>📰 NewsLite UI (PoC)</h1>
        ${articles.map(a => `
          <div>
            <a href="${a.url}">${a.title}</a>
            <p>${a.summary}</p>
          </div>
        `).join("")}
      </body>
      </html>
    `;

    return new Response(html, {
      headers: { "Content-Type": "text/html; charset=UTF-8" },
    });
  },
};
```

---

## 7. Running the Worker (Dev Mode)

```bash
npx wrangler dev
```

Expected log:

```
Your Worker has access to:
env.test_kv  KV Namespace   remote
```

When this finally changed from `local` → `remote`, KV data began appearing in the browser.

---

## 8. Result

Visiting:

```
http://localhost:8787/
```

Correctly displayed:

```
📰 NewsLite UI (PoC)

Example Article
This is a test summary.
```

This confirmed:

- ✅ Preview KV is working
- ✅ Worker code is correct
- ✅ wrangler.toml is correct
- ✅ remote binding is functioning

---

## 9. v3 → v4: Important Behavioral Differences

This section captures the conceptual differences that caused debugging difficulty.

### 9.1. KV Preview Behavior

| Wrangler version | Behavior |
|------------------|----------|
| **v3.x** | `preview_id` → automatically used in `wrangler dev` |
| **v4.x** | Local KV (Miniflare sqlite) is used **even if preview_id is set** |

### 9.2. Remote KV Access

| Action | v3 Behavior | v4 Behavior |
|--------|-------------|-------------|
| dev mode KV read | Remote Preview KV | Local KV unless `remote = true` |
| preview_id required? | Optional | Required for remote dev |
| Need explicit remote flag? | No | Yes (`remote = true`) |

### 9.3. Consequence

In Wrangler v4, you can have correct KV data in Cloudflare Dashboard, but Worker will still read empty local KV → `RAW: null`

This was exactly the issue we debugged.

---

## 10. Final Notes

### Key Takeaways

- ⚠️ **Wrangler 4 introduces breaking changes not clearly documented**
- 📚 Many online tutorials still describe Wrangler 3 behavior
- 🔧 **Preview KV must be explicitly created**
- 🌐 **`remote = true` is essential for real remote KV access**

I hope this guide helps ensure reproducibility for Cloudflare Workers + KV development.