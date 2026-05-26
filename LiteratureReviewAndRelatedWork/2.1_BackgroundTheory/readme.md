# 2.1 Background Theory

This section establishes the theoretical foundations on which R.A.Y.A v2.4 is constructed. Each subsection introduces a concept, explains how it works, and then connects it directly to a specific module or design decision in the project.

---

## 2.1.1 Large Language Models (LLMs)

A **Large Language Model** is a deep neural network — typically a Transformer with billions to trillions of parameters — trained on enormous corpora of text to predict the next token given a context. Modern LLMs are *autoregressive*: they generate output one token at a time, where each new token is conditioned on all preceding tokens. The probability distribution over the vocabulary is computed by a stack of self-attention layers, allowing the model to weigh distant context when making each prediction.

Key properties of LLMs that R.A.Y.A relies on:

| Property | Why it matters for R.A.Y.A |
|---|---|
| **In-context learning** | The model adapts behavior from instructions in the prompt (`core/prompt.txt`) without retraining |
| **Few-shot learning** | The planner prompt (`agent/planner.py`) shows 7 example goal→plan mappings to bias the model toward correct JSON |
| **System instructions** | The `LiveConnectConfig.system_instruction` field anchors the assistant's persona across an entire session |
| **Temperature / sampling** | Implicitly controlled by the API; lower temperatures favor deterministic tool-call generation |

R.A.Y.A specifically uses **Google's Gemini 2.5 family**, the latest generation of Google's multimodal models, in three roles:

1. **Gemini 2.5 Flash Native Audio Preview** — the primary conversation model in `main.py` (`LIVE_MODEL`).
2. **Gemini 2.5 Flash Lite** — the cheap, fast planner in `agent/planner.py`.
3. **Gemini 2.5 Flash** — the standard model used for code generation (`agent/executor.py::_run_generated_code`), replanning, and translation.

---

## 2.1.2 Multimodal Models and the Live API

A **multimodal model** is one that natively accepts and produces more than one type of media — typically text, images, audio, and (in newer models) video. Gemini 2.5 is multimodal: a single forward pass can ingest a spoken question, look at an attached image, and emit a spoken response.

The **Live API** is Google's bidirectional streaming protocol that exposes this capability over a long-lived WebSocket-like connection. Theoretical building blocks:

- **Streaming I/O.** The client sends audio chunks (e.g., 1024 samples of 16-bit PCM at 16 kHz) as they are captured, and the server streams audio chunks back as the model speaks. There is no fixed "request → response" round trip — the conversation is continuous.
- **Server-side VAD (Voice Activity Detection).** The Live API detects end-of-utterance on the server, eliminating the need for client-side wake-word logic.
- **Tool calling.** Every turn can include a `tool_call` message that names a function and its arguments; the client executes the function and replies with a `tool_response`.
- **Session resumption.** The `SessionResumptionConfig` lets the server gracefully resume a dropped connection, used in `RayaLive._build_config`.

R.A.Y.A uses all four of these primitives in `main.py`, with four concurrent asyncio tasks: `_send_realtime`, `_listen_audio`, `_receive_audio`, and `_play_audio`.

---

## 2.1.3 Speech Processing Pipeline

Even with a native-audio LLM, R.A.Y.A still has to do considerable signal handling on the client side:

- **Capture.** `sounddevice.InputStream` reads PCM at 16 kHz mono, 16-bit, with a `blocksize=1024` block (≈ 64 ms of audio).
- **Echo suppression.** When the model is speaking, the microphone callback drops incoming frames (`_is_speaking` lock) to prevent the assistant from hearing itself.
- **Mute gate.** A user-controlled mute flag (`ui.muted`) cuts the audio stream at the same callback level — the muted audio never enters the Python queue.
- **Playback.** `sounddevice.RawOutputStream` plays back the server's 24 kHz, 16-bit PCM stream.

Theoretical concepts at work:

- **Pulse Code Modulation (PCM):** the simplest digital audio representation — uniform samples of the analog signal's amplitude. R.A.Y.A uses `int16` PCM throughout, the native format Gemini Live expects.
- **Nyquist sampling theorem:** to faithfully represent a signal of frequency *f*, the sampling rate must exceed 2*f*. 16 kHz captures speech (which mostly lives below 8 kHz) cleanly.
- **Asynchronous queues:** producer/consumer decoupling between the capture callback, the network sender, the network receiver, and the playback loop. R.A.Y.A uses `asyncio.Queue` with `maxsize=10` for back-pressure.

---

## 2.1.4 Tool / Function Calling

**Function calling** is the technique whereby an LLM, instead of answering a user directly, emits a structured request to invoke an external function. The host application executes the function and feeds the result back into the conversation as a new message. This converts the LLM from a pure text predictor into an **agentic system**.

The contract in Gemini Live consists of:

1. **A declaration array** — JSON Schema for each function (name, description, parameters with types). R.A.Y.A registers 20 declarations in `TOOL_DECLARATIONS`.
2. **A `tool_call` event** — emitted by the model when it decides a tool is appropriate.
3. **A `tool_response` event** — emitted by the client after running the function, carrying the result.

Key theoretical considerations:

- **Tool grounding via description.** The LLM's choice of tool depends almost entirely on the natural-language description string. R.A.Y.A's descriptions are deliberately opinionated (e.g., *"THE ONLY tool for ANY Steam or Epic Games request"*) to disambiguate overlapping tools.
- **Schema strictness.** Loose schemas lead to hallucinated arguments. R.A.Y.A's schemas mark required fields explicitly and constrain enums in the description (e.g., `"go_to | search | click | type | scroll …"`).
- **One-call policy.** R.A.Y.A's system prompt instructs the model to call each tool *exactly once* per turn to avoid runaway loops — a known failure mode in agentic LLMs.

---

## 2.1.5 Agent Architectures: Planner, Executor, ReAct

R.A.Y.A's `agent_task` subsystem implements a classic **Planner-Executor** pattern, sometimes called **deliberative agent architecture**:

```
   ┌────────────┐    ┌──────────────┐    ┌─────────────┐
   │   Goal     │ -> │   Planner    │ -> │   Plan      │
   └────────────┘    └──────────────┘    └─────────────┘
                                                │
                                                v
                                       ┌─────────────────┐
                                       │   Executor      │
                                       │   step-by-step  │
                                       └─────────────────┘
                                                │
                              ┌─────────────────┴────────────────┐
                              │                                  │
                        success path                       failure path
                              │                                  │
                              v                                  v
                       ┌────────────┐                  ┌──────────────────┐
                       │  Summary   │                  │  Error analyzer  │
                       └────────────┘                  │  RETRY/SKIP/FIX  │
                                                       │   /ABORT/REPLAN  │
                                                       └──────────────────┘
```

Related theoretical work:

- **ReAct (Reason + Act)** — interleaves reasoning traces with tool calls; R.A.Y.A's main Live loop is closer to ReAct (think briefly, then act), while `agent_task` is a stricter Planner-Executor (think once, then act many times).
- **AutoGPT / BabyAGI** — recursive task-decomposition agents from 2023; influential but prone to loops. R.A.Y.A bounds plan length to **5 steps** and replan attempts to **2** to prevent this failure mode.
- **Reflexion / Self-Refine** — feedback loops where the agent critiques its own outputs. R.A.Y.A's `error_handler.analyze_error` plays a similar role on a per-step basis.

---

## 2.1.6 Error Handling and Recovery in Autonomous Agents

A serious agent must treat errors as first-class signals, not exceptions. R.A.Y.A's `agent/error_handler.py` exposes a four-way decision space:

| Decision | When | Behavior |
|---|---|---|
| **RETRY** | Transient / network errors | Wait 2 s, try again (up to 3 attempts) |
| **SKIP** | Non-critical step failed | Mark as done, continue |
| **FIX** | Wrong tool / wrong parameters | Generate an alternative tool call |
| **ABORT** | Unrecoverable | Stop and report to the user |

Underlying theory:

- **Exponential backoff / jitter** for retries (R.A.Y.A uses fixed 2 s for simplicity).
- **Saga pattern** in distributed systems — each step is a small unit of work that can be compensated or skipped without rolling back the whole plan.
- **Replanning** as a high-level retry — if local recovery fails, regenerate the remaining plan with knowledge of which steps already succeeded (`agent/planner.py::replan`).

---

## 2.1.7 Persistent Memory in Conversational Agents

Modern conversational agents face a fundamental limitation: the LLM is stateless. To create the illusion of continuity, the host application must store relevant facts externally and re-inject them into the context.

R.A.Y.A's memory system in `memory/memory_manager.py` is built on three theoretical ideas:

1. **Schema-based long-term memory.** Six fixed categories (identity, preferences, projects, relationships, wishes, notes) provide a stable taxonomy, mirroring how human assistants organize what they know about a person.
2. **Recency-biased eviction.** When the memory exceeds 2200 characters, the oldest entries are dropped first (`_trim_to_limit`) — analogous to LRU caching.
3. **Prompt-time formatting.** At session start, the structured memory is rendered into natural-language hints (`format_memory_for_prompt`) and prepended to the system instruction, so the model can use the facts naturally without being told "this is your memory."

This is closely related to **Retrieval-Augmented Generation (RAG)** in spirit — the difference being that R.A.Y.A's memory is small enough to inject in full, so no embedding-based retrieval is needed.

---

## 2.1.8 Vision Pipelines and Multimodal Grounding

The `screen_process` tool turns R.A.Y.A into a multimodal agent on demand. Theoretical components:

- **Frame capture.** `mss` provides fast, cross-platform screen capture by hooking the OS compositor's framebuffer. Webcam capture goes through `opencv-python` (`cv2.VideoCapture`).
- **Image encoding.** Captured frames are encoded as PNG/JPEG and uploaded to the Gemini Vision API as inline `image/png` parts.
- **Question-conditioned vision.** Unlike pure image captioning, the model is given a specific natural-language question (`text` parameter) and produces a focused answer.
- **Audio-out coupling.** The vision module speaks its answer through the same `RayaUI.speak` path, while the main Live model is instructed to stay silent — preventing the two voices from talking over each other.

The relevant background theory comes from **Vision-Language Models (VLMs)** — Transformers that jointly encode pixels and text into a shared embedding space (e.g., CLIP, Flamingo, Gemini Vision). The user does not need to understand this to use R.A.Y.A, but the architecture relies on the VLM doing its job within a single API call.

---

## 2.1.9 Asynchronous Concurrency in Python

R.A.Y.A is fundamentally an **I/O-bound application** — most time is spent waiting on the network, the microphone, the speakers, or external subprocesses. Python's `asyncio` is the natural fit:

- **Event loop.** A single thread runs an event loop that multiplexes many awaitable tasks. R.A.Y.A's `RayaLive.run` uses `asyncio.TaskGroup` to spawn four cooperative tasks.
- **Bridging sync and async.** The microphone callback runs in a `sounddevice` worker thread; it forwards data to the asyncio loop via `loop.call_soon_threadsafe`. Tool calls, which are blocking, are dispatched through `loop.run_in_executor`.
- **Back-pressure.** Bounded `asyncio.Queue(maxsize=10)` ensures that a slow consumer cannot run the producer out of memory.
- **Graceful cancellation.** `TaskGroup` propagates exceptions and cancels sibling tasks deterministically — important when the Live session drops.

Threading is also used selectively: the UI runs on the Tkinter main thread, the audio worker runs on a `threading.Thread`, and the task queue worker (`agent/task_queue.py::_worker_loop`) runs in its own thread.

---

## 2.1.10 GUI Application Theory: Tkinter, Event Loops, State Machines

R.A.Y.A's UI (`ui.py`) is a Tkinter application with three theoretical underpinnings:

- **Single-threaded event loop.** Tkinter requires all widget operations to happen on the main thread; cross-thread updates use `root.after(0, ...)` to schedule work.
- **State machine.** The UI explicitly tracks three states — `LISTENING`, `THINKING`, `SPEAKING` — and renders different visuals (animated face, color cues) for each. Transitions are triggered by the audio loop.
- **Activity logging.** Append-only log widget that records every tool call and user/assistant utterance, providing a transparent audit trail.

---

## 2.1.11 Packaging and Deployment

The final theoretical block is **application packaging**:

- **PyInstaller** freezes a Python interpreter and all dependencies into a single executable folder. The `getattr(sys, "frozen", False)` check at the top of `main.py` (`get_base_dir`) makes the codebase aware of whether it's running from source or from a bundle.
- **Inno Setup** produces a Windows installer that places the frozen bundle into Program Files, creates Start Menu shortcuts, and writes user data into local AppData.
- **Bundled Playwright browsers.** A Playwright Chromium build is copied into `ms-playwright/` and the `PLAYWRIGHT_BROWSERS_PATH` environment variable is set at startup (`_configure_playwright_browser_path`) so web automation works offline.

---

## Summary of Theoretical Foundations

R.A.Y.A v2.4 sits at the intersection of **eight theoretical domains**:

1. Large language models and in-context learning
2. Multimodal streaming AI via the Gemini Live API
3. Real-time PCM speech capture and playback
4. Function / tool calling and structured agent protocols
5. Planner-Executor agent architectures with replanning and error recovery
6. Schema-based persistent memory with size-bounded eviction
7. Vision-language models for question-conditioned image understanding
8. Asynchronous concurrency, GUI event loops, and application packaging

Each domain contributes a specific module to the codebase, and together they form the complete system that the next sections of this report will dissect in detail.
