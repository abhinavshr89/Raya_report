# R.A.Y.A v2.4 — System Design Diagrams (compact)

Trimmed Mermaid diagrams — only the essential nodes, so they render small enough
to paste into the report. Each is also a standalone `.mmd` file in this folder.
All reflect the **actual v2.4 code** (Gemini-only; JSON + in-memory state, no SQLite).

---

## 1. Level 0 Data Flow (Context)

```mermaid
flowchart LR
    User([User]) -- "voice/text ⇄ audio·UI" --> RAYA(((R.A.Y.A System)))
    RAYA -- "stream ⇄ tool_calls·plans" --> LLM([Gemini Cloud])
    RAYA -- "actions ⇄ results" --> OS([OS · Hardware · Internet])
    RAYA -- "read / write" --> Store[("long_term.json<br/>api_keys.json")]
```

## 2. Level 1 Data Flow

```mermaid
flowchart TD
    User([User]) <--> Disp["Dispatch<br/>_execute_tool"]
    Disp <--> LLM([Gemini])
    Disp -->|single tool| Act["Action Modules"]
    Disp -->|agent_task| Q[("Task Queue")]
    Q --> Exec["Executor"]
    Exec <--> Plan["Planner"]
    Exec --> Act
    Exec -->|on failure| Err["Error Handler<br/>retry / skip / replan / abort"]
    Err --> Exec
    Disp -->|save_memory| Mem[("long_term.json")]
    Exec -->|final summary| Disp
```

## 3. Use Case

```mermaid
flowchart LR
    U(("User"))
    subgraph RAYA["R.A.Y.A System"]
        a([Converse / command])
        b([Control system & apps])
        c([Web & browser])
        d([Vision: screen / webcam])
        e([Files & messages])
        f([Code / dev projects])
        g([Multi-step task])
        h([Memory & privacy])
    end
    U --- a & b & c & d
    U --- e & f & g & h
```

## 4. Activity

```mermaid
flowchart TD
    A([Start]) --> B[Request → Gemini]
    B --> C{Tool call?}
    C -->|No| Z([Reply & End])
    C -->|Single tool| D[Run action] --> Z
    C -->|agent_task| E[Queue → Plan]
    E --> F[Next step:<br/>inject + run tool]
    F --> G{Step OK?}
    G -->|Yes| H{More steps?}
    H -->|Yes| F
    H -->|No| I[Summarize] --> Z
    G -->|No| J{Recover?}
    J -->|retry ≤3| F
    J -->|replan ≤2| E
    J -->|abort| Z
```

## 5. Sequence

```mermaid
sequenceDiagram
    actor User
    participant Orch as RayaLive
    participant Live as Gemini
    participant Q as TaskQueue
    participant Exec as Executor+Actions

    User->>Orch: voice / text
    Orch->>Live: stream
    Live-->>Orch: tool_call(agent_task)
    Orch->>Q: submit(goal)
    Q->>Exec: execute
    Exec->>Live: plan request
    Live-->>Exec: JSON plan
    loop each step (retry / replan)
        Exec->>Exec: run tool + store result
    end
    Exec-->>Orch: summary
    Orch->>Live: speak
    Live-->>User: audio reply
```

## 6. Entity Relationship (real data model — no SQLite)

```mermaid
erDiagram
    CATEGORY  ||--o{ ENTRY       : holds
    TASK      ||--o{ PLAN_STEP   : has
    PLAN_STEP ||--o| STEP_RESULT : produces

    CATEGORY {
        string name "6 fixed categories"
    }
    ENTRY {
        string key
        string value
        string updated
    }
    TASK {
        string task_id PK
        int    priority
        string goal
        string status
    }
    PLAN_STEP {
        int    step
        string tool
        json   parameters
    }
    STEP_RESULT {
        int    step_num FK
        string output
    }
    CONFIG {
        string gemini_api_key
    }
```
