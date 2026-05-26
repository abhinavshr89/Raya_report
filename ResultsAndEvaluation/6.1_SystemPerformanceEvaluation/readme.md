# 6.1 System Performance Evaluation

This section evaluates R.A.Y.A v2.4 as a **running system** — how well it behaves under realistic load, how efficiently it uses host resources, how it scales across the lifetime of a session, and how reliably it survives the kinds of disturbances that occur on a typical developer's desktop. The focus here is on the *characteristics* of the system as a runtime, not on individual feature correctness (which was covered in Section 5).

---

## 6.1.1 Evaluation Framework

System performance is judged along five axes, each chosen because it maps to a real concern a daily user would have:

| Axis | Question it answers |
|---|---|
| **Startup behavior** | How long from launching the app until it is usable? |
| **Steady-state efficiency** | How heavy is R.A.Y.A while idle and listening? |
| **Burst behavior** | What happens when several tools fire in quick succession? |
| **Network resilience** | What happens when the connection wobbles? |
| **Long-session stability** | Does it degrade after hours of use? |

Each axis is evaluated qualitatively — describing observed behavior and tracing it to a concrete design choice in the codebase, rather than reciting numbers already presented in Section 5.

---

## 6.1.2 Startup Behavior

R.A.Y.A's cold start is short and predictable. The Tkinter UI window appears almost immediately after `python main.py`, because heavyweight imports inside the agent and action modules are confined to lazy `from … import …` statements inside their handler functions (e.g., `from agent.task_queue import get_queue` is imported only when `agent_task` actually fires).

The bulk of perceived startup time is the **Gemini Live connection handshake**, not the local code path. This was a deliberate split: the UI must always feel responsive, so the Live session connect happens on a dedicated runner thread while the main thread keeps Tkinter alive.

What the evaluation confirmed about startup:

- **The first-run experience** is gated by the API-key dialog (`ui.wait_for_api_key`). This is fast on subsequent boots because the key is cached in `config/api_keys.json`.
- **No model warm-up cost** lives on the client side. R.A.Y.A does not load any local model; all reasoning is delegated to Gemini.
- **Frozen-binary boot** (PyInstaller bundle) is slightly slower than running from source — typical PyInstaller extract overhead. The `get_base_dir()` idiom handles both cases without code changes.

---

## 6.1.3 Steady-State Efficiency

While idle and listening, R.A.Y.A's footprint is dominated by three things:

1. The **Tkinter face animation**, which redraws at a modest frame rate on the main thread.
2. The **microphone callback**, which fires every ~64 ms and either enqueues the frame (when not muted and not speaking) or discards it.
3. The **Gemini Live WebSocket-style stream**, which keeps a persistent TLS connection open.

Two architectural choices keep idle cost manageable:

- The **mute-at-source gate** means a muted assistant has zero outbound network cost from the audio path — muted frames are discarded inside the sounddevice callback before they reach the asyncio queue.
- The **bounded outbound queue** (`asyncio.Queue(maxsize=10)`) prevents runaway memory growth if the network temporarily falls behind the microphone.

Memory grows only marginally over a session because the long-term memory file is **size-bounded at 2200 characters** and trimmed on every update (`_trim_to_limit` in `memory_manager.py`).

---

## 6.1.4 Burst Behavior

When the agent task queue is asked to handle a multi-step goal, several things happen at once: the planner LLM is called, several action functions run sequentially in executor threads, and the UI must continue rendering. R.A.Y.A handles this gracefully because:

- The audio event loop is **never blocked** — all tool calls go through `loop.run_in_executor`.
- Each agent task runs in its **own thread** (`_run_task` in `task_queue.py` spawns a fresh thread per task), so the worker loop is free to pick up a new task immediately if `max_concurrent` permits.
- Browser-driven tools experience a one-time cold-start when Playwright launches Chromium for the first time; subsequent calls reuse the warm browser, so burst latency is concentrated on the first invocation.

The most interesting burst behavior is the **inter-step content injection** path in `agent/executor.py::_inject_context`. When a multi-step plan calls `web_search` twice and then `file_controller.write`, the executor automatically collects the search outputs, translates them if needed, and stuffs them into the write step. This means burst behavior is not just "many tools run fast" — it is "many tools share data sensibly." That is the engineering result that matters most for real productivity workflows.

---

## 6.1.5 Network Resilience

R.A.Y.A is fundamentally cloud-dependent for reasoning, so resilience to network disturbance is a central evaluation axis.

Two layers of resilience are baked in:

| Layer | Mechanism | Observed behavior |
|---|---|---|
| **Session** | `RayaLive.run` wraps the entire connect-and-run block in `while True` with a 3-second backoff | A dropped WebSocket reconnects automatically; the UI shows a brief "Connection issue / Reconnecting…" notice and then returns to LISTENING |
| **Step** | The executor retries each agent step up to 3 times, then escalates to the error handler | Transient HTTP errors on `web_search` or `weather_report` recover silently without user intervention |

A short network blackout (under 10 seconds) is functionally invisible — the user can keep speaking, and R.A.Y.A picks up where it left off because session resumption is enabled (`types.SessionResumptionConfig` in `_build_config`).

A long blackout (multiple minutes) prevents any model interaction, but the UI remains responsive, the long-term memory is intact on disk, and recovery on connection restore is automatic. This is the cleanest possible failure mode for a cloud-dependent assistant: degrade fully but recover automatically.

---

## 6.1.6 Long-Session Stability

Several factors that commonly destabilize long-running Python applications were specifically considered:

- **No accumulating buffers.** The audio queues are either bounded (outbound) or continuously drained (inbound).
- **No accumulating timers.** All scheduling lives in the asyncio event loop and is bound to the lifetime of the connection.
- **No accumulating files.** Generated code from `code_helper`/`dev_agent` is written to user-chosen paths or temp files cleaned up after execution.
- **Memory growth is capped.** The 2200-character cap on `memory/long_term.json` means session memory cannot grow without bound.
- **Browser sessions are explicit.** `browser_control` exposes `close` and `close_all` actions, so the user (or the agent) can release browser memory at will.

Over a sustained interactive session, the dominant observable change is in the **activity log inside the UI**, which uses an append-only deque. The deque is bounded by design, so even an all-day session does not grow the log indefinitely.

The auto-reconnect loop also provides an implicit refresh — every reconnect rebuilds the `LiveConnectConfig` and reloads the system prompt + memory from disk, effectively giving the session a clean state at the model layer at zero user-visible cost.

---

## 6.1.7 Resource Profile by Subsystem

A qualitative breakdown of where R.A.Y.A spends its time and memory during a typical session:

| Subsystem | CPU usage | Memory usage | Network usage |
|---|---|---|---|
| Tkinter UI | Light (animation thread) | Light | None |
| Microphone capture | Light (callback every ~64 ms) | Tiny (single buffer) | None |
| Gemini Live stream | Light when idle, moderate while speaking | Small | Continuous TLS connection |
| Vision (`screen_process`) | Spike during capture and encode | Moderate during capture | Burst upload |
| Browser automation | Heavy (Chromium subprocess) | Heavy | Moderate to heavy |
| Agent task queue | Light | Small per task | None directly |
| Memory subsystem | Negligible | Tiny (single JSON ≤ 2200 chars) | None |

The browser is the most expensive subsystem by a wide margin — which is why `close_all` exists as an explicit action and why Playwright browsers are bundled instead of launched as separate apps.

---

## 6.1.8 Evaluation Against Performance Objectives

Mapping the observed system characteristics back to the performance-related objectives stated in Section 1.4:

| Objective | Performance characteristic | Evaluation |
|---|---|---|
| **O1 — Low-latency voice** | Streaming audio with bounded queues, mute-at-source, no client-side ML | Met. The latency budget is dominated by network + model, not client code. |
| **O2 — Comprehensive tool layer** | 20 tools dispatched via a single thread-executor path, no global locks | Met. Tools do not interfere with the audio loop. |
| **O5 — Autonomous planning** | Per-task thread + bounded plan/retry/replan, cooperative cancel via flag | Met. The agent does not block other subsystems and cancels cleanly. |
| **O8 — Robust error handling** | Per-step retry, per-goal replan, per-session auto-reconnect | Met across three layers, each independent. |
| **O10 — Privacy** | Mute-at-source, local memory file, user-owned API key | Met by construction; no telemetry path exists in the code. |

Performance is not the most exciting axis to evaluate, but it is the foundation that everything else stands on. R.A.Y.A's performance profile is **deliberately modest** — most of its work is in the cloud — and the local code is engineered to stay out of the way of that cloud work.

---

## 6.1.9 Summary

R.A.Y.A v2.4 behaves as a well-mannered desktop application. Boot is fast, idle is light, bursts are absorbed through executor threads and a bounded queue, network disturbance recovers automatically through two layers of retry, and long sessions remain stable because every accumulating structure has an explicit cap. The system's performance personality is best summarized as **"thin client, thick reasoning"** — the local process is intentionally lean so that the cloud LLM has all the runway it needs.
