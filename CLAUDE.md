# Repurpose App

Content Repurposing Engine — turns one piece of content into 5 ready-to-ship variants + creative brief.

## Stack
- Node.js + Express serving static files from `public/`
- Single-page app in `public/index.html` (all CSS/JS inline)
- Talks to anchor-bot API at `/api/repurpose` endpoint
- Deployed on Railway (Nixpacks builder)

## Design
- Jack OS dark mode aesthetic (see global CLAUDE.md for palette)
- Typography: Lora for headings, Inter for body
- Fixed bottom nav with blurred backdrop
- Cards surface: #242019

## API
- POST to anchor-bot's `/api/repurpose` with `{text, source_type, is_coaching}`
- GET `/api/repurpose/{id}` to retrieve stored jobs
- API_URL configured via `window.API_URL` or defaults to production anchor-bot URL
