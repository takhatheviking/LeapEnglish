# LeapEnglish — Project Changelog

---

## Session: April 9, 2026

### 🏗️ Architecture Overhaul

#### Blind Spots Audit
Conducted a full structural audit of the app and identified key blind spots across Critical, Architecture, Content Scaling, and UX categories. Decided to tackle Architecture + Content Scaling + UX as priority.

#### content.json → Per-Level JSON Files
- **Removed** `content.json` as the single navigation source of truth
- **Replaced** with per-level files at `/levels/level_{CODE}.json`
- Level code map: `beginner→B1`, `elementary→EL`, `preintermediate→PI`, `intermediate→IN`
- Each lesson now has a `screens[]` array — makes lesson sequence data-driven, not hardcoded
- `updateLessonProgress()` now calculates against `lesson.screens.length` instead of hardcoded 3

#### manifest.json
- Created `/manifest.json` at repo root
- Tracks per-lesson content readiness: `{ "B1U1L1": { reading: true, exercises: false } }`
- App checks manifest before fetching — shows "Coming Soon" instead of 404
- Both `fetchReading()` and `fetchExercises()` are manifest-gated
- Manifest cached in memory after first fetch via `getManifest()`

#### Reading Fragments — Unified to GitHub Static
- Killed `TEST_MODE` flag entirely — single code path only
- Reading now always fetches from `https://takhatheviking.github.io/LeapEnglish/reading/{lessonId}.html`
- Script-tag sanitization applied before injecting HTML into DOM
- Error UI with retry button on fetch failure

#### REPO URL Fix
- Changed `const REPO` from `raw.githubusercontent.com/takhatheviking/LeapEnglish/main` to `https://takhatheviking.github.io/LeapEnglish`
- `raw.githubusercontent.com` silently 404s even on public repos in some contexts; GitHub Pages URL is reliable

---

### 📊 Progress Tracking

#### Migrated from n8n → Telegram CloudStorage
- **Removed** n8n `/get-progress` and `/track-progress` webhooks from app
- **Added** full CloudStorage layer: `csGet()`, `csSet()`, `getProgressData()`, `saveProgressData()`
- Storage key per level: `progress_{LEVEL}` → `{ completedLessons[], streak, lastSeen, lastLesson }`
- Language preference stored/restored: key `lang` → `"en"` or `"uz"`
- `setLang()` now persists to CloudStorage; `loadUserProgress()` restores it on boot
- Progress is source of truth; n8n is analytics only

#### Analytics Relay (Option A — Fire-and-Forget)
- Added `logAnalyticsEvent(event, score)` — fires after CloudStorage save, never blocks UX
- Posts to `n8n /log-event` with: `ts, userId, level, unitId, lessonId, event, score, lang`
- n8n appends row to **Analytics** sheet + upserts **Users** sheet `last_active`
- If n8n is down, user experience is completely unaffected

#### n8n: Log Analytics Event Workflow
- Created workflow `vuCVVQv7TpzWMZcN` — `POST /log-event`
- Two parallel writes: Analytics sheet (append) + Users sheet (upsert by telegram_id)
- Responds instantly (`onReceived`) — mini-app never waits
- **Manual step required:** Add `Analytics` tab to spreadsheet `1jTO4VOtsgyHR8kuy1jaIt5-GkoF_XQ6WbnvdgNDAB_k` with headers: `Timestamp | User ID | Level | Unit ID | Lesson ID | Event | Score | Lang`

---

### 🔄 Resume On Open
- Added `attemptResume()` — runs after `renderHome()` on boot
- Reads `data.lastLesson` from CloudStorage
- Shows native Telegram popup: "Continue where you left off?"
- On confirm: opens unit → opens lesson directly
- Best-effort: never blocks app if it fails

---

### 📁 Content Publishing Pipeline

#### n8n Publish Workflows (PAT hardcoded)
- **`LeapEnglish — Publish Reading`** (`gPo9qXsvPn9oc24U`) — `POST /publish-reading`
  - Input: `{ lessonId, htmlContent }`
  - Fetches manifest → checks if file exists → writes `/reading/{id}.html` → updates manifest
- **`LeapEnglish — Publish Exercise`** (`V6x2AxZY62Nr1n5k`) — `POST /publish-exercise`
  - Input: `{ lessonId, exercises[] }`
  - Same pattern → writes `/exercises/{id}.json` → updates manifest
- **`LeapEnglish — Convert + Publish Reading`** (`itaLnsmro9b23Thq`) — `POST /convert-and-publish-reading`
  - Input: `{ lessonId, level, pdfBase64 }` OR `{ lessonId, htmlContent }` (skip AI)
  - AI generates HTML from PDF → publishes to GitHub automatically
- **`LeapEnglish — Convert + Publish Exercise`** (`gqwGmphtOx7UFwfQ`) — `POST /convert-and-publish-exercise`
  - Input: `{ lessonId, level, topic, count }` OR `{ lessonId, exercises[] }` (skip AI)
  - AI generates exercise JSON → publishes to GitHub automatically
- All use GitHub PAT hardcoded in Code node (Fine-grained token, Contents: Read+Write on LeapEnglish repo)
- **Decision:** Publishing is manual — team lead controls publishing, content team submits to lead

#### Content Team Guide
- Created `LeapEnglish-Content-Team-Guide.html` — full guide for content creators
- Includes: pipeline overview, step-by-step for reading + exercise creation, Claude prompt templates, quality checklists, quick reference table
- Team uses Claude to generate content → submits to team lead → lead publishes via webhook
- n8n Convert+Publish workflows kept for lead's use only

---

### 📋 Level JSON Files — Full Structure

Created all 4 level files with real unit/lesson counts:

| File | Units | Total Lessons |
|---|---|---|
| `level_B1.json` | 14 | 153 |
| `level_EL.json` | 12 | 143 |
| `level_PI.json` | 12 | 130 |
| `level_IN.json` | 12 | 178 |

**Beginner (B1):** U1×9, U2×12, U3×12, U4×12, U5×11, U6×13, U7×10, U8×12, U9×11, U10×12, U11×9, U12×11, U13×10, U14×9

**Elementary (EL):** U1×15, U2×12, U3×12, U4×10, U5×12, U6×12, U7×12, U8×11, U9×11, U10×11, U11×13, U12×12

**Pre-Intermediate (PI):** U1×14, U2×11, U3×13, U4×12, U5×7, U6×12, U7×12, U8×8, U9×11, U10×12, U11×12, U12×6

**Intermediate (IN):** U1×14, U2×11, U3×19, U4×19, U5×15, U6×13, U7×10, U8×16, U9×16, U10×14, U11×16, U12×15

---

### 🐛 Bugs Fixed

#### SyntaxError: Uzbek Apostrophe (Critical)
- **Bug:** `'Yo'q'` inside single-quoted JS string crashed entire app silently on startup
- **Fix:** Changed to `"Yo'q"` using double quotes
- **Rule:** All Uzbek text with apostrophes in JS must use double-quoted strings

#### REPO URL (Critical)
- **Bug:** `raw.githubusercontent.com` URL was used — doesn't reliably serve files; also blocked in some network contexts
- **Fix:** Changed to `https://takhatheviking.github.io/LeapEnglish` (GitHub Pages CDN)

#### GitHub Pages 404
- **Cause:** No `index.html` at repo root — GitHub Pages requires it
- **Fix:** Added minimal `index.html` placeholder

#### Level Switching (Pending Fix)
- All levels show Beginner because Telegram Bot doesn't pass `start_param` when opening mini-app
- **Partial fix:** miniapp.html now reads level from `?level=` URL query param as fallback
- **Remaining:** Telegram Bot "Send Mini-App Button" node needs URL updated to include `?level={{ extracted_level }}` — must be done manually in n8n UI (Telegram trigger = no MCP access)

---

### 📌 Known Limitations & Technical Debt

| Item | Status | Notes |
|---|---|---|
| Level switching via bot | ⚠️ Partial | Bot needs `?level=` added to mini-app URL |
| Publish workflows — n8n Code node HTTP | ⚠️ Untested | `$http` and `fetch` not available in n8n Code node sandbox — needs HTTP Request nodes instead. Workflows built but not end-to-end tested |
| GitHub PAT hardcoded in n8n | ⚠️ Tech debt | PAT in Code node string — should move to n8n Variables when available on plan |
| Converter for content team | 🔜 Pending | Team needs tool to upload PDF/image → get LeapEnglish HTML output |
| Cross-device user identity | 🔜 Pending | CloudStorage is per-device; no recovery if user switches device |
| Exercise JSON schema validation | 🔜 Pending | No validation before publish |
| Content versioning | 🔜 Pending | No draft/published states |

---

### 📂 GitHub Repo Structure (current state)

```
LeapEnglish/
├── index.html              ← placeholder for GitHub Pages
├── miniapp.html            ← main app (fully patched)
├── manifest.json           ← content readiness flags
├── levels/
│   ├── level_B1.json       ← 14 units, 153 lessons
│   ├── level_EL.json       ← 12 units, 143 lessons
│   ├── level_PI.json       ← 12 units, 130 lessons
│   └── level_IN.json       ← 12 units, 178 lessons
├── videos/
│   └── beginner.json       ← Cloudflare Stream UIDs (B1U1L1-L9 live)
├── reading/                ← HTML fragments (empty, ready for content)
└── exercises/              ← JSON arrays (empty, ready for content)
```

---

### 🔧 n8n Workflows — Current State

| Workflow | ID | Status | Purpose |
|---|---|---|---|
| LeapEnglish — Telegram Bot | g44uvekR1sfnHJih | ✅ Active | Bot trigger, level selection, open mini-app |
| LeapEnglish — Log Analytics Event | vuCVVQv7TpzWMZcN | ✅ Active | Append to Analytics + Users sheets |
| LeapEnglish — Publish Reading | gPo9qXsvPn9oc24U | ✅ Active | Write reading HTML to GitHub |
| LeapEnglish — Publish Exercise | V6x2AxZY62Nr1n5k | ✅ Active | Write exercise JSON to GitHub |
| LeapEnglish — Convert + Publish Reading | itaLnsmro9b23Thq | ✅ Active | PDF→AI→HTML→GitHub |
| LeapEnglish — Convert + Publish Exercise | gqwGmphtOx7UFwfQ | ✅ Active | Topic→AI→JSON→GitHub |
| LeapEnglish — convert-reading | eKbSoz7f0hpmctwa | ✅ Active | Legacy — PDF→HTML only, no publish |
| LeapEnglish — Cloudflare Stream Sync | JbT9QOcN4ia6inIx | ⏸ Inactive | Sync video UIDs to VideoMap sheet |
| LeapEnglish — 2. Reading Content Fetcher | o2UsKoagpbX6yAhU | ⏸ Inactive | Legacy — now replaced by static GitHub |
| LeapEnglish — 3. Progress Tracker | sSDGdxBsibXq0Z4H | ✅ Active | Legacy — can be deactivated (CloudStorage replaces) |
| LeapEnglish — 4. Progress Fetcher | h1e1YasDjb31VSUO | ✅ Active | Legacy — can be deactivated (CloudStorage replaces) |

---

### ⏭️ Next Session Priorities

1. **Fix level switching** — update Bot's "Send Mini-App Button" to pass `?level=` in URL
2. **Test publish pipeline** — end-to-end test of Publish Reading/Exercise workflows with real content
3. **Update converter tool** — team needs PDF/image → LeapEnglish HTML converter (Claude-powered, standalone HTML tool)
4. **Deactivate legacy n8n workflows** — Progress Tracker, Progress Fetcher, Reading Fetcher are now dead code
5. **Add unit/lesson titles** — level JSON files have placeholder titles ("Unit 1", "Lesson 1") — should be populated with real curriculum names
