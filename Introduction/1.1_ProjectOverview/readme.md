# 1.1 Project Overview

## Introduction

**R.A.Y.A (Responsive AI Yielding Assistance) v2.4** is a real-time, voice-driven, cross-platform personal AI assistant engineered to function as an intelligent layer between the user and their operating system. Unlike traditional voice assistants that are confined to scripted command-response patterns, R.A.Y.A is built on top of Google's **Gemini 2.5 Flash Native Audio** model and operates as a fully autonomous desktop intelligence — capable of hearing, seeing, understanding, planning, and executing complex multi-step workflows entirely on the user's machine.

The system is designed for **total autonomy and privacy**: it runs locally, requires no recurring subscription, supports Windows, macOS, and Linux, and gives the user full control through a physical mute button and a dedicated graphical interface. R.A.Y.A is not merely a chatbot with voice — it is a context-aware automation framework that combines natural language understanding, computer vision, system-level control, and persistent long-term memory into a single cohesive experience.

## Core Identity

R.A.Y.A is engineered around four pillars:

1. **Conversational Intelligence** — Ultra-low latency, full-duplex voice interaction in any language, powered by Gemini's native audio model with real-time transcription of both user input and assistant output.
2. **System-Level Agency** — Direct control of the host computer: launching applications, managing files, controlling browsers, sending messages on WhatsApp/Telegram, adjusting volume and brightness, and executing arbitrary keyboard/mouse actions.
3. **Visual Awareness** — Real-time screen capture and webcam processing, allowing R.A.Y.A to analyze on-screen content and respond to questions like *"What am I looking at?"* or *"Read this error message."*
4. **Persistent Memory** — A structured long-term memory system that captures identity, preferences, projects, relationships, wishes, and notes about the user, persisted to disk and reloaded into every session for true contextual continuity.

## System Architecture at a Glance

R.A.Y.A is organized as a modular Python application with a clear separation between the **conversational core**, the **tool layer**, the **planning agent**, and the **user interface**:

| Layer | Module | Responsibility |
|---|---|---|
| Entry Point | `main.py` | Boots the system, manages the Gemini Live session, routes tool calls |
| UI | `ui.py` (`RayaUI`) | Tkinter-based front-end with visual states (LISTENING, THINKING, SPEAKING), activity log, queue snapshot, and text-input fallback |
| Core | `core/prompt.txt` | The system prompt that defines R.A.Y.A's persona, execution rules, and tool-routing policy |
| Actions | `actions/` | 17 specialized tool modules (browser control, file management, web search, code helper, dev agent, game updater, etc.) |
| Agent | `agent/` | Multi-step task queue, planner (Gemini 2.5 Flash Lite), executor with retry/replan/error recovery |
| Memory | `memory/` | Long-term JSON-backed memory with category trimming and prompt formatting |
| Config | `config/api_keys.json` | API key storage (Gemini) |

The system is started with `python main.py` and is also distributable as a **Windows installer (.exe)** built via PyInstaller and Inno Setup, with Playwright browsers bundled for fully offline web automation.

## Interaction Modes

R.A.Y.A is **bimodal** by design — the user can switch seamlessly between two interaction channels:

- **Audio mode (primary):** The microphone continuously streams 16 kHz PCM audio to Gemini Live; responses are streamed back as 24 kHz audio and played through the speakers, with a hardware-style mute toggle (F4 key or UI button) for privacy.
- **Text mode (fallback):** The UI includes a text input field that sends prompts directly to the same live session, useful in noisy environments or when typing is faster than speaking.

This dual-channel design ensures that the assistant remains usable in any setting — from a quiet personal workspace to a shared office or library.

## Tool Ecosystem

The assistant has access to **20 declarative tools**, each defined with strict JSON schema parameters and routed through a single `_execute_tool` dispatcher. The tools cover:

- **Productivity:** `open_app`, `file_controller`, `desktop_control`, `reminder`, `send_message`, `weather_report`, `flight_finder`
- **Web & Browser:** `browser_control` (multi-browser: Chrome, Edge, Firefox, Opera, Brave), `web_search`, `youtube_video`
- **Vision:** `screen_process` (screen + webcam capture and analysis)
- **System Control:** `computer_settings`, `computer_control` (keyboard/mouse/screen automation)
- **Development:** `code_helper`, `dev_agent` (full multi-file project generation)
- **Gaming:** `game_updater` (Steam and Epic Games integration)
- **Meta:** `agent_task` (multi-step planning), `save_memory`, `shutdown_raya`

Each tool is invoked asynchronously, runs in an executor thread (so the audio loop is never blocked), and reports its progress back to the UI in real time.

## Key Differentiators

What sets R.A.Y.A apart from commercial assistants like Alexa, Siri, or Google Assistant:

- **Local execution and zero subscription** — the user owns the data and the runtime.
- **True multi-step autonomy** — the `agent_task` tool delegates goals to a dedicated planner that produces a JSON plan, executes each step with retry and re-plan logic, and translates the final output into the user's language.
- **Cross-application orchestration** — a single voice command can chain a web search, a file write, a desktop action, and a notification.
- **Persistent personalization** — the memory module silently learns who the user is and feeds that context back into every new session.
- **Open and extensible** — the action modules follow a uniform `(parameters, player, response)` interface, making it trivial to add a new tool by writing a single Python file and registering a declaration in `main.py`.

## Project Version and Licensing

The version documented in this report is **R.A.Y.A v2.4.0**, released under the **Creative Commons BY-NC 4.0** license (personal and non-commercial use). Notable improvements in this release include cross-platform support (experimental on macOS and Linux), graceful shutdown via voice command, strengthened long-term memory recall, and optimized tool-call latency.

## Summary

R.A.Y.A v2.4 is a comprehensive personal AI assistant that unifies real-time voice conversation, computer vision, system automation, multi-step planning, and persistent memory into a single locally-executed desktop application. It demonstrates how modern multimodal LLMs can be combined with classical OS automation libraries to produce an assistant that is genuinely useful — one that not only answers questions but *acts* on the user's behalf across the entire digital environment.
