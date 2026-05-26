# 2.2 Review of Related Work

This section surveys the most relevant prior art that shaped — or competes with — R.A.Y.A v2.4. The review is organized into six thematic groups, moving from the broadest commercial assistants down to specific open-source projects, research prototypes, and the academic literature on agentic LLMs.

---

## 2.2.1 Commercial Voice Assistants

These are the products R.A.Y.A is most directly compared against. Each represents a different design philosophy.

### Apple Siri (2011 → present)

Siri pioneered mass-market voice assistance and remains the most polished from a UX standpoint. Its strengths are tight system integration on macOS and iOS, on-device speech recognition for short queries (since Apple Intelligence in 2024), and a strong privacy posture relative to its peers.

**Limitations relevant to R.A.Y.A:**

- Tightly coupled to Apple's walled garden — no Windows or Linux support.
- Extremely shallow desktop automation: no general "open this app, then search the web, then save the result" flow.
- No persistent personalization beyond what the OS chooses to expose.
- Closed source; the user has no control over the toolset.

R.A.Y.A consciously breaks Siri's walled-garden constraint by being **cross-platform** and **fully open** at the code level.

### Amazon Alexa (2014 → present)

Alexa's design center is the smart home and one-shot commands. Its **Skills SDK** is one of the earliest large-scale function-calling ecosystems and is a clear conceptual ancestor of modern LLM tool-calling.

**Limitations:**

- Heavy bias toward shopping and IoT.
- Weak conversational continuity — multi-turn skills are notoriously brittle.
- All voice data is processed in the cloud and retained by default.
- No support for desktop productivity workflows.

R.A.Y.A inherits the **declarative tool model** from the Alexa Skills paradigm (each tool has a name, a description, and a JSON-style schema) but applies it to desktop control instead of home appliances.

### Google Assistant (2016 → present)

Strong general search, growing tool-calling capabilities, deep Android integration. The newer **Gemini Assistant** (2024+) replaces it on Android and benefits directly from Gemini's multimodal stack.

**Limitations:**

- On desktop (Windows / Linux / macOS), Google Assistant is essentially absent.
- Multi-step planning is hidden inside Google's stack — there is no developer-accessible plan-then-execute API.
- Personalization data is locked to a Google account.

R.A.Y.A uses **the same underlying Gemini model**, but exposes the multi-step planner explicitly (`agent/planner.py`) and runs the assistant on the user's own desktop.

### Microsoft Cortana (2014 → 2023, deprecated)

Cortana was the closest precedent to R.A.Y.A — a voice assistant designed for desktop productivity on Windows. Microsoft retired it in favor of Copilot. Cortana's deprecation arguably opened the niche that R.A.Y.A targets: **a serious voice assistant for Windows that is not tied to a single vendor's cloud**.

### Microsoft Copilot for Windows (2023 → present)

The current incumbent on Windows. Copilot can answer questions in a sidebar, edit text, and increasingly call tools via the **Copilot Studio** plugin system. It is closed source, requires a Microsoft account, and is moving toward a subscription model.

**Comparison with R.A.Y.A:**

| Dimension | Copilot | R.A.Y.A |
|---|---|---|
| Voice-first | Optional sidebar | Yes, always |
| Local memory | No | Yes (`memory/long_term.json`) |
| Open source | No | Yes (CC BY-NC 4.0) |
| User-owned API key | No | Yes |
| Cross-platform | Windows only | Windows / macOS / Linux |
| Tool extensibility | Plugins via Copilot Studio | Plain Python files in `actions/` |

### OpenAI ChatGPT Voice / Advanced Voice Mode (2024)

ChatGPT introduced real-time voice in late 2024 — conceptually identical to Gemini Live. It is excellent at conversation but has **no access to the user's local OS, files, or applications**. R.A.Y.A's central thesis is exactly this gap: a real-time voice model is necessary but not sufficient — it must be embedded in an OS-control layer to be useful for desktop productivity.

---

## 2.2.2 Open-Source Personal Assistants

A small but vibrant open-source ecosystem provides direct architectural inspiration for R.A.Y.A.

### Mycroft AI (2015 → 2023)

Mycroft was a pioneering open-source voice assistant with a privacy-first ethos, a Python skills framework, and on-device wake-word detection. Its company shut down in 2023, but the codebase lives on as **OpenVoiceOS** and **Neon AI**.

**Lessons R.A.Y.A inherits from Mycroft:**

- A flat **skill folder** as the unit of extensibility (R.A.Y.A's `actions/` folder mirrors this).
- A **single message bus** between modules.
- A **user-controlled privacy model** with local data by default.

**Where R.A.Y.A diverges:**

- Mycroft used classical NLU pipelines (Adapt, Padatious). R.A.Y.A delegates all NLU to Gemini Live, eliminating thousands of lines of intent-matching code.
- Mycroft's audio path is slower because of the STT → NLU → TTS pipeline. R.A.Y.A uses native-audio Gemini Live with sub-second latency.

### Leon AI

Leon is an open-source personal assistant in JavaScript/Python that supports modular skills, on-device speech recognition (DeepSpeech / Coqui), and an MQTT-based architecture. It is closer to a "smart-speaker on your computer" than a true desktop automation tool.

**Comparison with R.A.Y.A:**

- Similar plugin philosophy.
- Less integrated desktop control (no screen vision, no Steam/Epic, no Playwright browser automation).
- Heavier setup footprint (Node.js + Python + speech models).

### Jarvis (Sukeesh)

A popular Python personal assistant on GitHub that uses Google Speech Recognition + pyttsx3 + a hand-rolled command parser. It is a **pre-LLM design** — R.A.Y.A can be understood as the *post-LLM evolution* of this class of project, where the brittle command parser is replaced by Gemini's general-purpose understanding.

### Rhasspy

A fully **offline** voice assistant focused on home automation, with pluggable STT/TTS engines (Kaldi, Whisper, Pico). Rhasspy's offline-first stance is admirable but trades away the conversational quality of cloud LLMs.

R.A.Y.A occupies a different niche: **online for reasoning, local for everything else** — accepting the API dependency in exchange for vastly better conversation quality.

### NVIDIA Riva and the SpeechBrain Ecosystem

These are research/production toolkits for building custom speech assistants. They are heavier and lower-level than R.A.Y.A, but they inform the choice of sample rates (16 kHz in / 24 kHz out is the de-facto standard) and the audio I/O pipeline design.

---

## 2.2.3 LLM Agent Frameworks

R.A.Y.A's `agent/` subsystem (planner + executor + error handler + task queue) is closely related to the wave of LLM-agent frameworks that emerged in 2023–2025.

### LangChain (2022 → present)

The first widely-adopted Python framework for chaining LLM calls and tools. LangChain provides abstractions for agents, tools, prompt templates, memory, and retrieval.

**What R.A.Y.A borrows:**

- The notion of a **Tool** as a structured object with name + description + schema + handler.
- The **agent loop** of "decide on tool → call tool → observe result → decide next".

**What R.A.Y.A intentionally does *not* borrow:**

- LangChain's heavy class hierarchy.
- The "LCEL" composition syntax.
- A dependency on the LangChain runtime.

R.A.Y.A's approach is closer to **"LangChain, but with one Python file per module and no framework lock-in"**.

### LangGraph (2024)

A graph-based agent framework that models agent loops as explicit nodes/edges with conditional transitions. It is more rigorous than vanilla LangChain but adds even more conceptual overhead. R.A.Y.A's planner-then-executor design can be seen as a hand-coded LangGraph with two nodes (plan, execute) and one feedback edge (replan).

### AutoGPT and BabyAGI (2023)

The two most viral examples of "fully autonomous" LLM agents. Both were heavily criticized for **looping forever, exhausting credits, and producing nothing**. The lessons R.A.Y.A took directly from these failure modes:

- **Bound plan length** (`Max 5 steps` in `PLANNER_PROMPT`).
- **Bound replan attempts** (`MAX_REPLAN_ATTEMPTS = 2`).
- **Bound retries per step** (the `attempt <= 3` loop in `executor.py`).
- **Make the executor humble** — when in doubt, abort and tell the user, rather than spinning.

### CrewAI, AutoGen, OpenAI Assistants API

Newer frameworks centered on **multi-agent collaboration**. R.A.Y.A is single-agent on purpose — multi-agent designs introduce coordination overhead that is unjustified for a desktop assistant. The architecture could be extended to multi-agent (one agent per tool category, for example) in a future version.

### ReAct (Yao et al., 2022)

The foundational research paper that introduced **interleaved reasoning and acting** in LLMs. R.A.Y.A's main Live loop is a streamlined ReAct: think briefly (one or two short sentences), act (tool call), observe (tool response), repeat.

### Reflexion (Shinn et al., 2023)

Adds a self-criticism step where the agent reflects on a failed trajectory and updates its strategy. R.A.Y.A's `error_handler.analyze_error` performs a lightweight version of this: it inspects the failure, classifies it (RETRY/SKIP/FIX/ABORT), and optionally generates a fixed step (`generate_fix`).

---

## 2.2.4 Computer-Using Agents (CUAs)

In late 2024 and 2025, both Anthropic and OpenAI released **computer-using agents** — LLM agents with vision and keyboard/mouse control of a virtual desktop.

### Anthropic Claude Computer Use (Oct 2024)

The first widely-publicized CUA. Claude is given screen captures and emits structured `screenshot`, `click`, `type`, `key` actions. It runs in an isolated VM.

**Relevance to R.A.Y.A:**

- The **action vocabulary** is similar to R.A.Y.A's `computer_control` tool (`type`, `click`, `hotkey`, `scroll`, `screenshot`).
- Claude Computer Use is **vision-first** — every action is conditioned on a fresh screenshot. R.A.Y.A is **API-first** by default (calling specific tools), and falls back to vision only on demand via `screen_process`. This is a deliberate trade-off: API-first is faster and more reliable, vision-first is more general.

### OpenAI Operator (2025)

A browser-based CUA from OpenAI. Similar paradigm, similar trade-offs. Both Operator and Claude Computer Use are cloud-hosted services; R.A.Y.A runs on the user's own machine.

### Google Project Mariner (2024)

Google's experimental browser-based agent. Influenced the design of `browser_control` in R.A.Y.A — particularly the idea of multi-browser support and smart selectors driven by natural-language descriptions (`smart_click`, `smart_type`).

---

## 2.2.5 Desktop Automation Libraries

R.A.Y.A stands on the shoulders of a stack of mature automation libraries. Their existence is what makes the project tractable at the source level.

| Library | Role in R.A.Y.A | Alternative |
|---|---|---|
| **PyAutoGUI** | Mouse / keyboard automation | pynput |
| **PyWinAuto** | Windows UI Automation API | UIA via comtypes |
| **Playwright** | Cross-browser web automation | Selenium |
| **mss** | Fast screen capture | PIL.ImageGrab |
| **OpenCV** | Webcam + image processing | Pillow |
| **sounddevice** | Audio I/O | PyAudio |
| **psutil** | Process management | wmi |
| **pycaw / comtypes** | Windows volume control | nircmd |
| **send2trash** | Safe file deletion | shutil.rmtree |
| **win10toast** | Windows notifications | plyer |
| **pygetwindow** | Window enumeration / focus | pyautogui |

These libraries individually solve narrow problems; R.A.Y.A's contribution is the **unified voice-driven layer that orchestrates them**.

---

## 2.2.6 Long-Term Memory in Conversational Systems

R.A.Y.A's memory module is informed by several lines of work:

- **MemGPT (Packer et al., 2023)** — proposed treating LLM context as a hierarchical memory with a "core" always-loaded section and a "recall" RAG-style section. R.A.Y.A uses a simpler version: a single always-loaded JSON block, capped at 2200 characters, with no RAG retrieval.
- **LangChain Memory** classes (Conversation Buffer, Summary, Vector Store) — provide useful taxonomy but were rejected in favor of a category-based schema better suited to personal facts (identity, preferences, projects, …).
- **MIT MyTalk / Replika** — consumer chatbots with persistent personalization. They demonstrate the *user value* of memory in conversational products, validating R.A.Y.A's choice to make memory a first-class feature.

R.A.Y.A's contribution to this design space is the **silent `save_memory` tool**: the LLM itself decides what to remember, with no explicit "remember this" prompt from the user. The tool's description (in `TOOL_DECLARATIONS`) explicitly instructs the model not to announce that it is saving — preserving the conversational flow.

---

## 2.2.7 Real-Time Voice Agent Demos

Throughout 2024–2025 a number of demos showed real-time voice agents driven by streaming APIs:

- **OpenAI's "Sky" demo** (May 2024) showed real-time multimodal voice with Vision; R.A.Y.A's screen-vision integration parallels this.
- **Hume AI's empathic voice interface** focuses on emotional prosody — a complementary direction R.A.Y.A could borrow from in the future (see Section 7).
- **Pipecat / LiveKit Agents** — open-source frameworks for building voice agents on top of streaming APIs. R.A.Y.A's `RayaLive` class is conceptually similar but specialized for the Gemini Live protocol, with desktop tool calling on top.

---

## 2.2.8 Academic Papers That Shaped R.A.Y.A

For completeness, the following papers are cited as part of the intellectual lineage:

| Paper | Year | Relevant idea |
|---|---|---|
| *Attention Is All You Need* (Vaswani et al.) | 2017 | The Transformer underlying every modern LLM |
| *Language Models are Few-Shot Learners* (Brown et al., GPT-3) | 2020 | In-context learning — basis for R.A.Y.A's example-rich planner prompt |
| *ReAct: Synergizing Reasoning and Acting in Language Models* (Yao et al.) | 2022 | Reasoning + tool-calling loop |
| *Toolformer: Language Models Can Teach Themselves to Use Tools* (Schick et al.) | 2023 | Modern justification for tool-calling at scale |
| *Reflexion: Language Agents with Verbal Reinforcement Learning* (Shinn et al.) | 2023 | Error self-analysis — basis for `error_handler.py` |
| *MemGPT: Towards LLMs as Operating Systems* (Packer et al.) | 2023 | Hierarchical conversational memory |
| *Gemini 2.5 Technical Report* (Google DeepMind) | 2025 | The underlying foundation model |

---

## 2.2.9 Summary Comparison Table

| System | Voice | Local memory | Multi-step planning | Screen vision | OS automation | Open source | Cross-platform |
|---|---|---|---|---|---|---|---|
| Siri | Yes | No | No | No | Limited | No | macOS / iOS only |
| Alexa | Yes | No | Limited | No | No (IoT only) | No | Echo only |
| Google Assistant | Yes | No | Hidden | No | Limited | No | Android / Smart speakers |
| Copilot for Windows | Optional | No | Limited | Partial | Partial | No | Windows only |
| ChatGPT Voice | Yes | Limited | No | Limited | No | No | Mobile / Web |
| Mycroft / OVOS | Yes | Optional | No | No | Limited | Yes | Linux primarily |
| Leon | Yes | Optional | No | No | Limited | Yes | Cross-platform |
| Claude Computer Use | No | No | Yes | Yes | Full (in VM) | No | Cloud VM |
| AutoGPT | No | Optional | Yes | No | Partial | Yes | Cross-platform |
| **R.A.Y.A v2.4** | **Yes** | **Yes** | **Yes** | **Yes** | **Yes** | **Yes** | **Win / macOS / Linux** |

R.A.Y.A is the only system in this table that satisfies **all seven** dimensions simultaneously — which is precisely the contribution articulated in the next section.

---

## Summary

The landscape around R.A.Y.A v2.4 is rich but fragmented. Commercial assistants are polished but closed and shallow on desktop. Open-source assistants are extensible but pre-date the LLM era. LLM agent frameworks excel at reasoning but are not tied to real-time voice or persistent desktop state. Computer-using agents from major labs run in cloud VMs, not on the user's own machine. R.A.Y.A's design is best understood as **a deliberate synthesis** of these threads — taking the voice-first immediacy of Siri, the extensibility of Mycroft, the planning rigor of LangChain, the vision capability of Claude Computer Use, and the memory model of MemGPT, and packaging them into a single locally-executed Python application.
