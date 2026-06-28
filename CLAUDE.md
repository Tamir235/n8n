# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repo Is

A collection of **n8n workflow JSON files** exported from n8n Cloud (`https://n8nconsutinginternational.app.n8n.cloud`). All "code" is n8n workflow JSON — there is no build system, no package manager, and no tests. Development means editing these JSON files locally and deploying to n8n Cloud via the REST API or UI import.

- GitHub repo: `https://github.com/Tamir235/n8n`
- n8n API base: `https://n8nconsutinginternational.app.n8n.cloud/api/v1`
- n8n API auth header: `X-N8N-API-KEY: <key from n8n Settings → API Keys>`

**Note:** The many subdirectories in this repo (`brandkit/`, `caveman/`, `skills/`, `.agents/`, etc.) are Claude Code skill files — they are not n8n workflows and should be ignored when working on the automation pipelines.

## How to Deploy Changes

```bash
# Get a workflow (find ID in table below)
curl -H "X-N8N-API-KEY: $N8N_KEY" https://n8nconsutinginternational.app.n8n.cloud/api/v1/workflows/{id}

# Update a workflow (PUT replaces entirely — always GET first)
curl -X PUT -H "X-N8N-API-KEY: $N8N_KEY" -H "Content-Type: application/json" \
  -d @"workflow.json" \
  https://n8nconsutinginternational.app.n8n.cloud/api/v1/workflows/{id}

# Create a new workflow
curl -X POST -H "X-N8N-API-KEY: $N8N_KEY" -H "Content-Type: application/json" \
  -d @"new-workflow.json" \
  https://n8nconsutinginternational.app.n8n.cloud/api/v1/workflows

# List all workflows
curl -H "X-N8N-API-KEY: $N8N_KEY" \
  https://n8nconsutinginternational.app.n8n.cloud/api/v1/workflows?limit=50
```

Via UI: open workflow → ⋮ menu → **Import from file**. Deactivate before import, re-activate after.

## Workflow Architecture

**Crypto and Cooking are two completely independent projects.** They do not share sub-workflows, triggers, or data flows. Both use the same external services (S3/R2, ElevenLabs, Shotstack, Telegram) but with separate S3 prefixes and separate n8n workflow IDs. Never wire them together.

### Crypto Reel System (production, runs daily 07:30 Berlin time)

| File | n8n ID | Role |
|---|---|---|
| `Reels Master.json` | `XvwKX49S493TsBlH` | Master orchestrator — generates `scriptId`, calls sub-workflows in sequence, sends social captions in parallel |
| `News Intake.json` | `pT5wAtJd57wCEBhv` | Fetches RSS from 5 sources, parses XML, scores/clusters articles, runs GPT agents, outputs `clips[]` |
| `Voice and Image Generator.json` | `1nuI2fuS22Cq5w81` | Two `SplitInBatches` loops (audio + image) run sequentially; merges results by `clipIndex` |
| `Video rendering.json` | `VeXgfkihC7JYQr26` | Builds SRT, uploads to S3, submits Shotstack render, polls for completion, delivers via Telegram + Google Drive |

### Cooking Video System (in development)

| File | n8n ID | Role |
|---|---|---|
| `Cooking — AI Food Studio Test.json` | `Qzd8tZyFb4rVPOyu` | Test/PoC — AI Director agent (GPT-4.1) converts recipe → cinematic production plan with Kling AI prompts |
| `Cooking — Recipe Intake.json` | `B2BrjjbqFhUgeHZV` | **A1 ✅** — Dual-mode: Telegram URL scrape or Spoonacular API → normalized recipe + `scriptId` |
| `Cooking — Drehbuch Generator.json` | `9QSE0w8qgSWdlzSK` | **A2 ✅** — AI screenplay generator: recipe → full cinematic production plan (AI Food Studio v1.0, 8–12 clips, 60–90s) |
| `Cooking — Voice and Image Generator.json` | `M6FzHQY3YG7zxjjC` | **A3 ✅** — Parallel audio (ElevenLabs) + image (OpenAI gpt-image-1 high) loops for all clips; merges by clipIndex |

### The `scriptId` — correlation key across all sub-workflows

- **Crypto:** `reel_YYYY-MM-DDTHH-MM-SS-SSSZ` (Berlin timezone), created in Reels Master
- **Cooking:** `cooking_YYYY-MM-DDTHH-MM-SS-SSSZ` (Berlin timezone), created in Recipe Intake (A1)

S3/R2 asset paths are namespaced under it:
```
reels/{scriptId}/audio/clip_NN_{type}__{hash}.mp3     ← crypto
reels/{scriptId}/image/clip_NN_{type}__{hash}.png     ← crypto
reels/{scriptId}/srt/full_transcript_{ts}.srt         ← crypto

cooking/{scriptId}/audio/clip_NN_{type}.mp3           ← cooking
cooking/{scriptId}/image/clip_NN_{type}.png           ← cooking
cooking/{scriptId}/video/clip_NN_{type}.mp4           ← cooking (Kling AI output)
cooking/{scriptId}/srt/full_transcript_{ts}.srt       ← cooking
```

The crypto hash (SHA-256, 12 chars) in filenames is derived from content — enables idempotent re-runs.

## Inter-Workflow Data Contract

Each sub-workflow is called via `executeWorkflow` with `waitForSubWorkflow: true`. The data shape that flows between them:

**A1 → A2 (Recipe Intake → Drehbuch Generator):**
```json
{
  "scriptId": "cooking_2026-06-28T07-30-00-000Z",
  "recipe": {
    "name": "Beef Wellington",
    "servings": 2,
    "prepTime": "30 min",
    "cookTime": "45 min",
    "ingredients": ["500g Rinderfilet", "..."],
    "steps": ["Fleisch würzen", "..."]
  }
}
```

**A2 → A3 (Drehbuch → Voice + Image Generator):**
```json
{
  "scriptId": "cooking_...",
  "recipe_name": "Beef Wellington",
  "clips": [
    {
      "clipIndex": 0,
      "act": "hook",
      "duration_seconds": 5,
      "voiceover_text": "Dieses saftige Beef Wellington...",
      "kling_prompt": "Emma from Project Bible. Scandinavian Kitchen...",
      "camera": "close-up",
      "lens": "85mm",
      "continuity_note": "...",
      "transition_out": "..."
    }
  ]
}
```

**A3 → A4 (Voice+Image → Kling Animator):** adds `audio_url` and `image_url` per clip, preserves `kling_prompt`.

**A4 → A5 (Kling → Renderer):** adds `video_url` per clip (Kling `.mp4`), preserves `audio_url`.

**Crypto clip shape** (News Intake → Voice and Image Generator):
```json
{
  "scriptId": "reel_...",
  "clips": [
    { "type": "market_overview", "voiceover": "...", "image_prompt": "..." }
  ]
}
```

## Infrastructure Constants

| Resource | Value |
|---|---|
| S3 bucket | `reels-voiceovers` (Cloudflare R2) |
| CDN base URL | `https://pub-b9cab7c4281d4ca4948989323fa68b5d.r2.dev/` |
| Telegram Chat ID | `-1003735970138` |
| ElevenLabs Voice ID | `TUKJhQmz3RPYBNAgC5A1` (German voice, set in `Voice ID` Set node) |
| ElevenLabs model | `eleven_multilingual_v2`, stability `0.6`, similarity `0.8` |
| OpenAI image model (crypto) | `gpt-image-1`, size `1024x1536`, format `png`, quality `medium` |
| OpenAI image model (cooking) | `gpt-image-1`, size `1024x1536`, format `png`, quality `high` |
| Shotstack output | `1080×1920 mp4` (vertical/Reels format) |
| Clip count (crypto) | Always exactly 6 (padded with `outro` clips if fewer arrive) |
| Clip count (cooking) | Dynamic — 8–12 clips targeting 60–90s total; no hard cap |
| Kling AI model | `kling-v1-6`, aspect ratio `9:16`, duration 5s or 10s per clip |

## Development Tree

The two tracks are **independent** — B-series and A-series can run in parallel.

```
── Crypto Track ──────────────────────────────────────────────────
B1  ✅ RSS Filter Dedup (5 nodes → 1 unified, only sourceName differs)  GitHub #1
B2  ✅ Bug Fixes: Shotstack URL /edit/, dead node, Voice ID placeholder  GitHub #2
B3  ⏳ Parallelize audio + image SplitInBatches loops                   GitHub #3

── Cooking Track ─────────────────────────────────────────────────
A1  ✅ Recipe Intake (ID: B2BrjjbqFhUgeHZV)          GitHub #4
A2  ✅ Drehbuch Generator  ID: 9QSE0w8qgSWdlzSK       GitHub #5
A3  ✅ Cooking Voice + Image Generator (ID: M6FzHQY3YG7zxjjC)  GitHub #6
A4  ⏳ Kling AI Animator (image → 5-10s MP4)          GitHub #7  ← needs A3
A5  ⏳ Cooking Video Renderer (Shotstack assembly)    GitHub #8  ← needs A4
A6  ⏳ Cooking Master Orchestrator + Social Delivery  GitHub #9  ← needs A5
```

## Key Implementation Details

### Video Rendering (crypto)
`build srt file` generates `.srt` at 3 words/segment. Timing comes from ElevenLabs `alignment.character_end_times_seconds`. Falls back to `calcDuration()` (words/2.35 + linguistic complexity heuristics) if duration is missing. Shotstack polling: 30s initial wait → check → 10s retry loop. Delivery to Telegram and Google Drive runs in parallel.

### AI Food Studio Director (cooking A2)
The Drehbuch Generator agent uses a locked character (Emma, 28, European, white cotton shirt, beige linen apron) and a fixed Scandinavian kitchen that must be identical across all clips. Every Kling AI prompt follows a strict 13-block architecture: Project Bible reference → Scene → Food Details → Hand Movement → Camera → Lens → Focus → Light → Atmosphere → Video Style → Continuity → Transition → Negative Prompt.

Known agent output bugs (both fixed in A2 spec):
- `total_duration_seconds` is estimated, not summed — must be computed as `Σ clip.duration_seconds`
- Complex recipes may skip steps — a completeness gate must verify all recipe steps appear in `clips[]`

### Intro/Outro (crypto, hardcoded in `Intro/Outro` Set node)
```
intro_length: 7.5s  |  outro_length: 8.5s  |  captions_enabled: true
```
Assets at `reels/Intro/` and `reels/Outro/` in the CDN.

## Open Issues

| Issue | Detail |
|---|---|
| **B3** | Parallelize two sequential `SplitInBatches` loops in Voice and Image Generator — split before, merge after by `clipIndex` |
| **A1** | `SPOONACULAR_API_KEY_PLACEHOLDER` in Recipe Intake — replace with real credential before production |

## Session Log

| Date | Changes |
|---|---|
| 2026-06-28 | Initial build session: B1+B2 crypto bugs fixed; A1 Recipe Intake deployed; A2 Drehbuch Generator in progress; GitHub issues #1–#9 created; CLAUDE.md created and maintained |
| 2026-06-28 | A2 Drehbuch Generator deployed (ID: 9QSE0w8qgSWdlzSK) — 5-node workflow: executeWorkflowTrigger → Code (recipe format) → Agent (AI Food Studio v1.0, gpt-4.1) → Code (parse & validate); JSON saved to desktop |
| 2026-06-28 | A3 Cooking — Voice and Image Generator deployed (ID: M6FzHQY3YG7zxjjC) — 24-node workflow: parallel audio (ElevenLabs TTS + S3) and image (gpt-image-1 high quality + S3) loops, both starting from Prepare Clips, merged by clipIndex; JSON saved to desktop |
