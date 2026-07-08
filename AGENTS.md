# AGENTS.md

Guidance for AI coding agents working in this repo. For product/UX intent see
`PRODUCT.md`; for end-user/teacher-facing instructions see `README.md`.

## What this is

A single self-contained HTML file (`index.html`, no build step, no
framework, no `node_modules` at runtime) implementing a 2-team classroom
math-quiz game in Thai. Students answer by raising a hand toward an
on-screen answer bubble (webcam pose detection) or by tapping/keyboard as a
fallback. Meant to be distributed as one file or hosted on GitHub Pages.

**Every change happens in `index.html`.** There is no separate CSS/JS build;
`<style>` and `<script type="module">` are inline in that file. Don't
introduce a bundler or split it into multiple files unless explicitly asked
— staying single-file is a deliberate distribution requirement (see
PRODUCT.md's "แจกจ่ายง่ายเพราะเป็น HTML ไฟล์เดียว").

## Running it

No install/build. Serve statically and open in a browser:

```bash
python3 -m http.server 8000   # then open http://localhost:8000/index.html
```

Opening the file directly (`file://`) mostly works but breaks camera
(`getUserMedia` needs a secure context) and ES module imports in some
browsers — prefer a local server when testing.

## Verifying changes

This project has no test suite. The established pattern across this repo's
history is: **write a throwaway Playwright script, run it against a local
server, read actual DOM/computed-style state, then discard the script.**
Pure code review has repeatedly missed real bugs here (ARIA attribute/CSS
selector mismatches, TDZ ordering, narrow-viewport layout overlap, a
stray `stopPropagation()` swallowing unrelated clicks) that only showed up
once actually driven in a browser. Don't skip this step for UI changes.

For camera-related testing, launch Chromium with
`--use-fake-device-for-media-stream --use-fake-ui-for-media-stream` and
grant the `camera` permission on the context — real `getUserMedia` still
needs the MediaPipe CDN to resolve, which may be blocked in sandboxed
environments (a `timeout` in `initCamera()`'s catch block there is a network
restriction, not necessarily a code bug — verify against a real browser
before treating it as a regression).

## Architecture map (inside index.html)

- **Screens**: `.screen` sections toggled via `.is-active`, switched by
  `showScreen(name)` — `scrSplash` → `scrHome` → `scrGame` → `scrResult`.
- **State**: a single `S` object (search `const S = {`) holds game/session
  state (`S.cfg`, `S.phase`, `S.camOn`, `S.camBusy`, `S.matchDeadline`,
  scores, etc.). No framework/reactivity — DOM is imperatively updated
  wherever state changes (e.g. `updateScores()`, `syncCamUI()`).
- **Settings persistence**: `localStorage` under `SETTINGS_KEY =
  'mathduel.settings.v1'` via `loadSettings()`/`saveSettings()`.
- **Camera/pose pipeline**: `initCamera()`/`stopCamera()` own the
  `getUserMedia` stream + MediaPipe `PoseLandmarker` (lazy-loaded from CDN
  only when a camera is actually requested). `loop()` → `processPoses()` →
  `feedHitTest()` drive edge-triggered instant-tap hit detection against
  cached zone rects (`zoneCache`). `OneEuroFilter` smooths cursor jitter.
  **Camera UI has three independent buttons that must stay in sync**:
  in-game `#btnCam`, the "วิธีเล่น" dialog's `#btnCamPre`, and the home
  screen's `#btnCamHome`. `syncCamUI()` is the single function that updates
  the latter two; `initCamera()`/`stopCamera()` both call it in their
  success/failure/`finally` paths — if you add a fourth camera toggle
  somewhere, wire it through `syncCamUI()` too rather than duplicating logic.
- **Audio**: synthesized SFX via Web Audio API (`audio.*`, chiptune-style),
  separate from the two `<audio>` elements (`bgmMenu`, `bgmGame`) that play
  real music files from `audio/`. Volume widgets (`.vol-control`, one per
  screen: `#splashAudioWidget`, `#homeAudioWidget`) are collapsible —
  `wireAudioWidget()` wires each instance generically.
- **ARIA radio groups**: the settings segmented controls (`.seg`,
  `role="radiogroup"`) use `role="radio"` + `aria-checked` + roving
  `tabindex`, wired by `wireSeg()`/`syncRadioGroup()`. If you touch their
  CSS, the selected-state selector is `.seg button[aria-checked="true"]`
  — **not** `aria-pressed` (a past mismatch here silently broke the visual
  "selected" state while the underlying value still worked, which is a
  nasty bug to catch by inspection alone).
- **Design tokens**: CSS custom properties in `:root` (`--red`, `--blue`,
  `--gold`, `--ease-out`, etc.) plus a semantic z-index scale (`--z-hud`,
  `--z-card`, `--z-zone`, `--z-banner`, `--z-overlay`, `--z-modal`). Reuse
  these instead of hardcoding colors or ad-hoc z-index values.
- **The `#howto` dialog** (native `<dialog>`) is the "วิธีเล่น" instructions
  popup, opened from the home screen only (`openPreGame()`). It is purely
  informational — its "เข้าใจแล้ว" button (`#btnGo`) just closes the dialog.
  It does **not** gate starting a match: `#btnStart` ("เริ่มเล่นเลย!") calls
  `startGame()` directly. Don't re-couple these without checking that
  history — an earlier version routed both through this dialog and it was
  deliberately split apart per user feedback.

## Conventions / gotchas learned the hard way

- Declare any `const`/icon-string used inside functions that run during
  early settings sync (e.g. `syncSettingsUI()`) *before* that sync runs —
  a late `const` declaration referenced earlier caused a temporal-dead-zone
  crash previously.
- Don't attach `capture: true` + `stopPropagation()` to global "click
  outside to close" listeners; it can swallow clicks on unrelated buttons
  underneath. Use a plain bubble-phase listener that only closes panels the
  click landed outside of.
- Buttons whose text gets rewritten via `.textContent` in JS (state-label
  swaps like "เปิดกล้อง" → "กำลังเปิดกล้อง…") must not also contain a child
  `<svg>` icon directly — that write wipes the icon. Put the label in its
  own `<span class="btn-label">` and update only that.
- Mobile/narrow-viewport bugs here are usually geometric (a fixed control
  rail overlapping randomly-positioned answer zones), not stylistic —
  reproduce at real narrow widths (e.g. 320–390px) with Playwright rather
  than assuming a media query alone fixes it.
- `cover.jpg` / `หน้าหลัก.jpg` / `พื้นหลัง.jpg` contain Shin-chan-style
  character art with an acknowledged (open, not yet resolved) IP-risk flag
  — see PRODUCT.md's anti-references. Don't add more third-party character
  likenesses; flag but don't unilaterally replace the existing ones without
  the user's direction.

## Assets

- `cover.jpg` — splash screen art. `หน้าหลัก.jpg` — home screen background.
  `พื้นหลัง.jpg` — result screen background / in-game backgrounds.
  `.original-assets-backup/` holds pre-compression originals.
- `audio/menu-music.mp3`, `audio/game-music.mp3` — licensed by the user;
  don't source replacement music from YouTube or similar (ToS/copyright).
