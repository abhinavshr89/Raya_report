# 4.1 Tools and Technologies

This section catalogs every tool, language, framework, library, model, protocol, and external system that R.A.Y.A v2.4 either uses or runs on. Where Section 3.2 inventoried *imports*, this section takes a wider view — it includes development tooling, build tooling, runtime environments, AI services, communication protocols, and hardware. The goal is to give a reviewer a complete picture of the technology surface the project depends on.

Items are organized into ten functional groups; each group is summarized in a table for quick scanning, followed by short notes explaining the role each technology plays.

---

## 4.1.1 Group 1 — Programming Language and Runtime

| Technology | Version | Role |
|---|---|---|
| **Python** | 3.11 (primary) / 3.12 (supported) | Implementation language for the entire codebase |
| **CPython** | Reference interpreter | Default runtime used in development and in the PyInstaller bundle |
| **asyncio** | std lib | Streaming event loop for `RayaLive` |
| **threading** | std lib | UI thread, audio worker, task-queue worker |
| **pathlib** | std lib | Cross-platform path handling |
| **dataclasses / enum** | std lib | Structured records for `Task`, `TaskStatus`, `TaskPriority`, `ErrorDecision` |

### Notes
- **Python 3.11** was chosen because it introduces `asyncio.TaskGroup`, used in `RayaLive.run` for structured concurrency. The codebase's `try/except TaskGroup as tg` patterns require this version or later.
- **Why not 3.13?** Some binary wheels (`pyaudio`, certain `pyautogui` builds) lag behind on Windows. 3.11/3.12 hit a sweet spot of language features and ecosystem stability.
- **Why not async-first frameworks like FastAPI?** R.A.Y.A is not a web service. The plain `asyncio` event loop plus Tkinter is sufficient and minimizes runtime overhead.

---

## 4.1.2 Group 2 — AI Models and AI Services

| Technology | Provider | Role |
|---|---|---|
| **Gemini 2.5 Flash Native Audio Preview** | Google | Real-time voice conversation in `main.py` (`LIVE_MODEL = "models/gemini-2.5-flash-native-audio-preview-12-2025"`) |
| **Gemini 2.5 Flash Lite** | Google | Cheap planner LLM in `agent/planner.py`; error analyzer in `agent/error_handler.py` |
| **Gemini 2.5 Flash** | Google | Code generation, replanning, and translation in `agent/executor.py` |
| **Gemini Vision (multimodal)** | Google | Screen and webcam image understanding in `actions/screen_processor.py` |
| **Charon voice** | Google | The synthetic voice preset configured in `SpeechConfig → PrebuiltVoiceConfig` |

### Notes
- **Why Gemini?** As of late 2025, Gemini's Live API offers the lowest end-to-end voice latency among the three major providers (Google, OpenAI, Anthropic), a generous free quota for student-level usage, and clean Python SDK ergonomics.
- **Three different Gemini models are used intentionally.** The Live model is expensive per minute but unmatched at conversation. Flash-Lite is two orders of magnitude cheaper and ideal for structured-JSON tasks like planning. Flash sits in between and is used when the planner needs more depth (replanning, code synthesis).
- **No fine-tuning.** R.A.Y.A uses the models *as-is*; all customization happens through prompting (`core/prompt.txt`, the planner prompt, the error-analyst prompt).
- **No self-hosted LLM.** This is a deliberate scope cut — see Section 7 (Future Scope) for a local-model variant.

---

## 4.1.3 Group 3 — Communication Protocols and Network Standards

| Technology | Role |
|---|---|
| **WebSocket-style streaming** | Underlying transport for `client.aio.live.connect`; bidirectional audio + text + tool calls |
| **HTTPS / TLS 1.3** | All Gemini API traffic |
| **HTTP(S)** | `requests` calls for weather, web search, and image downloads |
| **JSON** | Tool argument format, planner output, memory persistence, API key storage, error analyst output |
| **PCM (16-bit little-endian)** | The audio format Gemini Live expects on both input (16 kHz) and output (24 kHz) |
| **PNG / JPEG (inline)** | The image format used to send screen and webcam frames to Gemini Vision |
| **REST API endpoints** | DuckDuckGo, weather services, YouTube transcript API |

### Notes
- R.A.Y.A makes **no peer-to-peer connections**, opens **no listening ports**, and serves **no inbound traffic**. All network activity is outbound from the user's machine.

---

## 4.1.4 Group 4 — AI / SDK Libraries

| Technology | Purpose |
|---|---|
| **google-genai** | New (2025) Google Generative AI SDK — required for the Live API |
| **google-generativeai** | Older synchronous SDK — used for one-shot `generate_content` calls in planner/executor/error handler |
| **types.LiveConnectConfig** | Strongly-typed config object: response modalities, system instruction, tools, voice config, session resumption |
| **types.FunctionResponse** | Strongly-typed wrapper used to send tool results back into the Live session |

### Notes
- Both SDKs are imported because Gemini's Live API is in the *new* `google-genai` package while many `actions/` modules still use the *classic* `google.generativeai.GenerativeModel` interface, which is more concise for synchronous one-shot calls.

---

## 4.1.5 Group 5 — Audio / Speech Stack

| Technology | Purpose |
|---|---|
| **sounddevice** | Microphone capture (`InputStream`) and speaker playback (`RawOutputStream`) |
| **PortAudio** | The cross-platform audio backend that `sounddevice` wraps |
| **pyaudio** | Listed as fallback in `requirements.txt` for environments where `sounddevice` fails to install |
| **Hardware: microphone** | Required for voice mode |
| **Hardware: speakers / headphones** | Required for voice mode |

### Audio settings (constants in `main.py`)
| Setting | Value | Reason |
|---|---|---|
| Channels | 1 (mono) | Voice is mono; saves bandwidth |
| Send sample rate | 16,000 Hz | Standard for speech; Gemini Live input requirement |
| Receive sample rate | 24,000 Hz | Higher fidelity for synthesized output |
| Chunk size | 1024 samples | ≈ 64 ms per frame — small enough for responsive turn-taking |
| Sample format | int16 PCM | Native format Gemini Live expects |

---

## 4.1.6 Group 6 — Computer Vision Stack

| Technology | Purpose |
|---|---|
| **mss** | Fast cross-platform screen capture; hooks the OS compositor's framebuffer |
| **opencv-python (cv2)** | Webcam acquisition (`VideoCapture`), basic image processing |
| **pillow (PIL)** | Image encoding/decoding, drawing the UI face, in-memory image conversion |
| **numpy** | Backing array type for image buffers from OpenCV |
| **Gemini Vision API** | The actual visual reasoning — receives a PNG and a question, returns an answer |
| **Hardware: monitor** | Source for `screen_process` capture |
| **Hardware: webcam** | Optional, for `angle="camera"` mode |

### Notes
- Vision is **on-demand**, not streaming. A frame is captured only when the `screen_process` tool is invoked. This keeps bandwidth and battery usage minimal compared to a continuous vision pipeline.

---

## 4.1.7 Group 7 — Web and Browser Automation

| Technology | Purpose |
|---|---|
| **Playwright (Python)** | Cross-browser automation engine driving Chrome, Edge, Firefox, Opera, Brave, Vivaldi, Safari |
| **Chromium** | The actual browser binary; bundled into the installer at `ms-playwright/` for offline use |
| **requests** | Lightweight HTTP for REST APIs (weather, search) |
| **beautifulsoup4** | HTML parsing for scraped content |
| **duckduckgo-search** | Quota-free DuckDuckGo search API |
| **youtube-transcript-api** | Fetches YouTube subtitles without downloading the video |

### Notes
- **Why Playwright over Selenium?** Faster startup, more reliable selectors on modern JS sites, and a cleaner async API. Playwright also supports `smart_click` / `smart_type` via its accessibility tree, used in `browser_control.py`.
- **Why DuckDuckGo over Google?** No API key required; no per-IP throttling for normal usage; results are clean for programmatic consumption.

---

## 4.1.8 Group 8 — OS-Level Automation

These tools allow R.A.Y.A to act on the operating system directly, beyond what the browser can reach.

| Technology | Purpose | Platforms |
|---|---|---|
| **pyautogui** | Generic keyboard / mouse / screen automation | Win / macOS / Linux |
| **pyperclip** | Clipboard read/write | Win / macOS / Linux |
| **pygetwindow** | Window enumeration, focus, minimize/maximize | Primarily Windows |
| **psutil** | Process management — list, kill, query CPU/RAM | Win / macOS / Linux |
| **send2trash** | Trash/Recycle-Bin-aware file deletion | Win / macOS / Linux |
| **pywinauto** | Microsoft UI Automation API wrapper for window-control tasks | Windows |
| **comtypes** | COM bridge for Windows scripting | Windows |
| **pycaw** | Windows Core Audio API for volume control | Windows |
| **win10toast** | Native Windows toast notifications | Windows 10/11 |
| **winreg (std lib)** | Read registry keys (e.g., resolve real Desktop path) | Windows |

### Notes
- Cross-platform automation lives in `actions/computer_control.py` (`pyautogui`-driven), while Windows-only refinements live in `actions/computer_settings.py` and `actions/send_message.py`.
- macOS and Linux are supported on the "common" path (pyautogui, sounddevice, Playwright); some advanced features (Steam library inspection, taskbar UI Automation) are Windows-specific.

---

## 4.1.9 Group 9 — User Interface

| Technology | Purpose |
|---|---|
| **Tkinter** (std lib) | Main GUI toolkit — window, canvas, log widget, text entry, mute button |
| **Tk** | Underlying C library wrapped by Tkinter |
| **PIL.ImageTk** | Bridge between Pillow images and Tkinter canvas widgets |
| **Custom theming** | Nord-inspired color palette in `ui.py` (`C_BG`, `C_PRI`, `C_ACC`, etc.) |
| **Animation loop** | Frame-driven update tied to `root.after(...)` for the face animation |
| **Keyboard shortcuts** | F4 (mute toggle), Enter (submit text) |

### Notes
- **Why Tkinter?** Ships with Python, no extra installer footprint, sufficient for a low-resolution animated face plus log widget. A heavier framework (PyQt, Electron) was rejected as overkill.
- **Animated face.** A circular canvas with rotating scan rings, breathing halo, and orbiting particles. The animation state evolves with `tick`, `breathe_phase`, and `orbit_phase` variables updated each frame.

---

## 4.1.10 Group 10 — Application Data and Files

| Path | Format | Purpose |
|---|---|---|
| `core/prompt.txt` | Plain text | System prompt — persona and tool-routing rules |
| `config/api_keys.json` | JSON | Stores the user's Gemini API key |
| `memory/long_term.json` | JSON | Six-category persistent memory (identity, preferences, projects, relationships, wishes, notes) |
| `face.png` | PNG | Idle/listening face image rendered in the UI |
| `setup.py` | Python | Project metadata; queried by `ui.py` to display the version badge |
| `requirements.txt` | Plain text | Runtime dependencies |
| `requirements-build.txt` | Plain text | Build-time dependencies (PyInstaller, hooks) |
| `environment.yml` | YAML | Conda environment specification |
| `ms-playwright/` | Folder | Bundled Chromium browser (in installer builds only) |

---

## 4.1.11 Development Tools

These are the tools used to *write* R.A.Y.A, not to *run* it.

| Tool | Role |
|---|---|
| **VS Code** | Primary code editor (with Python and Pylance extensions) |
| **Claude Code (Anthropic CLI)** | AI-assisted code authoring and documentation drafting |
| **PowerShell 7+** | Primary shell on the Windows development machine; runs `scripts/build_windows.ps1` |
| **Git** | Version control |
| **GitHub** | Remote repository hosting |
| **conda / pip** | Python package management |
| **Playwright CLI** | One-time `playwright install` to fetch browsers during development |

---

## 4.1.12 Build and Packaging Tools

| Tool | Role |
|---|---|
| **PyInstaller** | Freezes Python + dependencies into a portable `dist/RAYA/` folder |
| **pyinstaller-hooks-contrib** | Provides import hooks for libraries PyInstaller does not handle natively |
| **Inno Setup 6** | Windows installer authoring tool — produces `RAYA-Setup-v2.4.0.exe` |
| **scripts/build_windows.ps1** | PowerShell driver that runs PyInstaller, copies Playwright browsers into the bundle, and invokes Inno Setup |
| **Output artifact** | `installer/output/RAYA-Setup-v2.4.0.exe` |
| **Install target** | User-local AppData (so the user does not need admin rights) |

### Notes
- **First-run config.** When the installer is launched, the app data directory is empty. The UI's `wait_for_api_key()` shows a dialog, the user pastes their Gemini key, and `config/api_keys.json` is created in place.
- **Offline Playwright.** The build script runs `playwright install chromium`, then copies `%LOCALAPPDATA%\ms-playwright` into the bundle next to `RAYA.exe`. At runtime, `_configure_playwright_browser_path` sets the environment variable that tells Playwright to use this bundled folder.

---

## 4.1.13 Concurrency, Synchronization, and Reliability Primitives

| Primitive | Where used |
|---|---|
| **asyncio.TaskGroup** | `RayaLive.run` — structured concurrency over the four streaming tasks |
| **asyncio.Queue** | `out_queue` (mic → server, bounded at 10) and `audio_in_queue` (server → speakers, unbounded) |
| **asyncio.Event** | `_turn_done_event` — signals when a server turn is complete |
| **threading.Lock** | `_speaking_lock` (R.A.Y.A's speaking state); `_lock` in `memory_manager.py` for file I/O |
| **threading.Condition** | `_condition` in `task_queue.py` for producer/consumer signalling |
| **threading.Event** | `cancel_flag` on each agent task — cooperative cancellation |
| **loop.run_in_executor** | All tool calls run on the default executor so the audio loop is never blocked |
| **loop.call_soon_threadsafe** | Bridges the sounddevice callback thread to the asyncio event loop |
| **Retry / backoff** | 3-retry per step inside the agent executor; 3-second reconnect backoff at the session level |
| **Bounded plan length** | `Max 5 steps` enforced by the planner prompt |
| **Bounded replan attempts** | `MAX_REPLAN_ATTEMPTS = 2` in `executor.py` |

---

## 4.1.14 Security and Privacy Mechanisms

| Mechanism | Implementation |
|---|---|
| **Mute-at-source** | The `_listen_audio` callback checks `ui.muted` *before* placing audio in any queue; muted audio never enters Python memory |
| **Local-only memory** | `memory/long_term.json` lives on the user's disk; no cloud sync |
| **User-owned API key** | The Gemini API key is the user's own; R.A.Y.A never bundles or shares one |
| **No telemetry** | The codebase issues zero outbound requests to any URL except the user's chosen APIs (Gemini, weather, DuckDuckGo, etc.) |
| **Safe file deletion** | `send2trash` instead of `shutil.rmtree` |
| **Local AppData install path** | Avoids requiring admin rights; respects user-scope filesystem permissions |
| **Bounded plan/retry/replan** | Defensive limits to prevent runaway tool calls |
| **Transcript sanitization** | `_clean_transcript` strips control characters and Gemini's `<ctrlNN>` artifacts before logging |
| **One-call policy** | `core/prompt.txt` instructs the model to call each tool exactly once per turn |

---

## 4.1.15 Hardware Requirements

| Hardware | Minimum | Recommended | Used by |
|---|---|---|---|
| CPU | Dual-core x86_64 | Quad-core | All |
| RAM | 4 GB | 8 GB | Playwright Chromium drives this up |
| Disk | 1 GB free | 2 GB free | Bundled Chromium accounts for most of the installer size |
| Microphone | Any USB / built-in | Noise-cancelling headset | Voice input |
| Speakers / headphones | Any | Stereo headphones | Voice output |
| Monitor | 1280×720 | 1920×1080 | UI and `screen_process` |
| Webcam | None | Any USB / built-in | Optional, for `screen_process angle="camera"` |
| Internet | Required for Gemini | 10 Mbps + | Live API streaming |
| GPU | None required | None required | All ML runs in the cloud |

---

## 4.1.16 Supported Operating Systems

| OS | Status | Notes |
|---|---|---|
| **Windows 10 / 11** | Fully supported, primary platform | All features verified |
| **macOS** | Experimental | Core features work; some Windows-only tools (volume via pycaw, win10toast) are no-ops |
| **Linux** | Experimental | Same caveats as macOS |
| iOS / Android | Not supported | Out of scope |

---

## 4.1.17 External Online Services

| Service | Used by | Auth |
|---|---|---|
| **Google Gemini API** | Conversation, planning, vision, codegen | User-provided key in `config/api_keys.json` |
| **DuckDuckGo** | `actions/web_search.py` | None |
| **YouTube** | `actions/youtube_video.py` (transcripts) | None |
| **Google Flights** | `actions/flight_finder.py` (via Playwright scraping) | None |
| **Weather services** | `actions/weather_report.py` | Optional API key, depending on backend chosen |
| **Steam / Epic Games** | `actions/game_updater.py` | Uses the user's already-installed client |
| **WhatsApp Web / Telegram Web / etc.** | `actions/send_message.py` | Uses the user's existing logged-in browser session |

---

## 4.1.18 Technology Stack at a Glance

A compact one-screen view of the entire tech stack:

```
┌──────────────────────────────────────────────────────────────────────────┐
│                          R.A.Y.A v2.4 Tech Stack                          │
├──────────────────────────────────────────────────────────────────────────┤
│ Language    : Python 3.11 / 3.12                                          │
│ Runtime     : CPython, asyncio, threading                                 │
│ AI Models   : Gemini 2.5 Flash Native Audio / Flash / Flash Lite / Vision │
│ AI SDKs     : google-genai (live), google-generativeai (classic)          │
│ Audio       : sounddevice + PortAudio                                     │
│ Vision      : mss + opencv-python + Pillow + Gemini Vision API            │
│ Browser     : Playwright + bundled Chromium                               │
│ OS Auto     : pyautogui · pywinauto · pycaw · psutil · pygetwindow        │
│ UI          : Tkinter + PIL.ImageTk                                       │
│ Storage     : Plain JSON (memory, config) on local disk                   │
│ Concurrency : asyncio.TaskGroup, asyncio.Queue/Event, threading.Lock      │
│ Packaging   : PyInstaller + Inno Setup 6 + bundled Playwright             │
│ Dev Tools   : VS Code · Claude Code · PowerShell · Git · GitHub · conda   │
│ OS Support  : Windows 10/11 (primary), macOS/Linux (experimental)         │
│ License     : Creative Commons BY-NC 4.0                                  │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## 4.1.19 Justification Summary

For every technology choice in the stack, the justification follows one of three patterns:

1. **"Best in class for the job."** — Gemini Live for streaming voice; Playwright for browser automation; sounddevice for audio I/O.
2. **"Standard library / zero install."** — Tkinter, asyncio, threading, pathlib. Chosen to keep the install footprint small.
3. **"Mature and proven."** — Pillow, OpenCV, requests, BeautifulSoup. Used because reinventing them would consume disproportionate time.

No technology in the stack is experimental or unmaintained. Every component is either a major commercial offering (Gemini, Playwright) or a long-established open-source project (the rest).

---

## Summary

R.A.Y.A v2.4 stands on a deliberately compact but capable technology stack: **Python 3.11** as the implementation language, **Gemini 2.5** as the cognitive engine, **sounddevice** and **Playwright** as the primary I/O channels, **Tkinter** as the UI, and a handful of OS-automation libraries to reach the rest of the desktop. The entire system is packaged with **PyInstaller + Inno Setup** into a single Windows installer, with the **Chromium browser bundled** to enable offline web automation. Every layer of the block diagram in Section 3.1 maps to a small, well-justified set of technologies — together forming a tech stack that is broad enough to deliver a full personal-assistant experience while remaining narrow enough for a single developer to maintain.
