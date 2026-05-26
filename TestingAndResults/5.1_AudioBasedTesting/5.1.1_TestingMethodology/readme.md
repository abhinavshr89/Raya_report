# 5.1.1 Audio-Based Testing — Methodology

This section documents how R.A.Y.A v2.4 was tested through its primary interface — **voice**. Because the assistant is fundamentally interactive, multimodal, and tied to a streaming LLM, traditional unit testing was not the right instrument. Instead, the system was evaluated through a **structured manual test campaign**, run on the developer's own hardware over the course of project development.

The methodology described here defines: the test environment, the test categories, the test-case format, the evaluation rubric, the metrics collected, and the limitations of the approach.

---

## 5.1.1.1 Test Environment

A single reference machine was used as the canonical test bench to make latency and reliability numbers comparable across categories.

| Component | Specification |
|---|---|
| **CPU** | Intel Core i5 (8th–10th gen class) |
| **RAM** | 8 GB |
| **Storage** | SSD |
| **OS** | Windows 11 Home Single Language (10.0.26200) |
| **Python** | 3.11 |
| **Microphone** | Built-in laptop microphone + a USB headset (tested separately) |
| **Speakers** | Built-in laptop speakers + the same USB headset |
| **Webcam** | Built-in 720p webcam |
| **Network** | 25–50 Mbps consumer broadband, < 30 ms RTT to Gemini endpoint |
| **Browser** | Bundled Playwright Chromium (no separate browser install required) |
| **Background load** | Quiet room; no other heavy applications running |

Every voice test was performed against this configuration unless explicitly noted.

---

## 5.1.1.2 Scope of Audio Testing

Audio-based testing covers everything the user can do through the **microphone + speakers** channel:

1. The full voice round-trip (capture → Gemini Live → tool call → audio response).
2. Tool-call selection accuracy under spoken commands.
3. Multilingual voice support.
4. The interaction between the main Live audio stream and the secondary vision-tool audio stream.
5. The mute control and echo-suppression behavior.
6. Auto-reconnect behavior under simulated network drops.

Behaviors that are **out of scope** for audio testing (and covered instead by 5.2 — Text-Based Testing):

- Pasting long input.
- Inputs that contain special characters / code / URLs.
- Headless / quiet-mode use of the assistant.

---

## 5.1.1.3 Test Categories

The audio test campaign is organized into eight categories. Each category targets a distinct subsystem or interaction pattern.

| ID | Category | Targets |
|---|---|---|
| **A1** | Smoke / Wake-up | Boot, mic capture, first response |
| **A2** | Conversational latency | End-to-end response time |
| **A3** | Tool-call accuracy | Selection of the right tool from a spoken command |
| **A4** | Multi-language voice | Conversation in non-English languages |
| **A5** | Vision via voice | The `screen_process` flow triggered by speech |
| **A6** | Multi-step goals via voice | `agent_task` invoked through spoken goals |
| **A7** | Memory persistence | Save → restart → recall a personal fact |
| **A8** | Robustness | Mute, echo, reconnect, interruption |

Every test case maps to exactly one category for clean reporting in Section 5.1.2.

---

## 5.1.1.4 Test-Case Format

Every test case is recorded with the following fields:

```
Test ID:        A3-07
Category:       A3 — Tool-call accuracy
Prompt (EN):    "Open Notepad and increase the volume to 60%."
Expected tool(s): open_app(app_name="Notepad"); computer_settings(action="set_volume", value="60")
Pre-conditions: Notepad not currently open
Steps:          1. Start R.A.Y.A.
                2. Wait for LISTENING state.
                3. Speak the prompt at conversational volume from 0.5 m away.
                4. Observe activity log and OS state.
Expected outcome: Notepad opens; system volume slider moves to ~60%.
Pass criteria:    Both tools executed in any order; final OS state correct.
Latency target:   < 2 s to first audio chunk; < 4 s to action completion.
Recorded result:  Pass / Partial / Fail
Notes:            free-text observations
```

Where a single test exercises more than one tool, *all* expected tools are listed and all must be invoked for a Pass.

---

## 5.1.1.5 Evaluation Rubric

The rubric is deliberately strict but allows a middle "Partial" tier so that *almost*-correct behaviors are captured without inflating the pass rate.

| Verdict | Definition |
|---|---|
| **Pass** | The system reached the intended end state, with no user re-prompts and within the latency target. |
| **Partial** | The system reached the intended end state but required a follow-up clarification, exceeded the latency target by ≤ 50 %, or used a slightly different but acceptable tool path. |
| **Fail** | The system did not reach the intended end state, hallucinated a result, called the wrong tool, or required a manual intervention beyond a single follow-up. |

For each test, the verdict is recorded alongside a one-line note explaining *why* — particularly for Partial or Fail outcomes.

---

## 5.1.1.6 Metrics Collected

Five quantitative metrics are collected across the campaign:

| Metric | Unit | How measured |
|---|---|---|
| **End-to-end response latency** | seconds | Time from end-of-user-utterance (operator's stopwatch / observed VAD cutoff) to first audio chunk played by `_play_audio` |
| **Tool dispatch latency** | seconds | Time from the `tool_call` event arriving in `_receive_audio` to the action function returning |
| **Tool-call accuracy** | % | Pass + Partial divided by total cases in category A3 |
| **Multi-step completion rate** | % | Goals fully completed without manual intervention in category A6 |
| **Reconnect time after network drop** | seconds | Time from forced WiFi-off → on, to "LISTENING" state restored |

Where exact instrumentation was not possible (e.g., observed VAD cutoff is approximate), latencies are reported as ranges rather than single numbers.

---

## 5.1.1.7 Prompt Set

The campaign uses a fixed **seed prompt set** of 60 spoken inputs distributed across the eight categories:

| Category | # of prompts | Examples |
|---|---|---|
| A1 — Smoke | 5 | "Hello", "Are you online?", "Status report" |
| A2 — Latency | 5 | Short factual questions: "What's 12 times 13?", "Who is the prime minister of India?" |
| A3 — Tool-call accuracy | 20 | "Open Chrome", "Send a WhatsApp to mom saying I'll be late", "What's the weather in Mumbai?", "Set a reminder for 5 PM tomorrow", "Play Shape of You on YouTube", "Take a screenshot" |
| A4 — Multilingual | 8 | Same prompts in Hindi, Turkish, Spanish, French |
| A5 — Vision | 5 | "What is on my screen?", "Read the error message", "Describe what's in front of the camera" |
| A6 — Multi-step | 7 | "Research mechanical engineering and save a summary to my desktop", "Find the cheapest flight from Delhi to Tokyo next month", "Install PUBG from Steam", "List my 5 largest files on Desktop" |
| A7 — Memory persistence | 5 | "My name is Abhinav", restart, "What's my name?" |
| A8 — Robustness | 5 | Mute mid-sentence, echo test, force WiFi off mid-conversation |

The seed set is intentionally **biased toward Indian-English vocabulary and accent**, since that matches the target user. A future English-American or English-British evaluation is left for further work.

---

## 5.1.1.8 Speaker and Environment Variability

Each prompt in category A3 is repeated three times under three speaker conditions to test acoustic robustness:

1. **Quiet room**, conversational tone, 0.5 m from the mic.
2. **Mild background noise** (a fan running ~50 dB), same distance.
3. **At distance** (~2 m from the mic), quiet room.

Per category A8, additional adversarial conditions include:

- Speaking immediately as R.A.Y.A starts replying (interruption / barge-in).
- Toggling F4 mute while speaking.
- Disabling WiFi for 10 seconds during a quiet moment.

---

## 5.1.1.9 Procedure for a Single Test Session

1. **Boot R.A.Y.A** with `python main.py` from a fresh terminal. Wait for the activity log to show `SYS: R.A.Y.A online.`.
2. **Wait** until the face state is `LISTENING` and the system notice clears.
3. **Run** the prompts of one category in order.
4. For each prompt: speak it once and only once (no repetitions unless the test explicitly allows them).
5. **Record** the verdict, latency, and any free-text observation in a test-log spreadsheet.
6. **Reset** between categories: close any windows opened by the test, clear the desktop of any new files, re-mute if needed.
7. Between days, **clear the long-term memory** (delete `memory/long_term.json`) before running category A7 to avoid carry-over.

A complete test session of 60 prompts takes roughly **45–60 minutes** of wall-clock time, including reset overhead.

---

## 5.1.1.10 Threats to Validity

A manual test campaign has well-known limitations. They are listed here to scope the conclusions in Section 5.1.2 honestly.

| Threat | Effect | Mitigation |
|---|---|---|
| **LLM non-determinism** | Same prompt may yield different tool selections across runs | Each A3 prompt is run **3 times**; results are averaged |
| **Network jitter** | Inflates the variance of latency measurements | Tests are run on a stable broadband link; outliers > 3σ are discarded |
| **Observer effect** | The developer knows what the "right" answer is | All Pass criteria are written **before** running the prompt |
| **Single-machine evaluation** | Results may not generalize across hardware | The reference machine is documented and treated as the lower bound; faster machines will see better numbers |
| **Single accent / language profile** | Tool-call accuracy may differ for other speaker populations | A4 covers four extra languages; broader accent evaluation is future work |
| **Subjective Partial / Fail boundary** | Rubric application can drift | The rubric is in Section 5.1.1.5 and every Partial includes the reason |

---

## 5.1.1.11 Data Recording

All raw test logs were recorded in a single spreadsheet (one row per (test ID, attempt)) with the following columns:

```
test_id | category | prompt | language | attempt | env (quiet/noisy/distance)
| verdict | e2e_latency_s | tool_latency_s | observed_tool(s)
| expected_tool(s) | notes
```

Aggregated tables in 5.1.2 are computed from this spreadsheet.

---

## 5.1.1.12 What This Methodology Does Not Try to Prove

To stay honest about the scope of the testing:

- It does **not** measure word-error-rate of the underlying Gemini ASR — that is a Google-owned black box and out of scope.
- It does **not** benchmark Gemini Live latency in isolation — only the **end-to-end** R.A.Y.A latency, which includes mic capture, queueing, network, model inference, tool dispatch, and playback.
- It does **not** claim coverage parity with an automated test suite. The campaign is a **representative sample**, not an exhaustive enumeration of every possible voice command.
- It does **not** evaluate user-acceptance metrics (likeability, perceived intelligence) — that would require a separate human-factors study with multiple participants.

---

## Summary

The audio-based testing methodology is a **structured manual campaign** of 60 spoken prompts across 8 categories, run on a documented reference machine using a defined three-tier rubric (Pass / Partial / Fail), with five quantitative metrics recorded in a spreadsheet. The campaign is designed to validate every major audio-driven capability of R.A.Y.A v2.4 — conversation, tool routing, multilingual support, vision-via-voice, multi-step goals, memory persistence, and robustness — while being explicit about the limitations of a single-developer, single-machine evaluation.
