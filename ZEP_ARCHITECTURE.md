# Zep Architecture: A Comprehensive Analysis

## Executive Summary

**Zep** is a context engineering platform for AI agents, designed to solve the critical problem of providing agents with the right contextual information at the right time. The platform is powered by **Graphiti**, an open-source temporal knowledge graph framework that enables real-time, incremental updates to dynamic knowledge graphs.

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           ZEP CLOUD PLATFORM                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐                     │
│  │   INGEST    │───▶│    GRAPH    │───▶│  ASSEMBLE   │                     │
│  │             │    │             │    │             │                     │
│  │ • Messages  │    │ • Entities  │    │ • Context   │                     │
│  │ • JSON Data │    │ • Relations │    │ • Retrieval │                     │
│  │ • Documents │    │ • Temporal  │    │ • Ranking   │                     │
│  └─────────────┘    └─────────────┘    └─────────────┘                     │
│                                                                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                         GRAPHITI (Core Engine)                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  Temporal Knowledge Graph Framework                                  │   │
│  │  • Real-time incremental updates                                     │   │
│  │  • Bi-temporal data model                                            │   │
│  │  • LLM-driven extraction & deduplication                             │   │
│  │  • Hybrid search (embedding + BM25 + graph traversal)                │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                           STORAGE LAYER                                     │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌────────────┐           │
│  │   Neo4j    │  │  FalkorDB  │  │    Kuzu    │  │  Neptune   │           │
│  │  (Primary) │  │  (Redis)   │  │ (Embedded) │  │   (AWS)    │           │
│  └────────────┘  └────────────┘  └────────────┘  └────────────┘           │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                              SDKs & INTEGRATIONS                            │
├─────────────────────────────────────────────────────────────────────────────┤
│  SDKs:        │  Integrations:     │  Frameworks:                          │
│  • Python     │  • AutoGen         │  • LangChain                          │
│  • TypeScript │  • CrewAI          │  • LlamaIndex                         │
│  • Go         │  • LiveKit         │  • LangGraph                          │
│               │  • OpenAI Agents   │  • Model Context Protocol (MCP)       │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Core Components

### 1. Graphiti: The Temporal Knowledge Graph Engine

Graphiti is the foundational technology powering Zep. It's an open-source framework for building temporally-aware knowledge graphs.

#### Three-Tier Graph Structure

```
┌─────────────────────────────────────────────────────────────────┐
│                    COMMUNITY SUBGRAPH                           │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐             │
│  │ Community A │  │ Community B │  │ Community C │             │
│  │  (Summary)  │  │  (Summary)  │  │  (Summary)  │             │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘             │
│         │ HAS_MEMBER     │               │                      │
├─────────┼────────────────┼───────────────┼──────────────────────┤
│         ▼                ▼               ▼                      │
│                 SEMANTIC ENTITY SUBGRAPH                        │
│  ┌────────┐     ┌────────┐     ┌────────┐     ┌────────┐       │
│  │ Entity │────▶│ Entity │────▶│ Entity │────▶│ Entity │       │
│  │  Node  │     │  Node  │     │  Node  │     │  Node  │       │
│  └────┬───┘     └────┬───┘     └────┬───┘     └────┬───┘       │
│       │ MENTIONS     │              │              │            │
├───────┼──────────────┼──────────────┼──────────────┼────────────┤
│       ▼              ▼              ▼              ▼            │
│                   EPISODE SUBGRAPH                              │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐       │
│  │ Episode  │─▶│ Episode  │─▶│ Episode  │─▶│ Episode  │       │
│  │ (Raw)    │  │ (Raw)    │  │ (Raw)    │  │ (Raw)    │       │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘       │
│                    NEXT_EPISODE                                 │
└─────────────────────────────────────────────────────────────────┘
```

#### Node Types

| Node Type | Purpose | Key Properties |
|-----------|---------|----------------|
| **EpisodicNode** | Raw input data (messages, text, JSON) | `valid_at`, `created_at`, `content`, `source` |
| **EntityNode** | Extracted entities | `name`, `name_embedding`, `summary`, `labels`, `attributes` |
| **CommunityNode** | Clustered entity groups | `name_embedding`, `summary`, member references |
| **SagaNode** | Sequence container | Links episodes in order |

#### Edge Types

| Edge Type | Connects | Purpose |
|-----------|----------|---------|
| **EntityEdge** (RELATES_TO) | Entity → Entity | Knowledge graph facts with temporal bounds |
| **EpisodicEdge** (MENTIONS) | Episode → Entity | Source attribution |
| **CommunityEdge** (HAS_MEMBER) | Community → Entity | Cluster membership |
| **NextEpisodeEdge** | Episode → Episode | Sequential ordering |

### 2. Bi-Temporal Data Model

A distinctive feature of Zep/Graphiti is tracking **two timelines**:

```
Timeline T  (Event Time)     │  Timeline T' (Ingestion Time)
─────────────────────────────┼─────────────────────────────────
When events actually         │  When the system learned
occurred in the real world   │  about the events
                             │
valid_at / invalid_at        │  created_at / expired_at
```

**Example: Tracking Relationship Changes**

```
Edge: "Alice is married to Bob"
├── valid_at: 2020-06-15      (wedding date)
├── invalid_at: 2024-03-01    (divorce date)
├── created_at: 2024-01-10    (when system learned)
└── expired_at: 2024-03-15    (when system learned of divorce)

Edge: "Alice is divorced from Bob"
├── valid_at: 2024-03-01      (divorce date)
├── invalid_at: NULL          (still current)
├── created_at: 2024-03-15    (when system learned)
└── expired_at: NULL          (still in system)
```

This enables:
- Point-in-time queries ("What did we know as of date X?")
- Historical relationship tracking
- Contradiction detection and resolution

### 3. Data Processing Pipeline

#### Episode Ingestion Flow

```
┌──────────────────────────────────────────────────────────────────────────┐
│                        ADD_EPISODE PIPELINE                              │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  Input Episode                                                           │
│       │                                                                  │
│       ▼                                                                  │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │ 1. EXTRACT ENTITIES                                              │    │
│  │    • LLM extracts entities from episode content                  │    │
│  │    • Reflexion step catches missed entities                      │    │
│  │    • Classification into entity types (Person, Location, etc.)   │    │
│  │    • Attribute extraction per entity type                        │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│       │                                                                  │
│       ▼                                                                  │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │ 2. RESOLVE & DEDUPLICATE ENTITIES                                │    │
│  │    • Search existing graph for similar entities                  │    │
│  │    • Embedding similarity + BM25 keyword matching                │    │
│  │    • LLM determines if entities are duplicates                   │    │
│  │    • Merge attributes from duplicates                            │    │
│  │    • UUID aliasing to maintain references                        │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│       │                                                                  │
│       ▼                                                                  │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │ 3. EXTRACT RELATIONSHIPS (EDGES)                                 │    │
│  │    • LLM extracts relationships between entities                 │    │
│  │    • Typed relationships via edge_type_map                       │    │
│  │    • Fact embedding generation                                   │    │
│  │    • Date extraction for temporal bounds                         │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│       │                                                                  │
│       ▼                                                                  │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │ 4. RESOLVE & DEDUPLICATE EDGES                                   │    │
│  │    • Search for semantically similar existing edges              │    │
│  │    • Contradiction detection (invalidation)                      │    │
│  │    • Set invalid_at on contradicting old edges                   │    │
│  │    • Preserve historical records                                 │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│       │                                                                  │
│       ▼                                                                  │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │ 5. PERSIST TO GRAPH DATABASE                                     │    │
│  │    • Save episode, entities, edges in transaction                │    │
│  │    • Create MENTIONS edges (episode → entity)                    │    │
│  │    • Update embeddings and indices                               │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

### 4. Hybrid Search System

Zep implements a sophisticated multi-layer search architecture:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           SEARCH QUERY                                  │
│                              "query"                                    │
└───────────────────────────────────┬─────────────────────────────────────┘
                                    │
                    ┌───────────────┼───────────────┐
                    │               │               │
                    ▼               ▼               ▼
            ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
            │   COSINE     │ │    BM25      │ │    BFS       │
            │  SIMILARITY  │ │  FULLTEXT    │ │  TRAVERSAL   │
            │              │ │              │ │              │
            │ Embedding    │ │ Keyword      │ │ Graph        │
            │ vectors      │ │ matching     │ │ exploration  │
            └──────┬───────┘ └──────┬───────┘ └──────┬───────┘
                   │               │               │
                   └───────────────┼───────────────┘
                                   │
                                   ▼
                    ┌──────────────────────────────┐
                    │         RERANKING            │
                    │                              │
                    │ • Reciprocal Rank Fusion     │
                    │ • Maximal Marginal Relevance │
                    │ • Node Distance              │
                    │ • Episode Mentions           │
                    │ • Cross-Encoder (LLM)        │
                    └──────────────┬───────────────┘
                                   │
                                   ▼
                    ┌──────────────────────────────┐
                    │       SEARCH RESULTS         │
                    │                              │
                    │ • Edges (facts/relations)    │
                    │ • Nodes (entities)           │
                    │ • Episodes (raw content)     │
                    │ • Communities (clusters)     │
                    └──────────────────────────────┘
```

#### Search Scopes

| Scope | What it searches | Best for |
|-------|-----------------|----------|
| **edges** | Facts and relationships | "What is the relationship between X and Y?" |
| **nodes** | Entities and attributes | "Tell me about person X" |
| **episodes** | Raw conversation/data | "What was said about topic X?" |
| **communities** | Clustered summaries | "What topics have been discussed?" |

### 5. LLM Integration

Graphiti supports multiple LLM providers:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        LLM CLIENT LAYER                                 │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐     │
│   │ OpenAI  │  │  Azure  │  │Anthropic│  │ Gemini  │  │  Groq   │     │
│   │         │  │ OpenAI  │  │         │  │         │  │         │     │
│   └────┬────┘  └────┬────┘  └────┬────┘  └────┬────┘  └────┬────┘     │
│        │            │            │            │            │           │
│        └────────────┴────────────┴────────────┴────────────┘           │
│                                  │                                      │
│                                  ▼                                      │
│                    ┌─────────────────────────────┐                      │
│                    │    Unified LLM Interface    │                      │
│                    │                             │                      │
│                    │  • Structured outputs       │                      │
│                    │  • Pydantic validation      │                      │
│                    │  • Concurrent rate limiting │                      │
│                    │  • Model size selection     │                      │
│                    └─────────────────────────────┘                      │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

**LLM Usage in Pipeline:**
- Entity extraction and classification
- Relationship extraction
- Deduplication decisions
- Contradiction detection
- Community summarization
- Cross-encoder reranking

### 6. Database Layer

Multi-backend support with abstraction layer:

| Backend | Type | Use Case |
|---------|------|----------|
| **Neo4j** | Native graph | Production (primary) |
| **FalkorDB** | Redis-based graph | High-performance caching |
| **Kuzu** | Embedded graph | Local/development |
| **Neptune** | AWS managed | Cloud-native deployment |

**Index Strategy:**
```
Range Indices:
├── Entity: (uuid, group_id, name, created_at)
├── Episodic: (uuid, group_id, created_at, valid_at)
└── Edges: (uuid, group_id, name, temporal fields)

Fulltext Indices:
├── node_name_and_summary (Entity nodes)
├── community_name (Communities)
├── episode_content (Episodes)
└── edge_name_and_fact (Relationships)
```

## Zep Platform Features

### 1. Agent Memory

Thread-based conversational memory with automatic context injection:

```python
from zep_cloud.client import Zep

client = Zep(api_key="...")

# Add messages to a thread
client.memory.add(
    session_id="session-123",
    messages=[
        {"role": "user", "content": "My name is Alice"},
        {"role": "assistant", "content": "Hello Alice!"}
    ]
)

# Retrieve context for the agent
context = client.memory.get(session_id="session-123")
```

### 2. Graph RAG

Automatic extraction and maintenance of knowledge graphs:

```python
# Add data to the graph
client.graph.add(
    user_id="user-123",
    type="json",
    data={"name": "Alice", "occupation": "Engineer", "company": "Acme Inc"}
)

# Search the graph
results = client.graph.search(
    user_id="user-123",
    query="Where does Alice work?",
    scope="edges"  # or "nodes", "episodes"
)
```

### 3. Context Assembly

Automatic retrieval and formatting for LLMs:

```python
# Get pre-assembled context
context = client.memory.get_context(
    session_id="session-123",
    max_tokens=4000
)

# Use in your LLM prompt
response = llm.generate(
    system=f"Context: {context}",
    user="What do you know about Alice?"
)
```

## Framework Integrations

### AutoGen Integration

```python
from zep_autogen import ZepUserMemory, ZepGraphMemory

# Thread-based memory
memory = ZepUserMemory(
    session_id="session-123",
    api_key="...",
    context_mode="raw_messages"  # or "summary"
)

# Graph-based memory
graph_memory = ZepGraphMemory(
    user_id="user-123",
    api_key="...",
    search_scope="edges"
)
```

### CrewAI Integration

```python
from zep_crewai import ZepUserStorage, ZepGraphStorage

# Thread storage
storage = ZepUserStorage(
    session_id="session-123",
    api_key="..."
)

# Graph storage with custom ontology
graph_storage = ZepGraphStorage(
    user_id="user-123",
    api_key="...",
    node_labels=["Person", "Organization"],
    search_scope="nodes"
)
```

### LiveKit Integration

```python
from zep_livekit import ZepUserAgent, ZepGraphAgent

# Voice agent with thread memory
agent = ZepUserAgent(
    session_id="voice-session-123",
    api_key="..."
)

# Voice agent with graph memory
graph_agent = ZepGraphAgent(
    user_id="user-123",
    api_key="..."
)
```

## Default Ontology

Zep comes with a default set of entity and relationship types:

### Entity Types

| Type | Description |
|------|-------------|
| **User** | The Zep user (singleton, required) |
| **Assistant** | AI assistant (singleton) |
| **Preference** | User preferences/choices |
| **Location** | Physical or virtual places |
| **Event** | Time-bound activities |
| **Object** | Physical items/tools |
| **Topic** | Subjects of conversation |
| **Organization** | Companies/institutions |
| **Document** | Information content |

### Relationship Types

| Type | Description |
|------|-------------|
| **LOCATED_AT** | Entity at a location |
| **OCCURRED_AT** | Event happened at time/location |
| **PREFERS** | User preference relationship |
| **WORKS_AT** | Employment relationship |
| **KNOWS** | Personal relationship |

### Custom Ontology

```python
from zep_cloud import EntityModel, EntityText, EntityInt, EdgeModel

class Person(EntityModel):
    occupation: EntityText
    age: EntityInt

class Organization(EntityModel):
    industry: EntityText

class WorksAt(EdgeModel):
    role: EntityText
    start_date: EntityText

# Set custom ontology
client.graph.set_entity_types([Person, Organization])
client.graph.set_edge_types([WorksAt])
```

## Performance Characteristics

| Metric | Value |
|--------|-------|
| **Search Latency** | <200ms (typical) |
| **Single-shot Retrieval Accuracy** | ~80% |
| **Entity Extraction** | 1-3s per episode (LLM-bound) |
| **Edge Extraction** | 1-3s per episode |
| **Deduplication** | 0.5-2s |
| **Community Detection** | 10-30s |

## Key Design Patterns

| Pattern | Purpose |
|---------|---------|
| **Async/Await** | Concurrent I/O operations |
| **Semaphore Limiting** | Rate-limit LLM API calls |
| **Bulk Transactions** | Efficient database writes |
| **UUID Aliasing** | Handle deduplication without data loss |
| **Temporal Tracking** | Maintain history via timestamps |
| **Lazy Embedding** | Load embeddings on-demand |
| **Graph Partitioning** | Multi-tenant support via group_id |
| **Bi-Temporal Model** | Separate occurrence vs ingestion time |

## Architecture Summary

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              ZEP ECOSYSTEM                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   OPEN SOURCE                        │   CLOUD PLATFORM                     │
│   ─────────────                      │   ──────────────                     │
│                                      │                                      │
│   Graphiti                           │   Zep Cloud                          │
│   • Temporal KG framework            │   • Managed service                  │
│   • Multi-backend support            │   • SDKs (Python, TS, Go)            │
│   • LLM-driven extraction            │   • Framework integrations           │
│   • Hybrid search                    │   • Enterprise features              │
│   • GitHub: getzep/graphiti          │   • https://www.getzep.com           │
│                                      │                                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   KEY INNOVATIONS                                                           │
│   ───────────────                                                           │
│                                      │                                      │
│   1. Temporal Knowledge Graphs       │   4. Contradiction Detection         │
│      Track how facts change          │      Automatically invalidate        │
│      over time                       │      outdated facts                  │
│                                      │                                      │
│   2. Bi-Temporal Data Model          │   5. Community Detection             │
│      Separate event time from        │      Cluster related entities        │
│      ingestion time                  │      for hierarchical queries        │
│                                      │                                      │
│   3. Hybrid Search                   │   6. Real-time Updates               │
│      Combine embeddings, BM25,       │      Incremental graph updates       │
│      and graph traversal             │      without recomputation           │
│                                      │                                      │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Sources

- [Zep Website](https://www.getzep.com/)
- [Graphiti GitHub Repository](https://github.com/getzep/graphiti)
- [Zep GitHub Repository](https://github.com/getzep/zep)
- [Zep: A Temporal Knowledge Graph Architecture for Agent Memory (arXiv)](https://arxiv.org/abs/2501.13956)
