# LabGPT — Text-to-Visualization Workspace Analysis

> Generated: 2026-04-28
> Scope: the three sibling projects under `labgpt-t2v/`.

---

## 1. Workspace Summary

This workspace implements a complete **"natural language → chart"** stack for Roche's *LabGPT / Text-to-Data-Insights* initiative. A user types a question in English, and the system answers with a rendered visualization (table, line, bar, scatter, or pie plot) backed by data the LLM was allowed to query.

The stack is split into three deliberately decoupled deliverables, each published as its own artifact:

| # | Project | Language | Artifact | Role |
|---|---------|----------|----------|------|
| 1 | `text-to-visualization` | Python 3.11 | PyPI / GitLab wheel | **Core LLM pipeline library** (Haystack-based). Builds the prompt, calls the LLM, validates the output, and returns structured visualization instructions. |
| 2 | `text-to-visualization-backend-service` | Python 3.11 / FastAPI | Docker image | **HTTP service** that wraps the library, adds multi-tenant config management, RAG retrieval of example requests, OIDC authentication, and persistence (SQLite/SQLAlchemy + Haystack Document Store). |
| 3 | `text-to-visualization-react-lib` | TypeScript / React 18 | npm package | **Front-end component library** that consumes the backend's JSON response and renders it with Recharts (`TdiGraph`) or `@tanstack/react-table` (`TdiTable`). |

Together they form a clean three-tier pipeline: **React UI → FastAPI service → Python LLM library → LLM provider (Azure OpenAI / navify Alchemy / Anthropic Claude via OpenAI-compatible gateway)**.

---

## 2. High-Level Architecture

```mermaid
flowchart LR
    subgraph Host["Host Web App (Roche product, e.g. nacore)"]
        UI[User Input: NL question]
        REACT["@llmsquad_ris_dna/text-to-visualization-react<br/>(TdiTextToVisualization, TdiGraph, TdiTable)"]
    end

    subgraph Backend["text-to-visualization-backend-service (FastAPI, Docker)"]
        AUTH[OIDC Auth Middleware<br/>navify Access Control]
        ROUTES[/API Routes<br/>/products /admin /health/]
        SVC[TextToVisualizationService]
        REPO[(SQLAlchemy Repo<br/>Products • Configs • Examples)]
        DS[(Haystack Document Store<br/>Embeddings)]
    end

    subgraph Lib["text-to-visualization (Python lib, Haystack)"]
        PIPE[simple_pipeline / rag_pipeline]
        PROMPT[Prompt Builder<br/>SQL + Viz template]
        LLMC[LLM ChatGenerator<br/>OpenAI/Azure client]
        VAL[T2VValidator<br/>retry loop]
        POST[Visualization<br/>Postprocessing]
    end

    LLM[(LLM Provider<br/>Galileo • navify Alchemy<br/>GPT-4 • Claude 3.7)]
    DB[(SQLite / aiosqlite)]

    UI --> REACT
    REACT -- HTTPS POST /products/{p}/text_to_visualization/* --> AUTH
    AUTH --> ROUTES
    ROUTES --> SVC
    SVC <--> REPO
    SVC <--> DS
    REPO --- DB
    SVC --> PIPE
    PIPE --> PROMPT --> LLMC
    LLMC <--> LLM
    LLMC --> VAL
    VAL -.retry.-> PIPE
    VAL --> POST
    POST --> SVC
    SVC -- TextToVisualizationResponse JSON --> REACT
    REACT --> UI
```

---

## 3. Per-Project Analysis

### 3.1 `text-to-visualization` — Core Python Library

**Short summary.** A Haystack-AI-based Python library that turns a natural-language query plus a data-source schema/config into a structured visualization instruction (chosen plot type, axes, group-by, and the SQL/data-fetch spec). It is provider-agnostic over the LLM (anything OpenAI-compatible: Azure OpenAI/Galileo, navify Alchemy gateway hosting Claude 3.7, etc.) and produces output in a schema mirrored by the React lib.

**Purpose & functionality**
- Build prompts (system + user) from a product config (DB schema, allowed columns, examples).
- Call an LLM via `haystack` `ChatGenerator` to produce **(a)** a SQL/data-fetch instruction and **(b)** a visualization spec.
- **Validate** the LLM output (`T2VValidator`) — checks JSON structure, plot type, axes/columns consistency, SQL parseability via `sqlglot` — and **retry** the LLM with an error message in the conversation if invalid.
- Provide both a **simple pipeline** (all examples baked into the prompt) and a **RAG pipeline** (top-k retrieved examples).
- Optional **rendering** via `plotly` for notebook/standalone use (`components/rendering/`).
- Mock helpers (`mock/`) for offline development without an LLM.

**Tech stack**
- Python 3.11–3.12, Pydantic 2, pandas, numpy.
- `haystack-ai` 2.17 (Pipelines, ChatGenerator, ListJoiner).
- `openai` 1.98+ client (used for both OpenAI and Azure / navify-Alchemy gateway).
- `sqlglot` for SQL validation, `transformers` + `torch` (CPU) for embeddings/ranking in the RAG path.
- `plotly` for optional Python-side rendering.

**Key modules** (`src/text_to_visualization/`)

| Module | Responsibility |
|---|---|
| `pipelines/simple_pipeline.py` | `create_simple_pipeline`, `create_simple_pipeline_with_validation` (Prompt → ListJoiner ↔ LLM → T2VValidator with retry loop). |
| `pipelines/rag_pipeline.py` | RAG variant: Retriever → Ranker → Prompt → LLM → Validator. |
| `pipelines/components/prompt.py` | Jinja2 chat-prompt builder (`sql` template). |
| `pipelines/components/llm_client.py` | `create_chat_generator_from_config`, `LLMApiConfig` (azure/openai). |
| `pipelines/components/t2v_validator.py` | Output-validation Haystack component with retry-message emission. |
| `components/data_fetching/sql/` | SQL generation/validation helpers (`sqlglot`). |
| `components/visualization/postprocessing.py` | Cleans LLM viz spec into `LLMResponseVisClean`. |
| `components/llm_agent/llm_response.py` | LLM response parsing models. |
| `components/document_store/` | RAG document store (in-memory / persisted) for example requests. |
| `components/rendering/` | Optional Plotly rendering for notebooks. |
| `types.py`, `constants.py` | `PlotType` enum and response types — **mirrored 1:1 in the React lib**. |

**Integration points**
- Consumed directly by the backend service (`TextToVisualizationService`).
- Its response schema is the authoritative contract — `lib/types.ts` in the React lib annotates each TS type with the Python class it mirrors.

---

### 3.2 `text-to-visualization-backend-service` — FastAPI Service

**Short summary.** A FastAPI service that exposes the core library over HTTPS for multiple "products" (tenants). It manages product/config lifecycle, stores RAG example requests, authenticates via navify OIDC, caches product configs, and orchestrates either a *simple* or *RAG* pipeline per request.

**Purpose & functionality**
- Multi-tenant **product & config CRUD** (`/admin/product`, `/admin/config`, `/products/{p}/config/...`).
- Two text-to-visualization endpoints per product:
  - `POST /products/{p}/text_to_visualization/simple/config/{config_name}` — simple pipeline against a stored config.
  - `POST /products/{p}/text_to_visualization/rag/config/{config_name}` — RAG pipeline with `retriever_top_k` / `ranker_top_k` query params.
  - `POST /products/{p}/text_to_visualization/simple` — same, but the config is **sent in the request body** (dynamic).
  - `POST /products/{p}/text_to_visualization/simple/rse` — CloudEvent-wrapped variant of the above.
  - `POST /products/{p}/text_to_visualization/clear_cache/{config_name}` — drops the in-process config cache.
- **Request examples** management for the RAG pipeline: `/products/{p}/example_requests/...` (CRUD).
- **Auth** via navify Access Control OIDC (client-credentials *or* authorization-code flow), aud-claim filtering (`AUTH_AUD`).
- **Health probes** (`/health/ready`, `/health/live`) for k8s-style deployment.
- **Persistence**: `SQLAlchemy` async (default `sqlite+aiosqlite:///database.db`) for products/configs/examples; Haystack `DocumentStoreService` for embeddings.
- **Caching**: per-config in-process cache (`CACHE_PRODUCT_CONFIG` seconds).
- Auto-creates DB tables and seeds initial products on FastAPI `lifespan` startup.

**Tech stack**
- FastAPI, Pydantic v2 settings.
- SQLAlchemy (async) + aiosqlite.
- The `text-to-visualization` library (installed from Roche GitLab via Poetry HTTP-basic auth).
- Haystack document store (in-memory or persisted) for RAG.
- Docker (multi-stage: builder + production with Roche CA certs baked in), Docker Compose for dev/test/deployment.

**Key modules** (`src/text_to_visualization_api/`)

| Module | Responsibility |
|---|---|
| `main.py` | FastAPI app factory, `lifespan` (DB init + DocumentStore startup + initial products), middleware. |
| `api/main.py` + `api/routes/` | Router aggregation: `health`, `debug`, `admin/`, `products/`. |
| `api/routes/products/text_to_visualization.py` | The five t2v endpoints (see above). |
| `api/routes/products/config.py` | Per-product config CRUD. |
| `api/routes/products/request_examples.py` | RAG examples CRUD. |
| `api/routes/admin/` | Cross-tenant product/config administration. |
| `service/text_to_visualization.py` | `TextToVisualizationService` — wraps the library pipeline, applies caching, fetches config from repo, calls `get_llm_response`. |
| `service/document_store.py` | Lifecycle + writes/reads against the Haystack store for RAG examples. |
| `repository/` | Async SQLAlchemy repositories (Product, Config, ExampleRequest). |
| `models/`, `schemas/` | SQLAlchemy ORM models + Pydantic request/response schemas. |
| `auth/` | OIDC token validation, claim → role mapping (user / product-admin / admin). |
| `core/settings.py`, `core/database.py` | Pydantic settings + async engine/session. |
| `utils/middleware.py` | `GlobalContextMiddleware` (correlation/context propagation). |

**Integration points**
- **Upstream**: imports `text_to_visualization` (the library above) and instantiates its `simple_pipeline` / `rag_pipeline`.
- **Downstream**: returns `TextToVisualizationResponse` JSON consumed by the React lib (`TDIResponse` type).
- **External**: navify Access Control (OIDC), Galileo / navify Alchemy LLM gateways.

---

### 3.3 `text-to-visualization-react-lib` — React Component Library

**Short summary.** A small Vite-built React/TypeScript component library, published as `@llmsquad_ris_dna/text-to-visualization-react`, that renders the backend's response. It is **stateless / transport-agnostic** — the host application performs the HTTP call and passes the resulting `TDIResponse` object as a prop.

**Purpose & functionality**
- Provide drop-in React components that turn a `TDIResponse` into UI:
  - `TdiTextToVisualization` — the umbrella component; switches on `response.type` / `response.visualization.plot_type` to render either an error message, a table, or a chart. Handles `status` of `idle | pending | error | success` with overridable placeholder slots.
  - `TdiGraph` — Recharts-based renderer for `line plot | bar plot | scatter plot | pie plot` (mirrors `text_to_visualization.constants.PlotType`).
  - `TdiTable` — `@tanstack/react-table`-based table renderer for `TDIRecordsData`.
  - `placeholders/` (`StatusPending`, `ErrorDataLoad`, `NotEnoughData`) with bundled SVG assets.
  - `ui/` shadcn-style primitives (Tailwind + `class-variance-authority`).
- Ship with **Storybook** stories (`lib/stories/`, `*.stories.tsx`) and a published Storybook site for documentation.

**Tech stack**
- React 18, TypeScript 5.5, Vite 5 (lib mode, `vite-plugin-dts`, CSS-injected-by-JS).
- Tailwind CSS 4, shadcn-style primitives (`class-variance-authority`, `clsx`, `tailwind-merge`).
- **Recharts 2.15** for charts (note: README mentions Plotly but the actual implementation uses Recharts — Plotly is only used on the Python side).
- `@tanstack/react-table` 8 for tables, `lucide-react` for icons.
- Storybook 8.1 for component docs.

**Key components / files** (`lib/`)

| File | Responsibility |
|---|---|
| `main.tsx` | Library entry — re-exports public components, types, and CSS. |
| `components/index.tsx` | Exports `TdiTable`, `TdiGraph`, `TdiTextToVisualization`. |
| `components/TdiTextToVisualization/TdiTextToVisualization.tsx` | Top-level dispatcher — picks renderer based on `status` + `response`. |
| `components/TdiGraph/` | Recharts wrapper, `graph.layouts.tsx` for default layout. |
| `components/TdiTable/` | TanStack-Table wrapper. |
| `components/placeholders/` | Pending / error / no-data states. |
| `components/ui/` | Reusable shadcn primitives. |
| `types.ts` | `TDIResponse`, `TDIPlotType`, `TDIRecordsData` — **explicitly mirrors the Python types**. |
| `constants.ts`, `plot_stylesheet.ts` | Shared constants and chart styling. |
| `tests/dataExampleRequests.ts`, `tests/dataExamplePlots.ts` | Sample request/response fixtures. |

**Integration points**
- **Input contract**: consumes the JSON returned by the backend's `/text_to_visualization/...` endpoints (`TDIResponse` matches the backend's `TextToVisualizationResponse`).
- **Out of scope**: the lib does **not** make the HTTP call or handle auth — that is the host app's responsibility. This keeps it embeddable in any tenant product.

---

## 4. End-to-End Request Trace

```mermaid
sequenceDiagram
    autonumber
    actor User
    participant App as Host Web App
    participant React as TdiTextToVisualization (React lib)
    participant API as FastAPI Service
    participant Auth as OIDC (navify AC)
    participant DB as SQLAlchemy DB
    participant DS as Haystack Document Store
    participant Lib as text_to_visualization (Haystack Pipeline)
    participant Val as T2VValidator
    participant LLM as LLM Provider

    User->>App: types "Show monthly sample counts in 2025"
    App->>App: build TextToInsightsRequest, attach OIDC token
    App->>API: POST /products/nacore/text_to_visualization/rag/config/full_db
    API->>Auth: validate Bearer token (aud, claims)
    Auth-->>API: user identity + roles
    API->>DB: load Product + Config (cached up to CACHE_PRODUCT_CONFIG s)
    DB-->>API: schema, examples, llm allow-list
    API->>Lib: TextToVisualizationService.get_llm_response(request)

    rect rgba(220,235,255,0.5)
        Note over Lib,DS: RAG branch only
        Lib->>DS: embed query, retrieve top-10 example requests
        DS-->>Lib: candidate examples
        Lib->>Lib: Ranker → top-5
    end

    Lib->>Lib: PromptBuilder(query, schema, examples)
    Lib->>LLM: ChatGenerator.run(messages)
    LLM-->>Lib: assistant reply (JSON: {sql, visualization})

    loop until valid OR max_runs reached
        Lib->>Val: validate(reply, history)
        alt invalid (bad JSON / unknown column / SQL parse fail)
            Val-->>Lib: retry_messages (error feedback)
            Lib->>LLM: ChatGenerator.run(history + retry msg)
            LLM-->>Lib: corrected reply
        else valid
            Val-->>Lib: clean LLMResponseVisClean
        end
    end

    Lib->>Lib: postprocessing → TextToVisualizationResponse
    Lib-->>API: response (data + visualization)
    API-->>App: 200 OK, JSON TDIResponse
    App->>React: <TdiTextToVisualization response={...} status="success" />
    React->>React: switch(plot_type) → TdiGraph | TdiTable | error
    React-->>User: rendered chart / table
```

> **Note on `data`.** The current pipeline emits *visualization instructions plus the data fetch spec*; whether the backend executes that SQL itself or expects the host app to run it depends on the product config (`config_data_fetching`). The React lib simply consumes whatever rows the backend returns in `response.data.records`.

---

## 5. Python Library Component View

```mermaid
classDiagram
    class Pipeline {
        +run(inputs) dict
    }
    class PromptBuilder {
        +template: ChatMessage[]
        +run(query, schema, examples) messages
    }
    class ChatGenerator {
        +api_config: LLMApiConfig
        +run(messages) replies
    }
    class T2VValidator {
        +max_retries
        +run(replies, history) (clean | retry_messages)
    }
    class Retriever {
        +top_k
        +run(query) docs
    }
    class Ranker {
        +top_k
        +run(query, docs) ranked_docs
    }
    class DocumentStore {
        +write(examples)
        +query(embedding) docs
    }
    class LLMResponseVisClean {
        +plot_type
        +x_axis
        +y_axis
        +group_by
    }
    class TextToVisualizationResponse {
        +type
        +data: TDIRecordsData
        +visualization: LLMResponseVisClean
    }
    class SqlValidator {
        +parse(sql) ast
        +check_columns(schema)
    }

    Pipeline --> PromptBuilder
    Pipeline --> ChatGenerator
    Pipeline --> T2VValidator
    Pipeline --> Retriever : RAG only
    Pipeline --> Ranker : RAG only
    Retriever --> DocumentStore
    T2VValidator --> SqlValidator
    T2VValidator --> LLMResponseVisClean
    LLMResponseVisClean <.. TextToVisualizationResponse
```

---

## 6. Deployment View (Backend Service)

```mermaid
flowchart TB
    subgraph Build["Docker multi-stage build"]
        B1[python:3.11-slim<br/>builder]
        B2[alpine + Roche CA certs]
        B3[python:3.11-slim<br/>production]
        B1 -- /app/.venv + src --> B3
        B2 -- ca-certificates.crt --> B3
    end

    subgraph Runtime["Container at runtime (USER appuser, EXPOSE 8000)"]
        EP[entrypoint.sh<br/>uvicorn / gunicorn]
        APP[FastAPI app<br/>text_to_visualization_api.main:app]
        DBV[(database.db<br/>volume-mounted)]
        CFG[(galileo.gpt4_8k_api_configs.json<br/>volume-mounted)]
    end

    subgraph Compose["docker-compose.yml env"]
        E1[DATABASE_URI]
        E2[LLM_API_CONFIG]
        E3[LLM_API_AVAILABLE_MODELS]
        E4[CACHE_PRODUCT_CONFIG=30]
        E5[AUTH_OIDC_CLIENT_ID/SECRET]
        E6[AUTH_URL_API • AUTH_APP_ALIAS • AUTH_AUD]
    end

    subgraph External["External services"]
        OIDC[(navify Access Control<br/>OIDC)]
        LLMP[(Galileo / navify Alchemy<br/>LLM gateway)]
    end

    Compose --> Runtime
    EP --> APP
    APP <--> DBV
    APP --> CFG
    APP <--> OIDC
    APP <--> LLMP

    Build --> Runtime
```

Key deployment facts (from `Dockerfile`, `docker-compose.yml`, `entrypoint.sh`):
- Multi-stage build; final image runs as **non-root** `appuser` (uid 1001) on **port 8000**.
- Roche internal CA bundle is baked in (`REQUESTS_CA_BUNDLE`, `SSL_CERT_FILE`).
- Library installed from Roche GitLab via Poetry HTTP-basic with a build secret (`GITLAB_SECRET`).
- Default DB is **SQLite via aiosqlite**; can be swapped via `DATABASE_URI`.
- LLM provider is selected by mounting an `LLM_API_CONFIG` JSON (Azure/Galileo or navify Alchemy).
- Several compose overlays exist: `docker-compose.dev.yml`, `docker-compose.test.yml`, `docker-compose.deployment-vm.yml`, `docker-compose.build.yml`, `docker-compose.overwrite.yml` — for local dev, CI, VM deployment, and image build respectively.

---

## 7. Per-Project One-Liner Summaries

- **`text-to-visualization`** — Haystack-based Python library that converts a natural-language question + a data-source config into a validated visualization spec (plot type, axes, SQL) by orchestrating prompt → LLM → validator (with retry) and an optional RAG branch.
- **`text-to-visualization-backend-service`** — FastAPI/Docker service that exposes the library to multiple Roche products with OIDC auth, multi-tenant product/config CRUD, RAG example management, async SQLAlchemy persistence, and caching.
- **`text-to-visualization-react-lib`** — Stateless React/TypeScript component library (`TdiTextToVisualization`, `TdiGraph`, `TdiTable`) that renders the backend's JSON response using Recharts and TanStack Table, ready to be embedded in any host app.

---

## 8. Workspace Takeaways

1. **Clean three-tier separation.** Each layer can be developed, versioned, and released independently. The library has its own version (`0.11.6`), the backend its own Docker image, the React lib its own npm package (`0.7.1`).
2. **Single source of truth for the response schema.** The Python `LLMResponseVisClean` / `TextToVisualizationResponse` types are mirrored verbatim in `lib/types.ts` (`TDIResponse`, `TDIPlotType`). Keeping these in sync is the most important cross-repo contract.
3. **LLM-provider agnosticism.** Both Azure (Galileo) and OpenAI-compatible gateways (navify Alchemy → Claude 3.7) are supported by the same `LLMApiConfig`/`ChatGenerator`. The provider is a runtime config, not a code change.
4. **Robustness via validate-and-retry.** `T2VValidator` plus the Haystack `ListJoiner` self-loop is the key trick that keeps LLM hallucinations from breaking the contract. SQL is parsed with `sqlglot` before being trusted.
5. **Multi-tenancy is built-in.** "Products" and "configs" are first-class concepts in the backend; per-config caching (`CACHE_PRODUCT_CONFIG`) and a manual `clear_cache` endpoint let operators tune freshness without restarts.
6. **React lib is intentionally transport-free.** It receives `response` as a prop — host apps own the HTTP call, auth, and any caching/streaming. This lets it be reused inside very different Roche product shells.
7. **Roche-internal infrastructure is assumed.** GitLab package registry, navify Access Control, internal CA bundle, and the Galileo LLM endpoint are all baked into the build/deploy. Outside Roche, configs would need to be substituted.

### Items worth confirming with the team
- Whether the backend executes the generated SQL itself or only emits the data-fetch instruction for the host app to run — both patterns appear plausible from the code; the truth lives in `service/text_to_visualization.py` and the per-product `config_data_fetching`.
- The README of the React lib mentions `TdiPlotlyVisualization`, but the codebase ships `TdiGraph` (Recharts) instead — the README is out-of-date.

