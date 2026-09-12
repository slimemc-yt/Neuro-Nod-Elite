# Neuro-Nod Elite V11 "Focus Shield"

**Universal Driver Monitoring System, Distraction Shield & Adaptive Coaching Engine**

AI-powered drowsiness, fatigue, and distraction detection that runs entirely in the browser using real-time 3D computer vision — no cloud, no install, no data upload. Safety, moved from expensive dedicated hardware to the "Edge": your existing phone or laptop browser.

🌐 **Live site:** [upaharmishra.com.np](https://upaharmishra.com.np) <br>
👤 **Lead Developer:** Upahar Mishra <br>
KIST Fair Sci-Tech Exhibition 2082

---

## The Mission

Advanced Driver Assistance Systems (ADAS) have historically been a luxury, locked behind vehicles priced above $60,000, such as Tesla Autopilot and Volvo Pilot Assist. This leaves the vast majority of commercial drivers, including truck, bus, and machinery operators using older vehicles, completely unprotected, even though fatigue and distraction related accidents account for over 25% of highway fatalities worldwide.

Neuro-Nod Elite exists to **democratize safety**: no environment to set up, no libraries to install, no terminal commands — just a browser tab.

---

## What's New in V11 "Focus Shield"

1. **Gaze / Distraction Guard (Yaw Tracking):**
   Continuous 3D facial orientation tracking detects when the driver turns away from the road (>25° yaw offset). Features a graduated response: non-alarming amber coaching notice after 1.8s, escalating to audio alarms if sustained past 3.5s.
2. **PERCLOS (Percentage of Eyelid Closure) Tiered 60s Window:**
   Tracks cumulative eyelid closure rates ($sEAR < baseEAR \times 0.70$) across a 60-second rolling buffer. Features a graduated clinical hierarchy: `<8.0%` Optimal, `8.0-15.0%` Mild/Rising, `15.0-25.0%` Elevated Fatigue Coaching Alert, and `>25.0%` Critical Severe Microsleep Hazard.
3. **Speech-Filtered Debounced Yawn Engine:**
   Rejects transient conversational speech via dual geometric aperture filtering ($\max(baseMAR \times 2.2, 0.18)$) and continuous hold duration verification ($\ge 800\text{ms}$ hold with single-cycle rising-edge latching). Tracks fatigue frequency across rolling 10-minute windows ($\ge 3$ yawns or compound yawn/PERCLOS interaction elevates coaching alarms).
4. **Proactive Break Coaching & Continuous Drive Clock:**
   Monitors continuous driving duration and proactively nudges operators to take 15-minute breaks at 60-minute intervals. Dedicated "Take a Break" action resets drive timers and logs rest events.
5. **Real-Time Driver Safety Score:**
   Live dynamic scoring index starting at 100 with weighted deductions for incidents (Head Drop: -8, Side Slump: -8, Drowsy Eye: -6, Distraction: -5, Face Lost: -4, Harsh Motion: -3, Fatigue Trend: -2, Yawn: -1, Break Rest: +2).
6. **Comprehensive Coaching Report & Client-Side JSON Export:**
   Full telemetry analytics modal featuring personalized AI coaching takeaways, duration KPIs, PERCLOS averages/peaks, and single-click client-side JSON export for fleet safety logs.

---

## Core Features & Watchdog Architecture

### 1. 3D Landmark Geometry
Powered by MediaPipe's 468-point 3D Face Mesh, the system treats the driver's face as a real object moving through 3D space rather than a flat image — using Z-coordinates for genuine depth awareness.

### 2. The Four Mathematical Pillars
- **EAR (Eye Aspect Ratio):**  
  Monitors Euclidean distance between eyelids normalized by horizontal eye span:
  $$EAR = \frac{||p_2 - p_6|| + ||p_3 - p_5||}{2||p_1 - p_4||}$$

- **3D Pose Orientation (Pitch / Roll / Yaw):**  
  - *Pitch (Head Drop / Nodding):* $\text{atan2}(\text{nose}.y - \text{chin}.y, \text{nose}.z - \text{chin}.z) \times \frac{180}{\pi}$
  - *Roll (Side Slump):* $\text{atan2}(\text{eyeR}.y - \text{eyeL}.y, \text{eyeR}.x - \text{eyeL}.x) \times \frac{180}{\pi}$
  - *Yaw (Distraction / Gaze Offset):* $\text{atan2}(\text{edgeR}.z - \text{edgeL}.z, \text{edgeR}.x - \text{edgeL}.x) \times \frac{180}{\pi}$

- **MAR (Mouth Aspect Ratio):**  
  $$MAR = \frac{||\text{lipUpper} - \text{lipLower}||}{||\text{nose} - \text{chin}||}$$

- **Signal Processing (EMA Smoothing):**  
  $$S_t = \alpha \cdot Y_t + (1 - \alpha) \cdot S_{t-1} \quad (\alpha = 0.15)$$

### 3. Parallel Watchdogs
- **Watchdog 1: Deep Drop Guard** — Pitch drop $> 43^\circ$ below baseline for $>800\text{ms}$ triggers critical alarm.
- **Watchdog 2: Posture Guard** — Pitch $>28^\circ$ or Roll $>15^\circ$ triggers posture alarm (1200ms in Sunglasses mode / 1500ms standard).
- **Watchdog 3: Eye Guard** — $sEAR < 60\%$ baseline for $>1500\text{ms}$ triggers acute drowsiness alert.
- **Watchdog 4: Gaze / Distraction Guard** — Yaw $>25^\circ$ triggers visual warning at 1.8s, escalating to audio at 3.5s.
- **Watchdog 5: Failsafe Guard** — Missing face landmarks $>1500\text{ms}$ triggers camera position alarm.
- **Context-Aware Sunglasses Paradox** — Low eye openness without posture deviation automatically routes primary detection to head posture, ensuring tinted lenses never compromise safety.

---

## 📁 Project Structure

```
.
├── index.html                     # Root portfolio landing page
├── logic.pdf                      # Full technical white paper
├── favicon.svg
├── neuro-nod-elite-v11-build-brief.md
└── projects/
    └── neuro-nod/
        ├── index.html             # Project details & feature list
        └── demo.html              # Core V11 Edge AI DMS HUD & Logic
```

---

## 🚀 How to Run Locally

Because Neuro-Nod Elite is 100% client-side with zero dependencies, run it with any static server:

```bash
cd projects/neuro-nod
python3 -m http.server 8080
# Open http://localhost:8080/demo.html in any modern browser with webcam access
```

---

*© 2026 Neuro-Nod Technologies — Democratizing Life-Saving Edge AI*
