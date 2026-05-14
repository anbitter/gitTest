# Assessment of T2V API for Generic Text-to-Artefact Platform Capability

---

## 1. Background

This document assesses whether the existing **Text-to-Visualization (T2V) backend API** — originally built to convert natural language queries into SQL and basic visualization instructions — can be reused or evolved into a **generic Text-to-Artefact (T2A) platform capability** serving multiple product teams across the GenAI platform.

**Context:** The GenAI platform requires a governed, multi-tenant service that converts natural language requests into structured artefacts (SQL queries, visualization specs, XML, JSON schemas, etc.) with validation, schema grounding, and observability. Rather than building from scratch, this assessment evaluates the existing T2V API as a candidate starting point.

**Scope of this assessment:**
- Analyze the current T2V API architecture, capabilities, and limitations
- Determine what is reusable, what needs modification, and what requires new design for generic T2A
- Identify key gaps and risks for platform adoption
- Propose a target architecture and phased evolution roadmap
- Surface technical areas requiring further exploration before implementation

**Out of scope:** Deep implementation planning, sprint-level task breakdown, cost modeling, and vendor selection.

---

## 2. Technology Stack

The following major libraries and technologies underpin the current T2V API:

| Technology | Version | Purpose |
|------------|---------|---------|
| **FastAPI** | 0.116.x | Web framework; async REST API with automatic OpenAPI docs |
| **Pydantic** | 2.11.x | Request/response validation, settings management, schema generation |
| **haystack-ai** | 2.17.x | Pipeline orchestration framework; provides `Pipeline`, `Component`, `Document`, `InMemoryDocumentStore`, `InMemoryBM25Retriever`, `TransformersSimilarityRanker`, `ChatPromptBuilder`, `ChatGenerator` |
| **SQLAlchemy** | 2.0.x | Async ORM for product/config/example persistence |
| **aiosqlite** / **asyncpg** | — | Async DB drivers (SQLite for local dev, PostgreSQL for prod) |
| **sqlglot** | 27.x | SQL parsing, AST analysis, dialect transpilation, and validation in the T2V validator |
| **sentence-transformers** | — | Powers `TransformersSimilarityRanker` cross-encoder reranking in RAG pipeline |
| **Jinja2** | — | Prompt template rendering (SQL schema context + user query → LLM prompt) |
| **aiocache** | 0.12.x | TTL-based caching decorator on service methods (config resolution) |
| **cloudevents** | 1.12.x | Pydantic-based CloudEvent envelope for API responses |
| **uvicorn** | — | ASGI server (single-worker only due to in-memory document store) |
| **PyTorch** | 2.8.x | Runtime dependency for transformer-based ranker model inference |
| **Azure OpenAI SDK** / **OpenAI SDK** | — | LLM provider integration via Haystack's `AzureOpenAIChatGenerator` and `OpenAIChatGenerator` |
| **PostgreSQL** | — | Production database for product, config, and example storage |
| **navify Access Control** | — | OIDC token introspection for authentication and role-based authorization |

---

## 3. Summary

- **Current capability:** Production-grade Text-to-SQL + basic visualization instruction API with schema grounding, multi-tenant configuration, and a validated retry-loop pipeline.
- **What it does NOT do:** Execute SQL, produce complex/interactive visuals, support non-SQL artefact types, or support multi-turn conversation.
- **Reuse recommendation:** **Extend via phased evolution.** The API is a strong starting point for governed Text-to-SQL/T2V. It should not be positioned as a generic Text-to-Any-Artefact service today.
- **Key risk:** Fixed output contract, SQL-only validation, in-memory RAG, and missing observability limit immediate platform-wide adoption.
- **Conclusion:** Adopt for T2SQL/T2V with targeted operational fixes (Phase 1), evolve toward enhanced visualization contracts (Phase 2), and evaluate type-agnostic artefact generation only after Phase 2 demand is validated (Phase 3).

---

## 4. Current T2V API Capability

The API provides a **secure, structured, API-contract-driven** way to generate:

| Capability | Description |
|------------|-------------|
| SQL generation | Converts natural language → syntactically valid SQL query grounded against a provided schema |
| Visualization instructions | Returns plot type, axes, and grouping metadata (not rendered visuals) |
| Explanation | Natural language explanation of the generated query logic |
| Simple pipeline | Uses predefined config (schema, dialect, examples) baked into the prompt |
| RAG pipeline | Retrieves examples using keyword-based BM25 search (not embedding-based), then reranks with a cross-encoder model before prompting. Lacks semantic retrieval. |
| Configuration-driven execution | Products register schemas, dialects, and examples via CRUD APIs; pipelines execute against stored config |
| Validation loop | Up to 4 retries: JSON extraction → SQL syntax → destructive-statement detection → column allow-list enforcement |

**Does not provide:** free-text responses, complex visualization rendering, non-SQL query formats, multi-turn conversation, or structure-agnostic metadata models.

---

## 5. High-Level Architecture

```mermaid
flowchart LR
    subgraph Consumers
        ProductApp[Product Application]
    end

    subgraph T2V API Service
        Auth[Auth Guard]
        Routes[API Routes]
        Orchestrator[Pipeline Orchestrator]
    end

    subgraph Pipeline Engine
        Simple[Simple Pipeline]
        RAG[RAG Pipeline]
        Validator[Validation Layer]
    end

    subgraph Data & Config
        DB[(PostgreSQL)]
        DocStore[(In-Memory Doc Store)]
    end

    subgraph External Services
        LLM[LLM Provider]
        Navify[navify Access Control]
    end

    ProductApp --> Auth
    Auth --> Routes
    Routes --> Orchestrator
    Orchestrator --> Simple
    Orchestrator --> RAG
    Simple --> Validator
    RAG --> Validator
    RAG --> DocStore
    Validator --> LLM
    Orchestrator --> DB
    Auth -.-> Navify
```

---

## 6. Client Interaction Lifecycle

### 6.1 Pipeline Execution

1. **Client authenticates** via navify Access Control (OIDC bearer token)
2. **Client sends request** with natural language query to a product-scoped endpoint
3. **API resolves configuration** (schema, examples, dialect) from DB or inline request
4. **Pipeline renders prompt** using Jinja2 templates with schema + examples + query
5. **LLM generates response** (SQL + visualization JSON)
6. **Validator checks response** — retries on failure (up to 4×)
7. **Structured response returned** to client (SQL query + visualization metadata + explanation)

```mermaid
sequenceDiagram
    participant Client as 🖥️ Client / Product App
    participant API as 🔒 T2V API
    participant Config as 🗄️ Config Store (DB)
    participant Pipeline as ⚙️ Pipeline Engine
    participant LLM as 🤖 LLM Provider
    participant Validator as ✅ Validation Layer

    Client->>API: ① POST /products/{product}/text_to_visualization/simple/config/{config}
    API->>API: ② Authenticate & authorize (navify OIDC)
    API->>Config: ③ Resolve product config (schema, examples, dialect)
    Config-->>API: ④ Return config JSON
    API->>Pipeline: ⑤ Execute pipeline with query + config
    Pipeline->>LLM: ⑥ Rendered prompt → chat completion
    LLM-->>Validator: ⑦ Raw LLM response

    loop ⑧ Up to 4 retries
        Validator->>Validator: JSON check → SQL syntax → security → column allow-list
        alt Validation fails
            Validator-->>LLM: Retry with corrective prompt
        end
    end

    Validator-->>API: ⑨ Validated response (LLMResponseClean)
    API->>API: ⑩ Format → TextToVisualizationResponse
    API-->>Client: ⑪ 200 OK — Structured JSON (SQL + visualization + explanation)
```

### 6.2 Configuration Creation

1. **Client authenticates** with product-admin claims
2. **Client sends config** containing SQL schema, dialect, examples, and plot definitions
3. **API validates** the request contract (Pydantic)
4. **Service resolves product** and persists config to database
5. **Config available** for pipeline execution on subsequent requests

```mermaid
sequenceDiagram
    participant Client as 🖥️ Client / Product Admin
    participant API as 🔒 T2V API
    participant Service as ⚙️ ProductConfigService
    participant DB as 🗄️ PostgreSQL

    Client->>API: ① POST /products/{product_name}/config/
    Note right of Client: Body: ProductConfigInput<br/>(name, config JSON with<br/>tables, dialect, examples)
    API->>API: ② Authenticate & authorize (product-admin claims)
    API->>API: ③ Validate request contract (Pydantic v2)
    API->>Service: ④ create(product, data)
    Service->>DB: ⑤ Resolve product by slug
    DB-->>Service: ⑥ Product record
    Service->>DB: ⑦ INSERT product_config (product_id, name, config JSON)
    DB-->>Service: ⑧ Created record
    Service-->>API: ⑨ ProductConfigOutput
    API-->>Client: ⑩ 201 Created — Config registered
```

### 6.3 Request Example Creation (for RAG Pipeline)

1. **Client authenticates** with product-admin claims
2. **Client sends batch of examples** (request/response pairs)
3. **API validates** and persists examples to database
4. **Examples are indexed** into the in-memory document store for RAG retrieval
5. **Examples available** for RAG pipeline on subsequent requests

```mermaid
sequenceDiagram
    participant Client as 🖥️ Client / Product Admin
    participant API as 🔒 T2V API
    participant Service as ⚙️ ProductExampleService
    participant DB as 🗄️ PostgreSQL
    participant DocStore as 📚 In-Memory Document Store

    Client->>API: ① POST /products/{product_name}/example_requests/
    Note right of Client: Body: List of ExampleRequest<br/>(request + structured response)
    API->>API: ② Authenticate & authorize (product-admin claims)
    API->>API: ③ Validate request contract (Pydantic v2)
    API->>Service: ④ create_batch(product, examples)
    Service->>DB: ⑤ Resolve product by slug
    DB-->>Service: ⑥ Product record
    Service->>DB: ⑦ UPSERT product_example records
    DB-->>Service: ⑧ Created records
    Service-->>API: ⑨ List[ExampleRequestOutput]
    API->>DocStore: ⑩ add_examples_to_docs(examples, product)
    Note right of DocStore: Converts to Haystack Documents<br/>with product_slug metadata filter
    DocStore-->>API: ⑪ Indexed for BM25 retrieval
    API-->>Client: ⑫ 201 Created — Examples registered & indexed
```

---

## 7. Component and Module Overview

```mermaid
flowchart LR
    subgraph External["<b>EXTERNAL</b>"]
        Client["<b>Client / Product App</b>"]
        LLM["<b>LLM Provider</b><br/>Azure OpenAI / OpenAI"]
    end

    subgraph API["<b>API LAYER</b>"]
        Auth["<b>Auth & Security</b><br/>navify OIDC"]
        Routes["<b>REST Endpoints</b><br/>FastAPI"]
        Contracts["<b>Request/Response Contracts</b><br/>Pydantic v2"]
    end

    subgraph Engine["<b>CORE ENGINE</b>"]
        ConfigSvc["<b>Product & Config Service</b><br/>CRUD + Caching"]
        PromptPrep["<b>Prompt Preparation</b><br/>Jinja2 Templates"]
        SimplePipe["<b>Simple Pipeline</b>"]
        RAGPipe["<b>RAG Pipeline</b>"]
        DocStore["<b>Example / Document Store</b><br/>In-Memory BM25 + Ranker"]
        Validation["<b>Validation Layer</b><br/>JSON + SQL + Security"]
        Formatter["<b>Response Formatter</b>"]
    end

    subgraph Data["<b>DATA LAYER</b>"]
        DB[("<b>PostgreSQL / SQLite</b><br/>Products, Configs, Examples")]
    end

    Client -->|"① Request"| Auth
    Auth -->|"② Authorize"| Routes
    Routes -->|"③ Validate contract"| Contracts
    Routes -->|"④ Load config"| ConfigSvc
    ConfigSvc -->|"⑤ Query DB"| DB
    Routes -->|"⑥a Simple path"| SimplePipe
    Routes -->|"⑥b RAG path"| RAGPipe
    RAGPipe -->|"⑦ Retrieve examples"| DocStore
    SimplePipe -->|"⑧ Build prompt"| PromptPrep
    RAGPipe -->|"⑧ Build prompt"| PromptPrep
    PromptPrep -->|"⑨ Call LLM"| LLM
    LLM -->|"⑩ Raw response"| Validation
    Validation -->|"⑪ Retry"| LLM
    Validation -->|"⑫ Format"| Formatter
    Formatter -->|"⑬ Return response"| Routes
```

| Component | Role |
|-----------|------|
| **API Layer** | FastAPI endpoints with OIDC auth, request validation, threadpool delegation |
| **Request/Response Contracts** | Fixed Pydantic schemas for request (`TextToInsightsRequest`) and response (`TextToVisualizationResponse`) |
| **Product & Config Service** | Product → Config → Examples hierarchy in PostgreSQL with TTL caching |
| **Prompt Preparation** | Jinja2 templates inject SQL schema, dialect, examples, and query into system/user prompts |
| **Simple Pipeline** | Schema + all config examples → LLM → validator |
| **RAG Pipeline** | BM25 keyword retrieval (top-10) + cross-encoder reranking (top-5) → LLM → validator. No embedding-based ingestion or retrieval — semantically similar but lexically different examples may not be retrieved. |
| **Validation Layer** | JSON parsing, SQL syntax (sqlglot), destructive-statement detection, column allow-list enforcement |
| **LLM Integration** | Config-driven Azure OpenAI / OpenAI-compatible via Haystack ChatGenerator |
| **Response Formatter** | Maps validated LLM output to fixed `TextToVisualizationResponse` contract |

---

## 8. Request Workflow

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

## 9. Suitability for Generic Text-to-Artefact

**Short answer:** Partially suitable as a starting point; not sufficient as-is for generic Text-to-Artefact (T2A).

### What is reusable

The following components are artefact-agnostic and can serve a generic T2A platform without major changes:

- **FastAPI framework + auth infrastructure** — route structure, OIDC token introspection, three-tier claim-based authorization (`valid_admin_user_token`, `valid_product_admin_user_token`, `valid_product_user_token`), and product-audience scoping are not SQL-specific.
- **Multi-tenant config CRUD + caching** — the `Product → ProductConfig → ProductExample` data model, async SQLAlchemy repositories, `@cached(ttl=...)` decorator pattern, and per-key cache invalidation work for any configuration structure.
- **LLM client factory** — `create_chat_generator_from_config()` in `llm_client.py` is a clean config-driven abstraction over Azure OpenAI and OpenAI-compatible APIs. It accepts any `LLMApiConfig` dict and returns a Haystack `Component`. No SQL coupling.
- **Haystack pipeline orchestration pattern** — the pipeline graph structure (prompt → joiner → LLM → validator with retry loop) and `run_in_threadpool` async delegation are reusable for any artefact type.
- **RAG document store infrastructure** — `DocumentStoreService` class-level singleton, `create_documents_for_examples()`, and product-scoped filtering via `create_filter_example()` are structurally reusable, though the retrieval mechanism (BM25) needs replacement.
- **Request middleware** — `GlobalContextMiddleware` (request UUID, latency logging, error catch-all) is fully generic.

### What requires modification

- **Response contract** — `TextToVisualizationResponse` is hardcoded with `DataFetchingSQL` (requires `data_query` string) and `VisualizationInstructionLLM` (requires `plot_type`, `x_axis`, `y_axis`, `group_by`). A T2A platform needs versioned, format-selectable response schemas. The `from_llm_response()` class method pattern is reusable but must be generalized.
- **Validation layer** — `T2VValidator` (`t2v_validator.py`) is a single Haystack `@component` with 5 hardcoded SQL validation steps: (1) JSON extraction, (2) `clean_llm_t2v_response()` plot normalization, (3) `check_valid_sql_statement()` via sqlglot transpile, (4) `security_check_sql()` for destructive statements + column allow-list, (5) `check_uses_table_column()`. Steps 1 is generic; steps 2–5 are entirely SQL-specific. There is no validator interface, no registry, and no way to swap validators per artefact type.
- **Prompt templates** — `create_chat_prompt_builder()` in `prompt.py` supports custom `ChatMessage` templates via the `prompt_type` parameter, but the only predefined prompt is `"sql"` (`PREDEFINED_PROMPTS = {"sql": sql_prompt}`). The API never exposes prompt selection — all routes hardcode `prompt_type="sql"`. The underlying mechanism is extensible; the API integration is not.
- **RAG retrieval** — `InMemoryBM25Retriever` performs keyword-only retrieval. Documents are stored as plain text (`content=example["request"]`) with no embedding vectors. The `TransformersSimilarityRanker` only reranks what BM25 returns, so semantically relevant examples phrased differently are lost. Needs embedding-based ingestion and retrieval.
- **Configuration schema** — `SQLProductConfig` requires `tables` (list of `DBTable` with `table_name`, `columns`, `type`) and `data_fetching` (with `sql_dialect`). Non-SQL products cannot describe their context using this structure.

### What likely needs new design

- **Artefact type registry and task-type routing** — pipeline selection is `Literal["simple_pipeline", "rag_pipeline"]` in `service/text_to_visualization.py`. There is no concept of artefact type, task type, or routing logic. Adding a new artefact type today requires building an entirely parallel vertical slice (new route, new service method, new pipeline factory, new validator, new response schema).
- **Caller-defined output schema** — the caller can only provide input context (query + config). There is no field to specify expected response structure, output format, or validation schema. The response is always the fixed `LLMResponseClean` TypedDict with fields: `data_query`, `explanation`, `plot_type`, `x_axis`, `y_axis`, `group_by`.
- **Multi-turn conversation state** — every request is stateless. There is no session ID, no conversation history storage, and no mechanism to reference prior LLM responses. The pipeline `list_joiner` only maintains history within a single request's retry loop, not across requests.
- **Generic metadata/context model** — the system prompt template (`template_prompt_context` in `sql/prompt.py`) renders SQL table schemas with column descriptions. The user prompt (`template_prompt_request`) asks for SQL dialect output and a fixed JSON response format. Both templates are SQL-specific Jinja2, not parameterizable by artefact type.

### Why current design limits T2A

- **Validator is monolithic and SQL-hardcoded.** `T2VValidator.run()` directly imports and calls `security_check_sql()`, `check_valid_sql_statement()`, `check_uses_table_column()`, and `clean_llm_t2v_response()`. These are not behind an interface — they are inline function calls on specific SQL fields (`t2v_response["data_query"]`, `config_data_fetching["sql_dialect"]`). Replacing them requires rewriting the component, not configuring it.
- **Prompt templates are SQL-locked at the API level.** While the library's `create_chat_prompt_builder()` accepts custom templates, every API route and service constructor passes `prompt_type="sql"` implicitly (it's the default, and no route overrides it). The prompt renders SQL schema tables (`template_prompt_context`) and asks "Write a SQL query" with a fixed JSON output format (`template_prompt_request`).
- **Response types are structurally fixed.** `LLMResponseRaw` is a TypedDict requiring exactly: `data_query`, `explanation`, `plot_type`, `x_axis`, `y_axis`, `group_by`. `TextToVisualizationResponse.from_llm_response()` maps these directly. There is no polymorphism, no format parameter, no content negotiation in any route.
- **~60% of pipeline code is SQL-specific.** The entire `components/data_fetching/sql/` directory (postprocessing, preprocessing, prompt, types), `T2VValidator`, `clean_llm_t2v_response()`, and SQL-specific TypedDicts are not reusable for non-SQL artefacts.
- **No task-type routing exists.** `TPipelines = Literal["simple_pipeline", "rag_pipeline"]` is the only pipeline selector. Both variants produce the same SQL output. There is no artefact-type parameter in any request schema, no factory that selects different validators or prompts based on artefact type.

### Gap Analysis

| # | Area | Current Implementation | T2A Gap | Impact | Recommendation |
|---|------|----------------------|---------|--------|----------------|
| 1 | Output contract | `TextToVisualizationResponse` with fixed `DataFetchingSQL` + `VisualizationInstructionLLM` | No format selection, no versioning, no caller-defined schema | Cannot serve non-SQL consumers without API change | Add artefact-type param, versioned schemas, polymorphic response |
| 2 | Artefact types | `TPipelines = Literal["simple_pipeline", "rag_pipeline"]` — both produce SQL | No XML, JSON schema, free-text, or any non-SQL path | Adding new artefact types requires parallel vertical slices | Add artefact type registry with per-type pipeline factories |
| 3 | Validation | `T2VValidator` with 5 inline SQL checks (`check_valid_sql_statement`, `security_check_sql`, etc.) | No validator interface; cannot swap validation per artefact | Cannot validate XML, JSON Schema, or other formats | Abstract to pluggable validator registry |
| 4 | Visualization schema | Only `plot_type` + `x_axis` / `y_axis` / `group_by` | No colors, controls, interactivity, or open-ended viz structures | Cannot support rich visuals beyond basic chart coordinates | Enhanced visualization contract (Phase 2) |
| 5 | Prompt management | `PREDEFINED_PROMPTS = {"sql": sql_prompt}` — API never exposes prompt selection | Cannot select or configure prompts per artefact type | Prompt changes risk regressions with no evaluation framework | Expose prompt type selection via API; add predefined prompts for new artefact types |
| 6 | Context model | `SQLProductConfig` requires `tables` (DBTable list) + `data_fetching` (sql_dialect) | Non-SQL products cannot provide context in their own format | Fixed metadata structure blocks non-SQL product adoption | Generalize config schema to accept arbitrary metadata structures |
| 7 | Conversation | Stateless; no session ID, no history store, no cross-request context | Cannot support follow-up questions on generated insights | Users cannot refine or iterate on previous results | Implement conversation state manager with external store |
| 8 | RAG retrieval | `InMemoryBM25Retriever` keyword search → `TransformersSimilarityRanker` rerank | No embedding-based semantic retrieval; misses lexically different but relevant examples | Lower quality few-shot examples, reducing LLM accuracy for varied phrasing | Replace with hybrid lexical+embedding retriever and external vector store |
| 9 | RAG storage | `InMemoryDocumentStore` class-level singleton, single-node, lost on restart | Cannot scale horizontally; data lost on process restart | Prevents horizontal scaling; data loss risk on deploy | Externalize to persistent document/vector store |
| 10 | Observability | `GlobalContextMiddleware` with request UUID + latency logging | No OTel tracing, no metrics, no SQL audit trail | Cannot meet platform SLA or compliance requirements | Add OpenTelemetry across all layers |
| 11 | Operational stability | Worker count default > 1 breaks in-memory state; sync auth blocks event loop | Known bugs impact service reliability under load | Service instability under concurrent requests | Immediate fixes required before platform adoption |

---

## 10. Proposed High-Level Architecture for T2A

The diagram below illustrates the target architecture for a generic Text-to-Artefact platform, incorporating the gaps and recommendations identified in this assessment. New or significantly modified components are highlighted.

```mermaid
flowchart LR
    subgraph Consumers["<b>CONSUMERS</b>"]
        ProductApp["<b>Product Application</b>"]
    end

    subgraph T2A["<b>T2A API SERVICE</b>"]
        Auth["<b>Auth Guard</b><br/>Platform IAM"]
        Routes["<b>API Routes</b><br/>Versioned /v1/"]
        ContractResolver["<b>Contract Resolver</b><br/>Artefact Type Registry"]
        ConvoMgr["<b>Conversation Manager</b><br/>Multi-Turn State"]
    end

    subgraph Orchestration["<b>PIPELINE ORCHESTRATION</b>"]
        Router["<b>Pipeline Router</b><br/>Task-Type Dispatch"]
        SimplePipe["<b>Simple Pipeline</b>"]
        RAGPipe["<b>RAG Pipeline</b>"]
    end

    subgraph Prompts["<b>PROMPT ENGINE</b>"]
        PromptRegistry["<b>Prompt Template Registry</b><br/>SQL / XML / JSON / Custom"]
        PromptBuilder["<b>Prompt Builder</b><br/>Jinja2 + Schema Injection"]
    end

    subgraph Validation["<b>VALIDATION FRAMEWORK</b>"]
        ValidatorRegistry["<b>Validator Registry</b><br/>Pluggable Validators"]
        SQLValidator["<b>SQL Validator</b><br/>sqlglot AST"]
        JSONValidator["<b>JSON Schema Validator</b>"]
        CustomValidator["<b>Custom Validator</b><br/>XML / other"]
    end

    subgraph Data["<b>DATA & CONFIG</b>"]
        DB[("<b>PostgreSQL</b><br/>Products, Configs,<br/>Artefact Types")]
        ExtDocStore[("<b>External Doc Store</b><br/>Elasticsearch / OpenSearch")]
        ConvoStore[("<b>Conversation Store</b><br/>Redis / PostgreSQL")]
    end

    subgraph External["<b>EXTERNAL SERVICES</b>"]
        LLM["<b>LLM Provider</b><br/>Azure OpenAI / OpenAI"]
        IAM["<b>Platform IAM</b><br/>Async Auth"]
        OTel["<b>OpenTelemetry</b><br/>Tracing + Metrics"]
    end

    ProductApp -->|"① Request"| Auth
    Auth -.->|"② Verify token"| IAM
    Auth -->|"③ Authorized"| Routes
    Routes -->|"④ Resolve contract"| ContractResolver
    Routes -->|"⑤ Load conversation"| ConvoMgr
    ConvoMgr -->|"⑥ Fetch/store state"| ConvoStore
    ContractResolver -->|"⑦ Route to pipeline"| Router
    Router -->|"⑧a Simple path"| SimplePipe
    Router -->|"⑧b RAG path"| RAGPipe
    Router -->|"⑨ Load config"| DB
    RAGPipe -->|"⑩ Retrieve examples"| ExtDocStore
    SimplePipe -->|"⑪ Select template"| PromptRegistry
    RAGPipe -->|"⑪ Select template"| PromptRegistry
    PromptRegistry -->|"⑫ Build prompt"| PromptBuilder
    PromptBuilder -->|"⑬ Call LLM"| LLM
    LLM -->|"⑭ Raw response"| ValidatorRegistry
    ValidatorRegistry -->|"⑮a"| SQLValidator
    ValidatorRegistry -->|"⑮b"| JSONValidator
    ValidatorRegistry -->|"⑮c"| CustomValidator
    ValidatorRegistry -->|"⑯ Retry"| LLM
    ValidatorRegistry -->|"⑰ Return validated"| Routes
    Auth -.->|"trace"| OTel
    Routes -.->|"trace"| OTel
    Router -.->|"trace"| OTel
    ValidatorRegistry -.->|"trace"| OTel
```

**Key architectural differences from current T2V:**

| Area | Current T2V | Proposed T2A                                                                                                                   |
|------|-------------|--------------------------------------------------------------------------------------------------------------------------------|
| **Artefact routing** | Hardcoded `simple_pipeline` / `rag_pipeline`, both produce SQL | Pipeline Router dispatches based on artefact type (SQL, JSON, XML, custom)                                                     |
| **Prompt management** | Single hardcoded SQL prompt template | Prompt Template Registry allows per-artefact-type and per-product prompt selection                                             |
| **Validation** | Monolithic `T2VValidator` with 5 SQL-specific checks | Pluggable Validator Registry selects the appropriate validator chain per artefact type                                         |
| **Output contract** | Fixed `TextToVisualizationResponse` | Contract Resolver maps artefact type to versioned response schema; supports caller-defined schemas                             |
| **Conversation** | Stateless, single-turn only | Conversation Manager maintains multi-turn state in an external store                                                           |
| **Document store** | In-memory `InMemoryDocumentStore` with BM25 keyword retrieval only (single-node, lost on restart, no semantic search) | External persistent vector store (Elasticsearch/OpenSearch) with semantic+lexical hybrid retrieval at ingestion and query time |
| **Auth** | Synchronous navify OIDC introspection blocking the event loop | Async platform IAM integration with TTL-based token caching                                                                    |
| **Observability** | Request UUID + latency logging only | OpenTelemetry tracing and metrics integrated across all layers                                                                 |
| **Config model** | SQL-specific schema (tables, columns, dialect) | Generic metadata model supporting SQL schemas, XML element definitions, JSON schemas, or free-form context                     |

---

## 11. Recommended Roadmap and Next Steps

### Phased Evolution

| Phase | Focus | Key Changes                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   | Timeline Estimate |
|-------|-------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------|
| **Phase 1** | SQL-only response with schema grounding + basic multi-turn | • Implement conversation memory so T2V becomes stateful for multi-turn interactions<br/>• Move the in-memory RAG document store to an external persistent service<br/>• Implement vector embedding based ingestion/retrieval along with lexical based <br/>• Update authentication and security mechanisms for platform-grade access control<br/>• Add OpenTelemetry-based tracing and metrics for observability<br/>• Make hardcoded values configurable, including model configuration and SQL validation rules<br/>• Conduct performance testing, fix known bugs, and perform code cleanup | 2–3 months |
| **Phase 2** | SQL-only response with enhanced visualization | • Introduce new API contracts that support richer visualization responses, including free-text visualization instructions<br/>• Support more open-ended visualization structures while retaining schema grounding<br/>• Implement new end-to-end simple and RAG-based pipelines tailored for enhanced visualization output                                                                                                                                                                                                                                                                    | 2–3 months |
| **Phase 3** | Type-agnostic response ± schema grounding | • Build an artefact type registry and pluggable validator framework to support non-SQL output types<br/>• Implement new end-to-end pipelines for type-agnostic responses such as XML and JavaScript<br/>• Enable caller-defined output schemas so products can specify their own response structures<br/>• Example use case: a product provides an XML schema with element definitions and optional examples, then resolves generated elements against its own systems                                                                                                                        | 3–4 months |



---

## 12. Areas Requiring Technical Exploration

The following technical areas require targeted evaluation or proof-of-concept before concrete implementation.

| # | Exploration Area | Current State | What Needs Investigation | Expected Outcome |
|---|-----------------|---------------|--------------------------|------------------|
| 1 | **Multi-turn conversation design** — Phase 1 proposes conversation memory, but session semantics, history window size, context carryover strategy (full history vs. summary), and how visualization context chains across turns are undefined. | Stateless; no session ID, no history store. Pipeline `list_joiner` only maintains retry history within a single request. | Define session duration policy, context carryover strategy (append full history vs. LLM-summarized context), memory store choice (Redis vs. PostgreSQL), and how prior SQL/visualization results feed into subsequent turns. | Clear conversation memory contract and storage design before implementation begins. |
| 2 | **Agentic RAG orchestration** — Current RAG is a static retrieve-then-generate pipeline (`BM25 → ranker → prompt → LLM → validator`). Agentic RAG patterns (query decomposition, self-reflection, iterative retrieval, tool-calling) could significantly improve output quality for complex queries. | Single-pass retrieval: `InMemoryBM25Retriever` → `TransformersSimilarityRanker` → fixed prompt. No query decomposition, no self-correction loop beyond validation retries, no ability to retrieve additional context mid-generation. | Evaluate whether an agentic orchestration layer (e.g., Haystack Agent, LangGraph, or custom tool-calling loop) that decomposes complex queries, retrieves context iteratively, and self-corrects before final output improves accuracy for multi-table joins and complex aggregations. | Decision on whether Phase 1 RAG upgrade should be simple embedding swap or full agentic orchestration. |
| 3 | **Embedding model selection for semantic retrieval** — Replacing BM25 with embedding-based retrieval requires choosing an embedding model, defining ingestion pipeline, and evaluating retrieval quality vs. latency tradeoffs. | No embeddings anywhere. Documents stored as plain text with `content=example["request"]`. No embedding generation at ingestion or query time. | Evaluate embedding models (e.g., `sentence-transformers`, OpenAI embeddings, domain-tuned models) for T2V example similarity. Benchmark retrieval recall against current BM25+ranker setup. Determine whether embeddings should be generated locally (adds torch/GPU dependency) or via API (adds latency + cost). | Embedding model choice, ingestion pipeline design, and retrieval quality baseline for Phase 1 RAG externalization. |
| 4 | **Streaming LLM responses** — Current pipeline waits for full LLM completion before validation begins. For enhanced visualization (Phase 2) with longer outputs, streaming could improve perceived latency. | Synchronous full-completion via `run_in_threadpool`. No SSE/WebSocket support. Haystack `ChatGenerator` components support streaming but it is not wired through. | Evaluate whether streaming is compatible with the validation-retry loop (validator needs full response). Explore partial streaming (stream explanation while validating SQL) or streaming only for non-validated endpoints. | Decision on whether streaming is architecturally viable given the validation requirement, and if so, which endpoints benefit. |
| 5 | **Validator abstraction interface design** — Phase 3 requires pluggable validators, but the interface contract (input/output types, retry signaling, error semantics) needs design before any non-SQL validator can be implemented. | `T2VValidator` is a single Haystack `@component` with `@component.output_types(t2v_response=LLMResponseClean, retry_messages=List[ChatMessage])`. Retry signaling is via output connection topology, not an explicit interface. | Design a validator interface that supports: (a) artefact-type-specific validation logic, (b) consistent retry/error signaling, (c) configurable validation strictness, (d) composable validation chains (JSON check → domain-specific check → security check). Determine if this stays as Haystack components or becomes a framework-agnostic abstraction. | Validator interface specification that Phase 2/3 implementations can target. |
| 6 | **Context window budget management** — With large SQL schemas (50+ tables), full schema + examples + query can exceed LLM context limits. Dynamic schema pruning or selective column inclusion is not implemented. | System prompt renders ALL tables and ALL columns from config into context unconditionally (`template_prompt_context` iterates `{% for table in schema %}`). No token counting, no truncation, no prioritization. | Evaluate schema pruning strategies: (a) query-relevant table/column selection via lightweight NLP, (b) token budget allocation across schema/examples/query, (c) multi-step approach where LLM first identifies relevant tables then generates SQL against a focused schema. | Schema management strategy for products with large databases, applicable to both Phase 1 (T2SQL) and Phase 3 (generic context). |

---

## 13. Final Recommendation

| Statement | Detail                                                                                                                                                                                                                                                       |
|-----------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **Strong starting point** | The existing API delivers production-grade, governed Text-to-SQL/Text-to-Visualization with schema grounding, multi-tenant isolation, and validated retry-loop safety. Rebuilding this would take significant initial effort.                                |
| **Not a generic T2A service today** | ~60% of pipeline code is SQL-specific. No pluggable validators, no artefact type routing, no caller-defined output schemas.                                                                                                                                  |
| **Recommended path** | **Phased evolution:** <br/>(1) Stabilize and externalize for platform-grade multi-turn conversation T2SQL/T2V <br/>(2) Introduce enhanced visualization contracts <br/>(3) Evaluate and implement type-agnostic artefact generation as per validated demand. |

---

*Document based on source code analysis of `text-to-visualization-backend-service` v0.10.7 and vendored library v0.11.x. All architectural facts derived from repository. Roadmap and recommendations incorporate platform strategy context.*
