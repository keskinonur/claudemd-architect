# CLAUDE.md — Bellek (bellek.ai)

> **M365 Backup & RAG Platform**
> *Beka korudu. Bellek hem koruyor, hem hatırlıyor, hem buluyor.*

> **Reading guide for AI assistants:** This document is the single source of truth for all architectural decisions. When in doubt, re-read the relevant ADR before writing code. Do NOT invent alternative approaches — every major choice has been deliberately made and documented with rationale. If you find yourself unsure, ask the developer rather than guessing.

---

## Project Vision

A self-hosted, open-source Microsoft 365 backup and restore tool with an integrated RAG (Retrieval-Augmented Generation) layer for enterprise search and AI agent access. Built as a single Go binary with an HTMX-based web UI, designed for NAS deployment with zero recurring license costs.

**Three layered capabilities (each builds on the previous, independently toggleable):**

1. **M365 Backup & Restore** — Core mission: reliable, metadata-preserving backup with faithful restoration.
2. **Enterprise Search** — Semantic search over all backed-up content via locally-running embedding models.
3. **AI-Ready Data Layer** — REST API endpoints enabling AI agents to query the corporate knowledge base.

**Core Principle:**
> "Backup's golden rule is restore capability." Every design decision prioritizes faithful, verifiable restoration over feature count.

---

## Critical Constraints (Read First)

These are non-negotiable rules. Violating any of these means the implementation is wrong, regardless of how elegant or clever the code appears.

### MUST Rules
- **MUST** use Go as the only backend language. No Python scripts, no Node.js sidecars, no shell script business logic.
- **MUST** use explicit error handling. Every function that can fail returns an error. Every returned error is checked. Never use `_` to discard errors in production code paths.
- **MUST** preserve all metadata during backup. Losing an email's read/unread flag or a file's modified timestamp is a bug, not a tradeoff.
- **MUST** keep backup.db and vectors.db as separate SQLite files. The backup database is sacred — RAG indexing failures must never corrupt backup metadata.
- **MUST** use interfaces for all boundary components (Embedder, VectorStore, TextExtractor, EventBus). This is not optional "nice to have" — it's required for the model-swap and deployment-flexibility architecture.
- **MUST** handle Graph API throttling (HTTP 429) with exponential backoff respecting the `Retry-After` header. Never retry immediately. Never ignore 429 responses.

### MUST NOT Rules
- **MUST NOT** add React, Vue, Svelte, or any JavaScript framework. The UI is HTMX + Go templates + Tabler CSS. Vanilla JS only for charts and confirmation dialogs.
- **MUST NOT** add PostgreSQL, MySQL, Redis, or any external database. SQLite only.
- **MUST NOT** send any corporate data to external APIs for embedding. All embedding is local via Ollama. This is a KVKK (Turkish GDPR) compliance requirement.
- **MUST NOT** create separate microservices. This is a single Go binary that embeds all functionality. Ollama and Tika are the only allowed external services.
- **MUST NOT** implement features from later phases while working on an earlier phase. Phase 1 must be solid before Phase 2 begins. Premature abstraction for future phases is acceptable only via interfaces — not via actual implementation.
- **MUST NOT** use `panic()` or `log.Fatal()` in library code. These are only acceptable in `main()` for startup failures.

### Verification Checkpoints
Before considering any phase complete, verify:
- [ ] All error paths are handled (no `_` discards, no unhandled errors)
- [ ] All interfaces are satisfied (compile-time checks via `var _ Interface = (*Impl)(nil)`)
- [ ] Tests exist for the happy path AND at least two error/edge cases per public function
- [ ] The binary still compiles and runs as a single executable
- [ ] No new external service dependencies were introduced

---

## Architecture Decisions (ADR Log)

These decisions were made through deliberate discussion and represent final consensus. Each ADR includes the rationale to prevent reasoning drift — if you understand WHY a decision was made, you are less likely to accidentally undermine it.

### ADR-001: Language — Go

**Decision:** Go for the entire application.

**Why (memorize this — it prevents "wouldn't it be better in Python?" drift):**
- `if err != nil` forces handling every failure — critical for backup where silent data loss is catastrophic.
- Goroutines + channels = natural concurrency model for parallel user backups with throttle control.
- Single static binary (~15-20MB), zero runtime dependencies, runs on any Linux including NAS devices.
- Predictable memory usage for 24/7 daemon operation (no GC pauses like V8, no GIL like Python).
- Microsoft Graph SDK exists: `github.com/microsoftgraph/msgraph-sdk-go`.

**Rejected:** Python (GIL, runtime dep, implicit errors), Bun/TypeScript (too young for infra, V8 GC).

### ADR-002: Web UI — HTMX + Go Templates + Tabler

**Decision:** Server-rendered HTML. All assets embedded via `go:embed`. PocketBase-style single binary.

**Implementation details (be specific to prevent ambiguity):**
- Template engine: Go `html/template` or [Templ](https://templ.guide/) for type-safe templates.
- CSS framework: [Tabler](https://tabler.io/) (Bootstrap 5-based admin UI kit).
- Dynamic behavior: HTMX attributes on server-rendered HTML. No client-side routing.
- Embedding: `//go:embed ui/static/*` and `//go:embed ui/templates/*` in an `embed.go` file.
- Charts (if needed): A lightweight vanilla JS library like Chart.js loaded from embedded static files.

**Concrete example of how a page works:**
```go
// server/pages.go — every page handler follows this exact pattern
func (s *Server) handleDashboard(w http.ResponseWriter, r *http.Request) {
    jobs, err := s.store.RecentJobs(r.Context(), 10)
    if err != nil {
        s.renderError(w, r, err) // NEVER swallow this error
        return
    }
    s.render(w, r, "dashboard.html", map[string]any{
        "Jobs": jobs,
        "Stats": s.backup.Stats(),
    })
}
```
```html
<!-- templates/dashboard.html -->
<div hx-ext="sse" sse-connect="/events" sse-swap="backup-progress">
    <!-- This div auto-updates when SSE sends "backup-progress" events -->
    <p>Waiting for backup events...</p>
</div>
<button hx-post="/api/backup/start" hx-swap="none">Start Backup</button>
```

### ADR-003: Real-time — SSE (not WebSocket)

**Decision:** Server-Sent Events as the only real-time channel. WebSocket deferred indefinitely.

**Why SSE wins for this project:**
- Backup monitoring is **one-way** (server → client). The dashboard watches, it doesn't talk back.
- All user actions (start backup, trigger restore, save settings) are normal HTTP POSTs — no bidirectional channel needed.
- HTMX SSE extension is clean: `sse-connect="/events"` + `sse-swap="event-name"`.
- Auto-reconnection is built into the SSE spec. No custom reconnection logic.

**Go implementation pattern:**
```go
// eventbus/bus.go
type Event struct {
    Type      string    `json:"type"`      // "backup.user.started", "backup.user.done", etc.
    Timestamp time.Time `json:"timestamp"`
    Payload   any       `json:"payload"`
}

type Bus struct {
    subscribers map[string][]chan Event
    mu          sync.RWMutex
}

// Publish sends event to all matching subscribers. Non-blocking.
func (b *Bus) Publish(e Event) {
    b.mu.RLock()
    defer b.mu.RUnlock()
    for pattern, chans := range b.subscribers {
        if matchPattern(pattern, e.Type) {
            for _, ch := range chans {
                select {
                case ch <- e:
                default: // drop if consumer is slow — never block the publisher
                }
            }
        }
    }
}
```

```go
// server/sse.go — SSE endpoint that bridges event bus to browser
func (s *Server) handleSSE(w http.ResponseWriter, r *http.Request) {
    flusher, ok := w.(http.Flusher)
    if !ok {
        http.Error(w, "SSE not supported", http.StatusInternalServerError)
        return
    }
    w.Header().Set("Content-Type", "text/event-stream")
    w.Header().Set("Cache-Control", "no-cache")
    w.Header().Set("Connection", "keep-alive")

    ch := s.bus.Subscribe("backup.*")
    defer s.bus.Unsubscribe(ch)

    for {
        select {
        case event := <-ch:
            html := s.renderFragment("backup-progress", event)
            fmt.Fprintf(w, "event: backup-progress\ndata: %s\n\n", html)
            flusher.Flush()
        case <-r.Context().Done():
            return // client disconnected — clean up
        }
    }
}
```

**Event bus consumers (loosely coupled):**
- SSE handler → live UI updates
- SQLite writer → persistent event log (for viewing history after page refresh)
- Future: Slack/email alerter → notifications on failures

### ADR-004: Database — SQLite (Two Files)

**Decision:** Two separate SQLite database files.

| File | Purpose | Criticality | Can be rebuilt? |
|---|---|---|---|
| `backup.db` | Backup metadata, job history, delta tokens, M365 sync state, config | **Sacred** — data loss here means backup integrity loss | NO |
| `vectors.db` | Embedding vectors, RAG index (sqlite-vec) | Important but derived | YES — rebuild from existing backup data |

**SQLite driver:** `modernc.org/sqlite` (pure Go, easier cross-compilation) preferred over `mattn/go-sqlite3` (CGo).

**Why not separate DBs like Postgres?** Zero external dependencies. SQLite handles single-writer workloads well, which is exactly our pattern.

### ADR-005: Embedding — EmbeddingGemma via Ollama

**Decision:** Google EmbeddingGemma (308M params) served by Ollama sidecar.

**Key specs to remember:**
- 308M parameters, 100+ languages including Turkish
- 768-dim output (adjustable: 512, 256, 128 via Matryoshka)
- <200MB RAM when quantized
- 2048 token context window
- Ollama model name: `embeddinggemma`

**Integration is an HTTP call, nothing more:**
```go
// index/ollama.go
func (o *OllamaEmbedder) Embed(ctx context.Context, text string) ([]float32, error) {
    body := map[string]any{
        "model": o.model,  // "embeddinggemma"
        "input": text,
    }
    resp, err := o.httpClient.Post(o.url+"/api/embed", "application/json", marshal(body))
    if err != nil {
        return nil, fmt.Errorf("ollama embed request: %w", err)
    }
    defer resp.Body.Close()

    var result struct {
        Embeddings [][]float32 `json:"embeddings"`
    }
    if err := json.NewDecoder(resp.Body).Decode(&result); err != nil {
        return nil, fmt.Errorf("ollama embed decode: %w", err)
    }
    if len(result.Embeddings) == 0 {
        return nil, fmt.Errorf("ollama returned no embeddings")
    }
    return result.Embeddings[0], nil
}
```

**KVKK compliance:** All embedding happens locally. No data leaves the corporate network. This is non-negotiable.

**Fallback models (if Turkish quality insufficient):**
1. `BAAI/bge-m3` — 568M, dense+sparse, 8192 context, MIT license
2. `Alibaba GTE-Qwen3-Embedding 0.6B` — Apache-2.0
3. Hybrid: `newmindai/TurkColBERT` for Turkish + EmbeddingGemma for English

**Model swap is config-only:** Change `EMBED_MODEL` env var + `ollama pull <new-model>`. The `Embedder` interface ensures zero code changes.

### ADR-006: Vector Storage — sqlite-vec

**Decision:** sqlite-vec extension in the separate `vectors.db`.

**Scale calculation (prevents "do we need Qdrant?" questions):**
- 100 users × ~12,700 chunks each ≈ 1.27M vectors
- At 768-dim float32: 1.27M × 3KB = ~3.7GB
- At 256-dim (Matryoshka): ~1.2GB
- SA3200D has 16GB RAM — entire index fits in memory
- **sqlite-vec is sufficient. Do not add Qdrant, Milvus, or ChromaDB unless scale exceeds 10M+ vectors.**

### ADR-007: Document Extraction — Go-native + Tika Fallback

**Two-tier strategy:**

| Tier | Formats | Implementation | When Used |
|---|---|---|---|
| **Tier 1: Go-native** | .txt, .md, .csv, .docx, .xlsx, .html | Go libraries (excelize, go-docx) | Always tried first (fast, no dependency) |
| **Tier 2: Tika** | PDF, legacy .doc/.xls, .pptx, .rtf, .eml, 1000+ others | HTTP POST to Tika container | Fallback when Tier 1 can't handle format |

**Chain pattern (this exact pattern must be used):**
```go
type ChainExtractor struct {
    native   []TextExtractor  // Go-native extractors, tried in order
    fallback TextExtractor    // Tika — always last resort
}

func (c *ChainExtractor) Extract(r io.Reader, mimeType string) (string, error) {
    for _, ext := range c.native {
        if ext.Supports(mimeType) {
            text, err := ext.Extract(r, mimeType)
            if err == nil {
                return text, nil
            }
            // Log the error but continue to next extractor
            slog.Warn("native extractor failed, trying next", "mime", mimeType, "err", err)
        }
    }
    return c.fallback.Extract(r, mimeType)
}
```

### ADR-008: Chunking — Format-Aware

**Each content type has a specific chunking strategy. Do NOT apply a generic "split every N tokens" approach:**

| Content Type | Strategy | Key Detail |
|---|---|---|
| Email | Single chunk: subject + sender + recipients + date + body (~1500 chars). Attachments = separate linked chunks. | Parent email ID in attachment chunk metadata |
| Word (.docx) | Split by headings (H1/H2/H3). Each section = chunk with breadcrumb path. >2048 tokens → overlap split (15%). | Preserve heading hierarchy as metadata |
| Excel (.xlsx) | Per-sheet. Column headers repeated with each row group. Summary/pivot rows = separate chunk. | Column context prevents meaningless number-only chunks |
| PDF | Page-aware chunks via Tika. Preserve page numbers. Large pages split with overlap. | Page number in metadata enables "found on page X" UX |
| Teams | Thread-level grouping (not individual messages). | Thread context is more meaningful than isolated messages |
| SharePoint | Page title + content as single chunk. | Most pages fit within 2048 tokens |
| Markdown/TXT | Heading-based or fixed-window (500-1000 words, 15% overlap). | Simplest case |

### ADR-009: Deployment — Flexible Topology

**Three deployment scenarios via config changes only (zero code changes):**

```
Scenario 1 (start here):
  NAS ──── Go binary + Ollama (EmbeddingGemma) + Tika
           Everything local. CPU-only. No LLM answers, search-only.

Scenario 2 (future):
  NAS ──── Go binary + Tika
  GPU ──── Ollama (EmbeddingGemma + LLM like Gemma 3 12B)
           Change: OLLAMA_URL=http://gpu-node:11434

Scenario 3 (future):
  NAS ──── Go binary + Tika
  GPU ──── Ollama
  Agent ── n8n / custom → REST API → RAG queries
           Add: LLM_MODEL=gemma3:12b + API auth
```

### ADR-010: Web Framework — Go stdlib

**Decision:** Go 1.22+ standard library routing with pattern matching.

```go
mux := http.NewServeMux()
mux.HandleFunc("GET /",                  s.handleDashboard)
mux.HandleFunc("GET /api/jobs/{id}",     s.handleGetJob)
mux.HandleFunc("POST /api/backup/start", s.handleStartBackup)
mux.HandleFunc("GET /events",            s.handleSSE)
mux.HandleFunc("POST /api/v1/search",    s.handleSearch)
```

**No Echo, no Chi, no Gin.** Stdlib is sufficient. Graph API latency dwarfs any framework overhead difference.

---

## Key Interfaces

These interfaces are the contracts between layers. **Implement them exactly as defined.** Adding methods requires discussion with the developer.

```go
// --- Embedding ---
type Embedder interface {
    Embed(ctx context.Context, text string) ([]float32, error)
    EmbedBatch(ctx context.Context, texts []string) ([][]float32, error)
    Dimensions() int
}

// --- Vector Storage ---
type VectorStore interface {
    Insert(ctx context.Context, id string, vector []float32, metadata map[string]string) error
    Search(ctx context.Context, query []float32, topK int, filters map[string]string) ([]SearchResult, error)
    Delete(ctx context.Context, id string) error
    Count(ctx context.Context) (int64, error)
}

type SearchResult struct {
    ID       string
    Score    float32
    Metadata map[string]string
}

// --- Text Extraction ---
type TextExtractor interface {
    Extract(reader io.Reader, mimeType string) (string, error)
    Supports(mimeType string) bool
}

// --- Chunking ---
type Chunker interface {
    Chunk(content string, contentType string, metadata map[string]string) ([]Chunk, error)
}

type Chunk struct {
    Text     string
    Metadata map[string]string  // source_type, date, owner, path, page, parent_id, etc.
}

// --- Event Bus ---
type EventBus interface {
    Publish(event Event)
    Subscribe(pattern string) <-chan Event  // "backup.*", "index.*", etc.
    Unsubscribe(ch <-chan Event)
}

type Event struct {
    Type      string    // "backup.started", "backup.user.complete", "index.chunk.added"
    Timestamp time.Time
    Payload   any
}
```

**Compile-time interface checks (mandatory in every implementation file):**
```go
var _ Embedder      = (*OllamaEmbedder)(nil)
var _ VectorStore   = (*SqliteVecStore)(nil)
var _ TextExtractor = (*TikaExtractor)(nil)
var _ EventBus      = (*Bus)(nil)
```

---

## Project Structure

```
bellek/
├── CLAUDE.md                    # THIS FILE — single source of truth
├── go.mod
├── go.sum
├── main.go                      # Entry point: bellek CLI — parse flags, load config, wire deps, start
├── Dockerfile                   # Multi-stage: build in golang image → copy bellek binary to scratch/alpine
├── docker-compose.yml           # bellek + Ollama + Tika
│
├── internal/                    # All application code (not importable by external packages)
│   ├── config/
│   │   └── config.go            # Config struct with env/file loading. No defaults hidden elsewhere.
│   │
│   ├── graph/                   # Microsoft Graph API — the ONLY package that talks to M365
│   │   ├── client.go            # Auth (client credentials), HTTP client, throttle + retry
│   │   ├── mail.go              # Exchange: list, get, download attachment
│   │   ├── onedrive.go          # OneDrive: list, download, permissions
│   │   ├── sharepoint.go        # SharePoint: sites, pages, document libraries
│   │   ├── teams.go             # Teams: channels, messages, threads
│   │   └── delta.go             # Delta query state machine per resource type
│   │
│   ├── backup/                  # Backup engine — writes to disk + backup.db
│   │   ├── engine.go            # Orchestrator: schedules users, manages concurrency
│   │   ├── job.go               # Single backup job lifecycle with status tracking
│   │   ├── writer.go            # Disk write: content-addressed layout, dedup-friendly
│   │   └── snapshot.go          # Point-in-time snapshot metadata
│   │
│   ├── restore/                 # Restore engine — reads from disk, writes to M365 via Graph
│   │   ├── engine.go            # Restore orchestrator
│   │   ├── mail.go              # Recreate mail with attachments, flags, folder placement
│   │   ├── onedrive.go          # Restore files preserving hierarchy, metadata, sharing
│   │   └── verify.go            # Post-restore comparison: source hash vs restored hash
│   │
│   ├── extract/                 # Text extraction from file content
│   │   ├── extractor.go         # TextExtractor interface + ChainExtractor
│   │   ├── plaintext.go         # .txt, .md, .csv (with encoding detection)
│   │   ├── docx.go              # Word: paragraph-level, preserving heading hierarchy
│   │   ├── xlsx.go              # Excel: sheet-aware, column headers preserved
│   │   ├── tika.go              # Tika HTTP client (POST file → receive text)
│   │   └── chunker.go           # Format-aware chunking per ADR-008
│   │
│   ├── index/                   # RAG indexing layer
│   │   ├── embedder.go          # Embedder interface definition
│   │   ├── ollama.go            # OllamaEmbedder implementation
│   │   ├── vectorstore.go       # VectorStore interface definition
│   │   ├── sqlitevec.go         # sqlite-vec implementation
│   │   ├── indexer.go           # Worker: event bus consumer → extract → chunk → embed → store
│   │   └── search.go            # Query embed → vector search → result ranking
│   │
│   ├── eventbus/
│   │   └── bus.go               # Publish/Subscribe with pattern matching. Non-blocking sends.
│   │
│   ├── scheduler/
│   │   └── scheduler.go         # Cron-like scheduling (backup at 2 AM daily, etc.)
│   │
│   ├── store/                   # SQLite data access layer for backup.db
│   │   ├── db.go                # Open, migrate, close. Migrations via embedded SQL files.
│   │   ├── jobs.go              # Job CRUD: create, update status, list recent
│   │   ├── users.go             # M365 user enrollment and status
│   │   ├── deltas.go            # Delta token read/write per user per resource type
│   │   └── events.go            # Event log persistence (event bus consumer)
│   │
│   └── server/                  # HTTP server + web UI
│       ├── server.go            # http.Server setup, middleware (logging, recovery)
│       ├── routes.go            # All route registrations in one place
│       ├── api.go               # REST API handlers (JSON responses)
│       ├── sse.go               # SSE endpoint: event bus → HTML fragments → browser
│       ├── pages.go             # HTML page handlers: render template with data
│       └── search.go            # Search page handler + search API
│
├── ui/                          # Embedded static assets
│   ├── embed.go                 # //go:embed directives
│   ├── templates/               # Go HTML templates
│   │   ├── layout.html          # Base layout with Tabler CSS/JS, HTMX
│   │   ├── dashboard.html       # Overview: active jobs, storage usage, recent events
│   │   ├── jobs.html            # Job list and detail views
│   │   ├── restore.html         # Restore wizard
│   │   ├── search.html          # Semantic search interface
│   │   └── settings.html        # Configuration UI
│   └── static/
│       ├── css/                 # Tabler CSS (vendored)
│       ├── js/                  # HTMX + htmx-ext-sse + minimal vanilla JS
│       └── img/
│
├── migrations/                  # SQL migration files (embedded via go:embed)
│   ├── backup/                  # Migrations for backup.db
│   │   ├── 001_initial.sql
│   │   └── ...
│   └── vectors/                 # Migrations for vectors.db
│       ├── 001_initial.sql
│       └── ...
│
└── scripts/                     # Developer utilities (not part of the binary)
    ├── setup-azure-app.sh       # Azure AD app registration + permission grant
    └── benchmark-embeddings.sh  # Test Turkish embedding quality with sample texts
```

---

## Configuration

```go
type Config struct {
    // Microsoft Graph API — all three are required, app won't start without them
    AzureTenantID     string `env:"AZURE_TENANT_ID"     required:"true"`
    AzureClientID     string `env:"AZURE_CLIENT_ID"     required:"true"`
    AzureClientSecret string `env:"AZURE_CLIENT_SECRET" required:"true"`

    // Storage paths
    DataDir           string `env:"DATA_DIR"    default:"/data"`
    DBDir             string `env:"DB_DIR"      default:"/db"`

    // Ollama (embedding + optional LLM)
    OllamaURL         string `env:"OLLAMA_URL"  default:"http://localhost:11434"`
    EmbedModel        string `env:"EMBED_MODEL" default:"embeddinggemma"`
    EmbedDimensions   int    `env:"EMBED_DIMENSIONS" default:"768"`
    LLMModel          string `env:"LLM_MODEL"`  // empty = search-only mode, no LLM answers

    // Tika (document extraction)
    TikaURL           string `env:"TIKA_URL"    default:"http://localhost:9998"`

    // Server
    ListenAddr        string `env:"LISTEN_ADDR" default:":8080"`

    // Backup behavior
    BackupCron        string `env:"BACKUP_CRON" default:"0 2 * * *"` // 2 AM daily
    MaxConcurrentUsers int   `env:"MAX_CONCURRENT_USERS" default:"5"`
    GraphRateLimit     int   `env:"GRAPH_RATE_LIMIT"     default:"100"` // requests/sec
}
```

---

## Docker Compose

```yaml
services:
  bellek:
    build: .
    ports:
      - "8080:8080"
    volumes:
      - ./data:/data
      - ./db:/db
    environment:
      - AZURE_TENANT_ID=${AZURE_TENANT_ID}
      - AZURE_CLIENT_ID=${AZURE_CLIENT_ID}
      - AZURE_CLIENT_SECRET=${AZURE_CLIENT_SECRET}
      - OLLAMA_URL=http://ollama:11434
      - TIKA_URL=http://tika:9998
      - EMBED_MODEL=embeddinggemma
    depends_on:
      - ollama
      - tika

  ollama:
    image: ollama/ollama:latest
    volumes:
      - ollama_models:/root/.ollama
    # GPU passthrough (uncomment when GPU node is available):
    # deploy:
    #   resources:
    #     reservations:
    #       devices:
    #         - capabilities: [gpu]

  tika:
    image: apache/tika:latest
    # Lightweight: ~200MB image, minimal resource usage

volumes:
  ollama_models:
```

---

## Graph API Integration Notes

**Authentication:** Azure AD client credentials flow (application permissions, not delegated).

**Required permissions:**
- `Mail.Read` — Exchange mailboxes
- `Files.Read.All` — OneDrive
- `Sites.Read.All` — SharePoint
- `ChannelMessage.Read.All` — Teams
- `User.Read.All` — enumerate users

**Throttling pattern (critical — get this right):**
```go
func (c *Client) doWithRetry(ctx context.Context, req *http.Request) (*http.Response, error) {
    for attempt := 0; attempt < maxRetries; attempt++ {
        resp, err := c.httpClient.Do(req.WithContext(ctx))
        if err != nil {
            return nil, fmt.Errorf("graph request failed: %w", err)
        }
        if resp.StatusCode != http.StatusTooManyRequests {
            return resp, nil
        }
        resp.Body.Close() // important: close body before retry

        retryAfter := resp.Header.Get("Retry-After")
        delay, _ := strconv.Atoi(retryAfter)
        if delay == 0 {
            delay = int(math.Pow(2, float64(attempt))) // exponential backoff fallback
        }

        slog.Warn("graph API throttled", "attempt", attempt, "retry_after_sec", delay)

        select {
        case <-time.After(time.Duration(delay) * time.Second):
            continue
        case <-ctx.Done():
            return nil, ctx.Err()
        }
    }
    return nil, fmt.Errorf("graph API: max retries (%d) exceeded", maxRetries)
}
```

**Delta queries:** Each resource type (mail, OneDrive, SharePoint) has its own delta token. Store per-user, per-resource in backup.db. Never share delta tokens across users or resource types.

**Pagination:** Most Graph API endpoints return paginated results. **Always** follow `@odata.nextLink` until it is absent. A backup that only captures the first page is silently incomplete — this is a critical bug, not an edge case.

**Restore must preserve:**
- Email: Full MIME, attachments, folder structure, read/unread, categories, importance, conversation threading
- OneDrive: File content, folder hierarchy, created/modified dates, sharing permissions, version history
- SharePoint: Page content, site structure, document libraries, list data, permissions
- Teams: Channel messages, threaded replies, file attachments, reactions (where Graph API allows)

---

## Development Phases & TODO

> **Rule: Complete each phase before starting the next.** Phase N+1 assumes Phase N is solid and tested. Do not build Phase 3 features while Phase 1 has gaps.

### Phase 1: Foundation — Single User Backup + Restore (MVP)

**Goal:** Prove the core loop: backup one user's Exchange mailbox → restore it → verify fidelity.

- [ ] `go mod init bellek` (or `github.com/<org>/bellek`) with project structure as defined above
- [ ] Config loading from environment variables with validation
- [ ] Azure AD authentication: client credentials flow → access token
- [ ] Graph API client with throttling (429 handling) and retry logic
- [ ] **Backup:** Download all mail + attachments for one user to disk
  - [ ] Folder structure preserved on disk (Inbox/, Sent/, etc.)
  - [ ] Each email: metadata JSON + .eml or structured file + attachments
  - [ ] Pagination: follow ALL `@odata.nextLink` pages
  - [ ] Delta token saved after full sync
- [ ] **Restore:** Upload backed-up mail back to M365 (different folder or test mailbox)
  - [ ] Attachments reattached
  - [ ] Folder placement correct
  - [ ] Read/unread flags preserved
- [ ] **Verify:** Compare original vs restored (count, subjects, attachment sizes, flags)
- [ ] SQLite schema: jobs, users, delta_tokens, events tables
- [ ] Event bus: basic publish/subscribe with pattern matching
- [ ] HTMX dashboard: start backup button, job list, SSE live progress
- [ ] Dockerfile + docker-compose.yml (app only, no Ollama/Tika yet)

**Phase 1 exit criteria:**
- [ ] Full backup of one user's mailbox completes without error
- [ ] Restore of that backup creates identical mailbox content
- [ ] Dashboard shows live progress via SSE during backup
- [ ] All errors are handled and visible in UI event log

### Phase 2: Full M365 Coverage + Multi-User

**Goal:** Backup and restore all M365 data types for all 100 users.

- [ ] OneDrive backup: files, folders, permissions, versions
- [ ] OneDrive restore with hierarchy, metadata, sharing settings
- [ ] SharePoint backup: sites, document libraries, pages, lists
- [ ] SharePoint restore
- [ ] Teams backup: channels, messages, threads, file attachments
- [ ] Multi-user parallel backup (configurable concurrency via `MaxConcurrentUsers`)
- [ ] Delta queries for incremental backup per user per resource type
- [ ] Backup scheduling (cron-based via `BackupCron` config)
- [ ] Job history and status tracking in UI
- [ ] Storage usage reporting

**Phase 2 exit criteria:**
- [ ] All 100 users' Exchange + OneDrive + SharePoint + Teams backed up successfully
- [ ] Incremental (delta) backup works — second run is significantly faster
- [ ] Scheduled backup runs automatically at configured time
- [ ] Restore verified for at least one item of each resource type

### Phase 3: RAG Layer — Extraction, Indexing, Search

**Goal:** Make all backed-up content semantically searchable.

- [ ] Go-native extractors: TXT, MD, CSV, DOCX (heading-aware), XLSX (sheet-aware)
- [ ] Tika sidecar integration + ChainExtractor pattern
- [ ] Format-aware chunking per ADR-008 table
- [ ] Ollama client for EmbeddingGemma
- [ ] sqlite-vec setup in separate vectors.db
- [ ] Indexer worker as event bus consumer (`backup.file.written` → extract → chunk → embed → store)
- [ ] Semantic search: query → embed → vector similarity search → ranked results
- [ ] Search UI in HTMX dashboard with result snippets and source links
- [ ] Add Ollama + Tika containers to docker-compose.yml
- [ ] Benchmark Turkish embedding quality with real company email/document samples

**Phase 3 exit criteria:**
- [ ] Natural language search for a known email topic returns relevant results
- [ ] Cross-format search works (find content originally in a Word doc via search query)
- [ ] Indexing runs in background without degrading backup performance
- [ ] vectors.db can be deleted and fully rebuilt from existing backup data

### Phase 4: Search API + External Access

- [ ] REST API: `POST /api/v1/search` (query → results with metadata and scores)
- [ ] REST API: `POST /api/v1/retrieve` (chunk ID → full source content)
- [ ] API key authentication for external agent access
- [ ] Metadata filters: date range, source type (mail/file/page/message), user
- [ ] Rate limiting on API endpoints
- [ ] OpenAPI / Swagger documentation for the API

### Phase 5: Monitoring & Hardening

- [ ] Dashboard charts: backup size over time, job success/failure rates
- [ ] Alerting: email and/or Slack on backup failure (new event bus consumer)
- [ ] Health check endpoint (`GET /healthz`) for monitoring systems
- [ ] Periodic backup integrity verification (spot-check random samples)
- [ ] Block-level deduplication for storage efficiency
- [ ] Configuration UI in dashboard (manage settings without env vars)
- [ ] User management UI (enroll/remove M365 users from backup scope)

### Phase 6: Advanced (Future)

- [ ] LLM-based answer generation (requires GPU node + `LLM_MODEL` config)
- [ ] n8n / workflow integration examples and documentation
- [ ] Multi-tenant support (multiple M365 tenants in one instance)
- [ ] Encryption at rest for backup data on disk
- [ ] KVKK/GDPR compliance exports (data subject access/deletion requests)
- [ ] Embedding model hot-swap without reindex downtime
- [ ] `bellek reindex` CLI command to rebuild vectors.db from scratch
- [ ] Open-source release preparation (license, CI/CD pipeline, contributor docs, demo)

---

## Common Pitfalls to Avoid

These are drawn from known LLM reasoning failure patterns (ref: Song et al., "Large Language Model Reasoning Failures", TMLR 2026). Pay special attention to these during implementation.

### Compositional Reasoning Failures
1. **Graph API pagination.** The most dangerous silent failure. Most endpoints return 10-50 items per page. A mailbox with 5,000 emails requires following ~100 `@odata.nextLink` pages. If you stop at page 1, the backup silently contains only 50 emails. **Always loop until `@odata.nextLink` is absent.**
2. **Delta token per (user, resource_type).** Each user has separate delta tokens for Mail, OneDrive, and SharePoint. A common mistake is sharing a single delta token across resource types. This leads to missed changes or duplicate downloads.

### Confirmation Bias / Path Dependency
3. **Don't assume the first working approach is correct.** If backup "works" for a test user with 20 emails, it doesn't mean it works for a user with 50,000 emails, nested folders, or large attachments. Test edge cases explicitly.
4. **Don't optimize before profiling.** The urge to add caching, connection pooling, or batch processing early is strong. Resist it. The bottleneck is always the Graph API rate limit, not Go code speed.

### Working Memory Limitations
5. **Keep functions short and focused.** A 200-line function that does backup + extraction + indexing in one flow will accumulate bugs. Each responsibility belongs in its own function, its own package.
6. **Consistent metadata schema.** Every chunk in the vector store must have the same set of metadata keys. Define the schema once as constants. A chunk missing `source_type` or `date` breaks filtering — this is a silent data quality bug.

### Inverse Inference Failures
7. **Test restore, not just backup.** It's easy to verify "backup ran successfully" and assume restore works. It doesn't necessarily. Always run the full cycle: backup → restore → verify equality. A backup that can't be restored is worthless.
8. **Test error paths, not just happy paths.** For every test that verifies "backup succeeds," write tests for: network timeout, 429 throttle, partial failure (3 of 5 users fail), invalid token, empty mailbox.

### Robustness / Sensitivity
9. **Handle empty and edge cases.** Users with zero emails, zero files, or no OneDrive. Emails with no body. Attachments with zero bytes. Files with non-UTF-8 names. SharePoint pages with no content. These must not crash the system.
10. **Context cancellation in long loops.** Backup jobs run for hours. Every loop that processes users, emails, or files must check `ctx.Done()`. Without this, the application cannot shut down gracefully or cancel a running job.

### Proactive Initiative (Scope Creep)
11. **Don't add "while we're at it" features.** When implementing Phase 1 (mail backup), do not add OneDrive support because "it's similar." Each phase boundary exists for a reason.
12. **Don't invent abstractions not in this document.** If the CLAUDE.md doesn't mention a factory pattern, a plugin system, or a middleware chain — don't create one. Ask the developer first.

---

## Hardware Target

**Synology SA3200D NAS**
- Intel Xeon D-1521 (4-core, 2.4GHz)
- 16GB DDR4 ECC RAM (expandable to 64GB/controller)
- 12 × SAS drive bays
- RAID-6: 6 × Seagate Exos 16TB SAS → ~64TB usable
- Btrfs filesystem

**Storage budget:**
| Component | Size |
|---|---|
| M365 backup data | 3-6 TB |
| backup.db | < 1 GB |
| vectors.db (768-dim) | ~3.7 GB |
| EmbeddingGemma model (quantized) | < 200 MB |
| **Total RAG overhead** | **~5 GB** |

---

## Quick Reference

| Component | Technology | Why This, Not That |
|---|---|---|
| Language | Go 1.22+ | Single binary, explicit errors, goroutines for concurrency |
| Web UI | HTMX + Go templates + Tabler | No build step, embedded in binary, admin-oriented |
| Real-time | SSE | One-way server→client, HTMX native, auto-reconnect |
| Database | SQLite (2 files) | Zero deps, sufficient for single-writer backup workload |
| Embedding | EmbeddingGemma 308M via Ollama | Open, multilingual (TR), <200MB, local-only (KVKK) |
| Vector DB | sqlite-vec | Embedded, no extra service, handles 1M+ vectors |
| Doc extraction | Go-native + Apache Tika | Speed for common formats + coverage for everything else |
| Auth | Azure AD client credentials | Application-level M365 access, no user interaction needed |
| HTTP framework | Go stdlib (1.22+) | Pattern matching built-in, no framework overhead |
| Deployment | Docker Compose on NAS | Portable, Ollama topology flexible via config |
