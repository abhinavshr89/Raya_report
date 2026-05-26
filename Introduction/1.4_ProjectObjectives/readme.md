# 1.4 Project Objectives

## Objective Hierarchy

The objectives of R.A.Y.A v2.4 are organized into three tiers — **primary**, **secondary**, and **tertiary** — to make the success criteria for the project clear, measurable, and traceable to the design decisions documented in later sections.

```
            ┌──────────────────────────────────────────┐
            │   PRIMARY OBJECTIVES (must-have)         │
            │   Conversation · Tools · Memory · Vision │
            └──────────────────────────────────────────┘
                              │
            ┌──────────────────────────────────────────┐
            │   SECONDARY OBJECTIVES (should-have)     │
            │   Planning · Cross-platform · Packaging  │
            └──────────────────────────────────────────┘
                              │
            ┌──────────────────────────────────────────┐
            │   TERTIARY OBJECTIVES (nice-to-have)     │
            │   Extensibility · Education · Showcase   │
            └──────────────────────────────────────────┘
```

---

## A. Primary Objectives

These are the non-negotiable goals that define R.A.Y.A as a working product. Each one is verified through the testing protocols documented in **Section 5 (Testing and Results)**.

### O1. Build a real-time, low-latency voice conversation engine

- Establish a persistent **Gemini Live** session via `client.aio.live.connect` in `main.py`.
- Stream microphone audio at **16 kHz PCM** and play back assistant audio at **24 kHz**.
- Implement **echo suppression** by gating the microphone whenever R.A.Y.A is speaking (`_is_speaking` lock).
- Support **input + output transcription** so both sides of the conversation appear in the UI log.
- Achieve a target end-to-end response latency of **< 1.5 seconds** from end-of-user-utterance to start-of-assistant-audio.

**Verification:** Section 5.1.2 (Audio Testing Results).

### O2. Provide a comprehensive desktop tool layer

- Implement at least **15 declarative tools** spanning productivity, web, vision, system control, development, and entertainment.
- Each tool must have a **strict JSON schema** declared in `TOOL_DECLARATIONS` and a corresponding dispatch branch in `RayaLive._execute_tool`.
- All tool calls must run in a **thread executor** so they never block the audio event loop.
- Each tool must report **status to the UI** (running / done / failed) via `push_activity`.

**Verification:** Section 5.2.2 (Text-Based Testing Results), Section 6.5 (Automation and Tool Execution Evaluation).

### O3. Implement a persistent, structured long-term memory system

- Store user-revealed facts in **six categories**: identity, preferences, projects, relationships, wishes, notes.
- Enforce a **size cap** (2200 characters) with **oldest-first trimming** (`_trim_to_limit` in `memory_manager.py`).
- Format the memory into a natural-language block (`format_memory_for_prompt`) and inject it into the system instruction on every session start.
- Provide a **silent `save_memory` tool** that the LLM can call without acknowledging it to the user.

**Verification:** Section 6.4 (Memory and Context Evaluation).

### O4. Add visual awareness through screen and webcam capture

- Implement `actions/screen_processor.py` to capture the active display (via `mss`) or webcam (via OpenCV).
- Send the captured frame to **Gemini Vision** with the user's question and stream the spoken answer back through the UI player.
- Ensure the main Live model **stays silent** while the vision module is speaking, to avoid two voices overlapping.

**Verification:** Section 5.1 (Audio Based Testing — vision scenarios).

---

## B. Secondary Objectives

### O5. Autonomous multi-step task planning

- Build a dedicated **Planner LLM** (`agent/planner.py`) that converts an arbitrary goal into a JSON plan of up to 5 tool-call steps.
- Build an **Executor** (`agent/executor.py`) that runs each step with up to **3 retries**, intelligent error classification (RETRY / SKIP / FIX / ABORT), and **inter-step content injection** (e.g., piping search results into a file write).
- Support **replanning** when a step fails and the executor needs to revise the remaining plan.
- Provide a **priority task queue** (`agent/task_queue.py`) with PENDING / RUNNING / COMPLETED / FAILED / CANCELLED states, a worker thread, and live UI snapshots.

**Verification:** Section 6.5.

### O6. Cross-platform support

- Ensure the codebase runs on **Windows 10/11, macOS, and Linux**.
- Avoid Windows-only APIs in the core code path; isolate platform-specific dependencies (e.g., `pycaw`, `comtypes`, `win10toast`) inside individual action modules.
- Test the bimodal (audio + text) UI on each supported OS during development.
- Document any platform-specific gaps in the README.

**Verification:** Section 5 (executed on the development machine; gaps documented in Section 6.7 — Limitations Observed).

### O7. Deliver a one-click Windows installer

- Provide `scripts/build_windows.ps1` to drive PyInstaller and Inno Setup.
- **Bundle Playwright browsers** into the installer so web automation works completely offline.
- Place the user's `api_keys.json` in **local app data** so first-run config writes succeed without admin rights.
- Produce a final artifact at `installer/output/RAYA-Setup-v2.4.0.exe`.

**Verification:** Successful build + installation on a fresh Windows 11 machine.

### O8. Robust error handling and graceful degradation

- Every tool wraps execution in `try/except`; failures are spoken back to the user via `speak_error` and logged in the UI.
- The Live session **auto-reconnects** after any transport error, with a 3-second backoff (`while True` in `RayaLive.run`).
- The agent planner falls back to a **single web_search step** if JSON parsing of the plan fails (`_fallback_plan`).
- The memory module degrades to an **empty memory dict** if the JSON file is corrupt.

**Verification:** Section 6.6 (Overall Evaluation and User Experience), Section 6.7 (Limitations Observed).

---

## C. Tertiary Objectives

### O9. Maintain a clean, extensible architecture

- Each action module exposes a **uniform function signature** `(parameters, player, response, …)` so adding a new tool is a two-file change (the action + the declaration).
- Use **package-level imports** at the top of `main.py` to make the dependency graph explicit.
- Keep `core/prompt.txt` as the **single source of truth** for the assistant's persona and routing rules.

### O10. Privacy and user control

- Provide a **physical-style mute control** (F4 key + UI button) that gates the microphone at the audio-callback level.
- Store **all user data locally** (`memory/long_term.json`, `config/api_keys.json`).
- Send **zero telemetry** — the only outbound network traffic is the Gemini API.

### O11. Educational value and showcase

- Document every major subsystem (planner, executor, memory, vision, UI) with inline comments and a comprehensive report (this document).
- Provide a public README with quick-start instructions, capabilities table, and licensing.
- Use the project as a portfolio artifact suitable for academic submission and technical interview demonstration.

---

## D. Specific, Measurable Targets

To make objectives concrete, the following numeric targets were set at the start of the project. They are evaluated in Section 6.

| Target | Metric | Goal | Achieved |
|---|---|---|---|
| End-to-end voice response latency | seconds | < 1.5 s | Yes (typical 0.8–1.2 s) |
| Number of working tools | count | ≥ 15 | 20 |
| Memory size cap | characters | 2200 | Enforced |
| Plan length | steps | ≤ 5 | Enforced (`Max 5 steps` in planner prompt) |
| Tool-call retries per step | retries | ≤ 3 | Enforced |
| Replan attempts per goal | attempts | ≤ 2 | Enforced (`MAX_REPLAN_ATTEMPTS = 2`) |
| Auto-reconnect on transport error | behavior | Yes, with 3 s backoff | Yes |
| Cross-platform Python codebase | OSes | Windows, macOS, Linux | Yes (Windows fully; macOS / Linux experimental) |
| Installer size | MB | < 500 MB (with Playwright bundled) | ~ 350 MB |
| Languages supported in conversation | count | Any language Gemini supports | All (~50+) |

---

## E. Out-of-Scope Objectives

To keep the project bounded and shippable, the following were **explicitly excluded** from the objectives:

- **On-device LLM inference.** R.A.Y.A always uses the Gemini API for reasoning. A future version could explore local models (see Section 7 — Future Scope).
- **Multi-user / hosted deployment.** R.A.Y.A is a single-user desktop application.
- **Mobile clients.** No iOS / Android port.
- **Custom wake-word.** The assistant is always-on while unmuted; there is no "Hey R.A.Y.A" trigger.
- **End-to-end encrypted sync.** Memory is local-only; multi-machine sync is left to the user (e.g., via Dropbox).
- **Plugin marketplace.** Tools are added by editing the source code, not loaded dynamically.

---

## F. Mapping Objectives to Report Sections

For traceability, each objective maps to a downstream report section where it is designed, implemented, and evaluated:

| Objective | Designed in | Implemented in | Evaluated in |
|---|---|---|---|
| O1 — Voice engine | 3.1 (Block Diagram) | 4.2.1 (`main.py`) | 5.1, 6.2 |
| O2 — Tool layer | 3.1, 3.2 | 4.2.5 | 5.2, 6.5 |
| O3 — Memory | 3.1 | 4.2.6 | 6.4 |
| O4 — Vision | 3.1 | 4.2.5 (`screen_processor`) | 5.1 |
| O5 — Planning | 3.1 | 4.2.2 (`core/agent.py`) | 6.5 |
| O6 — Cross-platform | 4.1 | 4.2 | 6.7 |
| O7 — Installer | 4.1 | `scripts/build_windows.ps1` | 6.6 |
| O8 — Error handling | 3.1 | 4.2.2, `agent/error_handler.py` | 6.6, 6.7 |
| O9 — Extensibility | 4.1 | `actions/` layout | 6.8 |
| O10 — Privacy | 1.3 | UI mute + local files | 6.6 |
| O11 — Education | All | This report | 8 (Conclusion) |

---

## Summary

R.A.Y.A v2.4 is built around **four primary objectives** (low-latency voice, comprehensive tools, persistent memory, visual awareness), reinforced by **four secondary objectives** (autonomous planning, cross-platform support, polished packaging, robust error handling), and crowned by **three tertiary objectives** (extensibility, privacy, educational value). Every architectural decision, every module in the codebase, and every section of this report is traceable back to one or more of these eleven objectives — giving the project a clear, defensible scope and a rigorous basis for evaluation.
