# 6.7 Limitations Observed

This section names the real limitations of R.A.Y.A v2.4 honestly. The goal is not to dwell on shortcomings but to give a clear-eyed picture of what the system **does not** do well — both so a reviewer can calibrate the rest of the report, and so the future-work agenda in Section 7 can be derived from real evidence rather than speculation.

Limitations are organized into seven categories, ranging from hard architectural constraints to soft UX gaps.

---

## 6.7.1 Architectural Limitations

These are properties baked into the system design. They are not bugs and cannot be fixed without significant rework.

### Cloud dependency for reasoning

R.A.Y.A relies on the Gemini API for every model call. If the network is down, the API is unreachable, or the user has exhausted their quota, the assistant cannot function — even though all the local code is intact. This is a deliberate scope cut (the project does not include any local model), but it is the single most consequential limitation in the system.

### Single-user, single-machine

R.A.Y.A is designed for one user on one machine. There is no multi-user model, no cloud-hosted variant, no synchronization of memory across devices. A user with a desktop and a laptop would have two independent R.A.Y.A instances with separate memory files.

### No on-device LLM fallback

Related to the cloud dependency: there is no degraded mode where R.A.Y.A could still execute simple commands without network access. Even `open_app` requires a tool-call decision from the model.

### Bounded plan length and replan attempts

The agent's bounds (5 steps, 3 retries, 2 replans) are deliberate guards against runaway behavior, but they also limit the maximum complexity of goals the assistant can pursue. A 12-step research project, for example, is out of reach without manual decomposition by the user.

### Single Live session at a time

Only one conversation runs at a time. If the user wanted two parallel conversations (for two different projects), there is no architectural support.

---

## 6.7.2 Platform Limitations

R.A.Y.A is targeted at Windows 10/11 first. The other supported platforms have known gaps.

### macOS and Linux are experimental

Cross-platform code paths exist for the core voice, text, memory, and basic OS automation. However:

- Volume control via `pycaw` is Windows-only.
- Notification toasts via `win10toast` are Windows-only.
- The `pywinauto` UIA paths in `actions/computer_settings.py` and `actions/send_message.py` are Windows-only.
- The Steam/Epic library paths in `game_updater.py` are Windows-conventional.

On macOS or Linux, the affected tools either silently no-op or return an error. The assistant remains usable for the majority of features, but parity is not claimed.

### Windows-only installer

The PyInstaller + Inno Setup build pipeline produces a Windows installer only. macOS and Linux users currently have to run from source. A signed `.app` bundle for macOS and a `.deb`/`.rpm` for Linux would each require their own packaging effort.

### No mobile

iOS and Android are explicitly out of scope. The assistant has no companion app, no remote-control surface, and no way to be triggered from a phone.

---

## 6.7.3 Voice and Audio Limitations

### Cold-tool latency spikes

The first invocation of a heavy tool (especially `browser_control`) after a fresh session is noticeably slower than steady-state turns. Subsequent invocations are fast because the underlying subprocess (Chromium) remains warm.

### Background noise degrades accuracy

In acoustic environments with significant background noise (music, chatter, fans at ~60 dB+), tool-call accuracy drops. The failure mode is graceful — the assistant re-prompts for clarification rather than hallucinating an action — but it does shift conversational burden back to the user.

### No barge-in cancel during tool execution

Once a long-running tool has started, the user cannot abort it by saying "stop." The user has to wait for the tool to finish or close the UI. This is a meaningful UX gap that is referenced repeatedly throughout the evaluation sections.

### Single voice option

The Charon voice is hard-coded. A future enhancement could expose voice selection through the UI, but currently the user gets one voice for all sessions.

### Audio device handling is minimal

The system uses the OS default input and output devices and does not provide a way to choose between them. Users who plug in a USB headset mid-session have to restart R.A.Y.A for the change to take effect.

---

## 6.7.4 Tool and Agent Limitations

### Agent plans are invisible to the user

The planner emits its JSON plan, and the executor walks through it, but neither surfaces the plan to the UI before execution begins. The user cannot review the plan or veto a particular step.

### No tool composition outside the agent

The single-tool path cannot chain two tools without going through `agent_task`. For users who want lightweight two-step actions ("take a screenshot and save it to Desktop"), the only option is to invoke the agent, which feels heavier than necessary.

### Tool descriptions are static

Tools are registered at session boot through `TOOL_DECLARATIONS`. There is no way to enable/disable tools at runtime, scope tools to particular contexts, or have the user customize the available toolset per session.

### Cooperative-only assumptions

`send_message` assumes the user is logged into WhatsApp Web; `game_updater` assumes Steam/Epic are installed. When these prerequisites are missing, the tool returns an error rather than offering to help fix the prerequisite.

### Generated-code execution is fully trusted

`code_helper` and `dev_agent` run generated Python code on the user's machine without sandboxing. This is acceptable for a single-user personal assistant on the user's own machine, but it means a malicious prompt (e.g., from a clipboard injection) could in principle cause harm. Defending against this is not addressed in v2.4.

---

## 6.7.5 Memory Limitations

### No explicit "forget" workflow

The `forget` function in `memory_manager.py` is callable from code, but there is no exposed tool or UI affordance for the user to say "forget what I told you about my school." The user has to edit `long_term.json` manually.

### Trimming is silent

When the 2200-character cap is reached, the oldest entry is dropped without notification. The user might be surprised later when a previously-known fact is no longer recalled.

### Single-category placement

The save_memory tool requires picking one of six categories. Some facts (e.g., "I usually code at night") do not fit cleanly into any of them and end up in the `notes` bucket where they are easy to miss later.

### No memory search beyond full-context injection

Memory is always loaded as a single block at session start. There is no facility for the agent to query "do I know the user's birthday?" mid-session except by inspecting the system prompt indirectly.

### No memory across devices

As noted in 6.7.1, there is no synchronization story. A user who reinstalls R.A.Y.A or moves to a new machine starts with empty memory.

---

## 6.7.6 UI and Discoverability Limitations

### No onboarding tour

The first-time user is dropped into a LISTENING state with no guidance. They have to discover capabilities through experimentation or by reading the README.

### No history view

Past agent tasks and their results are visible only in the live activity log, which is bounded. There is no persistent history pane to revisit "what did the assistant do for me yesterday?"

### No settings panel

Volume, voice, language preference, mute keybinding, and memory contents are all either hard-coded or hidden in JSON files. A UI settings tab would be a meaningful quality-of-life improvement.

### Text input is single-line

The Tkinter Entry widget allows only single-line input, so pasting multi-line content (e.g., code blocks with line breaks) requires using `\n` escapes or pasting the content as a single line. A multi-line text widget would be more natural.

### Activity log is text-only

The log shows tool names and results as text. Screenshots, file paths, and other rich content cannot be displayed inline.

---

## 6.7.7 Evaluation and Measurement Limitations

### No formal benchmarking

The system has been evaluated through a manual test campaign (Section 5). It has not been benchmarked against a fixed dataset, an automated test suite, or a third-party evaluation framework.

### Single-developer accent profile

The voice testing was performed primarily by one speaker with one accent (Indian English). Generalization to other accents, dialects, and speaker populations is plausible but unverified.

### No multi-user UX study

The UX claims in Sections 6.2–6.6 reflect the developer's own experience with the system. A formal study with multiple participants would be required to validate them more broadly.

### No long-haul reliability test

The system has been used for sessions of up to roughly an hour at a time. There has been no 24-hour stress run that would surface slow leaks, gradual desync, or rare failure modes.

---

## 6.7.8 Severity Classification

The limitations above vary in severity. A rough classification:

| Severity | Limitation |
|---|---|
| **High** (blocks meaningful workflows) | No barge-in cancel; no on-device fallback when offline |
| **Medium** (degrades but does not block) | macOS/Linux gaps; cold-tool latency spikes; no plan visibility in UI; single-line text input |
| **Low** (quality-of-life) | No settings panel; no history view; no onboarding tour; single voice option |
| **Latent** (no observed impact but worth noting) | Trusted execution of generated code; silent memory trimming; no multi-user UX validation |

The High-severity items are the most defensible targets for Section 7's future work.

---

## 6.7.9 What These Limitations Imply for v2.5

A concise translation of the limitations into the next-release backlog:

1. **Voice-driven cancel** for running agent tasks.
2. **Plan visibility** — surface the planner's JSON to the activity log as steps are decided.
3. **Pre-warmed browser** at startup to eliminate cold-tool latency.
4. **Settings panel** for voice, language, mute key, memory inspection.
5. **Explicit forget tool** with UI affordance.
6. **macOS / Linux verification** — actually run the full audio test campaign on those platforms.
7. **Multi-line text input** for pasting code and structured content.
8. **History view** — persistent agent task log with timestamps and outputs.
9. **Sandboxed code execution** — at minimum, a confirmation dialog for `code_helper`/`dev_agent` runs.
10. **Initial onboarding flow** — five-prompt tour shown on first launch.

These are tractable engineering items, not research challenges. The fact that the limitations resolve to a clean list of next steps is itself a sign that the v2.4 architecture is sound — the gaps are visible and patchable, not buried in foundational decisions.

---

## 6.7.10 Summary

R.A.Y.A v2.4 has real limitations across seven categories: architectural (cloud dependence, single-user), platform (macOS/Linux gaps, no mobile), audio (cold-tool spikes, no barge-in cancel, noise degradation), tool/agent (no plan visibility, no in-flight cancel, trusted code execution), memory (no explicit forget, silent trimming), UI/discoverability (no onboarding, no settings, no history), and evaluation (no formal benchmarks, single-accent testing). The limitations are honest scope cuts and not hidden defects, and they cluster into a clean v2.5 backlog of roughly ten engineering items. The project's foundations are sound enough that none of these limitations require redesign to address — which is the most important thing a limitations section can say.
