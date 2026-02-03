# Agent Cluster Architecture

This document details the internal architecture of the Agent Swarm (Cluster) system.

## Core Concepts

The system follows a **Multi-Agent System (MAS)** architecture where each agent operates as an independent "process" (Runner) managed by a central Runtime. Agents communicate exclusively through message passing (IM paradigm) and share a common persistent storage.

```mermaid
graph TD
    %% Styling
    classDef runtime fill:#e1f5fe,stroke:#0277bd,stroke-width:2px;
    classDef runner fill:#fff9c4,stroke:#fbc02d,stroke-width:2px;
    classDef llm fill:#e8f5e9,stroke:#2e7d32,stroke-width:2px;
    classDef tool fill:#f3e5f5,stroke:#7b1fa2,stroke-width:2px;
    classDef bus fill:#ffebee,stroke:#c62828,stroke-width:2px;
    classDef db fill:#e0e0e0,stroke:#616161,stroke-width:2px;

    subgraph Host ["Agent Runtime Host (Node.js)"]
        direction TB
        
        RuntimeManager["Agent Runtime Manager"]:::runtime
        
        subgraph Runners ["Active Agent Runners"]
            direction LR
            Runner1["Agent Runner (A)"]:::runner
            Runner2["Agent Runner (B)"]:::runner
            RunnerN["..."]:::runner
        end

        subgraph Communication ["Event System"]
            AgentBus["Agent Event Bus (Internal)"]:::bus
            UIBus["UI Event Bus (Frontend)"]:::bus
        end

        subgraph Tooling ["Tool Execution Environment"]
            BuiltIn["Built-in Tools\n(create, send, bash)"]:::tool
            MCP["MCP Registry\n(External Tools)"]:::tool
        end
    end

    subgraph External ["External Resources"]
        LLM_API["LLM Provider API\n(GLM/OpenAI/OpenRouter)"]:::llm
        DB[("PostgreSQL\n(State & History)")]:::db
    end

    %% Relationships
    RuntimeManager -->|Spawns/Wakes| Runners
    
    %% Runner Internal Loop
    Runner1 -- "1. Wakeup Signal" --> Runner1
    Runner1 -- "2. Fetch Unread" --> DB
    Runner1 -- "3. Generate Prompt" --> LLM_API
    LLM_API -- "4. Stream Tokens/Calls" --> Runner1
    
    %% Tool Execution
    Runner1 -- "5. Execute Tool" --> Tooling
    BuiltIn -->|Create Agent| RuntimeManager
    BuiltIn -->|Send Message| DB
    BuiltIn -->|Wake Peer| RuntimeManager
    MCP -->|Call| ExternalMCP["External MCP Servers"]:::tool
    
    %% Event Flow
    Runner1 -- "Events (Stream, Tool)" --> AgentBus
    Runner1 -- "UI Updates" --> UIBus
    
    %% Inter-Agent Communication
    Runner1 -.->|Send Message| DB
    DB -.->|Wakeup Event| RuntimeManager
    RuntimeManager -.->|Wakeup| Runner2
```

## Detailed Agent Lifecycle (The Loop)

Each Agent Runner executes a continuous event loop designed to be stateless between wakeups but stateful during execution (via DB history).

```mermaid
sequenceDiagram
    participant Trigger as Event Trigger
    participant Runner as Agent Runner
    participant DB as Database
    participant LLM as LLM Provider
    participant Tools as Tool Registry
    participant Bus as Event Bus

    Note over Runner: Idle State (Awaiting Promise)

    Trigger->>Runner: Wakeup (Manual / New Message)
    activate Runner
    
    loop Process Until Idle
        Runner->>DB: listUnreadByGroup(agentId)
        DB-->>Runner: Batch of Messages
        
        opt No Unread Messages
            Runner->>Runner: Go to Sleep
        end

        loop For Each Message Batch
            Runner->>DB: Load Agent History & Context
            Runner->>Runner: Append New User Messages
            
            loop Reasoning Round (Max 3)
                Runner->>LLM: Stream Completion (History + Tools)
                activate LLM
                LLM-->>Runner: Stream (Content + Reasoning + ToolCalls)
                LLM-->>Bus: Emit "agent.stream"
                deactivate LLM
                
                alt Has Tool Calls
                    Runner->>Tools: Execute Tool (e.g., send_message)
                    activate Tools
                    Tools-->>Runner: Tool Result
                    Tools-->>Bus: Emit "agent.tool_result"
                    deactivate Tools
                    
                    Runner->>Runner: Append Tool Result to History
                else No Tool Calls
                    Runner->>Runner: Break Reasoning Loop
                end
            end
            
            Runner->>DB: Persist Updated History
            Runner->>Bus: Emit "agent.done"
        end
    end
    deactivate Runner
```

## Key Mechanisms

### 1. The Wakeup Mechanism
Agents are **reactive**. They do not run in the background unless triggered.
- **Triggers**:
  - `manual`: Admin/User manually wakes an agent.
  - `group_message`: Receiving a message in a group.
  - `direct_message`: Receiving a DM.
- **Process**: When Agent A sends a message to Agent B, the `send` tool implementation explicitly calls `runtime.wakeAgent(B)`.

### 2. Tool-Use & Delegation
The swarm capabilities come from the built-in tools:
- **`create(role, guidance)`**: Spawns a new sub-agent. The runtime immediately instantiates a runner for it.
- **`send(to, content)`**: Direct P2P communication.
- **`send_group_message(groupId, content)`**: Broadcast communication.
- **`bash(command)`**: Ability to interact with the host system (files, git, etc.).

### 3. State Management
- **LLM History**: Stored as a JSON blob in the `agents` table. Includes System prompt, User messages, Assistant replies, and Tool outputs.
- **Context Window**: Managed by `drizzle-orm` and raw JSON manipulation.
- **Short-term Memory**: In-memory `HistoryMessage[]` array during the runner loop.
- **Long-term Memory**: Persisted to Postgres.

### 4. Event Bus
- **AgentEventBus**: High-frequency internal events (token streaming, tool status). Used for real-time logs.
- **UIBus**: Lower-frequency events for the frontend (new message, agent created).
