# CLAUDE.md — MERLIN

Context for Claude Code. Read this before doing anything in this repository.

## What this is

**MERLIN** — *Machine for Executing, Reasoning & Listening, Intelligent & Native* — is a local, voice-first AI assistant for the desktop (Windows and Linux, both native). You say "Merlin", speak naturally, and it listens (Faster-Whisper), reasons and acts with tools (local LLM via Ollama), and answers out loud (Piper). A 3D particle orb in the center of the app shows what the assistant is doing.

- Author: Thiago Camerato (GitHub `thiagocamerato757`). Speaks Brazilian Portuguese: **talk to him in Portuguese**; code, comments and commit messages stay in English.
- Repo: https://github.com/thiagocamerato757/M.E.R.L.I.N — license **GPL-3.0** (deliberate; do not suggest MIT).
- Formerly called **JAX**. The old repo `thiagocamerato757/Jax` is being deleted; MERLIN restarts here from scratch. Never reintroduce the name JAX.
- "Merlin" is also the **wake word**. App identifier: `com.cameratothiago.merlin`. The name lives in **one constant** (`ASSISTANT_NAME`), so renaming costs one line.
- Inspiration/benchmark: [I.S.A.A.C.](https://github.com/Matheussoranco/I.S.A.A.C), a colleague's agent. MERLIN aims to beat it where it is weak: **natural, offline voice conversation in Portuguese**.

### Current state (Oct 20, 2026)

The repo restarted from scratch: only `README.md`, `LICENSE` and `assets/`. **Focus is the backend first; the frontend (`app/`) is postponed.**

- The backend is built **CLI-first**: each piece (record, transcribe, ask the LLM, speak) runs from the terminal on its own, then they are chained into a terminal voice loop. The WebSocket server comes when the UI comes back.
- Thiago develops in **WSL Ubuntu** (`~/programs/M.E.R.L.I.N`) with Claude Code + Spec Kit. That is a dev setup only: the product still targets native Windows and Linux, and WSL2 is never a runtime requirement.
- A finished UI from the old repo exists outside this repo (see "Legacy UI"); it will seed `app/` later.

## How to work with Thiago (important)

- **He wants to learn by building.** The backend (voice pipeline, agent loop) is implemented **by him**. For backend work, explain *what, when and why* first, with concrete, simple numeric examples, then guide; don't dump finished code to paste unless he asks for it.
- UI / scaffolding / config / CI work can be done directly.
- Move in **small steps**, each with a clear, testable "done" criterion.
- Keep the single Jade theme and the orb as the center of every animation.
- **Never copy code from I.S.A.A.C.** Its license is CC BY-NC-SA 4.0 (non-commercial, share-alike), incompatible with GPL-3.0 and with MERLIN as a product. Study the ideas, reimplement.
- **Never reintroduce WSL2** as a requirement.

## Architecture (decided)

```
┌──────────────── MERLIN app (Tauri) ─────────────────┐
│  .exe/.msi on Windows · .deb, .rpm, AppImage on Linux│
│                                                      │
│  UI: React + Three.js          Rust core (Tauri)     │
│        ▲ local WebSocket            │ spawns with    │
│        │ 127.0.0.1 + token          │ port + token   │
│        ▼                            ▼                │
│  Python backend (Tauri sidecar, packaged w/ PyInstaller) ──HTTP──▶ Ollama (installed separately)
│  audio · Whisper → agent → Piper · tools             │
└──────────────────────▲───────────────────────────────┘
                       │ PortAudio (sounddevice)
                 microphone and speakers
```

Rules:

- **One repo**: `app/` (Tauri + React + TS, npm) and `backend/` (Python). Plus `assets/` (logo).
- **Backend is a Tauri sidecar** (`bundle.externalBin`, binary suffixed with the target triple). The app starts it on open and kills it on close.
- **Audio capture happens in the Python backend** (`sounddevice`), not in the frontend. On Linux, Tauri's WebView (WebKitGTK) does not grant microphone access without recompiling WebKitGTK.
- **Bind to `127.0.0.1`, random free port + token** chosen at launch and passed to the UI. Never `0.0.0.0:8000`: any website in a browser can try `ws://localhost:8000`, which becomes a hole once MERLIN can delete files.
- **CI matrix Windows + Linux**; each OS builds its own package. Build Linux on **Ubuntu 22.04** (glibc compatibility), not on Thiago's Arch (Arch is for testing).

Known pitfalls:

- **Echo**: without the browser there is no free echo cancellation. Phase 1: mute the mic while MERLIN speaks. Phase 3 (barge-in): evaluate `aec-rs` (SpeexDSP-based, has a Python API) or, Linux-only, PipeWire's echo-cancel module.
- **GPU**: faster-whisper on CUDA needs cuBLAS + cuDNN 9 (CUDA 12); pip on Windows doesn't ship them. Fallback: CPU with `compute_type="int8"`; the `doctor` command must report which one is in use.
- **Wayland**: global shortcuts and simulated typing into other apps are restricted; test on X11 too.

## Stack

| Layer | Tech |
| --- | --- |
| Desktop app | Tauri 2 + React 19 + TypeScript + Vite |
| Visuals | Three.js (`three` npm package), AdditiveBlending, per-vertex color buffers |
| Backend | Python 3.11+, FastAPI (WebSocket) |
| Speech-to-text | Faster-Whisper (Portuguese) |
| Wake word | openWakeWord (local, CPU) — planned |
| LLM | Ollama by default. One OpenAI-compatible adapter for Ollama, NVIDIA NIM, Groq, etc. `model_ref = "provider/model"`. Local-first; cloud only by explicit user choice |
| Text-to-speech | Piper (pt-BR voice), `talk_level` from real output amplitude |
| Resources | psutil, nvidia-smi, Ollama `/api/ps` |

## WebSocket protocol (contract)

The contract below comes from the legacy UI (`src/lib/types.ts`). It is the source of truth: extend it, don't fork it. It matters once the WebSocket server is built; until then the backend runs from the terminal.

```ts
export type OrbState = 'idle' | 'listening' | 'talking';   // planned: 'thinking', 'tool'

// backend → UI
export type ServerEvent =
  | { type: 'state'; state: OrbState }
  | { type: 'log'; who: 'user' | 'merlin'; text: string }
  | { type: 'agents'; agents: { name: string; task: string; status: 'standby' | 'running' | 'error' }[] }
  | { type: 'resources'; host: string; gauges: { id: string; label: string; used: number; total: number; unit: string }[] }
  | { type: 'talk_level'; level: number };   // 0..1, Piper output amplitude

// UI → backend
export type ClientEvent =
  | { type: 'user_message'; text: string }
  | { type: 'mic'; on: boolean };
```

- `state` drives the orb. `resources` gauges are decided by the backend; the UI renders whatever arrives (`total <= 0` = unavailable, shown as an em-dash).

## Visual identity (approved — don't change without asking)

- **Single theme: Jade** (emerald, hue 158). UI and particles share the same master hue (`THEME_H ≈ 0.44`); per-state colors caused visible seams, so they were removed. Must not look like JARVIS (no blue/cyan HUD, arc reactor, red-and-gold, hexagons).
- Palette tokens: bg `#090C0B`, raised `#101413`, hairline `#212B28`, accent `#5ED4A9`, highlight `#85F4D7` (glows stay in-hue, never blown-out white), text `#E2EEEA`, muted `#94ADA6`, error `#D46554`.
- **The orb is one sphere** (~3600 particles) modulated by state:
  - `idle`: vortex sphere — 6 loxodrome arms wrapping the ball and draining into a centered front "eye".
  - `listening`: three precessing gyroscope rings + breathing core.
  - `talking`: Fibonacci sphere that breathes with speech.
  - `thinking` / `tool`: not designed yet.
- Orb constants were tuned by eye. **Copy the numbers from `particleEngine.ts`, don't re-tune from scratch.** Camera at `z = 6.9`.
- Dark glassmorphism widgets (~16px backdrop blur, 1px borders at 10–14% white, faint 46px grid with radial vignette). Lean dashboard; widgets draggable between side columns with persisted layout.
- Custom frameless title bar (52px): wordmark left, date/time centered (monospace), window buttons right. Window 1900×1000, `decorations: false`.
- Settings (palette, visible widgets, layout, WS URL) persisted in `localStorage`; `DevControls` only in dev mode.
- **App icon (official, approved 2026-10-15): jade dot-matrix "M"** — 10×10 grid of glowing dots, brighter middle V, on a dark rounded plate. File: `assets/merlin-icon-jade.png` (1024px). Generate all Tauri icons with `npx tauri icon ../assets/merlin-icon-jade.png` from `app/`. The old JAX spiral icon is retired: the icon must never be a spiral/vortex/galaxy, and no wizard clichés (hats, staffs, stars-and-moons, medieval fonts).
- Open design items: wordmark + typeface pairing (today system fonts: Segoe UI / Cascadia Code), splash screen, risky-action confirmation card, privacy indicator, floating mini-orb window.

## Legacy UI (later, not now)

**Postponed: don't start `app/` until Thiago asks.** The finished UI from the old repo was saved, already renamed to MERLIN, outside this repo (Claude project files, `legado-jax-ui/`). When the frontend work starts, Thiago copies it to `legacy-ui/` at the repo root. It builds standalone: `npm ci && npm run build`; desktop: `npm run tauri dev`. It is the starting point for `app/`, not something to redesign from zero.

| File | What it has |
| --- | --- |
| `src/lib/particleEngine.ts` | Whole orb in pure Three.js, 3 states, tuned constants. Most valuable piece |
| `src/components/ParticleOrb.tsx` | Thin React wrapper |
| `src/lib/palettes.ts` | Palettes (Jade default `{h:158,s:58,bh:162,bs:13}`) + `applyPaletteVars` writing `--h --s --bh --bs` |
| `src/styles/merlin.css` | Glass theme + layout, scoped under `.merlin-app` |
| `src/lib/types.ts` | WebSocket contract |
| `src/hooks/useMerlinSocket.ts` | WS client with reconnect |
| `src/hooks/useWidgetDrag.ts` | Drag widgets between columns (pointer events) |
| `src/lib/settings.ts` | `localStorage` settings |
| `src/components/Header.tsx` | Custom title bar (drag, double-click maximize, clock, window buttons) |
| `src/components/*Panel.tsx` | Conversation, resources, sub-agents, settings panels |
| `src/lib/brand.ts` | `ASSISTANT_NAME` |
| `backend/merlin_ws.py` | Sample FastAPI backend speaking the protocol with real resources. Assumes WSL2 and `0.0.0.0:8000` — **obsolete on both counts**, use only as reference |
| `src-tauri/` | Tauri config (identifier `com.cameratothiago.merlin`), icons (old spiral — regenerate from the new icon) |

When frontend work starts and `legacy-ui/` is not present, ask Thiago for it instead of rewriting the orb.

## Roadmap (6 phases)

Each phase starts only when the previous one meets its criterion.

While the frontend is postponed, the UI parts of each phase (orb states, cards, widgets) are deferred; the backend part of each phase is still done in order, with output in the terminal instead of the UI. Phase 1 order: record 5 s to WAV → transcribe the WAV → text answer from Ollama → speak it with Piper → chain them into a push-to-talk loop in the terminal → hit the 2 s target.

| Phase | Builds | Done when | First step (one afternoon) |
| --- | --- | --- | --- |
| 1. Listen & respond | Mic → Faster-Whisper → Ollama → Piper, orb following real state | "Olá, tudo bem?" becomes a spoken answer in < 2 s on Windows and Linux | Record 5 s with `sounddevice` in the backend and save a `.wav` |
| 2. Act with tools | Hand-written agent loop + 3 tools (time, open app, files) with risk levels | "Cria a pasta Provas" works and a risk-4 action asks for confirmation | One `data_hora()` tool and the loop that calls it |
| 3. Natural conversation | Wake word, sentence-by-sentence streaming TTS, barge-in | Saying "para" silences it in < 0.5 s | Time each stage (STT, LLM, TTS) and show it in the log |
| 4. Remember & measure | SQLite + FTS5 conversations, facts table, 20-request eval suite, fake-LLM tests | Suite score recorded; remembers a fact the next day | Save each message in SQLite and search by word |
| 5. Ship it | Backend as sidecar, CI Win + Linux, `doctor` command, installers | Installs and runs on Windows and Linux (Ubuntu and Arch) first try | PyInstaller the backend and have Tauri launch it as sidecar |
| 6. Expand | MCP client, screen understanding (VLM), spoken reminders, specialists | — | Connect a ready MCP server (e.g. filesystem) |

Learning habit: for each phase, read the equivalent part of I.S.A.A.C. (without copying), write a minimal version, measure against the criterion, note surprises in `docs/aprendizados.md`.

## Project rules

- **Every tool has `risk_level` 1–5.** Read = 1, write = 3, delete / send email / shell = 4+, which requires confirmation (spoken + a card in the UI).
- **Local-first.** Nothing leaves the machine by default; cloud only when the user picks it.
- **Measure before claiming.** Every model or prompt change runs the eval suite (JSONL with deterministic checks), e.g.
  `{"id": "arq-001", "pedido": "Crie notas.txt com o texto oi", "checagem": {"tipo": "arquivo_contem", "arquivo": "notas.txt", "texto": "oi"}}`
- **Tests without GPU.** pytest with a fake LLM returning canned responses.
- **Agent loop written by hand first** (~150 lines). LangGraph only if the graph gets complex.

## Differentiators vs I.S.A.A.C.

Planned: native Windows **and** Linux; local wake word; barge-in; streaming speech (Piper starts on the first finished sentence); orb shows reasoning (`thinking`, `tool` states) + tool log; understands its own PC ("why is my PC slow?"); Portuguese and 100% offline; spoken reminders.

Backlog ideas: floating orb window, game mode (unload model with `keep_alive: 0` when GPU is busy), latency panel (p50/p95), dictation into any app, lecture/meeting transcription + summary, daily briefing, voice WER evaluation (tiny/small/medium), speaker verification, voice tests without speaking (Piper reads the suite into the pipeline), flight recorder (per-request timeline), tokens & GPU energy per answer, media control (MPRIS / Windows media), privacy indicator + mute key, local RAG over chosen folders, "where did I stop?" from git, spoken journal, language partner, phone as microphone. Suggested first after Phase 3: voice tests without speaking, flight recorder, tokens & energy.

I.S.A.A.C. files worth studying (ideas only): `src/isaac/agents/agent_loop.py`, `src/isaac/tools/base.py`, `src/isaac/multimodal/voice/audio_io.py` + `stt.py`, `src/isaac/memory/long_term.py`, `evals/golden_v1.jsonl`, `scripts/build_windows.ps1`, `src/isaac/doctor.py`, `src/isaac/mcp/client.py`, `LIMITATIONS.md`.

## Open decisions

| Decision | Suggestion | By |
| --- | --- | --- |
| Own agent loop vs LangGraph | Own loop first | Phase 2 |
| Echo cancellation with backend audio | Mute mic while speaking → test `aec-rs` on both OSes | Phase 3 |
| Wordmark / typeface | Open-source geometric sans + mono companion | Before first release |

## Target repo layout

```
M.E.R.L.I.N/
├── CLAUDE.md
├── README.md
├── LICENSE          # GPL-3.0
├── assets/          # merlin-icon-jade.png
├── backend/         # Python package `merlin`: audio, STT, agent, tools, TTS (WS server later)
│   ├── pyproject.toml
│   ├── merlin/
│   └── tests/
├── docs/            # aprendizados.md, guides
├── app/             # later: Tauri + React + Three.js (seeded from legacy-ui/)
└── specs/, .specify/  # Spec Kit artifacts
```
