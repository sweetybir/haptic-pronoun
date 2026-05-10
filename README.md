# 代词 · PRONOUN — Project Handbook

> **If you are Claude Code (or any other AI collaborator) starting a new session on this project, read this entire document before editing anything.**

This is a small series of art/research prototypes by Bailey, exploring mediated presence across different sensory channels. Each "work" in the series is one of those channels.

---

## TL;DR

- Repo hosts the **《代词》(Pronoun)** series — currently two works plus a portal.
- Each work is **one self-contained HTML file**. No build step, no bundler, no package manager.
- Files:
  - `index.html` — series portal (the front door at `/`)
  - `haptic.html` — 触觉代词 / Haptic Pronoun (two-device touch piece, **WebGL**)
  - `vocal.html` — 声音代词 / Vocal Pronoun (two-device voice piece, **Canvas 2D + Web Audio**)
- Two-device pairing uses **PeerJS** (WebRTC) with a 4-character room code.
- Deployed via **GitHub Pages** at `https://sweetybir.github.io/haptic-pronoun/`.
- Bailey communicates in mixed Chinese/English and prefers **concise, high-signal** responses.

---

## House rules (read carefully)

### 1. Additive-only by default

**Do not remove or "clean up" existing features unless explicitly asked.**

This is the single most important rule. Bailey iterates by adding and refining. Code that looks redundant or "unused" is often there because earlier user testing required it. Examples that LOOK removable but ARE NOT:

- The `localFading` / `remoteFading` arrays in haptic.html (visual ghost after lift)
- `iceCandidatePoolSize: 10` in PEER_CONFIG
- The `iceMonitorInterval` machinery (debug aid for stuck states)
- The state-label UI on each work
- The portal back-link in each work
- The colored tap-dots in haptic.html (each tap leaves a unique fleeting hue)

If unsure: **ask first.**

### 2. Single-file architecture is intentional

Each work has all CSS and JS inline. **Do not split into separate JS/CSS files.** Reasons:
- Bailey's deploy workflow is manual file replacement via GitHub Desktop
- Single-file prototypes are portable, archivable, and forkable
- The refactoring friction isn't worth the structural neatness

### 3. Don't break the paper-ink aesthetic

Shared visual language across all works:

| Element | Convention |
|---|---|
| Background | `#f2efe9` (paper cream) — NOT pure white |
| Ink | very dark warm values like `rgba(15,10,8,...)` — NOT pure black |
| Typography | `'Bricolage Grotesque', 'Songti SC', 'STSong', 'SimSun', serif` — Bricolage loaded from Google Fonts CDN (variable axes opsz/wght). EN/Latin → Bricolage; CN/CJK → Songti chain. No italic axis on Google Fonts → `font-style: italic` is browser-synthesized oblique. |
| UI text size | 9–11px |
| UI letter-spacing | 0.15–0.25em |
| UI opacity | 0.3–0.5 (deliberately faded) |
| English caps | UPPERCASE with letter-spacing for tags / labels |
| Chinese spacing | character-level spacing for atmosphere (e.g. `创 建 房 间`) |
| Paper grain | SVG `fractalNoise` overlay, opacity 0.06, `mix-blend-mode: multiply` |

What we DO NOT use: emoji, drop shadows, rounded corners on UI elements, gradient backgrounds (color gradients are reserved for the haptic state machine specifically), modern Sans fonts, icons.

When adding new UI elements, mimic existing ones. Don't introduce new fonts, colors, icon styles, or motion vocabularies without asking.

### 4. Don't suggest production best practices that contradict artist choices

This is an art project. Skip these suggestions:
- "Credentials are exposed in client code" — known and deliberate (see Networking)
- "Should use a build system" — no
- "Should use TypeScript" — no
- "Should add comprehensive accessibility" — ask first; some are appropriate, many break the aesthetic
- "Should split into modules" — no
- "Should switch to a frontend framework" — no

You may raise a concern once. Don't re-litigate after Bailey says no.

### 5. Response style

Bailey prefers:
- Telegraphic, mixed Chinese/English code-switching
- **Surfacing design tensions BEFORE executing** (especially for aesthetic / conceptual changes)
- One question at a time, max
- No throat-clearing, no "I'd be happy to help"
- Treat them as the artist — your job is to be a clear-headed implementer with taste, not a feature-suggester

When proposing changes that affect aesthetic or concept: name the trade-off, propose 2–3 options with what each preserves/loses, recommend one, then wait.

---

## Repo layout

```
haptic-pronoun/          ← repo name (kept for URL stability)
├── index.html           ← 代词 PRONOUN — series portal
├── haptic.html          ← 触觉代词 — touch piece
├── vocal.html           ← 声音代词 — voice piece
└── README.md            ← this file
```

The repo is still named `haptic-pronoun` even though it now hosts the whole series. Renaming is deferred to avoid breaking the deployed URL. Don't suggest renaming unless asked.

---

## Tech stack

| Concern | Tool |
|---|---|
| Pairing / signaling | PeerJS 1.5.4 (CDN) |
| Networking transport | WebRTC data channels |
| NAT traversal | STUN (Google + Cloudflare + Metered) + TURN (Metered authenticated) |
| Haptic rendering | WebGL 1.0, single-pass GLSL fragment shader |
| Vocal rendering | Canvas 2D |
| Audio analysis | Web Audio API (AnalyserNode, time + frequency domain) |
| Deployment | GitHub Pages (static) |
| Target browsers | iOS Safari (primary), Chrome Android, desktop Chrome/Safari |

---

## Networking — shared across all works

Both works use the same pairing pattern:

1. Host clicks "创建房间" — PeerJS registers an ID with the public PeerJS cloud
2. Joiner enters the 4-char code — PeerJS connects to that ID
3. WebRTC ICE negotiates a peer-to-peer data channel
4. Per-work message protocol flows over the data channel

### PEER_CONFIG

Both `haptic.html` and `vocal.html` contain an identical `const PEER_CONFIG = {...}` block with ICE servers. **If you change one, change both.**

Current ICE configuration:
- STUN: Google (×2), Cloudflare, Metered
- TURN: Metered with static credentials (username/credential strings inlined)

The Metered credentials ARE intentionally embedded in client-side source. This is known and accepted — the project's traffic is too low to make abuse practical, and rotation via the Metered dashboard is trivial if needed.

If TURN ever needs replacement: dashboard at https://dashboard.metered.ca/turnserver. Earlier we used `openrelay.metered.ca` with anonymous credentials — those died in 2024 and should not be used.

### Room ID prefixes

- haptic: `hpnn-` (e.g. `hpnn-AB23`)
- vocal: `vp-` (e.g. `vp-AB23`)

These prefixes namespace the two works so a vocal joiner can't accidentally land in a haptic room. **Keep prefixes distinct in any future works.**

### Common failure modes during testing

| Symptom | Probable cause |
|---|---|
| "通道未通 · 网络受限" | ICE timed out. Check TURN is alive; check iOS Private Relay isn't on; check WiFi AP isolation. |
| "代号不存在" | PeerJS signaling reached but host wasn't registered. Usually fine on retry. |
| "服务不通 · 刷新重试" | PeerJS public cloud is having an outage. Wait or refresh. |
| Same WiFi fails, hotspot works | Router has client isolation. Should still work via TURN relay. If not, try `iceTransportPolicy: 'relay'` to force relay-only mode. |

---

## haptic.html — 触觉代词

### What it does

Two participants each hold a device. Touching the screen produces ink-like fields rendered via a WebGL fragment shader. Local touches and remote touches are visually distinguishable (remote = softer, more diffuse via a `virtue` parameter that controls Gaussian falloff and noise distortion).

### Visual / mathematical model

Touches are **Gaussian potential fields** with **fBm noise** distortion. Each frame:
1. Active touches + remote touches + trails + bridges + fading ghosts are packed into a **32-slot uniform array** (`MAX_SLOTS = 32`)
2. The fragment shader sums Gaussian contributions per pixel
3. Output mixes ink (dark) onto paper (cream), with optional warm/cool tint driven by the `uWarmth` state machine

### Color state machine (`stepWarmth`)

Presence-based, not merge-based:

- `warmth` ranges **`0` (solo, cool blue) ↔ `1` (duet, warm orange)**
- `target = 1` when **both sides are touching** (`localActive.size > 0 && remoteActive.size > 0`); otherwise `target = 0`
- Eases toward `target` at ~500ms time constant (`dt * 2.5`)
- **No black/grey midtone** — direct cool→warm interpolation in the shader (`mapInk`)
- **Brief taps** still leave colored dots (random hue from `PALETTE`) — independent of the warmth machine
- The `merged` flag (cross-device touches at similar positions) is still computed, but **only drives the state-label text** (`相 遇 · MEET`); it no longer affects color.

### Cross-device size consistency

- Touch radius normalization in `norm()` divides by **`r.height`**, not `r.width` — because the shader's pos space is normalized so 1 unit = canvas height. Dividing by width caused iPad/iPhone discrepancies (same physical finger appeared smaller on the wider device).
- When `t.radiusX` is missing or absurdly small (iPad often reports `0`), it falls back to `FINGER_PX = 28` CSS pixels.

### Hot spots in the code

- **`MAX_SLOTS = 32`** — WebGL uniform array size. If you "need more slots," instead consider whether trail samples should age out faster, not raising the cap.
- **`emitTrail` function** — head-attached trail interpolation. An earlier version had a "trail lags behind live touch" bug; the current implementation lays samples from the previous trail head all the way TO the current touch position each frame. **Do not break this invariant.** `SPACING` is `radius * 0.2` for dense overlap; `0.3` left visible gaps on fast strokes.
- **`mapInk` function in the shader** — direct `mix(coolG, warmG, w)` between the cool steel-blue gradient (`C0/C1/C2`) and the warm amber gradient (`W0/W1/W2`). No midtone path.
- **`virtue` parameter** — represents real-vs-virtual / local-vs-remote. Higher virtue = softer edges + more noise displacement.
- **Velocity LP filter in `stepPoint`**: `dt * 25` (was `dt * 12`). Tightened to make the visual mark follow the finger more closely. Going much higher amplifies jitter.
- **Remote position LP**: `dt * 18` for x/y, `dt * 12` for radius. Tightened from `dt * 12 / dt * 8` for less perceived lag while still smoothing network jitter.

### Open design questions (do not auto-resolve)

These are surfaced from user testing and remain unresolved:
- The definition of "low pressure" interaction has conflicting interpretations
- Whether to add an **asynchronous trace-leaving mode** (surprise via residue, not just synchronous coincidence)
- ~~How the color state machine should evolve to support a **friendship** relationship framing~~ — **Resolved 2026-05-10**: replaced merge-driven warmth with presence-based blue (solo) ↔ orange (duet). The "couple/synchrony-required" semantic is gone; "just being there together" now drives the warm state.

---

## vocal.html — 声音代词

### What it does

Two participants speak into their devices. The system **transmits no audio and no text** — only paralinguistic data: volume (RMS), pitch (FFT peak in 80–500 Hz vocal range), tremor (rolling variance of volume), and active/silent state from VAD.

A canvas renders a **horizontal scrolling timeline**:
- The "now" is the right edge; history scrolls leftward and fades
- Self's voice draws an ink line bending **UP** from the midline
- Remote's voice draws **DOWN**
- When both speak simultaneously → lines collide into a **muddy black blob** (transient, scrolls away with the rest of history)

### Key concept (do not undermine)

> **Silence is the condition that lets the other develop.** (沉默是让对方显影的条件)

The overlap blob's **ugliness is the point**. Simultaneous speech = visual illegibility = the system embodying its own ethic ("don't speak over each other") at the medium level, not as a UI rule.

Bailey has explicitly considered and **rejected** these alternatives:
- Coloring the overlap (would reverse the work's ethical framing into "overlap is exciting")
- Each speaker having their own color identity (would shift this into a "two-voice harmony" piece — a different work)
- AI-voice-assistant style smooth glow (incompatible with paper-ink aesthetic)

Don't propose these again unless Bailey opens the conversation.

### VAD (Voice Activity Detection)

Tunable parameters at the top of the script:

```js
const VAD_ON      = 0.020;   // RMS threshold to enter "speaking"
const VAD_OFF     = 0.012;   // hysteresis lower bound (must drop below this)
const VAD_END_MS  = 700;     // ms of silence to confirm end of a turn
const VAD_START_F = 2;       // frames above VAD_ON to confirm start
```

Tuned conservatively. Mandarin speech with frequent natural pauses sometimes needs `VAD_END_MS` raised to 900–1000 to avoid splitting one utterance. **Don't auto-retune.** Bailey may want to expose these as user controls eventually.

### Paralinguistic features extracted

Only three, by deliberate design:

- **Volume**: RMS of time-domain samples, soft-curved: `Math.pow(rms * 4.5, 0.85)`
- **Pitch**: FFT peak between 80–500 Hz, log-normalized to 0–1, smoothed with `0.6*old + 0.4*new`
- **Tremor**: rolling variance of recent volume samples, scaled

Adding more features (formants, spectral centroid, etc.) requires concept-level discussion first — the limited feature set is itself a design statement (cf. the same restraint in haptic: only velocity, area, dwell, distance).

### Render structure

Two-pass per-segment draw:
1. **Pass 0**: halo (soft outer ink bleed, 2 stacked thick low-alpha strokes)
2. **Pass 1**: core line (thin crisp ink stroke)

Overlap is drawn separately in `drawOverlap` — concentric stacked ellipses + an irregular polygon with random tremor displacement (the "torn film" look).

### Network protocol

Vocal sends per-frame messages at ~25 Hz with shape:
```js
{ t: 'f', a: active_bool, v: volume, p: pitch, r: tremor }
```
Plus the `{ type: 'bye' }` disconnect signal (shared with haptic).

### Open design questions

- Whether silent gaps should have a "breath" presence (subtle paper motion) or remain fully empty. **Currently: fully empty.**
- Async / leave-a-trace mode (parallel to the same open question in haptic) — should a closed turn leave a faint residue for the other person to "develop" later? Not yet implemented.

---

## index.html — series portal

Static page, no JS logic. Acts as the 扉页 (frontispiece) for the series.

Three slots:
- **I.** 触觉代词
- **II.** 声音代词
- **III.** forthcoming (placeholder for future work)

When a new work is added: replace slot III with the real work and add a new slot IV as the new "forthcoming."

Each work file should contain a `← 代 词` portal link back to `./` in the top of its pairing overlay.

---

## Workflow

Bailey's deploy loop:

1. Edit file locally in a code editor
2. Commit + push via **GitHub Desktop**
3. Wait ~30 seconds for GitHub Pages to redeploy
4. Test on real devices — typically phone + iPad, often over personal hotspot

There is **no staging environment**. Changes go live on push.

Local testing notes:
- Opening the HTML directly in a browser works for most rendering/UI logic
- WebRTC pairing requires actual deployment (or running a local HTTPS server) to test end-to-end
- iOS Safari sometimes has issues that don't show up in desktop Chrome; always test on the actual target device for any UI / touch / audio change

---

## Session decision log

Recent decisions (so a new AI session doesn't accidentally try to unwind them):

- ✅ **vocal.html exists as a SIBLING to haptic.html**, not an extension. Same design DNA, different ethical structure (touch = simultaneity, voice = alternation).
- ✅ **Vocal extracts volume + pitch + tremor only.** No further abstraction. Emotional leakage from pitch/tremor is acceptable.
- ✅ **Overlap blob is TRANSIENT** — it scrolls away with the rest of history. No permanent "damaged paper" mode.
- ✅ **Silence and absence are the same thing visually.** No special "I am listening" indicator. If you're not speaking, your line just isn't being drawn.
- ✅ **Both participants see both traces.** Symmetric visibility. (Considered: making self's own line invisible to self — rejected.)
- ✅ **Turn-based, not continuous.** VAD detects ~700ms of silence to end a turn. One utterance = one continuous line.
- ✅ **index.html is the series portal** (the front door). haptic.html and vocal.html are sub-pages with `← 代 词` links back.
- ✅ **Dead `openrelay.metered.ca` TURN replaced** with Metered authenticated static credentials. Exposure of credentials in client source is accepted.
- ✅ **Overlap stays black.** No color, no AI-voice gradient, no per-speaker color identity. **The ugliness is the work.** Bailey reconsidered and confirmed.
- ✅ **Exhibition text** for vocal.html has been drafted (bilingual; see project files or ask Bailey).

---

## When in doubt

| Situation | What to do |
|---|---|
| Aesthetic question | Check "House rules #3" above. If still unclear, ask. |
| Conceptual question (does this change what the work means?) | **Ask before doing.** |
| Bug fix or pure mechanic change | Just do it, but follow additive-only. |
| "Should we refactor X?" | Almost certainly no. |
| "Should I add this nice-to-have?" | Ask. |
| User asks to remove a feature | Confirm — make sure it's intentional, since additive-only is the norm. |

Bailey is the artist. You are the implementer-with-taste. Surface tensions, don't solve them unilaterally.
