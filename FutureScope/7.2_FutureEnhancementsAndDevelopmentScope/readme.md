# 7.2 Future Enhancements and Development Scope

This section turns the architectural future-scope diagram of 7.1 into a **concrete enhancement roadmap**. Each item is described with its motivation, the proposed approach, the technical considerations that come with it, and the dependencies that would need to be in place for it to ship. The enhancements are grouped into three horizons matching the rings in 7.1: **near-term (v2.5)**, **mid-term (v2.6 — v3.0)**, and **long-term (v3.x and beyond)**.

At the end of the section, three cross-cutting concerns (security, accessibility, internationalization) are addressed since they apply across all horizons.

---

## 7.2.1 Horizon 1 — Near-Term Enhancements (v2.5)

The near-term enhancements close the gaps identified in Section 6.7 (Limitations Observed). All of them are bounded engineering tasks rather than research problems, and together they form the v2.5 release backlog.

### E1. Voice-driven cancellation of running agent tasks

**Motivation.** Once a slow `agent_task` is in flight, the user cannot abort it without quitting the UI. This was the highest-severity limitation in the evaluation.

**Approach.** Add a new tool `cancel_task` (or extend `agent_task` with an `action: "cancel"` parameter) that calls `task_queue.cancel(task_id)`. The model will route phrases like *"stop the current task"*, *"cancel it"*, or *"abort"* to this tool. The cooperative cancellation flag already exists in `agent/task_queue.py`; only the surface needs to be exposed.

**Considerations.** The model needs unambiguous task identification when more than one task is queued. The UI should also expose a per-task cancel button.

---

### E2. Plan visibility in the activity log

**Motivation.** The agent's plan is invisible to the user before execution begins. A failing plan is harder to diagnose because the user only sees the symptom.

**Approach.** When `executor.execute` receives a plan, publish each step's tool + description to the UI via `ui.push_activity(...)` *before* invocation begins. Each step appears in a "planned" state and transitions to "running", then "done"/"failed" as it executes.

**Considerations.** Avoid overwhelming the log with planner JSON. Render the human-readable description, not the raw parameters.

---

### E3. Pre-warmed Playwright at startup

**Motivation.** The first invocation of `browser_control` (or `flight_finder`, `youtube_video`) after a fresh session is the slowest perceptible moment in the audio path.

**Approach.** During `RayaLive.run` initialization, asynchronously launch a Chromium instance in headless mode and hold a reference. `browser_control` reuses this pre-warmed instance on first call.

**Considerations.** Increases idle memory footprint. Should be opt-in through a settings flag so users on lower-spec machines can disable it.

---

### E4. Settings panel in the UI

**Motivation.** Voice, language preference, mute keybinding, pre-warm toggle, and memory inspection are all currently hidden in JSON files or hard-coded. A dedicated UI surface would solve this.

**Approach.** Add a tabbed panel to the Tkinter UI:
- **Voice tab** — choose from available Gemini voice presets.
- **Behavior tab** — mute key, pre-warm toggle, auto-reconnect cadence.
- **Memory tab** — show current memory contents grouped by category; allow per-entry delete.
- **About tab** — version, API key reset.

**Considerations.** Settings persistence in `config/settings.json`; reload on change without requiring an app restart.

---

### E5. Explicit forget tool

**Motivation.** The user has no in-conversation way to say "forget what I told you about X." The `memory_manager.forget` function exists but is not exposed.

**Approach.** Register a new `forget_memory(category, key)` tool with the same silent-execution pattern as `save_memory`. The tool also has a UI surface in the memory tab from E4.

**Considerations.** The model needs robust description text so that "forget about my school" maps to `forget_memory(category="identity", key="school")` without ambiguity.

---

### E6. Multi-line text input

**Motivation.** The single-line text input is awkward for pasting code, stack traces, or structured content.

**Approach.** Replace the Tkinter `Entry` widget with a `Text` widget bound to **Shift+Enter** for newlines and **Enter** alone for submission.

**Considerations.** Preserve the existing single-Enter submit behavior for users who never paste multi-line content.

---

### E7. History view of past agent tasks

**Motivation.** Users have no way to revisit "what did the assistant do for me yesterday?" Past tasks scroll out of the activity log.

**Approach.** Introduce a SQLite-backed task history table that records each agent task's goal, plan, step outcomes, final summary, and timestamps. Add a history tab to the UI for browsing and replay.

**Considerations.** Defines the first persistent state beyond the long-term memory file. The SQLite database lives in the same user-local AppData directory as the memory file.

---

### E8. Sandboxed code execution

**Motivation.** `code_helper` and `dev_agent` currently run generated Python in the same process / same user account as R.A.Y.A. This is acceptable for trusted single-user contexts but is a meaningful safety gap.

**Approach.** Run generated code in a **separate subprocess** with restricted environment (no network unless requested, working directory scoped to a temp dir, conservative resource limits). Optionally support a Docker-backed mode for users who have Docker installed.

**Considerations.** Backwards-compatible default: still runs in-process unless the user opts in. Sandboxing too aggressively breaks legitimate workflows where the code needs to touch the user's files.

---

### E9. macOS and Linux verification

**Motivation.** Cross-platform support is the one partially-met objective. Closing this requires actually running the full test campaign on the other platforms.

**Approach.** Acquire a macOS and a Linux test machine (physical or VM); run the 60-prompt audio campaign and the 45-prompt text campaign; identify platform-specific failures (especially `actions/computer_settings.py` and `actions/send_message.py`); patch them; document any unavoidable gaps.

**Considerations.** Some Windows-only libraries (pycaw, win10toast, pywinauto) have no direct cross-platform equivalents; for those tools, the goal is graceful degradation, not strict parity.

---

### E10. First-launch onboarding tour

**Motivation.** First-time users are dropped into LISTENING with no guidance on what to try.

**Approach.** Detect first launch (no `config/api_keys.json`, or a new flag in settings). After the API-key dialog, show a brief modal with five suggested prompts the user can try and a "Got it" dismiss button.

**Considerations.** Keep it skippable for repeat installs. The five prompts should cover one trivial action, one tool-driven action, one memory disclosure, one multi-step goal, and one vision command — together they exercise the most distinctive features of the assistant.

---

## 7.2.2 Horizon 2 — Mid-Term Enhancements (v2.6 — v3.0)

The mid-term enhancements expand R.A.Y.A's reach beyond the single desktop. These are larger features that each introduce a new dimension to the product.

### E11. Mobile companion app

**Motivation.** Today, R.A.Y.A is only reachable when the user is at their desktop. A mobile companion makes the assistant useful anywhere.

**Approach.** Build a lightweight iOS/Android client (Flutter or React Native) that:
- Captures a voice prompt.
- Sends it (via E12 below) to the user's desktop R.A.Y.A.
- Plays back the spoken response.
- Surfaces async notifications when long-running agent tasks complete.

**Considerations.** Authentication between mobile and desktop (paired QR scan); push notifications for results; offline buffering of prompts when the desktop is unreachable.

---

### E12. Cloud relay for cross-device tasks

**Motivation.** The mobile companion needs a way to reach the desktop even when both are on different networks.

**Approach.** A small user-hosted relay service (or a tightly scoped optional service) that brokers messages between the mobile and the desktop. The user runs the relay on a VPS, NAS, or always-on machine they control.

**Considerations.** End-to-end encryption of relayed messages; no message persistence on the relay; clear documentation of what the relay sees.

---

### E13. Multi-device memory sync

**Motivation.** A user with a desktop and a laptop currently has two independent R.A.Y.A instances with disjoint memories.

**Approach.** Sync `memory/long_term.json` (and the SQLite history DB from E7) across devices using an end-to-end-encrypted backend. Either build on syncthing-style P2P sync or use a thin user-hosted service.

**Considerations.** Conflict resolution when two devices have updated the same memory entry; clear "this device just learned X" UI cues so the user knows when sync has happened.

---

### E14. Plugin marketplace and runtime-loaded tools

**Motivation.** Currently, adding a new tool requires editing the source. A plugin API would let community-contributed tools be installed without touching the codebase.

**Approach.** Define a plugin manifest format (Python entry-point + JSON schema declaration). At startup, R.A.Y.A scans a `plugins/` directory, loads each plugin, and adds its declarations to `TOOL_DECLARATIONS`. The plugin's function is registered in `_execute_tool`.

**Considerations.** Plugin trust model — a plugin is essentially arbitrary code, so installation must be gated by a confirmation flow. Sandboxing (see E8) applies here too.

---

### E15. Web dashboard

**Motivation.** A browser-based UI complements the native Tkinter UI by enabling remote access (from another machine on the same LAN) and providing a richer visual surface for memory, history, and plan inspection.

**Approach.** A lightweight FastAPI service (running locally) that serves a small React or Vue front-end. The dashboard reads the same memory/history state and can submit text prompts.

**Considerations.** The dashboard must respect the same privacy properties as the native UI — bind to localhost by default, require authentication for LAN access.

---

### E16. Provider-neutral reasoning router

**Motivation.** Coupling to a single LLM provider is a strategic risk. A router enables resilience against pricing changes, deprecations, and policy shifts.

**Approach.** Introduce an abstraction layer (`reasoning/router.py`) that exposes `generate(text, model_class)` and `stream_voice(audio, model_class)` methods. Backends include Gemini (Live + classic), OpenAI, Anthropic, and local. The user selects a default per model class in the settings panel.

**Considerations.** Voice streaming is the hardest case — not all providers support it; for non-streaming providers, fall back to the old STT→LLM→TTS pipeline. Tool-calling schemas differ between providers, so a translation layer is needed.

---

### E17. Multi-user mode

**Motivation.** A family or small team might want a shared assistant with per-user memory.

**Approach.** Add a "user profile" concept. The UI presents a profile selector on launch; each profile has its own memory file, history DB, and settings. The system prompt receives the active profile's facts.

**Considerations.** Profile switching must be explicit and obvious; the assistant must never mix facts from different profiles.

---

### E18. New high-value tool integrations

**Motivation.** The current toolset is broad but not exhaustive. Several high-value tools would meaningfully expand daily usefulness:

- **Spotify control** (currently handled by browser_control, but a direct API would be cleaner).
- **Calendar integration** (Google Calendar, Outlook).
- **Notion / Obsidian sync** for capturing research output.
- **Email composition and sending** through the user's existing client.
- **Smart-home control** (Home Assistant, Philips Hue).

**Approach.** Each tool follows the existing `actions/` plug-in pattern. Authentication is per-tool — OAuth for Google/Microsoft services, API tokens for self-hosted ones.

**Considerations.** Token storage must follow the same local-only privacy model as `api_keys.json`.

---

## 7.2.3 Horizon 3 — Long-Term Enhancements (v3.x and beyond)

The long-term enhancements depend on factors outside R.A.Y.A's control — model capability progress, ecosystem maturity, or significant engineering investment. They are listed here as direction, not commitment.

### E19. On-device LLM fallback for offline operation

**Motivation.** R.A.Y.A's cloud dependency is the single most consequential limitation. Even basic commands fail when offline.

**Approach.** When the network is unavailable, route tool calls to a local model (Phi-4, Llama 3.x, or whichever local model crosses the agentic-capability threshold). The local model is loaded lazily and only when needed.

**Dependencies.** Requires a local model that can reliably do tool-calling with a 20-tool schema. As of early 2026, this is on the edge of feasibility for high-end consumer hardware.

---

### E20. Voice cloning of a user-preferred voice

**Motivation.** Today the user gets the Charon voice. A personalized voice could improve the perceived rapport.

**Approach.** Add a voice-cloning module (e.g., XTTS, Coqui, or a Gemini-side preset) that synthesizes the assistant's voice in a user-provided sample's likeness.

**Considerations.** Significant ethical considerations — voice cloning has obvious abuse potential. R.A.Y.A would require the user to provide their own voice sample only (not a third party's) and bind the cloned voice to that user's profile.

---

### E21. Continuous-learning feedback loop

**Motivation.** When the user corrects the assistant ("no, I meant the other Chrome window"), this signal is lost. A learning loop could capture it.

**Approach.** Log corrections as `(prior_action, user_correction, corrected_action)` triples; periodically synthesize them into examples added to the system prompt's few-shot section.

**Considerations.** Must be transparent to the user (what is the assistant learning from?), reversible (the user can erase learned behaviors), and bounded (the learned context cannot grow without limit).

---

### E22. Multi-agent topology

**Motivation.** A single planner+executor is sufficient for the current scope. As tools grow into the dozens or hundreds, a single planner becomes a bottleneck.

**Approach.** Decompose the agent into specialist agents per tool category (e.g., a "browser agent", a "code agent", a "file agent"). A meta-agent receives the user's goal and delegates to specialists, each of whom has tight knowledge of their tool subset.

**Considerations.** Inter-agent communication overhead; risk of conflicting plans; harder to debug. Worth pursuing only if the current single-planner architecture starts to show clear scaling pain.

---

### E23. Local vision model for screen analysis

**Motivation.** `screen_process` currently uses Gemini Vision in the cloud. Each invocation uploads a full screenshot, which has privacy and bandwidth implications.

**Approach.** Run a local vision model (e.g., Qwen2-VL, Phi-3-Vision) for routine screen-analysis queries; reserve cloud vision for queries that exceed local capability.

**Dependencies.** Requires a local vision model that is good enough for common screen-content queries (text recognition, UI layout description, error message reading).

---

## 7.2.4 Cross-Cutting Concerns

Three concerns apply to every enhancement listed above and deserve explicit treatment.

### CC1. Security

Every new tool, every new layer, every new sync path introduces a new surface. Specifically:

- **E8** (sandboxed code) is itself a security enhancement.
- **E11–E13** (mobile, relay, sync) all require careful authentication and encryption design.
- **E14** (plugin marketplace) introduces arbitrary third-party code into the runtime.
- **E15** (web dashboard) opens a local network surface.
- **E20** (voice cloning) introduces ethical considerations.

A dedicated **security review** should accompany each of these enhancements before it ships.

### CC2. Accessibility

R.A.Y.A's text channel already provides accessibility for users who cannot speak. Future enhancements should not regress this. Specifically:

- The settings panel (E4) should be keyboard-navigable.
- The history view (E7) should support screen readers.
- The mobile companion (E11) should support voice-over and TalkBack.
- Multi-line input (E6) should preserve all keyboard shortcuts.

### CC3. Internationalization

Gemini Live handles many languages out of the box. But the UI surfaces (settings, history, plan view) are currently English-only. As the surface area grows:

- All new UI strings should go through a `t(key)` lookup function (currently absent — would need to be introduced in v2.5).
- The translation files should live in `i18n/` and be community-contributable.

---

## 7.2.5 Prioritization Framework

A useful frame for sequencing this backlog:

- **Ship if reasonable.** E1, E2, E3, E4, E5, E6, E7, E8, E10 — small, bounded, high user-visible impact. These are the v2.5 default.
- **Ship if there is time.** E9 (cross-platform verification), E18 (new tools).
- **Ship in a deliberate cycle.** E11–E13 (mobile + sync) require a coordinated release. E16 (provider-neutral router) requires significant abstraction work.
- **Treat as research.** E19, E20, E21, E22, E23 — pursued only if external conditions align.

---

## 7.2.6 What Success Looks Like for v2.5

A reasonable definition of "v2.5 ships" is the completion of items E1–E10. Together they:

- Close every high- and medium-severity limitation identified in 6.7.
- Convert R.A.Y.A from a developer-friendly tool into something a non-developer student could comfortably use.
- Add the persistent task history that lays the foundation for the mid-term sync work.
- Demonstrate cross-platform parity, opening the door to non-Windows users.

Crucially, none of E1–E10 require redesigning the v2.4 architecture. Each one is additive. This is the cleanest possible signal that the foundation is solid.

---

## 7.2.7 Summary

R.A.Y.A's future scope is organized into ten near-term engineering items (v2.5), eight mid-term platform expansions (v2.6–3.0), and five long-term research-grade directions (v3.x+), each grounded in concrete motivations from the limitations evaluated in Section 6.7. The near-term items are tractable and would close all high-severity gaps; the mid-term items expand the product into a multi-device, multi-user, provider-neutral assistant; the long-term items depend on external progress in model capabilities. Cross-cutting concerns of security, accessibility, and internationalization apply across all horizons. The roadmap demonstrates that the v2.4 architecture has substantial growth headroom and that the path from where the project is today to a much more capable v3.0 is paved with engineering tasks rather than research bets.
