+++
date = '2025-12-17T17:43:32+09:00'
draft = false
title = 'Wrangler v4 Production Command Guide'
categories = ["cloudflare"]
+++


# Wrangler v4 Production Command Guide

This article organizes the commands and concepts used in a PoC with Cloudflare Workers + KV / R2.

The focus is on the **operational phase with Production KV / R2**, not development (wrangler dev).

In the [previous article](https://tarclog.com/en/posts/028_wrangler_v4_kv_preview_remote/), we covered preview mode. This Production edition organizes the commands that worked in actual operations.

## Prerequisites (Important)

With wrangler v4, you'll get confused without understanding these axes:

| Aspect | Values |
|--------|--------|
| Worker Execution | local or Cloudflare |
| KV / R2 | local (emulation) or remote (actual) |
| Namespace | preview or production |

This article prioritizes **reliably interacting with Production KV / R2**.
For KV/R2, this guide assumes remote mode (real resources). The local mode stores its state under .wrangler/state (you can reset it by deleting that directory if needed).

## 1. JSON PUT to Production KV

### Recommended Command (Final Version)

```bash
npx wrangler kv key put articles/2025-12-13 \
  "$(cat data/daily_summary_2025-12-13_with_audio.json)" \
  --binding=newslite_kv \
  --remote \
  --preview=false
```

### Key Points

- `--remote`: Ensures writing to KV on Cloudflare
- `--preview=false`: Explicitly specifies production namespace (id)
- Adopts `cat` method
  - Can verify the actual content on the spot
  - Easy to integrate into CI / shell scripts

## 2. GET from Production KV (Verification)

```bash
npx wrangler kv key get articles/2025-12-13 \
  --binding=newslite_kv \
  --remote \
  --preview=false
```

### Purpose

- Immediate confirmation of successful write
- Clarify "which namespace am I looking at?"

## 3. wrangler dev (Production Data Verification)

### Basic Form

```bash
npx wrangler dev
```

### Important Notes (v4 Behavior)

- `wrangler dev` starts the Worker Runtime locally
- However, **KV / R2 can reference remote depending on configuration**

When `wrangler.toml` contains:

```toml
[[kv_namespaces]]
binding = "newslite_kv"
id = "..."
preview_id = "..."
remote = true
```

➡ Even on localhost, it directly reads production / preview KV

### Actual Verification URL

```
http://localhost:8787/?date=2025-12-13
```

## 4. wrangler deploy (Reflecting Worker Changes)

```bash
npx wrangler deploy
```

### When is deploy necessary?

| Case | Deploy Required? |
|------|-----------------|
| KV / R2 updates only | ❌ Not needed |
| index.ts changes | ✅ Required |
| Redirect specification changes | ✅ Required |
| HTML / UI modifications | ✅ Required |

**Since KV / R2 are independent from the Worker, "deploy is unnecessary for data updates only"** remains true in v4.

## 5. Production URL Verification

### workers.dev (Provided by Cloudflare)

```
https://<worker-name>.<your-subdomain>.workers.dev/?date=2025-12-13
```

### Custom Domain (Production Use)

```
https://newslite.tarclog.com/?date=2025-12-13
```

※ Even after setting up a Custom Domain, workers.dev remains available
(This is a different concept from R2.dev)

## 6. Common Confusion Points (v4-Specific)

### ❌ `wrangler dev --remote`

Deprecated in v4. Designed to control with `remote = true` in toml

### ❌ `key:put` / `key:get`

Officially `key put` / `key get`
No colon needed (GPT often gets this wrong)

## 7. Operational Flow (NewsLite Example)

1. Generate JSON (locally)
2. Auto-attach audio URL
3. Upload mp3 to R2
4. KV (preview) put → verify on localhost
5. KV (production) put
6. Verify on workers.dev / custom domain
7. Deploy if there are index.ts changes

## Summary

- Wrangler v4 is "local-oriented" but can force remote
- For production operations, always specify `--remote` + `--preview=false`
- KV / R2 updates and Worker deploy are separate operations
- Always be conscious of "which namespace am I touching?"