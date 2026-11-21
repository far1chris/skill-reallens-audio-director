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

## 1. Narrator Voice Profile (`Puck`) & Character / Impersonation Exception System

### 1.1 Exact Vocal Chain & Sonic Character
- **Voice & Model Priority:** Gemini Interactions TTS (Narrator locked to `voice: "Puck"`), executed in **parallel workers across `GEMINI_API_KEYS`** (bounded up to 3 concurrent keys with `dailyExhausted` per-model/key tracking and single `lockedModelId` across the entire episode):
  1. `gemini-3.8-flash-tts` (uses `annotations: [{ type: 'speech_metadata', style: NARRATOR_STYLE }]`)
  2. `gemini-3.8-flash-lite-tts` (uses `speech_metadata`)
  3. `gemini-3.1-flash-tts-preview` (**must omit** `annotations` / `speech_metadata`, uses `response_format: { type: 'audio' }`)
- **Base Narrator Style Directive (`UNIFIED_STYLE` / `NARRATOR_STYLE`):**
  > `"same single male Thai narrator (Puck) across all scenes, locked low-mid baritone pitch register, calm deadpan documentary tone, steady even cadence, consistent close-mic proximity, zero pitch spikes or theatrical shouting"`
- **Post-Processing Signature (`unifyVocalTone` — Two-Pass Linear EBU R128 + `dynaudnorm`):**
  - **Speed & EQ (Narrator Lock):** `atempo=1.12` (**+12% faster**), `highpass=f=82`, `lowpass=f=11800`, **`+1.4 dB @ 175 Hz`** (chest warmth lock), **`-1.0 dB @ 420 Hz`** (mud control), **`-1.4 dB @ 3350 Hz`** (softens Thai sibilants `ซ/ส/ช`), `dynaudnorm=f=120:g=5:p=0.92:m=3.0`, `acompressor` (`-21 dB`, `3.8:1`, `makeup=2.5`), followed by **Two-Pass `loudnorm`** (`linear=true` measured pass -> target **`-18.0 LUFS`**, **`LRA=3.5`**, `TP=-1.5` for Horror; **`-16.0 LUFS`**, **`LRA=3.5`** for Thai Sitcom).
- **How This Voice "Feels" to Listeners:**
  - Detached, forensic, coldly rational male documentary host—never melodramatic, screaming, or cartoonishly spooky.
  - Tension comes from **short declarative sentences, concrete numbers, and sudden quiet pauses**, not exclamation marks.

### 1.2 Writing `scene.vo` for `Puck` + `chunkSubs()`
Because `scene.vo` feeds **both** Gemini TTS synthesis and `chunkSubs()` (`Intl.Segmenter('th')`, `maxLen = 24`, short-tail merge ceiling `<= 26` chars/cue for fast 1.2–1.8s vertical reading cadence):
1. **Write for Deadpan Analytical Delivery:**
   - ✅ Prefer crisp, clinical statements: `"อุณหภูมิลดลงสามองศา ภายในสิบวินาที"` / `"นี่ไม่ใช่เสียงเครื่องปรับอากาศ"`
   - ❌ Avoid theatrical filler (`"กรี๊ดดด!"`, `"โอ้โห น่ากลัวสุดๆ ไปเลยครับ!"`, excessive `!!!` or `???` chains).
2. **TTS Pronunciation & Rhythm Hygiene:**
   - Spell out ambiguous symbols/codes when clarity matters, or keep short tokens cleanly spaced (e.g., `"ซีสิบสาม"` or `"ห้อง C-13"` rather than dense symbol clusters like `~180°C/min@2.5x`).
   - Use **spaces** or **`…`** intentionally for short breath beats (`chunkSubs` converts `...` to `… ` and strips `‘’'`).
   - Avoid mixing long unspaced English phrases mid-Thai sentence; keep English terms short and isolated when needed.
3. **Zero-Delay Cold Open, Standalone `45–60s` Budgeting & Part 2 3s Recap:**
   - **Zero-Delay Cold Open & Part 2 Recap (`0.06s`):** `SCENES[0]`, `isClimax: true`, `act` openers, and **`isRecap: true`** scenes (Part 2 3s recap) automatically use `COLD_OPEN_LEAD = 0.06s` and display the **first subtitle cue at `t0 = 0.000s`** (pre-roll visual hook for muted feed scrollers).
   - **Standard Cut Timing:** Standard scenes use tight J-cut pacing (`LEAD = 0.20s` Horror / `0.18s` Sitcom, `DEFAULT_PAD = 0.28s` Horror / `0.25s` Sitcom) — override with explicit `s.lead` or `s.pad` only for intentional suspense/comedic silence.
   - Silent tension beats (`vo: ""`) last exactly `s.pad` seconds (skip `LEAD`).
   - Because `atempo=1.12` compresses raw narration, **keep standalone Shorts stories tight (~6–9 concise scenes, total `45–58s`)** so the hook, context, and payoff resolution conclude cleanly in **1 clip**. If genuinely longer (**max 2 EPs total**), reserve the **first 3 seconds (~8–12 Thai words)** of `act: 2` (`isRecap: true`) for a snappy `"ความเดิมจากตอนที่แล้ว..."` summary before jumping to the resolution.
   - **High-Retention Shorts Ending (`<= 5s` — NEVER Say `"ฝากกดไลก์ กดแชร์ กดติดตาม"`):** Close Shorts with a **(1) Side-Choosing Open Question** (`2–3s` debate prompt tied to the story), **(2) 1-Sentence Cliffhanger or Full-Video Bridge** (`"ฟังเรื่องเต็มแบบไม่ตัด ได้ที่คลิปยาวหน้าช่อง"`), and leave pure channel CTA to the silent **`drawShortsEndingCTA`** graphic in the final `2.25s` so listeners never detect a generic sign-off and swipe away early.

### 1.3 Strict Narrator Vocal Tone Lock (`Puck` + `sanitizeVoForTTS` + Two-Pass `LRA=3.5`)
To ensure the main narrator **never** shifts pitch, timbre, or emotional register across scenes:
1. **Narrator Locked Across All Narration Scenes:** Never override `voice` or `speech_metadata` on narrator lines (`role: 'narrator'`, or default scenes without `speaker`/`impersonate`). Every narrator segment uses `VOICE_NAME = 'Puck'` and the exact same `UNIFIED_STYLE` (`NARRATOR_STYLE`) string.
2. **Strip Pitch-Spiking Punctuation on Narrator Lines (`sanitizeVoForTTS(text, 'narrator')`):** Gemini TTS raises pitch or shouts when given `?!`, `!!`, `!`, `—`, or `...`. For `narrator` lines, `generate_audio.js` strips `[!?؟…~—–:;]+` before calling Gemini TTS (keeping calm breath spaces), while `stripVoMarkupForSubs()` + `chunkSubs()` preserves original punctuation cleanly in subtitles.
3. **Single-Model Episode Lock (`lockedModelId`):** The first uncached scene locks `lockedModelId` (prioritizing `gemini-3.8-flash-tts`, defaulting to `TTS_MODELS[0].id` before workers start) across all API keys and retries so an episode never mixes two TTS models mid-video.
4. **Two-Pass EBU R128 + `dynaudnorm` (`LRA=3.5`):** Every narrator WAV runs through fixed chest/sibilance EQ, dynamic range tightening, `3.8:1` compression, and **Two-Pass linear `loudnorm`** (`VOICE_NAME` + `UNIFIED_STYLE` hashed into SHA-256 cache key).

### 1.4 Explicit Exceptions: Other Characters (`role: 'character'`) & Narrator Impersonations (`role: 'impersonate'`)
Voice tone and speaker changes are **strictly allowed ONLY** when explicitly representing another character or the narrator impersonating someone else—never on ordinary narrator prose:
1. **Other Characters (`role: 'character'` or `speaker: '<id>'`):**
   - Built-in `DEFAULT_CHARACTER_VOICES` (extensible via `episode.CHARACTER_VOICES`):
     - `yai_nuan` / `elder_female` / `grandma` / `auntie` $\rightarrow$ `Aoede` (`elder_female`)
     - `fah` / `female` / `woman` / `mother` / `nurse` $\rightarrow$ `Kore` (`female`) / `Aoede`
     - `clinic_staff` / `sales_girl` / `cashier` $\rightarrow$ `Kore` (`sweet_sales`)
     - `boss` / `elder_male` / `guard` / `shaman` / `monk` / `deep_male` $\rightarrow$ `Charon` (`deep_male`)
     - `barista` / `rider` / `friend_male` / `vendor_male` / `young_male` $\rightarrow$ `Fenrir` (`young_male`)
     - `ghost` / `entity` / `whisper` $\rightarrow$ `Aoede` (`whisper_ghost`)
   - Unknown `speaker` names deterministically hash into `['Kore', 'Charon', 'Aoede', 'Fenrir']` so each character keeps an identical voice across every scene they speak.
2. **Narrator Doing a Voice Impression (`role: 'impersonate'` or `impersonate: '<desc>'`):**
   - Keeps the narrator's base voice (`'Puck'` by default) while switching `speech_metadata` to vocal acting (`impersonate`), preserving `?`, `!`, `…`, and applying optional acoustic `voicePreset` (`'sweet_sales'`, `'elder'`, `'deep'`, `'excited'`, `'whisper_ghost'`, `'phone_speaker'`) at `LRA=4.5`, before snapping right back to locked deadpan `'Puck'` (`LRA=3.5`) on the next narrator segment.
3. **Mixing Narrator + Character / Impersonation Inside One Scene (`voSegments`, Inline `[[...]]`, Auto-Quote Split):**
   - **Structured array (`scene.voSegments`):** `[{ role: 'narrator', text: '...' }, { role: 'character', speaker: 'yai_nuan', text: '...' }, { role: 'impersonate', impersonate: 'เสียงหวานพนักงานคลินิก', preset: 'sweet_sales', text: '...' }]`
   - **Inline markup in `scene.vo`:** `"พนักงานรีบบอก [[char:clinic_staff|โปรผ่อน 0% นะคะ!]] แล้วต้นก็เดินออกมา"` or `"[[impersonate:เสียงดุเลียนแบบหัวหน้า|พรุ่งนี้ส่งก่อนแปดโมง!]]"` (`stripVoMarkupForSubs()` automatically strips `[[...|...]]` tags from subtitles!).
   - **Speech-verb quote auto-split:** Phrases like `พูดว่า "..."` / `เล่าว่า "..."` automatically isolate the quoted line as `role: 'impersonate'` (or `quoteSpeaker` / `quoteRole`) while keeping surrounding narrator prose locked deadpan (`autoSplitQuotes: false` disables per scene). Multi-segment scenes normalize each segment separately and concatenate with a crisp `0.10s` pause.

### 1.5 AI Pronunciation & Speech Clarity Detector + Auto-Retry (`normalizeThaiForTTS`, `inspectVoClipClarity`, `--verify-asr`)
1. **Pre-TTS Thai Phonetic & Number/Symbol Normalizer (`normalizeThaiForTTS`):**
   - Automatically expands numbers (`2,490` $\rightarrow$ `สองพันสี่ร้อยเก้าสิบ`, decimals `.50` $\rightarrow$ `จุดห้าศูนย์`), Thai time (`03:14 น.` $\rightarrow$ `สามนาฬิกา สิบสี่นาที`), currencies & units (`฿` / `บ.`, `%`, `ชม.`, `กม.`, `ซม.`, `มม.`, `มล.`, `ม.`, `°C`, `dB`), abbreviations (`กทม.`, `รพ.`, `ฯลฯ`), English acronyms (`LINE`, `CCTV`, `GPS`, `AI`, `ATM`, `OTP`, `VIP`, `KPI`, `7-Eleven`), and Mai Yamok spacing (`ๆ `) **only for the TTS input** while keeping raw concise symbols intact in visual subtitles.
   - Authors can supply episode-wide overrides via `episode.PRONUNCIATION_MAP` or per-scene phonetic spelling via `scene.ttsVo` (leaving `scene.vo` for subtitles).
2. **Post-TTS Acoustic Clarity, Truncation & Dropout Detector (`inspectVoClipClarity`):**
   - Audits every generated/cached WAV (`24 kHz` normalized PCM) for speaking-rate anomalies (`CUTOFF_OR_SKIPPED` `> 22.0 ch/s`, `FAST_RUSHED_SPEECH` `> 19.0 ch/s`, `DRAWL_OR_BABBLE` `< 5.2 ch/s`), mid-sentence dropouts (`INTERNAL_DROPOUT_GAP` `> 1.25s`, `LOW_VOICE_ACTIVITY` `vadRatio < 0.48`), swallowed endings (`MUMBLED_TAIL` `tailDropDb > 10.5 dB`), muffled speech/hiss (`MUFFLED_SPEECH` `zcr < 0.022`, `HIGH_FREQ_NOISE` `zcr > 0.30`), and clipping (`> 0.5%`), scoring `0–100` (`PASS >= 82`, `WARN 70–81`, `FAIL < 70`) and writing `audio/pronunciation_report.json`.
3. **Optional Gemini Multimodal ASR Verification & Auto-Retry:**
   - Fresh takes failing `inspectVoClipClarity` (or `--verify-asr` Gemini `2.5-flash` round-trip transcription check `< 0.78` similarity / mispronounced word flag) automatically trigger up to 2 self-healing retries with spaced Thai clauses (`buildClearerRetryText`), caching the highest-scoring take.
   - CLI flags: `node generate_audio.js --audit-only` (free cached/WAV audit without regenerating audio), `--fix-unclear` (re-roll cached `FAIL` scenes), `--repronounce` (re-synthesize digits/abbreviations with `normalizeThaiForTTS`), `--verify-asr` (enable multimodal ASR transcript verification).

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

## 4. Mixing Physics & Dual-Bus Cue Placement Rules (Critical)

`generate_audio.js` uses **Dual-Bus Sidechain Ducking** followed by soft `tanh` limiting and EBU R128 mastering (`-18.0 LUFS` Horror / `-16.0 LUFS` Sitcom):
- **`bedMix` (up to `42%` ducking under speech):** All `s.bed` tracks and continuous/looping cues (`steps`, `mop`, `rumble`, `wind_gust`, `water_lap`, `heartbeat` / Sitcom `keyboard`, `slurp`) duck cleanly beneath narration so voice clarity stays sharp.
- **`impactSfxMix` (`<= 10%` max ducking):** Transient stingers and punchline cues (`ui_pop`, `boom`, `sting`, `blackout`, `flicker`, `thunk`, `door`, `alert`, `ice_crack`, `CUSTOM_SFX`, plus Sitcom `rimshot`, `record_scratch`, `vine_boom`, `cash_register`, `line_ping`, `chat_pop`) stay crisp and loud even when synced directly over a key spoken word!

Follow these 4 placement rules when authoring `episode.js`:
1. **Lock `ui_pop` / `tick` / `chapter` / Punchline SFX to Visual Triggers:** Set `sfx` `at` equal to `hud.at` (or `0` for `s.chapter` / `snapAt`) so sound, camera snap-zoom, and animation hit together.
2. **Place Cold-Open Hooks at `at: 0.00–0.15s`:** Because `impactSfxMix` is barely ducked (`<= 10%`), you can hit a sudden sting/scratch/knock right as `SCENES[0]` speech opens at `0.06s` — reserve silent beats (`vo: ""`, `pad: 1.5–2.2`) for deep cliffhanger pauses or heavy `rumble` tails.
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
