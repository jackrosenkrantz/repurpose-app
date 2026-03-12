# Brainstorm: Multi-Platform Content Posting Engine

**Date:** 2026-03-12
**Status:** Draft
**Author:** Jack + Claude

---

## What We're Building

A multi-platform content adaptation and posting engine that extends the existing repurpose app. After the repurpose engine generates LinkedIn-optimized variants, this system:

1. **Adapts** each variant for target platform norms (character limits, formatting, tone)
2. **Generates images** for Instagram (quote cards, text graphics, carousels)
3. **Presents** all adapted variants in a web dashboard with per-platform toggles
4. **Publishes** to X, Instagram, Threads, and Facebook via official APIs
5. **Excludes LinkedIn** — no automated posting. LinkedIn stays manual/copy-paste only.

## Why This Approach

- **Direct API integration** gives full control and avoids monthly middleman costs
- **Extending the existing repurpose app** keeps the system unified — one place to generate, review, and publish
- **Platform-specific adaptation** via Claude ensures each post feels native to its platform, not like a cross-posted afterthought
- **Web dashboard with approval** keeps Jack in control — nothing posts without explicit action

## Key Decisions

1. **LinkedIn is explicitly excluded** from automated posting. Too high-risk for account safety. LinkedIn variants remain copy-paste from the existing UI.

2. **Official APIs only.** No scraping, no browser automation, no unofficial endpoints. Platforms:
   - **X (Twitter):** v2 API, free tier (1,500 posts/month). Text + image support.
   - **Instagram:** Meta Graph API. Requires Facebook Page + linked IG Business/Creator account. Image required for every post.
   - **Threads:** Meta Threads API (launched 2024). Text-first, simpler than IG.
   - **Facebook:** Meta Graph API. Page posting (not personal profile — personal profile API is heavily restricted).

3. **Platform adaptation is AI-powered.** Claude reformats each variant per platform:
   - **X:** Compress to 280 chars or split into thread. No hashtags (Jack's rule). Punchier tone.
   - **Instagram:** Caption format. Line breaks for readability. Can include hashtags (Instagram-native). Pair with generated image.
   - **Threads:** Similar to LinkedIn but slightly more casual. 500 char limit.
   - **Facebook:** Longer-form OK. More conversational framing. Link-friendly.

4. **Image generation for Instagram** is part of the system, not a separate step. Initial approach: generate quote cards / text-overlay graphics using a template engine or AI image gen (Gemini, DALL-E). Start simple — styled text cards with Jack's brand colors.

5. **Architecture: extend repurpose app.** New modules added to anchor-bot + repurpose web app:
   - `bot/publishers/` — platform-specific posting clients (x_publisher.py, meta_publisher.py, etc.)
   - `bot/intelligence/platform_adapter.py` — Claude-powered text adaptation per platform
   - `bot/intelligence/image_generator.py` — generates Instagram visuals
   - Web UI gets a "Publish" panel after variant generation

6. **Web dashboard is the primary publishing interface.** Flow:
   - Generate variants (existing flow)
   - New "Adapt for platforms" step shows each variant reformatted per platform
   - Toggle which platforms to publish to
   - Preview adapted text + images
   - Hit "Publish" — posts go out

7. **No scheduling in v1.** Posts go out immediately on publish. Scheduling queue is a future enhancement.

## Platform API Requirements

### X (Twitter)
- Developer account + app (free tier)
- OAuth 2.0 with PKCE (user context) or OAuth 1.0a
- Endpoints: POST /2/tweets (text), media upload for images
- Rate limit: 1,500 tweets/month on free tier
- Cost: Free

### Meta (Instagram + Facebook + Threads)
- Meta Developer account + Facebook App
- Facebook Page required (IG must be linked as Business/Creator account)
- Graph API permissions: `pages_manage_posts`, `instagram_basic`, `instagram_content_publish`, `threads_manage_posts`
- App Review required for Instagram publishing (Meta reviews your app before granting publish access)
- Cost: Free (API access)

### Image Generation
- Options: Gemini API (already have access), template-based (HTML canvas → image), or dedicated service
- Start with styled quote cards using brand palette (sand, amber, deep text)
- Consider: Gemini's image gen for more creative visuals

## Resolved Questions

1. **Facebook Page:** Need to create a Facebook Business Page. No existing one. (Prerequisite before Meta API integration.)
2. **Instagram Account Type:** Already a Business account. Ready for API publishing.
3. **Image Style Direction:** Mix it up — rotate styles per variant type. Poetic = photo-backed, radical = dark brand cards, sales = minimalist, etc.

4. **Meta App Review:** Submit for review as part of setup. Build X + Threads in parallel while waiting for approval.
5. **Posting Frequency:** Daily (1 post/day per platform). Well within X free tier (1,500/month) and Meta limits.

## Open Questions

None — all resolved.

## Success Criteria

- Generate platform-adapted variants from a single repurpose run
- Preview all adapted content in web dashboard before publishing
- One-click publish to selected platforms
- Instagram posts include auto-generated images
- LinkedIn remains manual — no API integration, no risk
- All publishing uses official, sanctioned APIs
- Account safety: no behavior that could trigger platform flags
