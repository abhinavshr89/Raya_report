# 6.5 Automation and Tool Execution Evaluation

The most ambitious part of R.A.Y.A v2.4 is the layer between the model's intent and the operating system's response — the **tools** and the **agent** that orchestrates them. This section evaluates that machinery on its own terms: how well the routing works, how reliable execution is, how the agent's autonomous planning behaves, and where the architecture has earned its complexity.

---

## 6.5.1 Two Kinds of Automation in One System

R.A.Y.A actually contains two distinct automation pipelines, and it is important to evaluate them separately:

| Pipeline | Where it lives | When it fires |
|---|---|---|
| **Single-tool dispatch** | `RayaLive._execute_tool` in `main.py` | Every time the model calls a tool directly (the vast majority of interactions) |
| **Multi-step agent** | `agent/planner.py` + `agent/executor.py` + `agent/error_handler.py` + `agent/task_queue.py` | When the model calls `agent_task` for a complex goal |

The first pipeline is shallow and fast; the second is deep and deliberate. Both share the same underlying action modules in `actions/`. The evaluation below covers them separately and then assesses how cleanly they coexist.

---

## 6.5.2 Single-Tool Dispatch — What the Evaluation Reveals

The single-tool path is the workhorse. Most user commands are single-tool: open an app, search the web, send a message, check the weather, take a screenshot, set a reminder. The path from `response.tool_call` to the action function returning is short and well-instrumented.

Three observations stand out:

- **Routing is strong.** The model picks the right tool for the right command with high consistency. The strict, opinionated descriptions in `TOOL_DECLARATIONS` ("THE ONLY tool for ANY Steam or Epic Games request", "MUST be called when user asks what is on screen") do most of the heavy lifting here. Where two tools could both plausibly handle a command, the descriptions encode the preferred choice loudly enough that the model rarely picks the wrong one.
- **Execution is independent of the audio loop.** Every action runs in an executor thread (`loop.run_in_executor`). The audio path is never blocked, the UI keeps updating, and a slow tool does not interrupt the conversation.
- **UI feedback is consistent.** Every dispatch updates the UI through the same set of hooks (`set_state`, `push_activity`, `set_last_task_result`, `set_active_task`). The user always knows what is happening, even when the assistant's verbal acknowledgment is brief.

The honest evaluation: the single-tool pipeline is the most production-feeling part of the codebase. It is small, predictable, and well-bounded — exactly the qualities that real-world reliability requires.

---

## 6.5.3 Tool Routing as a Design Discipline

The quality of single-tool routing is not free — it is the product of a few specific decisions:

- **Strict JSON schemas.** Every tool defines required parameters and constrained enums in the description. The model rarely fabricates parameter shapes because the schema is unambiguous.
- **One-call policy in the system prompt.** `core/prompt.txt` instructs the model to call each tool exactly once per turn. This eliminates the loop-of-tool-calls failure mode common to early agent designs.
- **Description tone.** Tool descriptions are written like operational instructions, not like documentation. "Use for ANY single computer control command. NEVER route to agent_task." The imperative register biases the model toward decisive routing.
- **Disjoint scopes.** Where two tools could overlap (e.g., `computer_settings` vs `agent_task`, or `web_search` vs `browser_control`), the descriptions explicitly draw the boundary.

These choices do not eliminate routing errors, but they reduce them to a small enough rate that the user rarely notices.

---

## 6.5.4 Multi-Step Agent — What the Evaluation Reveals

The multi-step path is structurally different. It is the only place in the codebase where a single user utterance can spawn a sequence of model calls, tool invocations, and conditional branches. Several aspects deserve evaluation.

**The planner produces usable plans.** The few-shot examples in `PLANNER_PROMPT` (in `agent/planner.py`) cover enough goal shapes that the planner reliably emits plans in the requested JSON form. The `Max 5 steps` constraint keeps plans concise; the strict instruction to avoid `generated_code` keeps them within the documented tool surface.

**The executor's content injection is the most impactful single component.** `_inject_context` is the function that pipes the outputs of earlier steps into the inputs of later ones when the schemas match. Without it, the planner would have to predict the content of search results, which is impossible. With it, the planner can leave content fields blank and the executor fills them in. This is the engineering trick that turns "a sequence of independent tool calls" into "a coherent multi-step workflow."

**Error handling is structurally sound.** The `error_handler.analyze_error` four-decision space (retry, skip, replan, abort) is the right shape. Combined with the per-step retry counter (`attempt <= 3`) and the per-goal replan counter (`MAX_REPLAN_ATTEMPTS = 2`), the agent has well-bounded behavior even when individual steps fail.

**Translation in the loop is a quiet win.** The `_translate_to_goal_language` helper means that a Hindi goal produces a Hindi output file even when the intermediate search results were in English. This is invisible to the user, which is exactly the right level of integration.

**The task queue keeps the assistant responsive.** Each agent task runs on its own thread. The user can keep talking to the assistant while a complex task runs in the background, and the UI surfaces the running task through the queue snapshot. The cooperative cancellation via `cancel_flag` is also clean — a cancel does not leave the system in a half-state.

---

## 6.5.5 Where the Agent Earns Its Complexity

A multi-step planner-executor adds significant complexity to the codebase. It is fair to ask whether that complexity is justified.

The evaluation answer is yes, but with a sharp boundary. The complexity is justified for:

- **Goals that require chaining results.** Research-and-save, find-and-organize, compare-and-decide. Without the agent, these would not be possible at all.
- **Goals where the steps are uniform but the count is variable.** "List the 5 largest files and tell me their names" — the agent decides how many tool calls are needed.
- **Goals expressed at a high level.** "Update all my Steam games" — the agent picks the right tool and parameters even when the user has not.

It is *not* justified for:

- **Single-tool commands.** The system prompt is explicit: do not route single commands through `agent_task`. This boundary holds in practice.

Encoding this boundary in the prompt rather than in code is a key design choice. The model is the gatekeeper, and the system prompt's `ONLY for complex, multi-step planning (3+ steps)` instruction is the rule it follows.

---

## 6.5.6 The Action Layer as a Shared Substrate

A subtle but important architectural observation: every `actions/*.py` module has the same signature `(parameters, player=None, response=None, **kwargs) -> str`, and that uniformity pays off in the agent layer. The executor's `_call_tool` is a small switch over tool names that imports the action module lazily and calls its function. There is exactly one implementation of each tool, used by both the single-tool dispatch and the multi-step executor.

This means improvements to any tool (better error handling, better parameter parsing, new sub-features) are inherited by both pipelines automatically. The plug-in pattern is not a feature for end users; it is a feature for the project's maintainability.

---

## 6.5.7 Reliability and Failure Modes

A few specific reliability behaviors are worth calling out:

- **Tool errors are caught at the dispatcher level.** `_execute_tool` wraps every action in a try/except, surfaces the error to the UI, and speaks a short apology. The Live session does not crash.
- **Step errors are caught at the executor level.** `executor.execute` wraps every step in a try/except, decides retry/skip/replan/abort, and continues. The agent task does not crash.
- **Session errors are caught at the outermost level.** `RayaLive.run` wraps the entire connect-and-run block in a try/except, reconnects after a 3-second backoff. The application does not crash.

The three layers of error handling are independent. A tool can fail without breaking a step; a step can fail without breaking a goal; a session can fail without breaking the assistant. This nested defensiveness is the main reason R.A.Y.A behaves like a daily-driver tool rather than a demo.

---

## 6.5.8 Limitations of the Automation Layer

Honest limitations the evaluation surfaces:

- **No in-flight cancel via voice.** A running agent task can be cancelled through the queue API, but the voice/text interface does not expose a `cancel_task` tool. The user has to wait, or close the UI.
- **Tools assume cooperative external apps.** `send_message` assumes the user is already logged into WhatsApp/Telegram Web; `game_updater` assumes Steam/Epic are installed. When these assumptions break, the tool returns an error rather than recovering.
- **Browser warm-up is unavoidable.** The first Playwright invocation in a session is markedly slower than subsequent ones. A future enhancement could pre-warm the browser at startup, but the current trade-off (faster boot at the cost of slower first browser tool) is the right default for casual use.
- **The agent's plan is invisible to the user.** The plan is printed to the developer console but not shown in the UI. A future enhancement could surface plan steps to the activity log as they are decided.
- **No tool composition outside the agent.** The single-tool path cannot chain two tools without going through `agent_task`. For most commands this is fine, but the boundary between "single tool" and "multi-step" sometimes feels arbitrary.

None of these are showstoppers, and most map to clear future-work items.

---

## 6.5.9 What Tool Execution Tells Us About the System

Stepping back, the way tools execute reveals the overall design philosophy of R.A.Y.A v2.4:

- **Bounded autonomy.** The agent can plan, retry, and replan, but only within explicit bounds (5 steps, 3 retries, 2 replans). The system never enters an unbounded loop.
- **Transparent execution.** The UI surfaces every tool call as it happens. The user can audit what the assistant did, in order.
- **Local primacy.** Tools live in code on the user's machine. The model can call them, but it cannot extend the toolset at runtime. The user (and the developer) is in charge of what the assistant can do.
- **Graceful failure.** Three nested error-handling layers mean that any single failure is isolated. The assistant degrades step-by-step rather than collapsing.

These properties together make R.A.Y.A's automation layer feel **defensible** rather than experimental. The agent can do impressive things, but it is not allowed to do uncontrolled things. That is the most important quality a personal-desktop assistant can have.

---

## 6.5.10 Summary

R.A.Y.A's automation and tool execution layer is the heart of the system's usefulness. The single-tool dispatch is fast and dependable; the multi-step agent is bounded and recoverable; the action modules are uniformly shaped and shared across both pipelines; and three layers of error handling keep the assistant alive even when individual operations fail. Inter-step content injection and language-aware translation are the two most consequential engineering decisions in the agent layer, and both behave invisibly in everyday use. The remaining limitations — no voice-driven cancel, no in-UI plan visibility, no startup browser pre-warm — are well-scoped and tractable. As an automation surface for a single user's daily desktop work, the system clears the bar set by the project objectives and does so without compromising the safety properties that make casual use viable.
