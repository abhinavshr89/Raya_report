# R.A.Y.A v2.4 — System Design Diagrams

Mermaid sources for the System Design section. Each diagram below is also kept
as a standalone `.mmd` file in this folder. These reflect the **actual v2.4
codebase** (Gemini-only; JSON-file + in-memory persistence), not an idealized
SQLite/OpenRouter design.

| # | Diagram | File |
|---|---|---|
| 1 | Level 0 Data Flow (Context) | `01_dfd_level0.mmd` |
| 2 | Level 1 Data Flow | `02_dfd_level1.mmd` |
| 3 | Use Case | `03_use_case.mmd` |
| 4 | Activity | `04_activity.mmd` |
| 5 | Sequence | `05_sequence.mmd` |
| 6 | Entity Relationship (data model) | `06_er_diagram.mmd` |

---

## 1. Level 0 Data Flow Diagram (Context)

R.A.Y.A as a single process exchanging data with the User, the Gemini Cloud LLM
provider, the host OS/hardware, and its local JSON store.

```mermaid
flowchart LR
    User([User]):::ent
    LLM([Gemini Cloud<br/>LLM Provider]):::ent
    OS([OS · Apps · Internet<br/>Mic · Speakers · Screen · Webcam]):::ent
    Store[("Local Store<br/>long_term.json · api_keys.json")]:::store

    RAYA((("R.A.Y.A<br/>System"))):::proc

    User -->|voice / text command| RAYA
    RAYA -->|audio reply · UI activity| User
    RAYA -->|audio · text · tool declarations| LLM
    LLM -->|audio · transcripts · tool_calls · plans| RAYA
    RAYA -->|app launch · file ops · browser · capture| OS
    OS -->|screen / webcam frames · command results| RAYA
    Store -->|memory + API key at boot| RAYA
    RAYA -->|save_memory writes| Store

    classDef ent fill:#1f2937,stroke:#60a5fa,color:#fff;
    classDef proc fill:#0b3d2e,stroke:#34d399,color:#fff;
    classDef store fill:#3b2f0b,stroke:#fbbf24,color:#fff;
```

---

## 2. Level 1 Data Flow Diagram

The single process decomposed into seven sub-processes and four data stores
(D1 long-term memory, D2 config, D3 in-memory task queue, D4 in-memory step
results). Flows mirror the real call graph: dispatch → plan → execute →
error-handle → report.

```mermaid
flowchart TD
    User([User]):::ent
    LLM([Gemini Cloud]):::ent
    OS([OS · Internet · Hardware]):::ent

    P1(["1.0 Input Processing<br/>capture audio/text · build prompt"]):::proc
    P2(["2.0 Tool Dispatch<br/>RayaLive._execute_tool"]):::proc
    P3(["3.0 Plan Generation<br/>planner.create_plan"]):::proc
    P4(["4.0 Task Execution<br/>executor + actions/*"]):::proc
    P5(["5.0 Error Handling & Replan<br/>error_handler"]):::proc
    P6(["6.0 Memory Management<br/>memory_manager"]):::proc
    P7(["7.0 Result Reporting<br/>UI activity · speech"]):::proc

    D1[("D1 Long-term Memory<br/>long_term.json")]:::store
    D2[("D2 Config<br/>api_keys.json")]:::store
    D3[("D3 Task Queue<br/>in-memory priority queue")]:::store
    D4[("D4 Step Results<br/>in-memory dict")]:::store

    User -->|voice / text| P1
    D2 -->|API key at boot| P1
    P1 -->|PCM stream + tool decls| LLM
    LLM -->|tool_call| P2

    P2 -->|single-tool args| P4
    P2 -->|agent_task goal| D3
    D3 -->|next task| P4

    P4 -->|goal / replan request| P3
    P3 -->|JSON plan| P4
    P3 -. reads .-> D2

    P4 -->|step + params| OS
    OS -->|step output| P4
    P4 -->|store output| D4
    D4 -->|prior outputs injected| P4

    P4 -->|failed step + error| P5
    P5 -->|retry / skip / abort| P4
    P5 -->|revised plan| P3

    P2 -->|save_memory fact| P6
    P6 -->|read at boot · write| D1

    P4 -->|final summary| P7
    P2 -->|tool result| P7
    P7 -->|tool_response| LLM
    LLM -->|synthesized audio| P7
    P7 -->|spoken reply + UI| User

    classDef ent fill:#1f2937,stroke:#60a5fa,color:#fff;
    classDef proc fill:#0b3d2e,stroke:#34d399,color:#fff;
    classDef store fill:#3b2f0b,stroke:#fbbf24,color:#fff;
```

---

## 3. Use Case Diagram

Primary actor **User**; supporting actor **Gemini Cloud** backs the AI-driven
use cases. `«include»` edges show that conversation can silently save memory and
that a multi-step task draws on web-search and file tools.

```mermaid
flowchart LR
    user(("👤<br/>User")):::actor
    gem(("☁️<br/>Gemini Cloud")):::actor

    subgraph SYS["R.A.Y.A System"]
        direction TB
        uc1(["Converse by voice"])
        uc2(["Send text command"])
        uc3(["Open apps & control system"])
        uc4(["Search the web"])
        uc5(["Analyze screen / webcam"])
        uc6(["Manage files"])
        uc7(["Control browser"])
        uc8(["Send messages"])
        uc9(["Generate code / build project"])
        uc10(["Manage Steam / Epic games"])
        uc11(["Run multi-step task"])
        uc12(["Monitor task queue"])
        uc13(["Mute / privacy toggle"])
        uc14(["Shut down assistant"])
        uc15(["Save memory"])
        uc16(["Set API key (first run)"])
    end

    user --- uc1
    user --- uc2
    user --- uc3
    user --- uc4
    user --- uc5
    user --- uc6
    user --- uc7
    user --- uc8
    user --- uc9
    user --- uc10
    user --- uc11
    user --- uc12
    user --- uc13
    user --- uc14
    user --- uc16

    uc1 -. "«include»" .-> uc15
    uc11 -. "«include»" .-> uc4
    uc11 -. "«include»" .-> uc6

    uc1  --- gem
    uc5  --- gem
    uc9  --- gem
    uc11 --- gem

    classDef actor fill:#1f2937,stroke:#60a5fa,color:#fff;
```

---

## 4. Activity Diagram

The complete task lifecycle. Branches at "tool call needed?" into a
conversational reply, a single-tool dispatch, or the agent path with its three
bounded loops (step loop, retry ≤3, replan ≤2).

```mermaid
flowchart TD
    A([Start]) --> B[/User enters request: voice or text/]
    B --> C[Input processing & prompt construction]
    C --> D[Gemini Live interprets request]
    D --> E{Tool call needed?}

    E -->|No| F[Speak conversational reply]
    F --> Z([End])

    E -->|Single tool| G[Dispatch to action module]
    G --> H[Return result and speak]
    H --> Z

    E -->|agent_task| I[Submit goal to Task Queue]
    I --> J[Worker thread pulls task]
    J --> K[Planner generates JSON plan]
    K --> L{Plan has valid steps?}
    L -->|No| M[Use fallback plan]
    M --> N[Take next step]
    L -->|Yes| N

    N --> O[Inject prior results + translate]
    O --> P[Call step tool]
    P --> Q{Step succeeded?}

    Q -->|Yes| R[Store step result]
    R --> S{More steps?}
    S -->|Yes| N
    S -->|No| T[Summarize and speak]
    T --> Z

    Q -->|No| U[Error handler analyzes failure]
    U --> V{Recovery decision}
    V -->|retry &le;3| P
    V -->|skip| R
    V -->|replan &le;2| K
    V -->|abort| W[Inform user: task aborted]
    W --> Z
```

---

## 5. Sequence Diagram

Message timeline for an `agent_task` request, plus the silent `save_memory`
path. Single-tool requests skip the Queue/Planner/Executor lanes.

```mermaid
sequenceDiagram
    actor User
    participant GUI as RayaUI (GUI)
    participant Orch as RayaLive (Orchestrator)
    participant Live as Gemini Live
    participant Queue as TaskQueue
    participant Plan as Planner (Flash-Lite)
    participant Exec as AgentExecutor
    participant Act as Action Module
    participant Mem as MemoryManager

    User->>GUI: speak / type request
    GUI->>Orch: audio frames / on_text_command
    Orch->>Live: send_realtime_input / send_client_content
    Live-->>Orch: tool_call(agent_task, goal)
    Orch->>Queue: submit(goal, priority)
    Queue-->>Orch: task_id
    Orch->>Live: send_tool_response("Task started")

    Note over Queue: worker thread pulls task
    Queue->>Exec: execute(goal)
    Exec->>Plan: create_plan(goal)
    Plan->>Live: planning prompt
    Live-->>Plan: JSON plan
    Plan-->>Exec: steps[]

    loop each step (retry <=3, replan <=2)
        Exec->>Exec: inject prior results + translate
        Exec->>Act: _call_tool(tool, params)
        Act-->>Exec: step result
    end

    Exec->>Exec: _summarize(completed_steps)
    Exec->>Orch: speak(summary)
    Orch->>Live: send_client_content(summary)
    Live-->>Orch: synthesized audio
    Orch->>GUI: audio + activity/queue snapshot
    GUI-->>User: spoken result

    opt user reveals a personal fact
        Live-->>Orch: tool_call(save_memory)
        Orch->>Mem: update_memory(category/key/value)
        Note over Orch,Mem: silent — no spoken acknowledgement
    end
```

---

## 6. Entity Relationship Diagram (actual data model)

> **Important:** R.A.Y.A v2.4 has **no relational/SQLite database**. Persistent
> state is `memory/long_term.json` (six categories, 2200-char cap) and
> `config/api_keys.json`. Tasks and their plan steps live **in memory** in the
> `TaskQueue`/`Task` objects. This ER models those real structures.

```mermaid
erDiagram
    MEMORY_FILE  ||--o{ CATEGORY    : contains
    CATEGORY     ||--o{ ENTRY       : holds
    TASK_QUEUE   ||--o{ TASK        : queues
    TASK         ||--o{ PLAN_STEP   : "plan has"
    PLAN_STEP    ||--o| STEP_RESULT : produces

    MEMORY_FILE {
        string path "memory/long_term.json"
        int    max_chars "2200 cap, oldest trimmed"
    }
    CATEGORY {
        string name "identity | preferences | projects | relationships | wishes | notes"
    }
    ENTRY {
        string key "snake_case"
        string value "<=380 chars"
        string updated "YYYY-MM-DD"
    }
    CONFIG {
        string gemini_api_key
    }
    TASK_QUEUE {
        int max_concurrent "1"
    }
    TASK {
        string task_id PK
        int    priority "1 high .. 3 low"
        string goal
        string status "pending|running|completed|failed|cancelled"
        float  created_at
        string result
        string error
    }
    PLAN_STEP {
        int    step
        string tool
        string description
        json   parameters
        bool   critical
    }
    STEP_RESULT {
        int    step_num FK
        string output
    }
```
