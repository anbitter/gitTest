# Assessment of T2V API for Generic Text-to-Artefact Platform Capability

---

## 1. Executive Summary

- **Current capability:** Production-grade Text-to-SQL + basic visualization instruction API with schema grounding, multi-tenant configuration, and a validated retry-loop pipeline.
- **What it does NOT do:** Execute SQL, produce complex/interactive visuals, support non-SQL artefact types, or support multi-turn conversation.
- **Reuse recommendation:** **Extend via phased evolution.** The API is a strong starting point for governed Text-to-SQL/T2V. It should not be positioned as a generic Text-to-Any-Artefact service today.
- **Key risk:** Fixed output contract, SQL-only validation, in-memory RAG, and missing observability limit immediate platform-wide adoption.
- **Conclusion:** Adopt for T2SQL/T2V with targeted operational fixes (Phase 1), evolve toward enhanced visualization contracts (Phase 2), and evaluate type-agnostic artefact generation only after Phase 2 demand is validated (Phase 3).

---

## 2. Current T2V API Capability

The API provides a **secure, structured, API-contract-driven** way to generate:

| Capability | Description |
|------------|-------------|
| SQL generation | Converts natural language → syntactically valid SQL query grounded against a provided schema |
| Visualization instructions | Returns plot type, axes, and grouping metadata (not rendered visuals) |
| Explanation | Natural language explanation of the generated query logic |
| Simple pipeline | Uses predefined config (schema, dialect, examples) baked into the prompt |
| RAG pipeline | Retrieves and ranks relevant examples from an in-memory document store before prompting |
| Configuration-driven execution | Products register schemas, dialects, and examples via CRUD APIs; pipelines execute against stored config |
| Validation loop | Up to 4 retries: JSON extraction → SQL syntax → destructive-statement detection → column allow-list enforcement |

**Does not provide:** free-text responses, complex visualization rendering, non-SQL query formats, multi-turn conversation, or structure-agnostic metadata models.

---

## 3. Client Interaction Lifecycle

1. **Client authenticates** via navify Access Control (OIDC bearer token)
2. **Client sends request** with natural language query to a product-scoped endpoint
3. **API resolves configuration** (schema, examples, dialect) from DB or inline request
4. **Pipeline renders prompt** using Jinja2 templates with schema + examples + query
5. **LLM generates response** (SQL + visualization JSON)
6. **Validator checks response** — retries on failure (up to 4×)
7. **Structured response returned** to client (SQL query + visualization metadata + explanation)

```mermaid
sequenceDiagram
    participant Client as Client / Product App
    participant API as T2V API
    participant Config as Config Store (DB)
    participant Pipeline as Pipeline Engine
    participant LLM as LLM Provider
    participant Validator as Validation Layer

    Client->>API: POST /products/{product}/text_to_visualization/...
    API->>API: Authenticate & authorize (navify OIDC)
    API->>Config: Resolve product config (schema, examples, dialect)
    API->>Pipeline: Execute pipeline with query + config
    Pipeline->>LLM: Rendered prompt → chat completion
    LLM-->>Validator: Raw LLM response
    loop Up to 4 retries
        Validator->>Validator: JSON check → SQL syntax → security → column allow-list
        alt Validation fails
            Validator-->>LLM: Retry with corrective prompt
        end
    end
    Validator-->>API: Validated response
    API-->>Client: Structured JSON (SQL + visualization + explanation)
```

---

## 4. Component and Module Overview

```mermaid
flowchart TB
    subgraph External
        Client[Client / Product App]
        LLM[LLM Provider<br/>Azure OpenAI / OpenAI]
    end

    subgraph API Layer
        Auth[Auth & Security<br/>navify OIDC]
        Routes[REST Endpoints<br/>FastAPI]
        Contracts[Request/Response Contracts<br/>Pydantic v2]
    end

    subgraph Core Engine
        ConfigSvc[Product & Config Service<br/>CRUD + Caching]
        PromptPrep[Prompt Preparation<br/>Jinja2 Templates]
        SimplePipe[Simple Pipeline]
        RAGPipe[RAG Pipeline]
        DocStore[Example/Document Store<br/>In-Memory BM25 + Ranker]
        Validation[Validation Layer<br/>JSON + SQL + Security]
        Formatter[Response Formatter]
    end

    subgraph Data Layer
        DB[(PostgreSQL / SQLite<br/>Products, Configs, Examples)]
    end

    Client --> Auth --> Routes
    Routes --> Contracts
    Routes --> ConfigSvc
    ConfigSvc --> DB
    Routes --> SimplePipe
    Routes --> RAGPipe
    RAGPipe --> DocStore
    SimplePipe --> PromptPrep
    RAGPipe --> PromptPrep
    PromptPrep --> LLM
    LLM --> Validation
    Validation -->|retry| LLM
    Validation --> Formatter
    Formatter --> Routes
```

| Component | Role |
|-----------|------|
| API layer | FastAPI endpoints with OIDC auth, request validation, threadpool delegation |
| Contracts | Fixed Pydantic schemas for request (`TextToInsightsRequest`) and response (`TextToVisualizationResponse`) |
| Config store | Product → Config → Examples hierarchy in PostgreSQL with TTL caching |
| Prompt preparation | Jinja2 templates inject SQL schema, dialect, examples, and query into system/user prompts |
| Simple pipeline | Schema + all config examples → LLM → validator |
| RAG pipeline | BM25 retrieval + cross-encoder ranking → top-k examples → LLM → validator |
| Validation layer | JSON parsing, SQL syntax (sqlglot), destructive-statement detection, column allow-list enforcement |
| LLM integration | Config-driven Azure OpenAI / OpenAI-compatible via Haystack ChatGenerator |
| Response formatter | Maps validated LLM output to fixed `TextToVisualizationResponse` contract |

---

## 5. High-Level Architecture

```mermaid
architecture-beta
    group client[Consumers]
        service product(server)[Product Application] in client
    end

    group api[T2V API Service]
        service auth(server)[Auth Guard] in api
        service routes(server)[API Routes] in api
        service orchestrator(server)[Pipeline Orchestrator] in api
    end

    group pipelines[Pipeline Engine]
        service simple(server)[Simple Pipeline] in pipelines
        service rag(server)[RAG Pipeline] in pipelines
        service validator(server)[Validation Layer] in pipelines
    end

    group data[Data & Config]
        service db(database)[PostgreSQL] in data
        service docstore(database)[In-Memory Doc Store] in data
    end

    group external[External Services]
        service llm(server)[LLM Provider] in external
        service navify(server)[navify Access Control] in external
    end

    product:R --> L:auth
    auth:R --> L:routes
    routes:B --> T:orchestrator
    orchestrator:R --> L:simple
    orchestrator:R --> L:rag
    simple:R --> L:validator
    rag:R --> L:validator
    rag:B --> T:docstore
    validator:R --> L:llm
    orchestrator:B --> T:db
    auth:B --> T:navify
```

---

## 6. Request Workflow

```mermaid
flowchart TD
    A[Request Received] --> B{Request Contract Valid?}
    B -->|No| Z1[422 Validation Error]
    B -->|Yes| C[Authenticate & Authorize]
    C -->|Fail| Z2[401 Unauthorized]
    C -->|Pass| D[Lookup Product Config]
    D -->|Not Found| Z3[404 Not Found]
    D -->|Found| E[Load Schema, Examples, Dialect]
    E --> F{Select Pipeline}
    F -->|Simple| G[Render Prompt<br/>Schema + Examples + Query]
    F -->|RAG| H[Retrieve & Rank Examples]
    H --> G
    G --> I[Send to LLM]
    I --> J[Receive LLM Response]
    J --> K{JSON Syntax Valid?}
    K -->|No| L{Retries < 4?}
    L -->|Yes| I
    L -->|No| Z4[500 Error]
    K -->|Yes| M{SQL Syntax Valid?}
    M -->|No| L
    M -->|Yes| N{Destructive SQL?}
    N -->|Yes| Z5[500 Immediate Error]
    N -->|No| O{Columns Allowed?}
    O -->|No| L
    O -->|Yes| P[Format Response]
    P --> Q[Return Structured JSON<br/>SQL + Visualization + Explanation]
```

---

## 7. Suitability for Generic Text-to-Artefact

**Short answer:** Partially suitable as a starting point; not sufficient as-is for generic Text-to-Artefact (T2A).

**What is reusable (~30–40% of codebase):**
- FastAPI framework, auth infrastructure, multi-tenant CRUD + caching
- LLM client factory and Haystack pipeline orchestration pattern
- RAG document store infrastructure and configuration-driven context injection

**What requires modification:**
- Response contract must become versioned and format-selectable
- Validation layer must become pluggable (not hardcoded SQL checks)
- Prompt templates must be selectable/configurable per artefact type
- In-memory document store must be externalized for scale

**What likely needs new design:**
- Artefact type registry and task-type routing
- Caller-defined output schema support
- Multi-turn conversation state management
- Generic metadata/context model beyond SQL schema

**Why current design limits T2A:** Every validator, prompt template, and response schema is tightly coupled to SQL output. ~60% of pipeline code is SQL-specific. There is no plugin interface, no task-type concept, and no caller-selectable output format.

| Area | Current Support | T2A Gap | Recommendation |
|------|----------------|---------|----------------|
| Output contract | Fixed SQL + viz JSON | No format selection or versioning | Add format param + versioned schemas |
| Artefact types | SQL only | No XML, JSON schema, or free-text | Add artefact type registry |
| Validation | SQL-specific (sqlglot AST) | No pluggable validator interface | Abstract to validator plugins |
| Prompt management | Hardcoded "sql" prompt type | Cannot select/configure per artefact | Expose prompt selection via API |
| Context model | SQL schema + dialect + examples | No generic metadata structure | Generalize config schema |
| Conversation | Stateless, single-turn only | No multi-turn memory | Implement conversation state |
| RAG store | In-memory, single-node only | Cannot scale horizontally | Externalize to persistent store |
| Observability | Request UUID + latency logging | No OTel, no metrics, no audit | Add OpenTelemetry |

---

## 8. Key Gaps and Risks

| # | Gap / Risk | Impact | Mitigation Path |
|---|-----------|--------|-----------------|
| 1 | **Fixed output contract** — always returns SQL + basic viz | Cannot serve non-SQL consumers without API change | Phased contract evolution (Phase 2–3) |
| 2 | **SQL-only artefact generation** — all validation is SQL-bound | Cannot validate XML, JSON Schema, or other formats | Pluggable validator abstraction |
| 3 | **Limited visualization schema** — only plot type + x/y/group_by | Cannot support rich visuals (colors, controls, interactivity) | Enhanced visualization contract (Phase 2) |
| 4 | **No artefact type abstraction** — no task-type routing | Adding new artefact types requires parallel vertical slices | Type registry + dynamic pipeline assembly |
| 5 | **Fixed metadata structure** — config schema is SQL-specific | Non-SQL products cannot describe their context naturally | Generalized context injection model |
| 6 | **No multi-turn capability** — stateless request/response | Cannot support follow-up questions on generated insights | Conversation memory service (Phase 1) |
| 7 | **In-memory RAG** — single-node, lost on restart | Prevents horizontal scaling; data loss risk | Externalize document store (Phase 1) |
| 8 | **Missing observability** — no OTel, no metrics, no audit trail | Cannot meet platform SLA or compliance requirements | Add OTel tracing + metrics (Phase 1) |
| 9 | **Operational bugs** — worker count default breaks state; sync auth blocks event loop | Service instability under load | Immediate fixes required |

---

## 9. Recommended Roadmap and Next Steps

### Phased Evolution

| Phase | Focus | Key Changes | Timeline Estimate |
|-------|-------|-------------|-------------------|
| **Phase 1** | SQL-only response with schema grounding + basic multi-turn | Conversation memory, externalize RAG, auth/security updates, OTel observability, configurable model/validation, performance testing, bug fixes | 2–3 months |
| **Phase 2** | SQL-only response with enhanced visualization | New API contracts for richer viz (free-text viz response, open-ended structures), new pipelines for enhanced visualization | 2–3 months |
| **Phase 3** | Type-agnostic response ± schema grounding | Artefact type registry, pluggable validators, new pipelines for XML/JSON/other formats, caller-defined output schemas | 3–4 months |

### 30 / 60 / 90-Day Next Steps

| Timeframe | Actions |
|-----------|---------|
| **30 days** | Fix critical operational bugs (worker count, sync auth). Add basic OTel tracing. Confirm ownership and contribution model with T2V team. |
| **60 days** | Externalize RAG document store. Implement conversation memory MVP. Add pipeline test coverage. Deploy as platform service for T2SQL/T2V. |
| **90 days** | Define enhanced visualization contract (Phase 2 design). Evaluate demand for non-SQL artefacts. Begin Phase 2 implementation if demand validated. |

### Management Decisions Needed

1. **Ownership model** — Will the T2V team maintain the service, or will the platform team fork/adopt?
2. **Scope boundary** — Confirm T2SQL/T2V as Phase 1 scope; defer generic T2A until demand is validated.
3. **Investment level** — Approve ~2–3 months of engineering for Phase 1 stabilization and platform readiness.
4. **Infrastructure** — Approve external document store (e.g., Elasticsearch/OpenSearch) for RAG scalability.
5. **Observability standards** — Confirm OTel + Prometheus as platform observability stack.

### Dependencies and Risks

| Dependency / Risk | Mitigation |
|-------------------|------------|
| T2V team cooperation for contribution model | Early engagement; define clear ownership boundaries |
| LLM provider availability and cost | Multi-provider support already exists; monitor usage |
| In-memory RAG migration complexity | Haystack supports pluggable document stores; migration path exists |
| Demand uncertainty for Phase 2–3 | Gate investment on validated consumer demand |
| Single-worker scaling ceiling | Acceptable for Phase 1 (<10 tenants); externalized RAG resolves for Phase 2+ |

---

## 10. Final Recommendation

| Statement | Detail |
|-----------|--------|
| **Strong starting point** | The API delivers production-grade, governed Text-to-SQL/Text-to-Visualization with schema grounding, multi-tenant isolation, and validated retry-loop safety. Rebuilding this would take significant effort. |
| **Not a generic T2A service today** | ~60% of pipeline code is SQL-specific. No pluggable validators, no artefact type routing, no caller-defined output schemas. Positioning it as generic T2A without modification would fail. |
| **Recommended path** | **Phased evolution:** (1) Stabilize and externalize for platform-grade T2SQL/T2V, (2) Introduce enhanced visualization contracts, (3) Evaluate and implement type-agnostic artefact generation only when demand is validated. |

---

*Document based on source code analysis of `text-to-visualization-backend-service` v0.10.7 and vendored library v0.11.x. All architectural facts derived from repository. Roadmap and recommendations incorporate platform strategy context.*

