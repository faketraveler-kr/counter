# CLAUDE.md — Web Counter PWA

## Project Overview

This is a **Korean-language Progressive Web App (PWA)** that functions as a tap counter. The app counts taps/clicks and visually transitions its background color from purple to deep red as the count increases. It is installable on iOS and Android home screens.

**Current version:** v1.1.0
**Language:** Korean (UI labels and code comments)
**Stack:** Vanilla HTML/CSS/JavaScript — no build tools, no frameworks, no dependencies

---

## Repository Structure

```
counter/
├── index.html       # Entire application: markup, CSS, and JavaScript in one file
├── manifest.json    # PWA web app manifest (name, icons, display mode, theme color)
├── sw.js            # Service worker — Network First caching strategy
├── icons/           # PNG app icons in 8 sizes (72–512 px) for PWA installation
│   ├── icon-72x72.png
│   ├── icon-96x96.png
│   ├── icon-128x128.png
│   ├── icon-144x144.png
│   ├── icon-152x152.png
│   ├── icon-192x192.png
│   ├── icon-384x384.png
│   └── icon-512x512.png
├── .gitignore       # Ignores OS files, IDE files, node_modules, dist, .env, etc.
└── CLAUDE.md        # This file
```

There is **no package.json**, no bundler, and no test suite. Everything runs directly in the browser.

---

## Core Application Logic (`index.html`)

All application code lives inside a single `<script>` block in `index.html`.

### Counter Behavior

- `count` (integer, starts at 0) is the sole piece of state.
- Taps anywhere on the screen (excluding the bottom 10% and the reset button) increment `count`.
- Each increment triggers:
  1. `counterElement.textContent` updated
  2. `updateColor()` — background color shift
  3. `addFlashEffect()` — 300 ms white flash animation on `<body>`

### Color Progression (`updateColor`)

| Count range | Behavior |
|-------------|----------|
| 0–100       | Hue shifts from 306° (purple) → 0° (red), saturation 60%, lightness 75% |
| 100–200     | Hue stays at 0°, saturation increases 60%→100%, lightness decreases 75%→50% |
| 200+        | Fixed deep red: `hsl(0, 100%, 50%)` |

The CSS custom property `--current-color` is kept in sync with the current background for use in the `@keyframes flash` animation.

### Touch Handling (`handleTap`)

Guards checked in order before incrementing:
1. PWA install prompt is **not** visible (`installPrompt.style.display !== 'block'`)
2. App is **active** (`isAppActive === true`, toggled via `visibilitychange`)
3. Multi-touch (>1 finger) — additional touches are **ignored** (only the first touch counts)
4. Touch originates in bottom 10% of screen (`clientY > windowHeight * 0.9`) — **ignored** (avoids iOS home indicator conflicts)

Double-tap prevention: a 300 ms `doubleTapDelay` window is checked via `lastTap`. When a double tap is detected, it still increments by 1 (not 2), because both paths in the `if/else` do the same thing — the branch exists to prevent the browser's native double-tap zoom.

Both `touchstart` (non-passive) and `click` events call `handleTap` so the app works with mouse and touch input.

### Reset Button

`resetButton` triggers `resetCounter`:
1. `fade` CSS class added to `<body>` and `#counter` (0.5 s opacity dip)
2. After 250 ms: `count = 0`, display updated, `updateColor()` called (returns to starting purple)
3. After another 500 ms: `fade` class removed

Both `touchstart` and `click` are wired to prevent double-firing.

### PWA Install Prompt

- **iOS Safari:** `isIOS` detection via `navigator.userAgent`; if not already in standalone mode, shows the install banner immediately with step-by-step share-button instructions.
- **Android Chrome:** Intercepts the `beforeinstallprompt` event, stores it in `deferredPrompt`, shows the banner; clicking "설치하기" calls `deferredPrompt.prompt()`.
- The `#overlay` div blocks background tap events while the prompt is visible; clicking the overlay dismisses the prompt.

### Service Worker Registration

Registered on `load`. If a new worker reaches `installed` state while a controller is active, the page auto-reloads (`window.location.reload()`).

---

## Service Worker (`sw.js`)

**Strategy:** Network First with cache fallback.

- **`install`:** Pre-caches all app assets (HTML, manifest, icons). Calls `self.skipWaiting()` to activate immediately.
- **`fetch`:** Attempts live network fetch first. On success (status 200), clones the response into the cache. On failure, serves from cache.
- **`activate`:** Deletes all caches not matching the current `CACHE_NAME`. Calls `clients.claim()` to take control of all open clients immediately.

**Cache naming:** `counter-app-<YYYY-MM-DD>-<SW_VERSION>` — the date component means the cache name changes daily, triggering re-caching. When updating the service worker, bump `SW_VERSION`.

---

## PWA Manifest (`manifest.json`)

| Field | Value |
|-------|-------|
| `name` | 웹 카운터 |
| `short_name` | 카운터 |
| `display` | standalone |
| `orientation` | portrait |
| `theme_color` | #f8e1ff (light purple) |
| `background_color` | #f8e1ff |
| `start_url` | ./ |

All 8 icons use `purpose: "any maskable"`.

---

## Development Workflow

### Running Locally

Open `index.html` directly in a browser **or** serve it with any static file server. HTTPS (or localhost) is required for service workers to register:

```bash
# Python 3
python3 -m http.server 8080

# Node.js (npx)
npx serve .

# Then open: http://localhost:8080
```

### Making Changes

1. Edit `index.html` (markup, styles, or script), `sw.js`, or `manifest.json` directly.
2. Hard-refresh (`Ctrl+Shift+R` / `Cmd+Shift+R`) to bypass the service worker cache during development, or use DevTools → Application → Service Workers → "Update on reload".
3. When changing `sw.js`, bump `SW_VERSION` at the top of the file so the new cache name invalidates the old one.

### No Build Step

There is no compilation, transpilation, bundling, or minification. What you edit is what gets served.

### No Tests

There is no automated test suite. Manual testing on mobile browsers (iOS Safari, Android Chrome) is the primary QA method given the heavy touch-interaction focus.

---

## Conventions and Key Constraints

- **Single-file HTML:** Keep all CSS and JS inside `index.html`. Do not split into separate `.css` or `.js` files unless there is a strong reason — the current architecture keeps deployment trivial.
- **No dependencies:** Do not introduce `npm`, frameworks, or external libraries. The app must remain a set of static files that can be served from any host.
- **Korean UI:** All user-visible strings are in Korean. Keep new UI text in Korean.
- **Mobile-first:** The primary target is mobile browsers. Any UI changes must be verified against mobile viewports, especially iOS Safari's quirks (viewport height, touch events, home indicator).
- **Touch event correctness:** Always attach both `touchstart` (with `{ passive: false }`) and `click` for interactive elements. Preserve existing `stopPropagation` / `preventDefault` calls on the reset button and overlay to avoid event bubbling into the counter.
- **Service worker versioning:** Increment `SW_VERSION` in `sw.js` for every deployment that changes cached assets.
- **Commit message language:** Commit messages are written in Korean (see git log for examples).
- **AI 답변 언어:** 이 저장소에 관한 모든 AI 응답은 한국어로 작성한다.

---

## Common Gotchas

- **iOS viewport height:** The app uses `-webkit-fill-available` alongside `100vh` to handle Safari's dynamic toolbar. Do not remove these duplicate rules.
- **Double-event firing:** The reset button and install buttons both listen to `touchstart` + `click`. The `stopPropagation` calls are intentional — removing them causes the tap to also fire the body's counter increment.
- **Multi-touch:** Only the first touch of a multi-touch gesture should count. The `event.touches.length > 1` guard in `handleTap` is intentional.
- **Daily cache rotation:** `sw.js` embeds `new Date().toISOString()` in the cache key at install time. This is evaluated once when the worker first installs for the day. On subsequent days a new cache is created and the old one is deleted on activation.
