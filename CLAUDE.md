# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

AgentTube (package name `youtube-automation-agent`, also branded "Lumen" in places): a Node.js 18+ (CommonJS) agent that runs a YouTube channel end to end — research → script → narration/visuals → video assembly → SEO metadata → human review → schedule → publish → learn from analytics and comments. Single Express server plus a static vanilla-JS dashboard; state in SQLite.

## Commands

```bash
npm install            # also fetches bundled FFmpeg (ffmpeg-static, optional dep)
npm test               # node test.js — full system suite, no credentials needed
npm run lint           # eslint . (flat config, eslint.config.js)
npm start              # node index.js — dashboard/API on http://localhost:3456 (PORT)
npm run setup          # interactive credential setup (setup.js)
npm run credentials:setup
npm run scheduler      # run schedules/daily-automation.js standalone
npm run agent:<name>   # run one agent standalone: strategy, script, thumbnail, seo, production, publishing, analytics
```

CI (`.github/workflows/ci.yml`) runs `npm ci && npm run lint && npm test` on Node 20.

**Tests:** there is no test framework. `test.js` is a single `SystemTest` class; `runAllTests()` holds an ordered list of `{ name, test }` entries and each `testXxx()` method throws on failure. To run one test:

```bash
node -e "new (require('./test').SystemTest)().testPublishingSafety().then(() => console.log('PASS'), e => { console.error(e); process.exit(1); })"
```

New tests: add a `testXxx()` method and register it in the `tests` array in `runAllTests()`. Bug fixes should come with a regression test there (per CONTRIBUTING.md). Tests use the real SQLite file at `data/youtube_automation.db` and stub agents/providers by injecting fakes (see `testSceneRepairStudio` / `testNarrationReliability` for the pattern).

## Architecture

**`index.js` — `YouTubeAutomationAgent`** is the orchestrator and the whole HTTP API. `initialize()` opens the DB, constructs the service layer, validates credentials, then instantiates the seven agents and the cron scheduler. If credentials are invalid it starts in **setup mode**: dashboard served, generation/publishing disabled. Mutating routes go through `requireAPIKey()` / `protect`, which checks the `x-api-key` header only when `API_KEY` is set.

**Generation pipeline** (`startGenerationJob` → `runGenerationJob` → `generateContent`): fixed stages `strategy → script → thumbnail → seo → production → quality_review` (`GENERATION_STAGES` in `utils/generation-recovery-service.js`). Each stage runs through `runGenerationStage`, which delegates to `GenerationRecoveryService` — stage outputs are checkpointed in `generation_checkpoints`, so a resumed job reuses verified artifacts instead of regenerating. Jobs run one at a time by default (`MAX_CONCURRENT_JOBS`); interrupted jobs are marked on startup (`db.markInterruptedJobs`).

**Agents** (`agents/*.js`): one class per pipeline role, each constructed with `(db, credentials)` and an `initialize()`. `ProductionManagementAgent` is the heavy one — narration (TTS), visuals, and video assembly via `utils/ai-video-generator.js`, `utils/media-generation-service.js`, `utils/ffmpeg.js`. `PublishingSchedulingAgent` owns YouTube upload/scheduling; `AnalyticsOptimizationAgent` reads YouTube Analytics.

**Services** (`utils/*-service.js`, `utils/*-engine.js`): feature "studios" layered on the pipeline, each taking `db` plus injected dependencies (which is what makes them testable with fakes). Notable: `production-readiness-service` (pre-publish gate), `provenance-service` (factual claims & media rights), `scene-repair-service` (per-scene regenerate/narration repair), `shorts-repurposing-service`, `audience-engagement-service` (comment sync, reply drafts, daily reply cap), `growth-experiment-service` (title/thumbnail A/B arms), `autonomous-channel-operator` + `operator-service` (strategy-driven runs, notifications), `channel-learning-engine`, `discoverability-service` + `utils/discoverability-adapters/darkzseo.js` (bundled JS audit by default; external Python DarkzSEO only if `DARKZSEO_PATH` is set, falling back to bundled on failure).

**Provider abstraction:**
- Text: `utils/ai-text-service.js` — any one of OpenAI, OpenRouter, Gemini, Moonshot/Kimi, MiMo, GLM (OpenAI-compatible base URLs, Gemini via `@google/genai`).
- Video: `utils/video-providers.js` — `VideoProviderRegistry` tries `VIDEO_PROVIDER_ORDER` (Seedance/Replicate, MiniMax, Google Omni, Kling, Wan) and always falls back to the local FFmpeg `slideshow` provider.
- TTS: OpenAI, ElevenLabs, Gemini, or Azure.
- Missing capabilities are **simulated** rather than crashing; `logCapabilitySummary()` prints the `🔎 Capability check:` block showing what is real vs. simulated.

**Safety gates (do not weaken):** publishing is blocked for simulated video, missing narration, unresolved factual claims, and unconfirmed media rights. Quality review sets productions to `needs_review` / `needs_attention`; auto-approval happens only when the `approval_required` setting is `'false'`. Covered by `testPublishingSafety`, `testProductionReadinessGate`, `testPlaceholderSchedulingGuard`.

**Database** (`database/db.js`): one `Database` class wrapping `sqlite3` at `data/youtube_automation.db`, with all tables created via `CREATE TABLE IF NOT EXISTS` in `initialize()`. Schema changes to existing tables go through `ensureColumns(table, {col: definition})` (additive `ALTER TABLE` migration) — add new columns there rather than only editing the `CREATE TABLE`.

**Scheduler** (`schedules/daily-automation.js`): `DailyAutomation` registers `node-cron` jobs (daily generation, 15-min publish queue checks, analytics, engagement sync, growth-experiment refresh). It receives callbacks from `index.js` (e.g. `generateContent` → `queueScheduledContent`) rather than calling the orchestrator directly.

**Credentials:** `utils/credential-manager.js` merges `.env` and `config/credentials.json` (template: `config/credentials.example.json`); YouTube OAuth tokens live in `config/tokens.json`. YouTube OAuth requires Desktop-app credentials using a dynamic loopback redirect (`oauth-server.js`, `modern-auth.js`). All of these are gitignored.

**Dashboard** (`dashboard/`): static `index.html` + `app.js` + `enhance.js` served by Express; talks to the `/api/*` routes.

Generated media goes under `data/{videos,audio,thumbnails,scripts,captions,assets}/` (gitignored, lint-ignored).

## Contribution conventions (from CONTRIBUTING.md)

- One concern per PR; don't regenerate `package-lock.json` unless the PR is about dependencies.
- Unused vars/args must be prefixed with `_` to silence the lint warning.
