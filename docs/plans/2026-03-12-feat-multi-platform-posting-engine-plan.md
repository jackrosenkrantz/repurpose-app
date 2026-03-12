---
title: Multi-Platform Content Posting Engine
type: feat
status: active
date: 2026-03-12
deepened: 2026-03-12
origin: docs/brainstorms/2026-03-12-multi-platform-posting-engine-brainstorm.md
---

# Multi-Platform Content Posting Engine

## Enhancement Summary

**Deepened on:** 2026-03-12
**Research agents used:** Security Sentinel, Architecture Strategist, Performance Oracle, Python Reviewer, Code Simplicity Reviewer, Data Integrity Guardian, OAuth Best Practices Researcher, Pillow Quote Card Researcher

### Key Improvements
1. **Simplified architecture** — Collapsed from 4 phases to 2. Eliminated separate `/api/adapt` endpoint, OAuth callback infrastructure, and PIN auth. Tokens stored as env vars for single-user simplicity.
2. **Security hardened** — Fernet encryption for any DB-stored tokens. Server-side API key auth on all mutating endpoints. OAuth state parameter validation.
3. **Performance optimized** — Parallel platform publishing via `asyncio.gather()`. SSE streaming for real-time per-platform status. Haiku + Gemini calls run concurrently.
4. **Concrete code patterns** — Full Pillow quote card generator with auto-scaling fonts, text wrapping, brand-styled templates. Production-ready Instagram image pipeline.

### Simplification Decisions (from Code Simplicity Review)
- **Tokens as env vars** for X (OAuth 1.0a static keys) and Meta (long-lived Page tokens). No `platform_tokens` Supabase table in v1. One-time local OAuth scripts store results as Railway env vars.
- **Merged adapt + publish** into single `/api/publish` endpoint. No separate adaptation step.
- **Image generation deferred to Phase 2.** Phase 1 ships text-only publishing to all platforms.
- **No PIN auth.** Use existing `PUBLISH_API_KEY` pattern (same as portal intake key). Server-side check on every mutating request.

---

## Overview

Extend the existing repurpose app to adapt LinkedIn-optimized content variants for X, Instagram, Threads, and Facebook — then publish directly via official APIs from a web dashboard. LinkedIn remains manual/copy-paste only. (see brainstorm: docs/brainstorms/2026-03-12-multi-platform-posting-engine-brainstorm.md)

## Problem Statement

The repurpose engine generates 5 excellent LinkedIn-optimized variants, but posting to other platforms requires manually reformatting and copy-pasting each one. This friction means Jack rarely cross-posts, leaving reach on the table across X, Instagram, Threads, and Facebook.

## Proposed Solution

A two-layer addition to the existing repurpose system:

1. **Platform Adapter + Publisher** (single `/api/publish` call) — adapts text via Claude Haiku, then posts to selected platforms
2. **Web UI** — publish panel in the existing repurpose app with preview, toggles, and one-click publish

Phase 2 adds: image generation for Instagram quote cards.

All gated behind `PUBLISH_API_KEY` server-side auth (same pattern as portal intake).

## Technical Approach

### Architecture

```
Existing Flow (unchanged):
  User Input → /api/repurpose → content_repurposer.py → 5 variants → Web UI

New Flow (Phase 1 — text only):
  User picks variant → clicks "Publish" → selects platforms
    → POST /api/publish {variant_text, variant_type, platforms[]}
    → platform_adapter.py adapts text (Claude Haiku)
    → publishers post to each platform concurrently (asyncio.gather)
    → SSE streams per-platform results back to UI

New Flow (Phase 2 — adds images):
  Same as above, plus:
    → image_generator.py creates quote card (Pillow)
    → image uploaded to R2 (public URL for Instagram)
    → Instagram publisher uses image URL
```

### Research Insights: Architecture

**From Architecture Review:**
- The `publishers/` directory semantic shifts from "Notion output" to "external platform output." Update anchor-bot CLAUDE.md to document this.
- Use the allowlist CORS pattern from `public_api.py` for new endpoints (not the wildcard `*` from `server.py`).
- All Supabase access must go through existing `bot/handlers/supabase_client.py` — do not create parallel DB access in new modules.

**From Simplicity Review:**
- For a single user posting 1/day, full OAuth token management infrastructure is overkill. Store tokens as Railway env vars. X uses static OAuth 1.0a keys (never expire). Meta Page tokens are long-lived (never expire if generated from long-lived user token).
- Threads tokens expire in 60 days — add a simple refresh function triggered via Telegram `/refresh_threads` command.

**New files in anchor-bot:**

| File | Purpose |
|---|---|
| `bot/intelligence/platform_adapter.py` | Claude Haiku: adapt variant text per platform |
| `bot/publishers/x_publisher.py` | X/Twitter v2 API posting |
| `bot/publishers/meta_publisher.py` | Meta Graph API: Facebook, Instagram, Threads |
| `bot/intelligence/image_generator.py` | Generate Instagram quote cards (Phase 2) |
| `bot/services/image_storage.py` | Upload images to Cloudflare R2 (Phase 2) |

**New API endpoints:**

| Endpoint | Method | Purpose |
|---|---|---|
| `/api/publish` | POST | Adapt + publish to selected platforms (SSE response) |
| `/api/publish/history` | GET | Recent publish history |

**Web UI changes (repurpose-app `public/index.html`):**
- New "Publish" button per variant (appears after generation)
- Publishing panel: platform toggles, preview of adapted text, publish button
- SSE-driven status indicators per platform
- API key sent via `X-Publish-Key` header on every request

### Research Insights: Security

**From Security Sentinel (3 Critical, 3 High findings):**

1. **CRITICAL: Server-side auth on ALL endpoints.** The web app is static HTML — any client-side check is bypassable. Every mutating endpoint MUST validate `X-Publish-Key` header server-side. Use `hmac.compare_digest()` for timing-safe comparison.

2. **CRITICAL: If tokens are ever stored in Supabase (e.g., Threads refresh), encrypt with Fernet.**
   ```python
   from cryptography.fernet import Fernet
   cipher = Fernet(os.environ["FERNET_KEY"])
   encrypted = cipher.encrypt(token.encode()).decode()
   decrypted = cipher.decrypt(encrypted.encode()).decode()
   ```
   Store `FERNET_KEY` in Railway env vars only. Never in code.

3. **CRITICAL: OAuth state parameter.** If OAuth callbacks are ever added, generate a random `state`, store server-side with TTL, validate on callback.

4. **HIGH: Rate limiting on publish endpoint.** Max 10 publishes per hour per API key. Log every publish attempt.

5. **HIGH: Input validation.** Validate content length per platform, platform parameter against allowlist, structured error responses.

### Research Insights: Performance

**From Performance Oracle:**

1. **Parallel publishing is mandatory.** Sequential publishing across 4 platforms takes 47-93 seconds. Concurrent via `asyncio.gather()` reduces to ~35 seconds (limited by Threads' 30-second mandatory delay).

   ```python
   async def publish_all(adapted_texts, platforms):
       tasks = []
       for platform in platforms:
           tasks.append(publish_to_platform(platform, adapted_texts[platform]))
       results = await asyncio.gather(*tasks, return_exceptions=True)
       return dict(zip(platforms, results))
   ```

2. **SSE for real-time results.** User sees X and Facebook complete in 3-5 seconds, Instagram in 10-30 seconds, Threads at ~35 seconds. Much better UX than waiting for all to complete.

3. **Haiku adaptation + Gemini image gen run concurrently** (they're independent). Saves 2-4 seconds.

4. **All waits must use `asyncio.sleep()`, never `time.sleep()`.** Blocking the event loop stalls the Telegram webhook handler.

5. **Reuse a single `httpx.AsyncClient`** with connection pooling instead of creating per-request clients.

6. **Supabase tracking writes are fire-and-forget** — don't block the publish response on DB writes.

### Research Insights: Python Patterns

**From Python Reviewer:**

1. **All sync SDKs must use `asyncio.to_thread()`.** This applies to: tweepy (X), boto3 (R2), Pillow (image compositing). Follow the existing `email_service.py` pattern.

2. **Return types must be typed.** Use `TypedDict` for adapter output, not bare dicts.

3. **Extract JSON parsing helper to `bot/utils/json_helpers.py`** (2x rule — already duplicated in `content_repurposer.py`).

4. **Publishers raise exceptions** (`XPublishError`, `MetaPublishError`). Intelligence modules return error dicts (existing pattern). Don't mix.

5. **Consider raw `httpx` with OAuth 1.0a instead of tweepy** — consistent with codebase pattern of avoiding SDKs for API calls. Tweepy is a large dependency for two endpoints.

### Implementation Phases

#### Phase 1: Text Publishing to All Platforms

**Goal:** Adapt and publish text content to X, Instagram (text-only/skip), Threads, and Facebook from the web dashboard.

**Prerequisites (manual, do in parallel with development):**
- [ ] Register X Developer account, create app (free tier), get OAuth 1.0a keys (4 static env vars: `X_API_KEY`, `X_API_SECRET`, `X_ACCESS_TOKEN`, `X_ACCESS_SECRET`)
- [ ] Create Facebook Business Page
- [ ] Register Meta Developer App
- [ ] Complete Meta OAuth flow locally (one-time script), get long-lived Page Access Token → `META_PAGE_ACCESS_TOKEN`, `META_PAGE_ID` env vars
- [ ] Complete Threads OAuth flow locally, get long-lived token → `THREADS_ACCESS_TOKEN`, `THREADS_USER_ID` env vars
- [ ] Get Instagram User ID → `INSTAGRAM_USER_ID` env var
- [ ] Submit Meta App Review for `instagram_content_publish` (start early — 1-6 week lead time)
- [ ] Set Railway env vars: `PUBLISH_API_KEY`, `FERNET_KEY`, all platform tokens above

**Tasks:**

- [ ] Create `published_posts` Supabase table (improved schema from Data Integrity review):
  ```sql
  CREATE TABLE published_posts (
    id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
    repurpose_job_id UUID NOT NULL REFERENCES repurpose_jobs(id),
    variant_type TEXT NOT NULL,
    platform TEXT NOT NULL,
    adapted_text TEXT NOT NULL,
    image_url TEXT,
    platform_post_id TEXT,
    status TEXT DEFAULT 'draft'
      CHECK (status IN ('draft', 'publishing', 'published', 'failed')),
    error_message TEXT,
    error_code TEXT,
    retry_count INT DEFAULT 0,
    published_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW(),
    CONSTRAINT uq_published_posts_job_variant_platform
      UNIQUE (repurpose_job_id, variant_type, platform)
  );

  CREATE INDEX idx_published_posts_job_id ON published_posts(repurpose_job_id);
  CREATE INDEX idx_published_posts_status ON published_posts(status)
    WHERE status IN ('draft', 'publishing', 'failed');

  ALTER TABLE published_posts ENABLE ROW LEVEL SECURITY;
  -- No policies = service_role only access
  ```

- [ ] Build `bot/intelligence/platform_adapter.py`:
  - Single Claude Haiku call via httpx (no SDK, consistent with codebase)
  - Input: variant text + variant_type + target platforms
  - Output: typed `dict[str, AdaptedContent]` with adapted text per platform
  - Platform rules:
    - **X:** Compress to 280 chars OR split into thread (return `list[str]`). Punchier tone. No hashtags.
    - **Instagram:** Caption format, line breaks for readability, 5-10 hashtags (Instagram-only exception).
    - **Threads:** Casual tone, 500 char max, no hashtags.
    - **Facebook:** Conversational framing, longer-form OK, no hashtags.
  - Extract JSON parsing to `bot/utils/json_helpers.py` (2x rule)

- [ ] Build `bot/publishers/x_publisher.py`:
  - Uses OAuth 1.0a static tokens from env vars (no refresh needed)
  - `post_tweet(text)` — `POST https://api.x.com/2/tweets` via httpx with OAuth 1.0a signing
  - `post_thread(tweets: list[str])` — sequential, each reply references previous tweet ID
  - All wrapped in `asyncio.to_thread()` if using tweepy, or native async if using httpx directly
  - Raises `XPublishError` on failure

- [ ] Build `bot/publishers/meta_publisher.py`:
  - Shared `_graph_api_post(endpoint, payload, access_token)` base method via httpx
  - `post_to_facebook(text, image_url=None)` — `POST /{PAGE_ID}/feed`
  - `post_to_threads(text, image_url=None)` — two-step: create container → `asyncio.sleep(30)` → publish via `graph.threads.net`
  - `post_to_instagram(image_url, caption)` — two-step: create container → poll with bounded timeout (60s, exponential backoff) → publish (Phase 2 only, skip in Phase 1)
  - Raises `MetaPublishError` on failure

  ```python
  # Instagram polling pattern (Phase 2)
  async def _wait_for_container(container_id, timeout=60):
      elapsed, interval = 0, 2
      while elapsed < timeout:
          status = await _check_status(container_id)
          if status == "FINISHED": return True
          if status == "ERROR": raise MetaPublishError("Container failed")
          await asyncio.sleep(interval)
          elapsed += interval
          interval = min(interval * 1.5, 10)
      raise TimeoutError(f"Container not ready after {timeout}s")
  ```

- [ ] Add `/api/publish` SSE endpoint to `server.py`:
  - Auth: validate `X-Publish-Key` header with `hmac.compare_digest()`
  - Input: `{repurpose_job_id, variant_type, variant_text, platforms[]}`
  - Adapts text (Haiku call)
  - Publishes to all platforms concurrently (`asyncio.gather()`)
  - Streams per-platform results as SSE events
  - Writes to `published_posts` table (fire-and-forget)
  - Duplicate check: query `published_posts` for existing job+variant+platform combo, warn via SSE if exists

- [ ] Add `/api/publish/history` endpoint:
  - Returns recent publishes from `published_posts` table
  - Auth: validate `X-Publish-Key` header

- [ ] Add repurpose-app origin to CORS allowlist (use `public_api.py` pattern, not `server.py` wildcard)

- [ ] Update web UI (`public/index.html`):
  - "Publish" button on each variant card
  - Publish panel: platform toggles (X, Threads, Facebook — Instagram grayed out until Phase 2)
  - API key input (stored in localStorage, sent as `X-Publish-Key` header)
  - SSE listener for real-time per-platform status updates
  - Retry button for failed platforms
  - Duplicate post warning
  - Use data-attribute + event delegation pattern for buttons (XSS prevention per `docs/solutions/`)

- [ ] Add Threads token refresh utility:
  - Simple `refresh_threads_token()` function in a utility module
  - Triggered via Telegram `/refresh_threads` command
  - Refreshes long-lived token via `GET graph.threads.net/refresh_access_token`
  - Updates Railway env var (or, if simpler, stores encrypted in Supabase)

**Success criteria:** Can pick a variant, select platforms, click publish, and see posts go live on X, Threads, and Facebook with real-time status updates.

#### Phase 2: Image Generation + Instagram Publishing

**Goal:** Generate Instagram quote cards and enable Instagram publishing (pending Meta App Review approval).

**Tasks:**

- [ ] Build `bot/intelligence/image_generator.py`:
  - **Hybrid approach:** Gemini generates artistic backgrounds, Pillow composites exact text
  - Style rotation by variant type:
    - `sales` → minimalist white/cream background, dark text
    - `poetic` → Gemini-generated nature/landscape background with dark overlay
    - `radical` → dark brand card (Jack OS palette: `#1a1714` bg, `#c4956a` text)
    - `weekend_bridge` → Gemini-generated warm scene with overlay
    - `short_contrarian` → bold high-contrast (dark bg, large amber text)
  - Output: 1080x1080 JPEG (quality=85, `subsampling=0` for text clarity)
  - Auto-scaling font size via binary search (24-80px range)
  - Text wrapping with pixel-accurate measurement via `draw.textlength()`
  - Vertical centering via `anchor="mm"` on `multiline_text()`
  - Fonts: Inter for body, Lora for headings (bundled as `.ttf` files)
  - Fallback: if Gemini fails, solid color background from brand palette
  - All Pillow work via `asyncio.to_thread()` (CPU-bound)
  - Gemini via `httpx.AsyncClient` directly (no SDK)

  ```python
  # Core generation pattern
  async def generate_quote_card(text: str, variant_type: str) -> bytes:
      style = STYLE_MAP[variant_type]
      if style.needs_ai_background:
          bg_bytes = await _generate_background(style.prompt)  # async Gemini call
      else:
          bg_bytes = None
      return await asyncio.to_thread(_composite, text, style, bg_bytes)

  def _composite(text: str, style: Style, bg_bytes: bytes | None) -> bytes:
      # All Pillow work here — runs in thread pool
      img = Image.new("RGB", (1080, 1080), style.bg_color)
      if bg_bytes:
          bg = Image.open(io.BytesIO(bg_bytes)).convert("RGBA")
          # ... overlay + crop logic
      draw = ImageDraw.Draw(img)
      font, wrapped = _auto_scale_font(text, draw, style)
      draw.multiline_text((540, 540), wrapped, font=font, fill=style.text_color,
                          anchor="mm", align="center", spacing=14)
      buf = io.BytesIO()
      img.save(buf, "JPEG", quality=85, optimize=True, subsampling=0)
      return buf.getvalue()
  ```

- [ ] Build `bot/services/image_storage.py`:
  - Upload JPEG to Cloudflare R2 bucket via `boto3` (wrapped in `asyncio.to_thread()`)
  - Return public URL (required by Instagram API)
  - R2 lifecycle rule: auto-delete objects older than 30 days
  - Uses `R2_ACCOUNT_ID`, `R2_ACCESS_KEY_ID`, `R2_SECRET_ACCESS_KEY`, `R2_BUCKET_NAME` env vars

- [ ] Enable Instagram in `/api/publish`:
  - After Meta App Review approval, add Instagram to the publish flow
  - Generate image → upload to R2 → pass public URL to `meta_publisher.post_to_instagram()`
  - Image generation runs concurrently with text adaptation (they're independent)

- [ ] Update web UI:
  - Enable Instagram toggle (was grayed out in Phase 1)
  - Image preview in publish panel
  - "Regenerate image" button
  - "Upload custom image" fallback (file input → upload to R2)

- [ ] Add image to Threads/Facebook posts (optional — both support image posts for higher engagement)

**Success criteria:** Can generate a branded quote card, preview it, publish to Instagram with the image. All 4 platforms fully operational.

## System-Wide Impact

- **Interaction graph:** User generates variants (existing) → clicks publish → Claude Haiku adapts text (concurrent with Gemini image gen in Phase 2) → publishes to all selected platforms concurrently → SSE streams results. Each publish writes to `published_posts` (fire-and-forget). Telegram notification on completion (fire-and-forget).
- **Error propagation:** Platform API failures are isolated per-platform via `asyncio.gather(return_exceptions=True)`. One failure doesn't block others. All errors written to `published_posts.error_message`. Token errors surface as clear messages in SSE stream.
- **State lifecycle risks:** Partial publish (some succeed, some fail) is the main risk. Mitigated by: per-platform status tracking, unique constraint prevents duplicates, retry capability on failed platforms, Instagram containers auto-expire in 24h if orphaned.
- **API surface parity:** `/api/publish` is a standalone API — can later be called from Telegram bot (`/publish` command), not just the web UI.
- **Security:** All mutating endpoints validate `X-Publish-Key` server-side with timing-safe comparison. OAuth tokens stored as Railway env vars (simplest, most secure for single user). Any tokens stored in Supabase (Threads refresh) are Fernet-encrypted. RLS enabled, service_role only.

## Acceptance Criteria

### Functional Requirements
- [ ] Adapt any of 5 variant types for X, Instagram, Threads, and Facebook
- [ ] X adaptation: compress to 280 chars or split into thread
- [ ] Instagram adaptation: caption format with hashtags + auto-generated quote card image (Phase 2)
- [ ] Threads adaptation: casual tone, 500 char limit
- [ ] Facebook adaptation: conversational, longer-form
- [ ] Preview adapted text before publishing
- [ ] Toggle which platforms to publish to
- [ ] One-click publish with real-time SSE status per platform
- [ ] Per-platform retry on failure
- [ ] Duplicate post warning (unique constraint on job+variant+platform)
- [ ] LinkedIn excluded from all automation
- [ ] Adapted text editable before publishing

### Non-Functional Requirements
- [ ] All publishing via official, sanctioned APIs only
- [ ] All mutating endpoints authenticated server-side (`X-Publish-Key` header, `hmac.compare_digest`)
- [ ] Tokens stored as env vars (X, Meta Page) or Fernet-encrypted in Supabase (Threads refresh)
- [ ] Rate limiting on publish endpoint (max 10/hour)
- [ ] All async waits use `asyncio.sleep()`, never `time.sleep()`
- [ ] All sync SDKs wrapped in `asyncio.to_thread()`
- [ ] No behavior that could trigger platform account flags

## Dependencies & Prerequisites

1. **X Developer account** — register at developer.x.com, create app (free tier). Get OAuth 1.0a keys.
2. **Facebook Business Page** — create manually before Meta API integration
3. **Meta Developer App** — register at developers.facebook.com
4. **Meta App Review** — submit for `instagram_content_publish` (1-6 week lead time, blocks Instagram only)
5. **One-time local OAuth scripts** — run locally to get Meta Page token + Threads token, store as Railway env vars
6. **Cloudflare R2 bucket** (Phase 2) — for hosting Instagram images at public URLs
7. **Gemini API key** (Phase 2) — already have access
8. **Railway env vars:**
   - `PUBLISH_API_KEY` — auth for publish endpoints (min 32 chars: `openssl rand -hex 32`)
   - `FERNET_KEY` — encryption key for any DB-stored tokens (`python -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())"`)
   - `X_API_KEY`, `X_API_SECRET`, `X_ACCESS_TOKEN`, `X_ACCESS_SECRET` — X OAuth 1.0a (static, never expire)
   - `META_PAGE_ACCESS_TOKEN`, `META_PAGE_ID` — Facebook/Instagram (long-lived, never expire)
   - `THREADS_ACCESS_TOKEN`, `THREADS_USER_ID` — Threads (60-day expiry, refresh via `/refresh_threads`)
   - `INSTAGRAM_USER_ID` — Instagram Business account ID
   - Phase 2: `R2_ACCOUNT_ID`, `R2_ACCESS_KEY_ID`, `R2_SECRET_ACCESS_KEY`, `R2_BUCKET_NAME`, `GEMINI_API_KEY`

## Risk Analysis

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Meta App Review rejected | Low | High (blocks Instagram) | Ship X + Threads + Facebook first. Instagram is additive. |
| X free tier limits reduced further | Medium | Low (1/day is well within limits) | Monitor. Free tier is 500 posts/month. |
| Token expiry (Threads) | Certain | Low | `/refresh_threads` Telegram command. Token lasts 60 days. |
| Gemini image gen poor quality (Phase 2) | Medium | Low | Fallback to solid-color brand cards. User can upload custom image. |
| Instagram container timing (Phase 2) | Low | Medium | Bounded polling with exponential backoff (60s timeout). |
| Sequential publish blocks event loop | Medium | High | `asyncio.gather()` + `asyncio.sleep()` — never `time.sleep()`. |

## Alternative Approaches Considered

1. **Buffer/Hootsuite as middleware** — Rejected: adds monthly cost and dependency. (see brainstorm)
2. **All-in-one Claude call** (generate all platform variants at once) — Rejected: V1 tried this, quality suffered. Two-step is better.
3. **Template-only image generation** (Phase 2) — Partially adopted: Gemini for backgrounds, Pillow for text compositing to avoid AI text-rendering errors.
4. **Full OAuth token management + DB storage** — Rejected for v1: overkill for single user. Env vars are simpler and more secure.
5. **Separate `/api/adapt` endpoint** — Rejected: no real use case for adapting without publishing. Merged into single `/api/publish`.

## API Reference

### X (Twitter) API v2
- **Auth:** OAuth 1.0a (static keys, never expire)
- **Post tweet:** `POST https://api.x.com/2/tweets` with `{"text": "..."}`
- **Media upload:** `POST https://api.x.com/2/media/upload` (v2 endpoint, free tier)
- **Rate limit:** 500 posts/month on free tier
- **Thread:** Chain tweets via `reply: {in_reply_to_tweet_id: "..."}`

### Meta Graph API (Facebook + Instagram)
- **Auth:** Long-lived Page Access Token (never expires)
- **Facebook post:** `POST https://graph.facebook.com/v22.0/{PAGE_ID}/feed` with `{message, access_token}`
- **Facebook photo:** `POST https://graph.facebook.com/v22.0/{PAGE_ID}/photos` with `{url, caption, access_token}`
- **Instagram publish:** Two-step: create container (`POST /{IG_USER_ID}/media` with `{image_url, caption}`) → poll status → publish (`POST /{IG_USER_ID}/media_publish` with `{creation_id}`)
- **Instagram requires:** Public image URL (JPEG, max 8MB, 1080x1080)
- **Rate limit:** 200 calls/hour/page (Facebook), 50 posts/24h (Instagram)

### Threads API
- **Auth:** Long-lived token (60-day expiry, refreshable)
- **Base URL:** `https://graph.threads.net/v1.0/` (NOT graph.facebook.com)
- **Post:** Two-step: create container (`POST /{USER_ID}/threads` with `{media_type: "TEXT", text}`) → wait 30 seconds → publish (`POST /{USER_ID}/threads_publish` with `{creation_id}`)
- **Rate limit:** 250 posts/24h
- **Refresh:** `GET graph.threads.net/refresh_access_token?grant_type=th_refresh_token&access_token={token}`

## Sources & References

### Origin
- **Brainstorm document:** [docs/brainstorms/2026-03-12-multi-platform-posting-engine-brainstorm.md](docs/brainstorms/2026-03-12-multi-platform-posting-engine-brainstorm.md) — Key decisions: LinkedIn excluded, direct APIs only, platform-adapted via Claude, image gen included, extend existing repurpose app.

### Internal References
- Repurpose API: `bot/handlers/repurpose.py`, `bot/webhooks/server.py`
- Intelligence pattern: `bot/intelligence/content_repurposer.py`
- Publisher pattern: `bot/publishers/prospect_publisher.py`
- Voice library: `bot/intelligence/voice_library.py`
- Email service (async wrapper pattern): `bot/services/email_service.py`
- CORS allowlist pattern: `bot/handlers/public_api.py`
- XSS prevention: `docs/solutions/integration-issues/web-touch-logging-xss-and-async.md`
- Fire-and-forget pattern: `docs/solutions/integration-issues/web-touch-logging-xss-and-async.md`
- Dedup pattern: `docs/solutions/runtime-errors/batch-prospect-dedup-and-cost-controls.md`
- Supabase RLS pattern: `docs/solutions/integration-issues/supabase-rls-pii-stripped-views.md`
- Async SDK wrapper: `docs/solutions/integration-issues/resend-python-sdk-gotchas.md`

### External References
- X API v2: https://developer.x.com/en/docs/x-api
- Meta Graph API: https://developers.facebook.com/docs/graph-api
- Instagram Content Publishing: https://developers.facebook.com/docs/instagram-platform/content-publishing/
- Threads API: https://developers.facebook.com/docs/threads/posts/
- Threads Long-Lived Tokens: https://developers.facebook.com/docs/threads/get-started/long-lived-tokens/
- Gemini Image Generation: https://ai.google.dev/gemini-api/docs/image-generation
- Fernet Encryption: https://cryptography.io/en/latest/fernet/
- Pillow Text Anchors: https://pillow.readthedocs.io/en/stable/handbook/text-anchors.html
