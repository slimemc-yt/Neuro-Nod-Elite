# Neuro-Nod Elite → V11 "Focus Shield" — Claude Code Build Brief

**Repo:** `slimemc-yt/Neuro-Nod-Elite` (already cloned locally — work in this repo, don't scaffold a new one)
**Entry point to edit:** `projects/neuro-nod/demo.html`
**Goal:** Turn the existing fatigue-only prototype into a full solution for the hackathon problem statement — *"detect driver fatigue, distraction, or unsafe driving patterns in real time and generate alerts or coaching feedback."* Right now the repo only covers fatigue + alerts. Distraction and coaching feedback are missing.

Paste this whole file to Claude Code as the task prompt.

---

## 1. Non-negotiable constraints

- **Zero backend, zero build step.** Everything must keep running as static files opened via a local server (`python3 -m http.server` or `npx serve`) or `file://`. No bundler, no npm install requirement, no server-side inference. This is the entire point of the "Edge AI / no paywall" pitch — do not introduce a Node backend, API calls, or cloud model calls.
- **All inference stays on-device.** Camera frames are never uploaded anywhere. Any new model (e.g. MediaPipe Hands) must load from the same CDN pattern already used (`cdn.jsdelivr.net/npm/@mediapipe/...`) and run locally in the browser.
- **Preserve real-time latency.** The current loop runs Face Mesh on every camera frame via `Camera` from `camera_utils.js` and reacts within the existing watchdog timers (800ms–1500ms). Any new detector must not visibly increase lag. If a new model is heavier (e.g. Hands), throttle it (e.g. run every 2nd–3rd frame) rather than slow down the main Face Mesh loop.
- **Do not break what already works.** Keep the existing calibration flow, EMA smoothing (`ALPHA = 0.15`), the three watchdogs (Eye Guard, Posture Guard, Failsafe Guard), the Sunglasses Paradox state machine, and the ACK/Pause protocol exactly as they behave today. New features are additive layers on top of `processLogicV10`, not replacements.
- **Keep the visual identity.** Dark glassmorphic "cyberpunk HUD" theme, existing CSS variables (`--neon-blue`, `--neon-red`, `--neon-amber`), existing metric-card / badge / dot patterns. New UI elements should reuse these classes, not introduce a new design language.
- You may split `demo.html`'s inline `<script>` into a few plain ES module files (e.g. `guards.js`, `scoring.js`, `report.js`) loaded via `<script type="module">` if it keeps the code manageable — but this is optional. If you do, it must still run with zero build tooling, just a static file server.

## 2. Current state — read this before writing any code

Open `projects/neuro-nod/demo.html` and read it fully first. Key things already implemented, so you don't reinvent them:

- **Landmarks used:** nose tip `1`, chin `152`, right-eye-inner `362`, left-eye-inner `133`, eyelids `159/145` and `386/374` for EAR, eye corners `33/133` and `362/263` for EAR normalization, lip landmarks `13/14` (vertical) and `1/152` (face height, MAR denominator).
- **Pitch:** `atan2(nose.y - chin.y, nose.z - chin.z)` — used for nod/head-drop detection.
- **Roll:** `atan2(rightEyeInner.y - leftEyeInner.y, rightEyeInner.x - leftEyeInner.x)` — used for side-slump detection.
- **EAR / MAR:** already computed and smoothed every frame (`sEAR`, `sMAR`), with `baseEAR`/`baseMAR` set during the 3s calibration window.
- **Watchdogs today:** Eye Guard (EAR < 60% baseline, 1500ms), Posture Guard (pitch/roll thresholds, 800–1200ms dynamic), Failsafe Guard (face lost, 1500ms), plus the Sunglasses auto-detect (EAR < 0.05 for 2s → posture becomes primary signal).
- **Audio/alert plumbing:** `handleAudio()`, `setAlarm()`, `muteUntil`/`suppressUntil`, ACK button, 20s pause button — reuse this, don't duplicate a second alarm system.
- **What's missing entirely:** yaw/gaze tracking, any distraction detection, PERCLOS or any rolling/trend metric, yawn *counting* (MAR is computed but only drives a one-off visual badge, not a watchdog or counter), session duration tracking, any coaching/report layer, and any driver score.

## 3. Feature scope — mapped to the rubric

The problem statement grades four things. Build in this order so a partial build still demos well:

| Rubric clause | Status today | What to add |
|---|---|---|
| Fatigue | ✅ done | Add PERCLOS + yawn frequency (Section 4.3, 4.4) |
| Distraction | ❌ missing | Gaze/Yaw Guard, optional phone-hand heuristic (4.1, 4.2) |
| Unsafe driving patterns | ⚠️ partial (only face-derived) | Optional stretch: harsh-motion via DeviceMotion (4.8) |
| Coaching feedback | ❌ missing | Session timer + break nudges + end-of-session report + driver score (4.5–4.7) |

---

## 4. Feature specifications

### 4.1 Gaze / Distraction Guard (Yaw) — required

Add a fourth Watchdog, styled exactly like the existing three (own entry in the `timers` object, own metric card, own badge).

- Compute raw yaw using the same `atan2` style as pitch/roll, e.g. using the outer face-edge landmarks `234` (left) and `454` (right):
  `rawYaw = atan2(rightEdge.z - leftEdge.z, rightEdge.x - leftEdge.x) * RAD_TO_DEG`
  Smooth it with the same EMA alpha (`sYaw`), and capture `baseYaw` during calibration exactly like pitch/roll/EAR/MAR are captured.
- **Threshold:** `|sYaw - baseYaw| > 25°` sustained for **> 1,800ms** → distraction event ("EYES OFF ROAD"). Make both numbers tunable constants near the other thresholds (28° pitch, 15° roll) so they're easy to demo-tune.
- Distraction should **not** trigger the full-screen red DANGER overlay the same way head-drop does — use the amber "visual warning" pattern already used for posture-warning-without-eyes-closed, escalating to the audio alarm only if sustained past a second, longer threshold (e.g. 3,500ms) to avoid false positives during normal mirror-checks and shoulder checks.
- Add a "Gaze" metric card to the panel (same markup pattern as `card-head`/`card-eye`/`card-mouth`) showing live yaw degrees and a badge (`ON ROAD` / `LOOKING AWAY`).
- Must coexist correctly with Sunglasses Mode and Posture Guard — a driver can be nodding AND looking away; don't let one Guard silently overwrite another's badge/message. Use the same message-priority pattern already implicit in `processLogicV10` (critical > drowsy > posture-visual > yawn > distraction, or similar — use your judgment but keep it deterministic).

### 4.2 Phone-in-hand heuristic — optional / stretch, off by default

Only build this if 4.1, 4.3–4.7 are solid and there's time left.

- Load `@mediapipe/hands` from the same CDN pattern as Face Mesh.
- Run Hands inference at a reduced rate (e.g. every 3rd frame) to protect latency, since running two heavy models every frame will visibly lag on mid-range laptops.
- Heuristic: a wrist/hand landmark persists within a bounding-box region near the ear/jaw line for > 1.5s, combined with sustained yaw > 15°, → "POSSIBLE PHONE USE" distraction sub-type. This is a heuristic, not object detection — label it as such in the UI, don't overclaim.
- Must be toggleable (like the existing `modeToggle` for Sunglasses Mode) so it can be switched off instantly if it hurts framerate during the demo.

### 4.3 PERCLOS — required

Layer this on top of the existing EAR pipeline; don't replace the instant Eye Guard.

- Maintain a rolling window (default 60 seconds, configurable constant) of per-frame boolean "eyes closed" state, where closed = `sEAR < baseEAR * 0.70` (slightly looser than the 60%-for-1500ms Eye Guard threshold, since PERCLOS is a trend metric, not an instant alarm).
- A simple ring buffer of timestamps or a decaying counter is fine — no need for a real circular buffer library.
- `PERCLOS% = (closed time in window / window length) * 100`.
- Display live in a metric card. Industry rule of thumb: PERCLOS > 15% over the window = elevated fatigue trend — surface this as a distinct, calmer "FATIGUE TREND RISING" indicator (amber, non-blocking) separate from the acute Eye Guard alarm. This is what gives the system a "trend vs event" story for judges.

### 4.4 Yawn frequency watchdog — required

The repo already computes `sMAR` and a one-off "MOUTH WIDE OPEN" badge. Extend it:

- When `sMAR` crosses the existing yawn threshold (`baseMAR * 3.5`) and stays above it for > 500ms then drops back down, count it as **one yawn event** (debounce so one long yawn isn't counted many times — track a boolean "in yawn" state with rising/falling edge detection).
- Keep a rolling yawn count per session and per most-recent-10-minutes window.
- 3+ yawns within 10 minutes → elevate to the same amber "FATIGUE TREND RISING" banner used by PERCLOS (they can share one indicator with a combined reason, e.g. "FATIGUE TREND RISING (Yawning + Eye Closure)").

### 4.5 Session timer & proactive break coaching — required

This is the piece that actually satisfies "coaching feedback," not just alerts.

- Start a session clock when `startBtn` is clicked (reuse the existing click handler).
- Track **continuous driving time** — resets when the user hits a new "Take a Break" action (add this button, distinct from the existing 20s debug `pauseBtn` — the existing pause is a demo/dev tool, this new one represents a real rest stop and should reset the continuous-drive clock and log a "break taken" event).
- At configurable intervals (default: every 60 minutes of continuous driving, since this targets long-haul drivers) show a **non-alarming coaching nudge**: a dismissible amber banner, no siren, no red overlay — e.g. "You've been driving 60 min — a short break improves reaction time." This must be visually and behaviorally distinct from the danger/drowsiness alerts; reusing the red `alarm-overlay` for this would undercut the "prevent alarm fatigue" pitch already in the whitepaper.
- Display total session time and continuous-drive time live in the panel.

### 4.6 Event logging & driver safety score — required

- Maintain an in-memory array of events for the session: `{ timestamp, type, severity }` for every watchdog trigger (drowsy-eye, head-drop, side-slump, distraction, phone-use if built, yawn, face-lost, break-taken).
- Compute a **Driver Safety Score** starting at 100, deducted per event, floored at 0:
  - Critical head-drop / side-slump alarm: −8
  - Drowsy-eye alarm (Eye Guard fired): −6
  - Distraction (gaze-away alarm fired): −5
  - Yawn event: −1
  - Face-lost alarm: −4
  - PERCLOS/yawn "trend" banner shown: −2 (only once per occurrence, not per frame)
  These weights are starting points — expose them as named constants so they're easy to justify/tune live if a judge asks "why these numbers."
- Show the live score somewhere persistent in the panel (small badge near System Status is fine), and recompute it in real time as events log.

### 4.7 Coaching report — required

- Add a "View Session Report" button that opens a modal (reuse the existing glass panel / overlay visual style — do not introduce a different design system).
- Report contents:
  - Total session duration, continuous-drive time at end, number of breaks taken.
  - Final Driver Safety Score, plus a short auto-generated one-line takeaway based on which event type had the highest count (e.g. "Mostly posture-related fatigue tonight — consider adjusting seat height" vs "Frequent distraction events — minimize phone handling while driving").
  - Event timeline: a simple scrollable list, each row = time offset + event type (no need for a charting library — a plain styled list is enough and keeps the zero-dependency promise).
  - PERCLOS average and peak, total yawn count.
- Add a "Export Report (JSON)" button that downloads the event log + summary as a `.json` file via a `Blob` + temporary `<a download>` — purely client-side, no backend.

### 4.8 Optional stretch features — only if time permits, clearly label as experimental

- **Harsh-motion detection:** if `window.DeviceMotionEvent` is available (phone browsers only — feature-detect and hide gracefully on desktop), watch for acceleration spikes above a threshold to flag hard-braking/swerving as an "unsafe driving pattern" event, logged into the same event array/report. This is the only way to extend beyond driver-face signals without new hardware.
- **rPPG heart-rate estimate** from subtle facial color changes: genuinely a stretch goal, high effort/uncertain payoff for the time available — only attempt if everything above is done and stable.

---

## 5. UI/UX requirements

- New metric cards (Gaze, PERCLOS, Yawn Count) follow the exact existing `.metric-card` / `.metric-label` / `.metric-value` / `.badge` markup and class-toggling pattern (`good` / `bad` classes) already used for Head Pose, Eye, Mouth.
- New non-alarm coaching nudges (break reminder, fatigue-trend banner) must be visually distinct from the existing red `#alarm-overlay` DANGER state — reuse the amber `#warning-bar` pattern, not the full-screen red overlay, so judges immediately see the "alerts vs coaching" distinction the rubric is asking about.
- Session Report modal: same `.panel` glass style, scrollable content, a close button, and the export button.
- Everything must remain usable on the existing 640×480 `.vid-frame` + 360px side-panel two-column layout — don't redesign the layout, extend it (new cards stack in the existing `.metric-grid`, a new row of session/coaching controls under the existing `.btn-group`).

## 6. Architecture guidance

- Preferred: keep `demo.html` as the entry point. If the file gets unwieldy, extract into `projects/neuro-nod/js/guards.js`, `scoring.js`, `report.js`, `state.js` as native ES modules (`<script type="module" src="...">`), sharing state via a small exported state object rather than globals scattered across files. No import maps, no bundler — plain relative-path ES module imports work fine served over `http://localhost`.
- Follow the existing naming conventions (`timers.*`, `s<Metric>` for smoothed values, `base<Metric>` for calibration baselines) for anything new — consistency matters more than cleverness here.

## 7. Performance budget

- Face Mesh must keep running every frame at the current `maxNumFaces:1, refineLandmarks:true` settings — don't downgrade its config to make room for new features.
- Any additional per-frame computation (PERCLOS ring buffer update, yaw calc, score recompute) should be O(1) or O(window size) with a small fixed window — avoid anything that scans a growing full-session array every frame. Keep the per-session event log append-only and only iterate it when the report modal is opened.
- If Hands (4.2) is implemented, it must be lazy-loaded only when the phone-heuristic toggle is turned on, not loaded/run by default.

## 8. Definition of done

- [ ] Looking away (>25° yaw) for a few seconds while eyes stay open triggers a distraction indicator, escalating to audio only if sustained.
- [ ] Sunglasses Mode, Eye Guard, Posture Guard, and Failsafe Guard behave exactly as before — no regressions.
- [ ] PERCLOS updates live and a "fatigue trend" banner appears distinct from acute drowsy alarms after sustained partial eye-closure.
- [ ] Yawning 3+ times in a short window shows a trend indicator; each yawn is counted once (no double-counting on a single long yawn).
- [ ] A visible session/continuous-drive timer runs, and a break reminder appears after the configured continuous-drive threshold, styled as non-alarming.
- [ ] Taking a "break" resets the continuous-drive timer and logs an event.
- [ ] Driver Safety Score updates live and is visible without opening the report.
- [ ] "View Session Report" opens a modal with duration, score, takeaway line, event timeline, and PERCLOS/yawn stats.
- [ ] "Export Report" downloads a valid JSON file with the session's events and summary.
- [ ] No network calls other than the initial CDN script loads — verify via browser dev tools Network tab with camera running.
- [ ] Frame rate / responsiveness feels the same as the current build (no visible added lag) on a normal laptop webcam.

## 9. How to run locally (keep this true after your changes)

```bash
cd projects/neuro-nod
python3 -m http.server 8080
# open http://localhost:8080/demo.html and allow camera access
```

No `npm install`, no build step, no `.env` file, no API keys — if your changes require any of those, that's a sign you've drifted from the brief.
