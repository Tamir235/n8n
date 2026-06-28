# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repo Is

A collection of **n8n workflow JSON files** exported from n8n Cloud (`https://n8nconsutinginternational.app.n8n.cloud`). All "code" is n8n workflow JSON — there is no build system, no package manager, and no tests. Development means editing these JSON files locally and importing them into n8n Cloud via the UI or API.

GitHub repo: `https://github.com/Tamir235/n8n`

## Workflow Architecture

**Crypto and Cooking are two completely independent projects.** They do not share sub-workflows, triggers, or data flows. Both happen to use the same external services (S3/R2, ElevenLabs, Shotstack, Telegram) but with separate credentials, separate S3 prefixes, and separate n8n workflow IDs. Never wire them together.

### Crypto Reel System (production, runs daily 07:30 Berlin time)

| File | n8n ID | Role |
|---|---|---|
| `Reels Master.json` | `XvwKX49S493TsBlH` | Master orchestrator — generates `scriptId`, calls sub-workflows in sequence, sends social captions in parallel |
| `News Intake.json` | `pT5wAtJd57wCEBhv` | Fetches RSS from CoinDesk + others, parses XML, produces clip array |
| `Voice and Image Generator.json` | `1nuI2fuS22Cq5w81` | Two independent `SplitInBatches` loops (audio + image) run sequentially; merges results by `clipIndex` |
| `Video rendering.json` | `VeXgfkihC7JYQr26` | Builds SRT, uploads to S3, submits Shotstack render, polls for completion, delivers via Telegram + Google Drive |

### Cooking Video System (in development)

| File | n8n ID | Role |
|---|---|---|
| `Cooking — AI Food Studio Test.json` | `Qzd8tZyFb4rVPOyu` | Test workflow — AI Director agent (GPT-4.1) converts recipe → full production plan with Kling AI prompts |
| `Cooking — Recipe Intake.json` | `B2BrjjbqFhUgeHZV` | **A1 ✅** — Dual-mode intake: Telegram URL scrape + Spoonacular API; outputs normalized recipe + `scriptId` |

### The `scriptId` — the correlation key

- **Crypto:** `reel_YYYY-MM-DDTHH-MM-SS-SSSZ` (Berlin timezone), created in Reels Master
- **Cooking:** `cooking_YYYY-MM-DDTHH-MM-SS-SSSZ` (Berlin timezone), created in Recipe Intake (A1)

Every downstream sub-workflow receives and passes this forward. S3 assets are namespaced under it:

```
reels/{scriptId}/audio/clip_NN_{type}__{hash}.mp3      ← crypto
reels/{scriptId}/image/clip_NN_{type}__{hash}.png      ← crypto
reels/{scriptId}/srt/full_transcript_{timestamp}.srt   ← crypto

cooking/{scriptId}/audio/clip_NN_{type}.mp3            ← cooking
cooking/{scriptId}/image/clip_NN_{type}.png            ← cooking
cooking/{scriptId}/video/clip_NN_{type}.mp4            ← cooking (Kling AI output)
cooking/{scriptId}/srt/full_transcript_{timestamp}.srt ← cooking
```

Hash is SHA-256(12 chars) of the content that determines the file — allows idempotent re-runs without collisions.

## Infrastructure Constants

| Resource | Value |
|---|---|
| S3 bucket | `reels-voiceovers` (Cloudflare R2) |
| CDN base URL | `https://pub-b9cab7c4281d4ca4948989323fa68b5d.r2.dev/` |
| Telegram Chat ID | `-1003735970138` |
| ElevenLabs Voice ID | `TUKJhQmz3RPYBNAgC5A1` (German voice, set in `Voice ID` node) |
| OpenAI image model | `gpt-image-1`, size `1024x1536`, format `png`, quality `medium` |
| ElevenLabs model | `eleven_multilingual_v2`, stability `0.6`, similarity `0.8` |
| Shotstack output | `1080×1920 mp4` (vertical/Reels format) |
| Clip count (crypto) | Always exactly 6 (padded with `outro` clips if fewer arrive) |
| Clip count (cooking) | Dynamic — 8–12 clips targeting 60–90s total; no hard cap |
| Kling AI model | `kling-v1-6`, aspect ratio `9:16`, duration 5s or 10s per clip |

## Development Tree

The two tracks are **independent** — Crypto Cleanup (B-series) and Cooking (A-series) can progress in parallel. There is no dependency between them.

```
── Crypto Track ──────────────────────────────────────────────────
B1  ✅ RSS Filter Dedup (5 nodes → 1 parametriert)   GitHub #1
B2  ✅ Bug Fixes (Shotstack URL, dead node, Voice ID)  GitHub #2
B3  ⏳ Audio + Image Loops parallelisieren             GitHub #3

── Cooking Track ─────────────────────────────────────────────────
A1  ✅ Recipe Intake (ID: B2BrjjbqFhUgeHZV)           GitHub #4
A2  ⏳ Drehbuch-Generator                              GitHub #5  ← A1 done → unblocked
A3  ⏳ Cooking Voice + Image Generator                 GitHub #6  ← needs A2
A4  ⏳ Kling AI Animator (Image → MP4)                 GitHub #7  ← needs A3
A5  ⏳ Cooking Video Renderer                          GitHub #8  ← needs A3 + A4
A6  ⏳ Cooking Master Orchestrator + Social Delivery   GitHub #9  ← needs A5
```

### Cooking dependencies
- **A1 → A2**: Drehbuch braucht normalisiertes Rezept-Objekt + `scriptId` von A1
- **A2 → A3**: Voice/Image braucht `clips[]` mit `voiceover_text` + `kling_prompt` von A2
- **A3 → A4**: Kling AI Animator braucht `image_url` pro Clip von A3
- **A3 + A4 → A5**: Renderer braucht `audio_url` (A3) + `video_url` (A4) pro Clip
- **A5 → A6**: Orchestrator kann erst verdrahtet werden wenn alle Sub-Workflows stehen

### Test-Workflow → A2 Mapping

`Cooking — AI Food Studio Test.json` ist der Proof-of-Concept für A2. Für die Produktion muss er:
1. `manualTrigger` + `Sample Recipe` Set-Node **ersetzen** durch `executeWorkflowTrigger` (passthrough)
2. Input: `recipe_name` + `recipe_text` kommen von A1 statt hardcoded Carbonara
3. Output: das geparste JSON direkt als Workflow-Output zurückgeben (kein `Parse Production Plan` nötig wenn Agent sauber JSON liefert — Regex-Fallback bleibt)
4. Workflow-ID in A6 eintragen

### Open issues
**B3 (GitHub #3):** Zwei sequentielle `SplitInBatches`-Loops in `Voice and Image Generator.json` → parallel via Split → zwei Branches → Merge by `clipIndex`.

**A2 — duration math (known):** Agent gibt falsches `total_duration_seconds` aus (schätzt statt summiert). Fix: Agent anweisen, Clip-Dauern explizit zu summieren. Im A2-Issue-Spec bereits vorgesehen.

**A2 — completeness gate (known):** Agent überspringt ggf. komplexe Rezeptschritte. Fix: Pflicht-Check ob alle Hauptschritte in clips[] abgedeckt sind vor JSON-Output.

**A1 — Spoonacular Key:** Placeholder `SPOONACULAR_API_KEY_PLACEHOLDER` in `Cooking — Recipe Intake.json` — vor Prod-Einsatz in n8n Credentials ersetzen.

## Video Rendering Logic

The `build srt file` node in `Video rendering.json` generates a `.srt` caption file at 3 words per segment, with timing derived from actual audio `duration` values (measured by ElevenLabs `alignment.character_end_times_seconds`). If `duration` is missing it falls back to a `calcDuration()` formula (words/2.35 + linguistic complexity heuristics).

Shotstack polling: initial 30 s wait → check status → if not `done`, wait 10 s and retry. Delivery runs in parallel to Telegram and Google Drive.

## Intro/Outro Config (hardcoded in `Intro/Outro` set-node)

```
intro_length: 7.5s   |  outro_length: 8.5s
captions_enabled: true
```
Intro/Outro assets live at `reels/Intro/` and `reels/Outro/` in the CDN.

## AI Food Studio Director (Cooking)

The `Cooking — AI Food Studio Test.json` agent uses a locked character ("Emma") and kitchen (Scandinavian) that must never change between clips. It produces a JSON production plan with `kling_prompt` per clip following a strict 13-block prompt architecture. Output is parsed by a `Parse Production Plan` code node that extracts the JSON object with a regex fallback.

## Session Log

| Datum | Was |
|---|---|
| 2026-06-28 | CLAUDE.md erstellt; Entwicklungsbaum (B+A-Series) mit GitHub Issues #1–#9 abgeglichen; Crypto ≠ Cooking Trennung klargestellt; Test-Workflow → A2 Mapping dokumentiert; B1/B2/A1 als ✅ bestätigt; A2 als nächster Schritt unblocked |

## How to Deploy Changes

1. Edit the JSON file locally.
2. In n8n Cloud UI: open the workflow → ⋮ menu → **Import from file** (overwrites existing nodes).
3. Alternatively use the n8n REST API: `PATCH /api/v1/workflows/{id}` with the updated JSON body.
4. All active workflows must be **deactivated before import** and re-activated after.
