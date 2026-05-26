# 5.2.1 Text-Based Testing — Methodology

R.A.Y.A v2.4 is voice-first, but the UI also exposes a **text-input channel** that routes typed messages into the same Gemini Live session. Text-based testing exists to (a) verify that the text channel achieves feature parity with voice, (b) probe edge cases that are awkward or impossible to speak — long inputs, code snippets, URLs, special characters — and (c) measure performance without the variance introduced by microphones and acoustic environments.

This section documents the methodology of that text-based campaign. The format mirrors 5.1.1 so the two campaigns are directly comparable.

---

## 5.2.1.1 Test Environment

The same reference machine described in 5.1.1.1 was used. Only the input channel differs.

| Component | Specification |
|---|---|
| **CPU / RAM / OS / Python** | Identical to 5.1.1.1 |
| **Input** | Tkinter `Entry` widget at the bottom of the UI; submitted with **Enter** |
| **Output** | Spoken audio through the same `_play_audio` pipeline + transcript line in the activity log |
| **Microphone** | Muted (F4) for the duration of the campaign — guarantees the only input the model sees is typed |
| **Browser** | Bundled Playwright Chromium |
| **Network** | Same broadband as the audio campaign |

Muting the mic during text testing removes one variable from the experiment and isolates the text path cleanly.

---

## 5.2.1.2 Scope of Text Testing

Text testing is structured to complement, not duplicate, the audio campaign. Categories are chosen so that each one either:

- **(a)** validates parity with an audio category, or
- **(b)** exercises a behavior that text is uniquely suited to test.

### Parity categories

| ID | Category | Mirrors |
|---|---|---|
| **T1** | Smoke / wake-up via text | A1 |
| **T2** | Tool-call accuracy via text | A3 |
| **T3** | Multi-step goals via text | A6 |

### Text-unique categories

| ID | Category | Why text is appropriate |
|---|---|---|
| **T4** | Long-input handling | A 2,000-character paste is awkward to dictate but trivial to paste |
| **T5** | Special-character & symbol handling | Code, URLs, file paths, regex — error-prone to speak |
| **T6** | Code- and dev-flow tests | `code_helper` and `dev_agent` are easier to evaluate with precise written prompts |
| **T7** | Latency baseline | Removes mic/speech variance, isolates network + model + dispatch latency |
| **T8** | Mixed-channel conversation | Alternate text and voice in the same session — verifies unified session state |

---

## 5.2.1.3 Test-Case Format

Identical to the audio campaign (5.1.1.4), with two field changes:

- `Prompt (EN):` is replaced by `Typed input:` and quotes the exact string entered.
- `Latency target` is tightened to **< 1.0 s** for no-tool prompts (no mic capture overhead).

Example:

```
Test ID:        T2-08
Category:       T2 — Tool-call accuracy via text
Typed input:    "open https://github.com/anthropics/claude-code in incognito"
Expected tool:  browser_control(action="go_to", url="https://github.com/anthropics/claude-code", incognito=true)
Pass criteria:  URL opens in a private window of the active browser
Latency target: < 3 s to URL navigation
Recorded result: Pass / Partial / Fail
```

---

## 5.2.1.4 Evaluation Rubric

Identical to 5.1.1.5 (Pass / Partial / Fail) so the two campaigns can be aggregated.

---

## 5.2.1.5 Metrics Collected

The same five metrics as 5.1.1.6 are collected, with two additions specific to text:

| Additional metric | Definition |
|---|---|
| **Time to first token** | From Enter keypress to the start of the response (audio or activity log line) |
| **Long-input truncation** | Whether inputs > 1500 characters reach the model intact (visible in the input transcription field) |

---

## 5.2.1.6 Prompt Set

A fixed seed set of **45 typed prompts** distributed across the eight categories:

| Category | # | Examples |
|---|---|---|
| T1 — Smoke | 3 | `hello`, `status`, `what can you do` |
| T2 — Tool-call accuracy | 15 | `open chrome`, `weather in tokyo`, `set volume to 30`, `play despacito on youtube`, `remind me at 7pm tomorrow to call dad`, `list files on desktop`, `screenshot this` |
| T3 — Multi-step | 5 | `research linux distros and save the comparison to a desktop file`, `find the largest 5 files on my desktop and tell me their names`, `compare iphone 15 and pixel 9 prices` |
| T4 — Long input | 5 | A 500-word product description with the prompt *"summarize this in 5 bullet points"*; a 1,500-character email draft with *"make this more concise"*; a 2,000-character paste with *"translate to Hindi"* |
| T5 — Special characters | 5 | `open file://C:\Users\Abhinav\Desktop\notes.txt`, `search for "self-attention" site:arxiv.org`, `regex to match email addresses`, `\n\n\n\n` (empty prompt with newlines), emojis-only input |
| T6 — Code / dev | 5 | `write a python script that lists all .pdf files in downloads`, `explain this code: def foo(x): return x*x`, `make a react todo app with localstorage` (dev_agent) |
| T7 — Latency baseline | 4 | Single-word prompts: `time`, `date`, `joke`, `who are you` |
| T8 — Mixed channel | 3 | Type prompt → unmute → speak follow-up → re-mute → type continuation |

---

## 5.2.1.7 Procedure for a Single Text Session

1. **Boot R.A.Y.A** and wait for `SYS: R.A.Y.A online`.
2. **Mute the microphone** (F4 — UI shows MUTED state).
3. **Type the first prompt** into the entry field; press Enter; record latency from Enter to first audible token.
4. **Wait** for the activity log to show the turn complete.
5. **Repeat** for the next prompt in the category.
6. For T8 only: **unmute** and continue with voice for the speaking step; re-mute before typing again.
7. **Record** verdict + latency + free-text observations in the same spreadsheet used for the audio campaign.

A full text session of 45 prompts takes roughly **30 minutes** of wall-clock time.

---

## 5.2.1.8 Threats to Validity

The same threats listed in 5.1.1.10 apply, with one notable addition:

| Threat | Note |
|---|---|
| **Copy/paste IME quirks** | On Windows, certain pasted content (e.g., text containing newlines copied from a code editor) may submit prematurely when Enter is interpreted by Tk. Mitigation: pastes use Ctrl+V then a manual Enter, never paste-with-Enter. |

---

## 5.2.1.9 What This Methodology Does Not Try to Prove

- It does **not** evaluate accessibility for users with motor or visual impairments — that requires a dedicated study.
- It does **not** stress-test for very long sessions (multi-hour); session-resumption behavior under such conditions is covered briefly in the robustness category A8 of audio testing.
- It does **not** evaluate the assistant's behavior when **both** text and voice arrive simultaneously — this is constrained by the UI (the entry only submits on Enter, the mic gate is per-turn).

---

## Summary

The text-based testing methodology mirrors the audio campaign for parity, then adds four text-unique categories (long input, special characters, code/dev workflows, mixed-channel). 45 typed prompts across 8 categories, same rubric and metrics, mic muted to isolate the text path, all results recorded into the same spreadsheet so totals can be combined with the audio campaign in 5.2.2.
