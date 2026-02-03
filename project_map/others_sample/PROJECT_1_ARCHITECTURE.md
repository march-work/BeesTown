# System Architecture

This document describes the high-level architecture of the Swarm IDE project.

## Overview

The system is built primarily on a **Next.js** full-stack architecture, integrating an AI Agent Runtime, Real-time communication, and a relational database.

```mermaid
graph TD
    %% Styling
    classDef client fill:#f9f,stroke:#333,stroke-width:2px;
    classDef backend fill:#e1f5fe,stroke:#0277bd,stroke-width:2px;
    classDef db fill:#e8f5e9,stroke:#2e7d32,stroke-width:2px;
    classDef external fill:#fff3e0,stroke:#ef6c00,stroke-width:2px;

    %% Client Layer
    subgraph Client ["Client Side"]
        Browser["Web Browser (User)"]:::client
    end

    %% Backend Layer
    subgraph Backend ["Backend (Next.js)"]
        direction TB
        
        %% API Routes
        subgraph API ["API Layer (Route Handlers)"]
            AdminAPI["/api/admin"]
            AgentAPI["/api/agents"]
            GroupAPI["/api/groups"]
            HealthAPI["/api/health"]
        end

        %% Core Runtime
        subgraph Runtime ["Agent Runtime Core"]
            AgentRuntime["Agent Runtime"]
            EventBus["Event Bus"]
            MCP["Model Context Protocol (MCP)"]
            UIBus["UI Bus"]
        end

        %% Services
        subgraph Services ["Internal Services"]
            StreamService["Stream Service"]
            RealtimeService["Upstash/Redis Realtime"]
        end
    end

    %% Data Layer
    subgraph Data ["Data Persistence"]
        Postgres[("PostgreSQL Database")]:::db
        Redis[("Redis Cache")]:::db
    end

    %% External
    subgraph External ["External Services"]
        LLM["LLM Providers (OpenAI, GLM, etc.)"]:::external
        MCPServers["External MCP Servers"]:::external
    end

    %% Connections
    Browser -->|HTTP/WebSocket| API
    Browser -->|Realtime Updates| RealtimeService

    API --> AgentRuntime
    API --> Services

    AgentRuntime --> EventBus
    AgentRuntime --> MCP
    AgentRuntime --> Postgres
    
    EventBus --> UIBus
    UIBus --> RealtimeService

    MCP --> MCPServers
    AgentRuntime --> LLM

    Services --> Postgres
    Services --> Redis
    
    %% Specific Data Flows
    AgentAPI -->|Manage| AgentRuntime
    GroupAPI -->|Chat History| Postgres
```

## Component Details

### 1. Client Side
- **Web Browser**: The user interface, built with React and Tailwind CSS.
- **Features**: 
  - IM Interface (`/app/im`) for chatting with agents.
  - Graph View (`/app/graph`) for visualizing agent relationships.
  - Admin controls.

### 2. Backend (Next.js)
- **API Layer**: Handles HTTP requests for managing agents, groups, and messages.
- **Agent Runtime**: The core brain.
  - Manages agent lifecycles.
  - Handles context and history (`llm_history`).
  - Integrates with MCP for tools and context.
- **Event Bus**: Decouples components, allowing asynchronous communication.
- **Realtime Service**: Pushes updates to the client (e.g., new messages, agent status) using Redis/Upstash.

### 3. Data Persistence
- **PostgreSQL**: Stores persistent data using Drizzle ORM.
  - `workspaces`: Projects/Tenants.
  - `agents`: AI entities.
  - `groups` & `messages`: Chat data.
- **Redis**: Used for caching and real-time pub/sub.

### 4. External Integrations
- **LLM Providers**: The actual intelligence behind agents (e.g., GPT-4, GLM).
- **MCP Servers**: Provides external tools and data context to agents via the Model Context Protocol.

## Directory Structure Mapping

- **`backend/app/api`**: API Layer.
- **`backend/src/runtime`**: Runtime Core (`agent-runtime.ts`, `event-bus.ts`).
- **`backend/src/db`**: Database configuration and schema.
- **`backend/src/lib`**: Utility libraries (LLM streams, storage).
