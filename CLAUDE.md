# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Running the App

No build step — open `index.html` directly in a browser, or serve it with any static file server:

```
npx serve .
# or
python -m http.server 8080
```

There are no tests, no linter, and no package.json.

## Architecture

This is a **single-file vanilla JS PWA**. All application logic lives in `index.html` (~1850 lines: HTML, `<style>`, and `<script>`). The only external JS file is `foods.js`, which exports `window.FOOD_DB` — a static array of ~300 food items with nutritional data per 100g.

**External CDNs loaded at runtime:**
- Chart.js 4.4.0 (trend charts)
- Firebase compat SDK 10.14.1 (auth + Firestore)

### State Model

All runtime state lives in a single `store` object:

```js
store = {
  profile: { name, gender, birthDate, height, targetWeight, goal, weight, activity },
  weightHistory: [{ date, weight }, ...],
  days: { "YYYY-MM-DD": { foods: [], exercises: [] } },
  apiKey: ""  // optional user-supplied GLM key; falls back to BUILTIN_API_KEY
}
```

`store` is read from `localStorage` on load (`STORAGE_KEY = "diet_tracker_v3"`), with a migration path from the legacy `"diet_tracker_v2"` key. Every mutation calls `saveStore()`, which writes to `localStorage` synchronously and then debounces a Firestore write (1.5s delay).

### Data Flow

1. User action → mutate `store.days[currentDate]` or `store.profile`
2. Call `saveStore()` → `localStorage` + deferred Firestore sync
3. Call `renderAll()` (or targeted render fn) → DOM update

There is no reactive framework; all rendering is imperative DOM manipulation via `$(id)` = `document.getElementById(id)`.

### Key Calculations

- **BMR**: Mifflin-St Jeor formula in `calcTargets()` (line ~768)
- **TDEE**: `BMR × activity`
- **Goal target**: `TDEE - 500` (cut) / `TDEE` (maintain) / `TDEE + 300` (bulk)
- **Effective daily target**: `goal target + today's exercise burn` — exercise calories are added back so the target scales with activity
- **Exercise burn**: `MET × weight_kg × minutes / 60`
- **Macro targets**: protein = 1.5g/kg, fat = 0.8g/kg, carbs = remaining calories ÷ 4

### AI Integration

Two GLM (智谱) API calls, both hitting `https://open.bigmodel.cn/api/paas/v4/chat/completions`:

| Function | Model | Input | Output |
|---|---|---|---|
| `callGLM()` | `glm-4-flash` | Natural-language food description | JSON array of food items |
| `callGLMVision()` | `glm-4v-flash` | Base64 nutrition label image | Single food object (per-100g values) |

The system prompt in `SYSTEM_PROMPT` (line ~1381) contains detailed rules for ingredient decomposition and is critical to AI parse quality — edit carefully.

`BUILTIN_API_KEY` is a pre-filled GLM key embedded in the source. Users can override it via the AI settings modal, which saves to `store.apiKey`.

### Firebase / Cloud Sync

- Firebase project: `wxxnutritiontracker`
- Firestore document path: `users/{uid}` — entire `store` (minus the API key being additive) is stored as one document
- Auth: Google Sign-In via popup
- On first login, local data is migrated to Firestore; subsequent logins load cloud data and overwrite local
- Firestore offline persistence is enabled (`enablePersistence({ synchronizeTabs: true })`)

### PWA

`manifest.json` + `icons/icon.svg` enable "Add to Home Screen". An install-tip banner (`#pwa-install-tip`) is shown to non-PWA sessions unless dismissed (tracked via `localStorage('pwa-tip-dismissed')`).
