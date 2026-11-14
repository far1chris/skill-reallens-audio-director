---
name: reallens-audio-director
description: >-
  Use when writing Thai voiceover scripts (`scene.vo`), designing sound effects (`sfx`,
  `CUSTOM_SFX`), selecting ambient beds (`bed`, `CUSTOM_BEDS`), tuning `speech_metadata`,
  estimating TTS duration/pacing, or profiling custom voice/SFX samples for Real Lens
  (@real_lens1) and `generate_audio.js` pipelines.
---

# Real Lens Audio Director: Voice Persona (`Puck`) & Procedural SFX/Bed Bible

Gives AI agents an exact **sonic mental model** of how the narrator voice, FFmpeg tone shaping, procedural SFX, ambient beds, and sidechain ducking in [`generate_audio.js`](../reallens-horror-animator/templates/generate_audio.js) actually sound and behave—so scripts, SFX cues, and visual timings lock together on the first pass.

---

## 1. Narrator Voice Profile (`Puck` + `unifyVocalTone`)

### 1.1 Exact Vocal Chain & Sonic Character
- **Voice & Model Priority:** Gemini Interactions TTS (`voice: "Puck"`):
  1. `gemini-3.8-flash-tts` (uses `annotations: [{ type: 'speech_metadata', style: UNIFIED_STYLE }]`)
  2. `gemini-3.8-flash-lite-tts` (uses `speech_metadata`)
  3. `gemini-3.1-flash-tts-preview` (**must omit** `annotations` / `speech_metadata`)
- **Base Style Directive (`UNIFIED_STYLE`):**
  > `"calm, composed, deadpan male survival-horror documentary narrator in Thai, steady pace, low and even pitch, quiet suspense"`
- **Post-Processing Signature (`unifyVocalTone`):**
  - **Speed:** `atempo=1.12` (**+12% faster** than raw TTS delivery — crisp, urgent analytical cadence without sounding rushed).
  - **EQ & Dynamics:** `highpass=f=80`, `lowpass=f=12000`, **`+1.2 dB @ 180 Hz`** (warm chest resonance), **`-1.0 dB @ 3500 Hz`** (softens Thai sibilants `ซ/ส/ช`), `acompressor` (`-18 dB`, `2.5:1`), normalized to **`-18.0 LUFS`** (`TP=-1.5`).
- **How This Voice "Feels" to Listeners:**
  - Detached, forensic, coldly rational male documentary host—never melodramatic, screaming, or cartoonishly spooky.
  - Tension comes from **short declarative sentences, concrete numbers, and sudden quiet pauses**, not exclamation marks.

### 1.2 Writing `scene.vo` for `Puck` + `chunkSubs()`
Because `scene.vo` feeds **both** Gemini TTS synthesis and `chunkSubs()` (`Intl.Segmenter('th')`, `maxLen = 34`, short-tail merge ceiling `<= 36` chars/cue):
1. **Write for Deadpan Analytical Delivery:**
   - ✅ Prefer crisp, clinical statements: `"อุณหภูมิลดลงสามองศา ภายในสิบวินาที"` / `"นี่ไม่ใช่เสียงเครื่องปรับอากาศ"`
   - ❌ Avoid theatrical filler (`"กรี๊ดดด!"`, `"โอ้โห น่ากลัวสุดๆ ไปเลยครับ!"`, excessive `!!!` or `???` chains).
2. **TTS Pronunciation & Rhythm Hygiene:**
   - Spell out ambiguous symbols/codes when clarity matters, or keep short tokens cleanly spaced (e.g., `"ซีสิบสาม"` or `"ห้อง C-13"` rather than dense symbol clusters like `~180°C/min@2.5x`).
   - Use **spaces** or **`…`** intentionally for short breath beats (`chunkSubs` converts `...` to `… ` and strips `‘’'`).
   - Avoid mixing long unspaced English phrases mid-Thai sentence; keep English terms short and isolated when needed.
3. **Timing & `<= 60s` Shorts Budgeting:**
   - Every voiced scene adds `LEAD = 0.35s` before speech starts + trimmed `voDur` + `pad` (default `0.5s`, typical `0.6s`).
   - Silent tension beats (`vo: ""`) last exactly `s.pad` seconds (skip `LEAD`).
   - Because `atempo=1.12` compresses raw narration, keep `episode.js` scenes tight (~12–28 Thai words per standard beat) so Acts stay comfortably under the `58s` `shorts_planner.js` ceiling.

---

## 2. Built-In Procedural SFX Catalog (`generate_audio.js`)

All built-in `SFX` cues are deterministic `44.1 kHz` math/noise synthesis (`Math.sin` + `nz()`). Never invent key names outside this table unless defined in `episode.CUSTOM_SFX`.

### 2.1 UI, HUD & Transition Cues
| Key | Duration | Sonic Character | Best Visual / Narrative Trigger |
| :--- | :--- | :--- | :--- |
| `ui_pop` / `pop` | `0.45s` | Crisp dual-sine chime (`1320 + 1980 Hz`, `4ms` attack) | **Sync `at === hud.at`** whenever a Rule / Math / Inventory / Anomaly card pops in |
| `tick` | `0.06s` | Ultra-short metallic click (`2400 Hz`) | Countdown ticks, clock seconds, telemetry updates, calculator steps |
| `tap` | `0.05s` | Dry transient noise tap (`8ms` decay) | Finger tap, switch press, subtle UI micro-cue |
| `chapter` | `1.4s` (auto `+0.2s` offset) | Warm major-fifth chime (`523 + 784 Hz`) | Always set `at: 0` on scenes with `s.chapter` (internal function already adds `+0.2s` delay) |
| `alert` | `~0.64s` | 2-cycle high-low warning beep (`880 → 660 Hz`) | `WARN` / `DANGER` / `CRITICAL` status flip or emergency alarm |
| `poweroff` | `0.45s` | Falling pitch sweep (`800 → 215 Hz`) | Flashlight dying, generator/monitor shutoff |

### 2.2 Action & Prop Foley Cues
| Key | Duration / Options | Sonic Character | Best Visual / Narrative Trigger |
| :--- | :--- | :--- | :--- |
| `step` | `0.18s` (`amp=0.12`) | Muffled low foot thud (`85 Hz` + filtered thump) | Single deliberate footstep |
| `steps` | `dur` (def `3.5s`), `{ slow: true, dur }` | Stepping cadence (`0.55s` normal, `0.9s` slow) | Walking corridors, stalking approach (`slow: true` for entity) |
| `squeak` | `~0.93s` | 2 vibrato chirps (`1750 ± 420 Hz`) | Rusty cart/hinge, wet sneaker squeak |
| `mop` | `0.90s` | Band-shaped noise sweep (`sin²` swell) | Janitor mopping, wiping glass/floor |
| `tape` | `0.55s` | High-passed tearing/pulling friction | Pulling tape, ripping plastic/paper |
| `marker` | `~0.60s` | 3 short dry-marker strokes (`2900–3500 Hz`) | Writing notes, circling map/clues |
| `door` | `0.70s` | Heavy low slam (`thunk` @ `amp=0.38`) | Heavy door/hatch closing |
| `creak` | `1.20s` | Low-mid wobbly wood groan (`190 ± 35 Hz`) | Opening door, floorboards, boat hull |

### 2.3 Tension, Impact & Environment Cues
| Key | Duration / Options | Sonic Character | Best Visual / Narrative Trigger |
| :--- | :--- | :--- | :--- |
| `thunk` | `0.70s` (`amp=0.35`) | Punchy sub-thud (`60 → 40 Hz` + click) | Object drop, muffled distant knock |
| `rumble` | `4.2s` (`2.6s` if `{ soft: true }`) | Deep brown noise + `32 Hz` sub-bass swell | Approaching threat, building vibration, storm build |
| `blackout` | `~1.84s` | 4 decaying `thunk` hits (`0.38s` apart) | Sequential breaker/corridor lights shutting off |
| `flicker` | `0.70s` | `120 Hz` fluorescent hum gated @ `11 Hz` + crackle | Overhead tube flickering (`flick()` visual sync) |
| `boom` | `2.40s` | Deep sub drop (`48 → 14 Hz` + filtered tail) | Major climax reveal, sudden hazard shock |
| `sting` | `2.20s` | Dissonant `880 + 932 Hz` minor-2nd + `thunk` | Chilling anomaly realization, twist reveal |
| `heartbeat` | `~3.60s` | 4 lub-dub pulse cycles (`52 / 46 Hz`, `0.9s` gap) | Hold-breath stealth, rising proximity tension |
| `wind_gust` | `2.20s` | Swelling low-pass wind rush | Cold draft, sudden blizzard/door gust |
| `water_lap` | `1.00s` | Soft decaying water splash/lap | Pier waves, wading, boat hull slap |
| `ice_crack` | `~0.30s` | 5 sharp rapid snaps (`2400 Hz` + noise) | Freezing surface crack, brittle ice/glass stress |

---

## 3. Built-In Ambient Beds (`s.bed`)

Every bed automatically crossfades (`F = 0.4s`) across scene boundaries:

| `s.bed` Key | Sonic Texture | Recommended Scene Context |
| :--- | :--- | :--- |
| `hum` | `100 + 200 Hz` electrical buzz + soft air noise | Lit hallways, offices, control rooms, convenience stores (Theme A) |
| `dark` / `tense_pulse` | `43 + 45.5 Hz` (`2.5 Hz` beating pulse) + deep brown noise | Stalking encounters, blackout corridors, high-danger dilemmas |
| `wind` / `freeze` | Undulating `0.28 Hz` wind + `68 Hz` cold drone | Sub-zero storms, snowy exteriors, unheated shelters (Theme B) |
| `water` / `pier_waves` | Rolling `0.16 Hz` wave swell + `54 Hz` deep hum | Flooded basements, piers, open water, coastal night |
| `parking` / `interior_hush` / `silence_room` | Quiet brown noise + subtle `58 Hz` room tone | Eerie dead-quiet rooms, stealth hiding, night-shift stillness (Theme C) |
| `none` | Pure silence | Cold-open shock pause or isolated pre-climax drop |

---

## 4. Mixing Physics & Cue Placement Rules (Critical)

`generate_audio.js` applies **35% sidechain ducking** to `sfxMix` whenever `vo` is speaking (`duck = 1 - Math.min(0.35, envF * 0.45)`), followed by soft `tanh` limiting and `-18.0 LUFS` mastering.

Follow these 4 placement rules when authoring `episode.js`:
1. **Lock `ui_pop` / `tick` / `chapter` to Visual Triggers:** Set `sfx` `at` equal to `hud.at` (or `0` for `s.chapter`) so sound and animation hit together.
2. **Never Bury Heavy Impacts Under Dense Speech:** Place `boom`, `sting`, `blackout`, or `door` in `LEAD` (`at: 0.05–0.25`), during `pad` (near end of scene), or inside a silent beat (`vo: ""`, `pad: 1.5–2.5`) so ducking does not mute the punch.
3. **Avoid Stacking Long Tails:** Do not trigger `rumble` (`4.2s`), `heartbeat` (`3.6s`), and `boom` (`2.4s`) on top of each other within the same short scene; alternate `bed: 'dark'` with targeted single-impact cues.
4. **Writing `episode.CUSTOM_SFX` / `episode.CUSTOM_BEDS`:** When a story needs a signature sound (e.g., radio static, sonar ping, metallic latch), layer **3 components** instead of a raw single sine wave: `(1) short noise transient + (2) pitch-swept body tone + (3) smooth exponential/sin² envelope`. Both hooks receive `{ SR, add, nz, env, TAU, rnd }` in their 4th argument:
   ```javascript
   const CUSTOM_SFX = {
     sonar_ping(m, t, cue, { add, env, nz, TAU }) {
       add(m, t, 0.8, x => 0.14 * env(x, 0.003, 0.22) * Math.sin(TAU * (1480 - 120 * x) * x) + (x < 0.015 ? 0.05 * nz() : 0));
     }
   };
   const CUSTOM_BEDS = {
     reactor_room(m, t0, t1, { add, nz, TAU }) {
       let lp = 0;
       add(m, t0, t1 - t0, x => { lp += 0.02 * (nz() - lp); return 0.025 * Math.sin(TAU * 60 * x) + 0.04 * lp; });
     }
   };
   module.exports = { TITLE, SUBTITLE, SCENES, CUSTOM_SFX, CUSTOM_BEDS };
   ```

---

## 5. Profiling New or Custom Audio Samples (Optional)

If the user supplies a reference voice clip or custom `.wav`/`.mp3` SFX sample and wants the agent to learn how it sounds before scripting:
1. Measure duration, peak/integrated loudness, and silence segments using `ffprobe` / `ffmpeg -af volumedetect` or `ebur128`.
2. When multimodal audio inspection is available, listen directly to characterize **pitch register, speaking cadence (syllables/sec), emotional restraint, breath pauses, and harsh frequencies**.
3. Save the distilled characteristics as a reusable **Voice/SFX Profile** note (matching Sections 1–3 above) before drafting `episode.js`.
