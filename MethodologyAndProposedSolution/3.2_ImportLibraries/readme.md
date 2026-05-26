# 3.2 Import Libraries

This section catalogs every third-party library R.A.Y.A v2.4 depends on, why it was chosen, and which subsystem uses it. The list is grounded in `requirements.txt`, `environment.yml`, and the actual `import` statements across the codebase.

The dependencies are organized into nine functional groups so a reviewer can quickly understand the role of each library.

---

## 3.2.1 Group A — AI Model Access

These libraries are how R.A.Y.A talks to Google's Gemini family.

### `google-genai`
- **Package on PyPI:** `google-genai`
- **Used in:** `main.py`
- **Role:** The new (2025) Google Generative AI SDK. Provides the **Live API client** — `genai.Client(...)` and the `client.aio.live.connect(...)` async context manager — that powers R.A.Y.A's real-time voice session.
- **Why chosen:** It is the only client that supports the bidirectional streaming Gemini Live protocol with native audio.

### `google-generativeai`
- **Package on PyPI:** `google-generativeai`
- **Used in:** `agent/planner.py`, `agent/executor.py`, `agent/error_handler.py`, several action modules.
- **Role:** The older (2024-era) but more mature synchronous SDK. Used for one-shot calls to `GenerativeModel.generate_content` — for planning, error analysis, code generation, translation, and vision.
- **Why both SDKs?** The Live API requires the new `google-genai`, while the simpler `generate_content` style of the older SDK is faster to invoke from synchronous tool code. Keeping both is intentional.

---

## 3.2.2 Group B — Audio I/O

### `sounddevice`
- **Used in:** `main.py` (`InputStream`, `RawOutputStream`).
- **Role:** Captures microphone audio at 16 kHz, 16-bit PCM, mono, in 1024-sample blocks (≈ 64 ms); plays back the assistant's 24 kHz audio.
- **Why chosen:** Lightweight, cross-platform binding over PortAudio. Cleaner Pythonic API than `pyaudio`, and the `callback` model integrates cleanly with asyncio via `loop.call_soon_threadsafe`.

### `pyaudio`
- **Used in:** Listed in `requirements.txt` as a fallback / secondary audio backend.
- **Role:** Alternative cross-platform audio I/O. Retained for compatibility with environments where `sounddevice` fails to install.

---

## 3.2.3 Group C — Image, Vision, and Screen Capture

### `pillow`
- **Imported as:** `from PIL import Image, ImageTk, ImageDraw`
- **Used in:** `ui.py` (face rendering), `actions/screen_processor.py` (image post-processing).
- **Role:** All raster image manipulation: loading the face PNG, drawing onto Tkinter canvases, converting between formats.

### `mss`
- **Used in:** `actions/screen_processor.py`.
- **Role:** Ultra-fast cross-platform screen capture. Hooks the OS compositor at a low level — significantly faster than `PIL.ImageGrab`. Single source of frames for the vision pipeline.

### `opencv-python`
- **Imported as:** `import cv2`
- **Used in:** `actions/screen_processor.py` (webcam capture via `cv2.VideoCapture`), occasional preprocessing.
- **Role:** Webcam frame acquisition and basic image operations.

### `numpy`
- **Used in:** `opencv-python` internals, optional buffer math in audio paths.
- **Role:** N-dimensional array support. A transitive dependency that R.A.Y.A pins explicitly so the binary wheel is matched to OpenCV's expected version.

---

## 3.2.4 Group D — Web and Browser Automation

### `playwright`
- **Used in:** `actions/browser_control.py`.
- **Role:** The cross-browser web automation engine. R.A.Y.A drives Chrome, Edge, Firefox, Opera, Brave, Vivaldi, and Safari through a uniform API. Supports headless and headed modes, multiple concurrent browsers, screenshots, form fills, and natural-language "smart" selectors via Playwright's accessibility tree.
- **Why chosen:** More reliable than Selenium for modern JS-heavy sites. The official Playwright Python binding is a first-class citizen. Browsers can be **bundled into the Windows installer** (`ms-playwright/` folder) so the user does not need to run `playwright install` separately.

### `requests`
- **Used in:** Several actions (`weather_report`, `web_search`, helpers).
- **Role:** Simple HTTP for REST APIs (weather services, search engines, etc.).

### `beautifulsoup4`
- **Imported as:** `from bs4 import BeautifulSoup`
- **Used in:** `actions/web_search.py`, helpers that parse HTML responses.
- **Role:** HTML parsing and DOM traversal.

### `duckduckgo-search`
- **Used in:** `actions/web_search.py`.
- **Role:** Programmatic DuckDuckGo search results — no API key needed, no quota throttling. Preferred over Google for unattended searches.

### `youtube-transcript-api`
- **Used in:** `actions/youtube_video.py`.
- **Role:** Fetches YouTube transcripts so R.A.Y.A can summarize videos without downloading them.

---

## 3.2.5 Group E — OS-Level Automation (Cross-Platform)

### `pyautogui`
- **Used in:** `actions/computer_control.py`, `actions/computer_settings.py`.
- **Role:** Cross-platform keyboard / mouse automation: typing, clicking, hotkeys, scrolling, dragging.

### `pyperclip`
- **Used in:** `actions/computer_control.py` (paste/copy operations).
- **Role:** Cross-platform clipboard access.

### `pygetwindow`
- **Used in:** `actions/computer_settings.py` (window focus, list, minimize/maximize).
- **Role:** Enumerate and manipulate OS windows.

### `psutil`
- **Used in:** Several modules for process management.
- **Role:** List processes, check CPU/RAM, kill processes by name. Used during graceful shutdown and conflict resolution (e.g., closing a browser that holds a lock).

### `send2trash`
- **Used in:** `actions/file_controller.py`.
- **Role:** Move-to-trash file deletion that respects the OS recycle bin — safer than `shutil.rmtree`.

---

## 3.2.6 Group F — Windows-Specific Libraries

These libraries are imported only on Windows or only used by Windows-specific code paths.

### `comtypes`
- **Used in:** Volume control, Windows COM automation.
- **Role:** Pure-Python COM bridge for Windows. Underlies `pycaw`.

### `pycaw`
- **Used in:** `actions/computer_settings.py` (volume up/down/mute).
- **Role:** Python wrapper for the Windows Core Audio API. Lets R.A.Y.A read and set the master volume.

### `win10toast`
- **Used in:** Reminder notifications.
- **Role:** Native Windows toast notifications.

### `pywinauto`
- **Used in:** `actions/send_message.py`, `actions/open_app.py`, edge cases of Windows window automation.
- **Role:** Microsoft UI Automation API wrapper. Used when GUI scripting needs to address controls by name/role rather than coordinates.

### `winreg` (standard library)
- **Used in:** `agent/executor.py::_run_generated_code` (resolving the Desktop path when the default does not exist).
- **Role:** Read Windows registry keys to locate well-known user folders.

---

## 3.2.7 Group G — User Interface

### `tkinter` (standard library)
- **Used in:** `ui.py`.
- **Role:** The entire UI layer. Chosen because it ships with Python — zero install cost, instant cold start, sufficient for an animated face + log + queue snapshot + text input.

### `PIL.ImageTk`
- **Used in:** `ui.py` (rendering PIL images inside Tkinter canvases).
- **Role:** Bridge between Pillow images and Tkinter widgets.

---

## 3.2.8 Group H — Concurrency and System

These are standard library modules but worth listing because R.A.Y.A leans heavily on them.

| Module | Role |
|---|---|
| `asyncio` | Backbone of `main.py`'s streaming model — `TaskGroup`, `Queue`, `Event`, `run_coroutine_threadsafe`. |
| `threading` | UI thread, audio worker, task queue worker, per-task threads. |
| `subprocess` | Used by `agent/executor.py::_run_generated_code` to run generated Python scripts in an isolated process. |
| `tempfile` | Temporary files for generated code, screenshots, downloads. |
| `pathlib.Path` | Cross-platform file paths everywhere — replacing `os.path` joins. |
| `json` | Memory file I/O, API config, tool argument parsing, planner output. |
| `re` | Transcript sanitization (`_CTRL_RE`), JSON cleanup from LLM responses. |
| `traceback` | Detailed error logging when tool calls fail. |
| `datetime` | Time context injected into the system prompt; reminder scheduling. |
| `uuid` | Short task IDs for the task queue. |
| `dataclasses` | The `Task` record in the task queue. |
| `enum` | `TaskStatus`, `TaskPriority`, `ErrorDecision`. |
| `collections.deque` | Activity log ring buffer in the UI. |
| `sys`, `os`, `platform` | Frozen-binary detection, env var configuration, OS branching. |

---

## 3.2.9 Group I — Packaging and Build

These are not runtime imports but are essential to the deployment story documented in Section 4.

| Tool | Role |
|---|---|
| `pyinstaller` | Freezes the Python runtime + all dependencies into `RAYA.exe`. Listed in `requirements-build.txt`. |
| `pyinstaller-hooks-contrib` | Extra import hooks for tricky dependencies. |
| **Inno Setup** | External (not pip) — Windows installer authoring tool driven by `scripts/build_windows.ps1`. |
| `playwright install` | Pre-fetches browser binaries into `ms-playwright/` before packaging. |

---

## 3.2.10 Complete Top-Level `requirements.txt` Mapping

For traceability, here is the file as it appears in the repository, annotated:

```
sounddevice              # Audio I/O — Group B
google-genai             # Gemini Live SDK — Group A
google-generativeai      # Gemini classic SDK — Group A
pillow                   # Imaging — Group C
requests                 # HTTP — Group D
beautifulsoup4           # HTML parsing — Group D
duckduckgo-search        # Web search — Group D
playwright               # Browser automation — Group D
pyautogui                # Mouse/keyboard — Group E
pyperclip                # Clipboard — Group E
pygetwindow              # Windows — Group E
opencv-python            # Vision/webcam — Group C
numpy                    # Array math — Group C
mss                      # Screen capture — Group C
psutil                   # Processes — Group E
comtypes                 # Windows COM — Group F
pycaw                    # Windows audio — Group F
win10toast               # Windows toasts — Group F
send2trash               # Safe delete — Group E
youtube-transcript-api   # YouTube — Group D
pywinauto                # Windows UIA — Group F
pyaudio                  # Audio fallback — Group B
```

The Conda `environment.yml` simply pins Python 3.11 and pulls the same list via pip:

```
name: raya-2.4.0
channels:
  - defaults
dependencies:
  - python=3.11
  - pip
  - pip:
      - -r requirements.txt
```

---

## 3.2.11 Imports Used in `main.py` (The Master Entry Point)

For concreteness, here is the actual import block of `main.py` and what each line contributes:

```python
import asyncio          # streaming event loop
import os               # env vars, paths
import re               # transcript cleanup
import threading        # speaking lock, async runner thread
import json             # API key + tool args
import sys              # frozen detection
import traceback        # error logging
from pathlib import Path  # cross-platform paths

import sounddevice as sd        # audio I/O
from google import genai        # new Live SDK
from google.genai import types  # LiveConnectConfig, SpeechConfig

from ui import RayaUI                       # UI layer
from memory.memory_manager import (         # persistent memory
    load_memory, update_memory, format_memory_for_prompt,
)

# 17 action modules — Layer 5 of the block diagram
from actions.flight_finder     import flight_finder
from actions.open_app          import open_app
from actions.weather_report    import weather_action
from actions.send_message      import send_message
from actions.reminder          import reminder
from actions.computer_settings import computer_settings
from actions.screen_processor  import screen_process
from actions.youtube_video     import youtube_video
from actions.desktop           import desktop_control
from actions.browser_control   import browser_control
from actions.file_controller   import file_controller
from actions.code_helper       import code_helper
from actions.dev_agent         import dev_agent
from actions.web_search        import web_search as web_search_action
from actions.computer_control  import computer_control
from actions.game_updater      import game_updater
```

This import block is itself a **mini system map**: by reading the imports, a new developer can identify every subsystem the orchestrator touches.

---

## 3.2.12 Optional / On-Demand Imports

Some libraries are **imported lazily** inside functions, not at module top level. This keeps the cold-start time low and avoids loading heavy modules unless the user actually triggers them.

| Library | Lazy-imported in | Reason |
|---|---|---|
| `google.generativeai` | `agent/planner.py::create_plan`, `agent/executor.py::_run_generated_code` | Avoids loading until the first multi-step task |
| `winreg` | `_run_generated_code` | Windows-only fallback |
| `agent.task_queue` | `main.py::_execute_tool` | Imported only when `agent_task` is called |
| `agent.executor.AgentExecutor` | `task_queue.py::_get_executor` | First-task lazy init |

---

## 3.2.13 Dependency Risk Notes

A brief risk assessment of the external dependencies:

| Risk | Mitigation |
|---|---|
| Gemini API price / rate-limit changes | The user provides their own API key — no shared quota; the planner uses the cheap `flash-lite` model |
| Breaking changes in `google-genai` SDK | Pinned by `pip` resolver; if a breaking version ships, the codebase has clear seams to upgrade |
| Playwright browser version drift | Bundled browser is shipped with the installer; offline users keep working even if the upstream browser updates |
| `pycaw` / `comtypes` are Windows-only | Wrapped behind platform-aware action modules; non-Windows users simply do not see volume control |
| `pyaudio` install pain on Windows | `sounddevice` is the default; `pyaudio` is the fallback, not the primary path |

---

## Summary

R.A.Y.A v2.4 stands on a stack of roughly **23 third-party libraries** organized into nine functional groups: AI access, audio I/O, vision and screen capture, web and browser automation, OS-level automation, Windows-specific helpers, user interface, concurrency/system utilities, and packaging/build. Two SDKs (`google-genai` for streaming, `google-generativeai` for one-shot calls) provide the LLM access; `sounddevice` powers the voice channel; `playwright` powers the web channel; `pyautogui`, `pywinauto`, and `pycaw` power the OS channel; and Tkinter (standard library) renders the UI. Together with around fifteen standard-library modules, these dependencies form the complete substrate on which R.A.Y.A's six-layer architecture is built.
