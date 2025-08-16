# Mac Touchpad Chromaplane

The Chromaplane is an electromagnetic syntesizer that is played using two electromagnetic pickup coils, that are moved over its surface to listen to a cloud of electromagnetic fields produced by the instrument. The instrument is fully analog and polyphonic, this makes the instrument ideal for both sculpting complex drones, and playing agile melodies.

There are 10 oscillators in a 3-4-3 hex grid type layout. Each plays a (configurable) note through an electronic coil oscilator thingee. A pickup (or two, but one is cool enough) is waved by the player over the device. As it hovers above the different oscillators, it picks them up - loud single tones when directly over one osc, combined tones when you're in-between.

I want to mimic this with a webapp that uses the mac touchpad's fine touch sensing on the touchpad or a mobile phones touch screen to mimic the effect. I want to use pressure (mac) or a slider (touchscreen phones with multi-touch) to control the volume (~height in the physical play style).

I also want to mimic the effects: "4-pole low pass filter that darkens the instrument’s sound, and a lo-fi delay/echo effect for space and ambiance."

Plan is to start with a mac touchpad version (what I'm coding on now) and then re-create it in mobile-friendly form later.

## Detailed Implementation Plan — Mac Touchpad Version (v1)

### 1) Product goals & scope
- **Goal:** Recreate the Chromaplane’s play style using a Mac trackpad as the “pickup” over a virtual 3‑4‑3 hex grid of oscillators, with expressive control via Force Touch pressure (Safari) and a fallback slider/keyboard when pressure isn’t available.
- **Non-goals (v1):** No true multi‑pickup tracking from the trackpad (macOS browsers don’t expose multi‑touch). No recording/looping. No mobile UI yet (to be a separate v2 effort).
- **Target environment:** Safari 17+ on macOS 14+ for full force support; Chrome/Firefox on macOS supported via fallback controls.

### 2) Architecture
- **Frontend only web app** (single‑page), built with vanilla TS/JS + WebAudio. Keep the audio graph static; modulate parameters (gains, cutoff, delay time) in response to input.
- **File layout (proposal):**
  - `index.html` – canvas UI + control panel
  - `main.ts` – bootstraps UI + input, orchestrates state
  - `audio.ts` – builds/owns the WebAudio graph & DSP params API
  - `geometry.ts` – hex grid layout & distance math
  - `worklets/bitcrusher.worklet.ts` – AudioWorklet for lo‑fi
  - `styles.css`
- **Performance strategy:**
  - Use `AudioContext({ latencyHint: 'interactive' })`.
  - Run pointer updates on `requestAnimationFrame` (rAF) to throttle to ~60 Hz.
  - Avoid node churn: 10 `OscillatorNode` → 10 `GainNode` → mix bus.
  - Parameter changes via `AudioParam.setTargetAtTime` for click‑free moves.

### 3) Hex grid & note mapping
- **Layout:** 10 oscillators in a 3‑4‑3 hex grid.
  - Define hex cell radius `R` (px). Canvas size computed from `R` and margins.
  - Centers computed in axial coords then mapped to pixel coords. Row layout:
    - Row 0: 3 nodes
    - Row 1: 4 nodes (offset by half hex width)
    - Row 2: 3 nodes
- **Tuning:** Default = equal‑tempered chromatic anchored at A4 = 440 Hz (configurable root + scale).
  - Example mapping (index → semitone offset relative to root):
    - Row 0: [-2, 0, +2]
    - Row 1: [-5, -3, -1, +1]
    - Row 2: [-9, -7, -5]
  - Provide presets: **Ionian (major)**, **Dorian**, **Pentatonic**, **Custom** (manual per‑node semitone offsets).

### 4) Pickup interaction & weighting
- **Pointer position** `(x,y)` from mouse/pointer events represents the pickup location.
- **Weighting:** For each oscillator `i`, compute distance `d_i` from `(x,y)` to center `c_i`. Convert to weight `w_i` via Gaussian RBF:
  - `w_i = exp(-(d_i^2) / (2*sigma^2))`, with `sigma ≈ 0.7*R` (tweakable).
  - Normalize: `W = sum(w_i)`, per‑oscillator gain = `(w_i / W)^gamma`, with `gamma ∈ [0.7, 1.5]` to shape blend (expose in Advanced panel).
- **Hover zones:** Snap boost when pointer is within `d_i < 0.35*R` to get strong single‑tone behavior.
- **Second (optional) drone pickup:** Press `D` to toggle a pinned pickup at the current cursor position; it keeps sounding until toggled off or moved with arrow keys (fine) / WASD (coarse).

### 5) Pressure → volume mapping
- **Safari Force Touch:** Listen for `webkitmouseforcechanged` / `event.webkitForce` (range typically ~0–3). Normalize to `[0,1]` by `p = clamp((force - f_min) / (f_max - f_min))`. Provide a **Calibrate** button: press hard/soft to set `f_min/f_max`.
- **Pointer Events path:** Where `PointerEvent.pressure` exists (`0–1`), use that directly.
- **Fallback:** On browsers without pressure, show a vertical **Volume/Height** slider bound to `p` and support `[`/`]` keys to nudge.
- **Master gain:** `G_master = p^curve`, with `curve ∈ [0.8,1.3]`.

### 6) WebAudio graph
```
[10x OscillatorNode] → [10x Per‑Osc Gain] → [Mix] → [Filter Stage] → [Delay Bus] → [Master Gain] → [Destination]
                                                       ↓ (send)
                                               [Delay Send Gain]
```
- **Oscillators:** default waveform `sine`; per‑node detune (cents) param for chorusing (range ±10 cents, default 0).
- **Envelopes:** Per‑oscillator amplitude envelope applied via `setTargetAtTime` to `Per‑Osc Gain.gain` when weights change.
  - Attack 5–20 ms; Release 40–120 ms. Configurable.
- **Filter (4‑pole LP):** Cascade two `BiquadFilterNode` lowpass. Expose cutoff `40–10,000 Hz`, resonance (Q) `0.5–12`.
- **Delay/Echo:** `DelayNode` time `0–800 ms`, feedback via `GainNode` (0–0.9), with tone control by placing a lowpass `Biquad` in the feedback path.
- **Lo‑fi (optional):** `AudioWorklet` bitcrusher (bit depth `4–16`, downsample factor `1–8`) on the delay return for character.

### 7) UI/UX
- **Canvas:** shows hex nodes. Intensity glow per node reflects current gain. Cursor/pickup drawn as a soft ring; force/volume indicated by ring thickness.
- **Controls panel:**
  - **Master:** Volume (if fallback), Output limiter toggle.
  - **Tuning:** root note, scale preset, detune range, save/load presets.
  - **Blend:** sigma, gamma, snap boost toggle.
  - **Filter:** cutoff, resonance, drive (pre‑filter gain), on/off.
  - **Delay:** time, feedback, tone (LP cutoff), mix, lo‑fi depth/rate.
  - **Advanced:** envelope A/R, rAF update rate, latency hint, calibration.
- **Keyboard shortcuts:**
  - `F` toggle filter; `E` toggle delay; `D` toggle drone pickup; `C` calibrate force; `[`/`]` volume (fallback); `1–5` presets; `Z/X` change root; `Shift+Z/X` transpose ±12.
- **Accessibility:** All controls reachable via keyboard and labeled; no color‑only cues.

### 8) State & persistence
- Keep full state object (tuning, FX, blend, envelopes, calibration, drone pos) in memory; mirror to `localStorage` on change and load on boot.
- Export/import presets as JSON.

### 9) Timing & smoothing
- Update per‑oscillator gains once per rAF frame. Apply `setTargetAtTime(value, now, timeConstant)` where `timeConstant ≈ 0.02–0.06` to prevent zipper noise.
- For filter/delay params, slew changes with `linearRampToValueAtTime` over 30–100 ms.

### 10) Error handling & fallbacks
- If `AudioContext` is suspended, show a “Click to Start Audio” overlay.
- If `AudioWorklet` fails to load, disable lo‑fi controls gracefully.
- If Force Touch unavailable, auto‑show Volume slider and hint text.

### 11) Testing checklist
- **Audio:** single‑node focus (center hover), in‑between blends (midpoints), fast glides (no clicks), max pressure stability, effect tails when leaving canvas.
- **Performance:** ≤ 5 ms rAF handler time on an M‑series Mac; no GC churn; CPU < 20% in Activity Monitor at 60 FPS.
- **Compatibility:** Safari (force), Chrome/Firefox (fallback), different trackpad pressures.

### 12) Milestones
1. **M0 (Scaffold):** Canvas + AudioContext + 10 oscillators mixed, static sine tones on.
2. **M1 (Grid & Pickup):** Hex layout + pointer → weights → per‑osc gains. Visual feedback.
3. **M2 (Pressure/Volume):** Force Touch + calibration + fallback slider; envelopes.
4. **M3 (FX):** 4‑pole LP + delay/feedback/tone; lo‑fi worklet.
5. **M4 (UX polish):** Presets, state save, keyboard shortcuts, limiter.
6. **M5 (QA):** Perf passes, edge cases, accessibility, doc & demo video.

### 13) Acceptance criteria
- Audible blend changes track cursor smoothly with no audible zipper/clicks.
- With pressure support, dynamic range clearly responds to force; calibration persists between sessions.
- Filter sounds 4‑pole‑ish (steeper slope vs single biquad) and delay/feedback are stable up to 0.85 feedback without runaway.
- rAF loop holds 60 FPS on a 13‑inch MacBook Pro M‑series.
- All controls operable via keyboard; presets save/load.

### 14) Risks & mitigations
- **Force Touch variance across devices:** Provide calibration + fallback slider; keep musical output independent of absolute force.
- **Browser differences:** Feature‑detect everything; isolate Safari code paths for `webkitForce`.
- **Audio glitches from parameter jumps:** Always slew (`setTargetAtTime` / ramps); avoid creating/destroying nodes during play.

### 15) Out of scope (for v1)
- Recording/looping, MIDI I/O, multiple independent pickups tied to actual multi‑touch, mobile UI.