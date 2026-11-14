# Real Lens Audio Director (Antigravity Skill)

[![Antigravity Skill](https://img.shields.io/badge/Antigravity-Skill-orange.svg)](#)
[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](https://opensource.org/licenses/MIT)

Voice persona (`Puck`), Thai narration hygiene (`Intl.Segmenter('th')`), procedural SFX & ambient bed bible, and sidechain ducking / EBU R128 (`-18.0 LUFS`) mastering guide for **Real Lens (@real_lens1)** survival horror animations in **Google Antigravity**.

---

## 📖 Key Capabilities

- **Unified `Puck` Narrator Chain:** Priority order (`gemini-3.8-flash-tts` → `gemini-3.8-flash-lite-tts` → `gemini-3.1-flash-tts-preview`), deadpan documentary style directive (`UNIFIED_STYLE`), `atempo=1.12` cadence, warm chest EQ (`+1.2 dB @ 180 Hz`), de-essing (`-1.0 dB @ 3500 Hz`), and `-18.0 LUFS` normalization.
- **Thai Voiceover & Subtitle Hygiene (`chunkSubs`):** Clinical writing rules, number/symbol spelling for clean Thai TTS pronunciation, word-boundary segmentation (`maxLen = 34`, short-tail ceiling `<= 36` chars), and `<= 60s` Shorts timing budgets.
- **Procedural SFX & Ambient Bed Catalog:** Complete acoustic breakdown of all built-in `44.1 kHz` procedural sound effects (`ui_pop`, `tick`, `tap`, `chapter`, `alert`, `poweroff`, `step`, `steps`, `squeak`, `mop`, `tape`, `marker`, `door`, `creak`, `thunk`, `rumble`, `blackout`, `flicker`, `boom`, `sting`, `heartbeat`, `wind_gust`, `water_lap`, `ice_crack`) and crossfaded ambient beds (`hum`, `dark`/`tense_pulse`, `wind`/`freeze`, `water`/`pier_waves`, `parking`/`interior_hush`/`silence_room`).
- **35% Sidechain Ducking & Custom Synthesis Contract:** Cue placement rules around `LEAD` (`0.35s`) and `pad` so heavy impacts aren't buried under speech, plus 3-layer synthesis patterns for `episode.CUSTOM_SFX` and `episode.CUSTOM_BEDS` (with `{ SR, add, nz, env, TAU, rnd }` helper injection).

---

## 🚀 Installation in Google Antigravity

```powershell
git clone https://github.com/far1chris/skill-reallens-audio-director.git "$HOME\.gemini\config\skills\reallens-audio-director"
```

---

## 📄 License

MIT License. See [LICENSE](LICENSE) for details.
