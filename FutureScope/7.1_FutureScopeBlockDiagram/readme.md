# 7.1 Future Scope Block Diagram

This section presents the **forward-looking architecture** of R.A.Y.A — what the system could become across the next several releases. The diagram shows the current v2.4 system as a stable core, with planned extensions surrounding it in three concentric rings of ambition: **near-term**, **mid-term**, and **long-term**.

The purpose of this block diagram is not to commit to a fixed roadmap but to make explicit *how* the v2.4 architecture is designed to grow — which layers absorb new features, which new layers may need to be added, and which subsystems remain unchanged.

---

## 7.1.1 Conventions Used in the Diagram

- **Solid boxes** are subsystems that exist in v2.4.
- **Dashed boxes** are subsystems planned for future releases.
- **Double-dashed boxes** are research-grade extensions whose feasibility depends on external evolution (e.g., a sufficiently capable on-device LLM).
- **Arrows** indicate the direction of data or control flow.
- **Concentric rings** group future work by horizon (near / mid / long).

---

## 7.1.2 Three Horizons of Future Work

```
            ┌───────────────────────────────────────────────────────┐
            │            LONG-TERM (v3.x and beyond)                │
            │  ╔══════════════════════════════════════════════════╗ │
            │  ║   On-device LLM fallback  ·  Local vision model  ║ │
            │  ║   Voice cloning  ·  Continuous learning loop     ║ │
            │  ╚══════════════════════════════════════════════════╝ │
            │                                                       │
            │   ┌───────────────────────────────────────────────┐   │
            │   │           MID-TERM (v2.6 — v3.0)              │   │
            │   │  ┌─────────────────────────────────────────┐  │   │
            │   │  │ Mobile companion · Multi-device sync   │  │   │
            │   │  │ Plugin marketplace · Multi-user mode    │  │   │
            │   │  │ Web dashboard · Cloud-relay agent      │  │   │
            │   │  └─────────────────────────────────────────┘  │   │
            │   │                                               │   │
            │   │   ┌─────────────────────────────────────────┐ │   │
            │   │   │           NEAR-TERM (v2.5)              │ │   │
            │   │   │ ┌─────────────────────────────────────┐ │ │   │
            │   │   │ │ Voice cancel · Plan view in UI     │ │ │   │
            │   │   │ │ Pre-warm browser · Settings panel  │ │ │   │
            │   │   │ │ Forget tool · Multi-line text      │ │ │   │
            │   │   │ │ History view · Sandboxed code      │ │ │   │
            │   │   │ │ macOS/Linux verified               │ │ │   │
            │   │   │ │ Onboarding tour                    │ │ │   │
            │   │   │ └─────────────────────────────────────┘ │ │   │
            │   │   │                                         │ │   │
            │   │   │       R.A.Y.A v2.4 (stable core)        │ │   │
            │   │   │                                         │ │   │
            │   │   └─────────────────────────────────────────┘ │   │
            │   │                                               │   │
            │   └───────────────────────────────────────────────┘   │
            │                                                       │
            └───────────────────────────────────────────────────────┘
```

---

## 7.1.3 Detailed Future-Architecture Block Diagram

```
┌───────────────────────────────────────────────────────────────────────────┐
│                          ENVIRONMENT (unchanged)                           │
│       User · Mic · Speakers · Webcam · Screen · OS · Internet · Files      │
└───────────────────────────────────────────────────────────────────────────┘
                ▲                                                          ▲
                │                                                          │
                ▼                                                          ▼
┌────────────────────────────────────┐         ┌──────────────────────────────────┐
│         LAYER 1 — UI LAYER          │         │     LAYER 5 — ACTIONS LAYER       │
│         ┌──────────────────┐        │         │      actions/*.py + new ones      │
│         │ ui.py · RayaUI    │ v2.4   │         │   17 existing tools (v2.4)         │
│         │ (states, log,    │        │         │   + spotify_control ────────┐    │
│         │  queue, mute)    │        │         │   + calendar_tool           │    │
│         └──────────────────┘        │         │   + notion_sync             │ ←──│ (v2.5+ tools)
│                                     │         │   + email_compose           │    │
│  ┌─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┐    │         │   + smart_home              │    │
│  │ Plan visualization panel  │ v2.5│         │   + custom_user_plugins  ───┘    │
│  │ Settings tab              │      │         └────────────────────────────────┬─┘
│  │ History view              │      │                                          │
│  │ Memory inspector          │      │                                          │
│  │ Onboarding tour           │      │                                          │
│  └─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┘    │                                          │
│                                     │                                          │
│  ┌═ ═ ═ ═ ═ ═ ═ ═ ═ ═ ═ ═ ═ ═ ┐    │                                          │
│  ║ Web dashboard (browser UI) ║ v3.x│                                          │
│  ║ Mobile companion app       ║    │                                          │
│  └═ ═ ═ ═ ═ ═ ═ ═ ═ ═ ═ ═ ═ ═ ┘    │                                          │
└────────────────────────────────────┘                                          │
                │                                                              │
                │                                                              │
                ▼                                                              │
┌─────────────────────────────────────────────────────────────────────┐         │
│      LAYER 2 — ORCHESTRATION (extended)                               │         │
│      main.py · RayaLive  +  new sub-controllers                       │         │
│   ┌─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┐                                          │         │
│   │ Cancel-task tool          │ v2.5                                    │ ────────┘
│   │ Plan-publication hook     │                                         │
│   │ Forget tool               │                                         │
│   │ Pre-warm browser at boot  │                                         │
│   └─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┘                                          │
└─────────────────────────────────────────────────────────────────────┘
                │
                │
                ▼
┌─────────────────────────────────────────────────────────────────────────┐
│      LAYER 3 — REASONING (extended)                                       │
│   ┌──────────────────────────┐   ┌─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┐    │
│   │  Gemini Live (v2.4)       │   │  On-device LLM fallback         │    │
│   │  Gemini Flash / Lite      │   │  (e.g., Phi-4, Llama 3.x local) │    │
│   │  Gemini Vision            │   │  for offline single-tool ops     │    │
│   └──────────────────────────┘   └ ═ ═ ═ ═ ═ ═ ═ ═ ═ ═ ═ ═ ═ ═ ═ ─ ┘    │
│                                                                          │
│   ┌─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┐ │
│   │  Model-selection router (Gemini / OpenAI / local) — provider-neutral │ │
│   └ ═ ═ ═ ═ ═ ═ ═ ═ ═ ═ ═ ═ ═ ═ ═ ═ ═ ═ ═ ═ ═ ═ ═ ═ ═ ═ ═ ═ ═ ═ ═ ═ ┘ │
└─────────────────────────────────────────────────────────────────────────┘
                │
                ▼
┌─────────────────────────────────────────────────────────────────────────┐
│    LAYER 4 — STATE & PERSISTENCE (extended)                              │
│   memory/long_term.json (v2.4)                                            │
│   ┌─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┐   │
│   │ Task history DB (SQLite)                                            │ v2.5
│   │ Vector store (embeddings of older memory entries)                   │
│   │ Encrypted sync backend (e.g., E2E syncthing-style)                  │ v2.6+
│   └ ═ ═ ═ ═ ═ ═ ═ ═ ═ ═ ═ ═ ═ ═ ═ ═ ═ ═ ═ ═ ═ ═ ═ ═ ═ ═ ═ ═ ═ ═ ═ ═ ┘   │
└─────────────────────────────────────────────────────────────────────────┘
                                                                                                                  ▲
                                                                                                                  │
┌─────────────────────────────────────────────────────────────────────────┐                                       │
│    LAYER 6 — AGENT SUBSYSTEM (extended)                                  │                                       │
│   planner.py · executor.py · error_handler.py · task_queue.py (v2.4)     │                                       │
│   ┌─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┐   │                                       │
│   │ Plan visualizer (publishes plan to UI before exec)                  │ v2.5                                  │
│   │ Sandboxed code executor (jailed subprocess for code_helper)         │                                       │
│   │ Longer-plan support (10+ steps with summary checkpoints)            │ v2.6+                                 │
│   │ Multi-agent fork (specialist agents per tool category)              │ v3.x                                  │
│   └ ═ ═ ═ ═ ═ ═ ═ ═ ═ ═ ═ ═ ═ ═ ═ ═ ═ ═ ═ ═ ═ ═ ═ ═ ═ ═ ═ ═ ═ ═ ═ ═ ┘   │                                       │
└─────────────────────────────────────────────────────────────────────────┘                                       │
                                                                                                                  │
┌─────────────────────────────────────────────────────────────────────────┐                                       │
│    LAYER 7 — NEW: COMPANION / SYNC FABRIC (dashed/future)                │                                       │
│   ┌─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┐   │                                       │
│   │ Mobile companion (iOS / Android) — voice trigger to desktop        │                                       │
│   │ Multi-device memory sync (e2e-encrypted)                            │ ──────────────────────────────────────┘
│   │ Cloud relay agent (optional, user-hosted) for cross-device tasks   │
│   │ Web dashboard for remote access via browser                         │
│   └ ═ ═ ═ ═ ═ ═ ═ ═ ═ ═ ═ ═ ═ ═ ═ ═ ═ ═ ═ ═ ═ ═ ═ ═ ═ ═ ═ ═ ═ ═ ═ ═ ┘   │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 7.1.4 Layer-by-Layer Reading of the Future Diagram

### Layer 1 — UI (extended)

The Tkinter UI absorbs four near-term additions (plan view, settings, history, memory inspector, onboarding) without changing its fundamental structure. The longer-term Web Dashboard and Mobile Companion are *new front-ends* talking to the same orchestrator through a future remote API.

### Layer 2 — Orchestration (extended)

`RayaLive` gains a small number of new tools (cancel, forget) and a startup hook for browser pre-warming. Crucially, no architectural change is required — these are additive declarations and dispatch branches.

### Layer 3 — Reasoning (extended)

The single most consequential future extension. The current single-provider (Gemini) reasoning layer becomes a **model-selection router** that can route a given call to Gemini, a local model (Phi-4, Llama 3.x, or a future Gemini Nano), or another cloud provider as configured. This unlocks the offline mode and provides resilience against vendor changes.

### Layer 4 — State (extended)

Adds three persistence mechanisms:

- **SQLite task history DB** — replaces the in-memory queue snapshot with a persistent log of past tasks and their outcomes.
- **Vector store** — overflow target for memory entries trimmed from the 2200-char JSON cap; enables semantic search of older facts when the agent needs them.
- **Encrypted sync backend** — allows the user to sync memory and task history across devices without trusting a cloud provider.

### Layer 5 — Actions (extended)

Three classes of additions:

- **Predefined new tools** (Spotify, calendar, Notion, email, smart home) following the same uniform signature.
- **User-defined plugins** — a runtime-loaded plugin API so users can drop tool files into a known folder without editing source.
- **Refinements to existing tools** (better Steam/Epic prerequisite checks, smarter file_controller defaults, etc.).

### Layer 6 — Agent (extended)

The agent layer grows in three directions:

- **Plan visibility** — publish the plan to the UI before execution.
- **Sandboxed execution** — `code_helper` and `dev_agent` run inside an isolated subprocess with restricted filesystem access.
- **Longer plans** — gradually relax the 5-step cap as confidence in the planner grows.
- **Specialist agents (long-term)** — a multi-agent topology where each tool category has its own specialist whose plans are coordinated by a meta-agent.

### Layer 7 — NEW: Companion / Sync Fabric

The biggest architectural addition. A new layer dedicated to **off-machine surfaces**:

- A **mobile companion** that records a voice prompt and routes it to the user's desktop R.A.Y.A.
- A **multi-device memory sync** so the same R.A.Y.A on the user's laptop and desktop share the same memory.
- A **cloud relay** (user-hosted, optional) for the mobile-to-desktop case when the desktop is on a different network.
- A **web dashboard** for remote monitoring and limited control.

This layer is new because v2.4 has no off-machine surfaces at all. Adding it touches Layer 1 (clients), Layer 2 (orchestration must accept remote turns), and Layer 4 (persistence must support sync).

---

## 7.1.5 What Stays the Same

It is equally important to identify what does **not** change:

- **The Live audio path.** Streaming voice through Gemini Live remains the primary interaction model.
- **The six-category memory schema.** It is small enough to remain the canonical structure even as a vector store overflow is added behind it.
- **The action-module plug-in pattern.** New tools continue to follow `(parameters, player, response, **kwargs) -> str`.
- **The system prompt as the single source of routing truth.** Future additions extend the prompt; they do not replace it.
- **The three-layer error recovery model** (tool / step / session). Each new feature is responsible for fitting into one of these layers.

These invariants are the reason the future diagram can be drawn as concentric rings of additions rather than as a redesign.

---

## 7.1.6 Provider-Neutral Reasoning — The Most Strategic Future Block

Of all the future additions, the **model-selection router** (in Layer 3) is the most architecturally consequential. Today, R.A.Y.A is tightly coupled to Gemini. Introducing a router with pluggable providers:

- Decouples the assistant's behavior from any single vendor's pricing or policy decisions.
- Enables offline operation through a local model on capable hardware.
- Lets the user choose models per task (e.g., a local Phi-4 for planning, Gemini for voice, a vision-specialist model for screen analysis).
- Makes R.A.Y.A defensible against the long-term risk that a single provider could deprecate or restrict access to the Live API.

The future-scope diagram puts this in the **mid-term** ring rather than the near-term because it requires careful abstraction of every model call in the codebase — a non-trivial refactor — and a thoughtful decision about how voice (which is currently Gemini-Live-exclusive) is handled when the chosen provider does not support streaming audio.

---

## 7.1.7 Companion Fabric — The Most Visible Future Block to End Users

A mobile companion is the single addition that would most change how R.A.Y.A is *used*. Today, the assistant lives on the user's desktop and is only reachable when the user is at the desk. A mobile app that captures voice and forwards it to the desktop (via the cloud relay) would:

- Make R.A.Y.A useful when the user is away from their machine.
- Allow voice triggers that route to long-running desktop tasks (e.g., "back home in 30 minutes, prep my workspace").
- Provide a notification surface for completed agent tasks.

This is a meaningful product expansion. It also introduces real security and privacy considerations (authentication between mobile and desktop, end-to-end encryption of the relay channel, account model for multi-device users) that v2.4's single-machine simplicity intentionally avoids.

---

## 7.1.8 Research-Grade Long-Term Blocks

The outermost ring (double-dashed) contains items whose feasibility depends on factors outside R.A.Y.A's control:

- **On-device LLM that is good enough.** Currently, local models lag substantially behind frontier cloud models in agentic tool calling. If this gap closes (Phi-5, Llama 4, Gemini Nano), the offline fallback becomes feasible.
- **Voice cloning of the user's preferred voice.** Possible today but with significant ethical and quality questions.
- **Continuous learning loop.** Adapting the model's behavior based on which tool calls the user accepts or corrects. Possible in principle but requires careful UX to avoid creating an opaque system.

These are flagged as "research-grade" because they are not guaranteed to ship — but the architecture is designed to accommodate them when they do.

---

## 7.1.9 What the Diagram Is Not

The future-scope diagram deliberately avoids three pitfalls:

1. **No timeline.** Software roadmaps drift; promising specific dates would be misleading.
2. **No fixed deliverables.** Each ring is a *direction*, not a contract.
3. **No redesigns of the v2.4 core.** Every future block sits *around* the existing layers, not in place of them. This is by design — the v2.4 architecture is the foundation, not the prototype.

---

## 7.1.10 Summary

The future-scope block diagram shows R.A.Y.A v2.4 as a stable core surrounded by three rings of planned extensions: near-term gap-closing (UI panels, voice cancel, settings, sandboxed execution), mid-term platform expansion (mobile companion, multi-device sync, plugin marketplace, provider-neutral reasoning), and long-term research-grade additions (offline on-device LLM, voice cloning, continuous learning). A new Companion/Sync Fabric layer is the most visible architectural addition; a provider-neutral model router is the most strategic. The diagram makes explicit that none of the planned work requires redesigning the existing architecture — every future block is additive, fitting cleanly into one of the six existing layers or extending the system with a seventh.
