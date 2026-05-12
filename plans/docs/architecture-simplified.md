# GitNexus Simplified Architecture

## High-Level Shape

GitNexus is a local-first code intelligence stack. The CLI builds and stores a graph. MCP, direct CLI tools, HTTP, and the web UI all read from the same persisted graph.

```text
                    +------------------+
                    |  Developer / AI  |
                    +---------+--------+
                              |
              +---------------+---------------+
              |               |               |
              v               v               v
        CLI commands      MCP stdio       Web browser
              |               |               |
              |               v               v
              |        +-------------+   +-------------+
              |        | MCP server  |   | React UI    |
              |        +------+------+   +------+------+
              |               |                 |
              v               v                 v
        +-----------------------------------------------+
        | LocalBackend / HTTP API / tool implementations |
        +----------------------+------------------------+
                               |
                               v
                  +--------------------------+
                  | LadybugDB graph index    |
                  +--------------------------+
                               ^
                               |
                  +--------------------------+
                  | Ingestion pipeline       |
                  +--------------------------+
                               ^
                               |
                        source repository
```

## Package Responsibilities

```text
GitNexus monorepo
|
+-- gitnexus/
|   +-- src/cli/                  CLI entry points and command wrappers
|   +-- src/core/ingestion/       scan, parse, resolve, graph construction
|   +-- src/core/lbug/            LadybugDB schema, load, query adapters
|   +-- src/core/search/          BM25, hybrid search, FTS indexes
|   +-- src/core/embeddings/      embedding model and batch pipeline
|   +-- src/core/group/           multi-repo groups and contract bridge
|   +-- src/mcp/                  MCP tool/resource server
|   +-- src/server/               Express HTTP API and MCP-over-HTTP
|
+-- gitnexus-web/
|   +-- src/App.tsx               main app shell and server connection flow
|   +-- src/services/             typed HTTP client
|   +-- src/components/           graph, panels, settings, file tree, chat UI
|   +-- src/core/llm/             browser-side LLM agent helpers
|   +-- src/core/graph/           client-side graph model
|
+-- gitnexus-shared/
    +-- src/graph/                shared graph types
    +-- src/lbug/                 schema constants
    +-- src/scope-resolution/     shared scope-resolution data contracts
    +-- src/integrations/         retry, circuit breaker, resilient fetch
```

## Indexing Flow

The `analyze` command is the write path. It is intentionally separated from read/query paths.

```text
gitnexus analyze
      |
      v
src/cli/analyze.ts
      |
      v
runFullAnalysis(repoPath, options)
      |
      +--> check existing meta / current commit
      +--> preserve or drop existing embeddings based on flags
      |
      v
runPipelineFromRepo()
      |
      v
in-memory KnowledgeGraph
      |
      v
loadGraphToLbug()
      |
      +--> create FTS indexes
      +--> restore/generate embeddings
      +--> save meta.json
      +--> register repo globally
      +--> generate AGENTS.md / CLAUDE.md context blocks
```

## Ingestion Pipeline

The pipeline is phase-based. Each phase declares dependencies and receives only those dependency outputs. The runner validates the graph and executes in topological order.

```text
scan
  |
  v
structure
  |
  +----------------+
  |                |
  v                v
markdown          cobol
  |                |
  +--------+-------+
           |
           v
         parse
           |
   +-------+-------+-------+
   |       |       |       |
   v       v       v       |
 routes  tools   orm      |
   |       |       |       |
   +-------+-------+-------+
           |
           v
       crossFile
           |
           v
    scopeResolution
           |
           v
          mro
           |
           v
     communities
           |
           v
       processes
```

### Phase Summary

| Phase | Simplified Responsibility |
| --- | --- |
| `scan` | Collect file paths and sizes. |
| `structure` | Create File/Folder nodes and containment edges. |
| `markdown` | Index Markdown sections and doc links. |
| `cobol` | Extract COBOL programs, sections, and paragraphs. |
| `parse` | Tree-sitter parse, symbol extraction, imports, calls, routes/tools/ORM candidates. |
| `routes` | Add route nodes and route-handler edges. |
| `tools` | Add tool nodes and tool-handler edges. |
| `orm` | Add ORM query relationships. |
| `crossFile` | Propagate types and bindings across imports. |
| `scopeResolution` | Registry-primary reference resolution for migrated languages. |
| `mro` | Add override/implementation relationships from inheritance. |
| `communities` | Detect functional clusters with graph algorithms. |
| `processes` | Detect execution flows and process steps. |

## Language Architecture

GitNexus keeps most ingestion logic language-agnostic. Language behavior is plugged in through providers, tree-sitter queries, import resolver configs, and scope resolvers.

```text
language source file
        |
        v
language detection
        |
        v
tree-sitter grammar + query captures
        |
        v
unified capture tags
        |
        v
shared extractors and semantic model
        |
        +----------------------+
        |                      |
        v                      v
legacy call-resolution DAG     registry-primary scope resolution
unmigrated languages           migrated languages
        |                      |
        +----------+-----------+
                   |
                   v
            KnowledgeGraph edges
```

Current registry-primary migrated languages in code are Python, CSharp, TypeScript, Go, and C. Other supported languages continue through the legacy call-resolution DAG unless their registry-primary flag is enabled.

## Call and Reference Resolution

There are two resolution paths during the transition to the scope-resolution pipeline.

### Legacy Call-Resolution DAG

```text
extract-call
     |
     v
classify-form
     |
     v
infer-receiver
     |
     v
select-dispatch
     |
     v
resolve-target
     |
     v
emit CALLS edge
```

Language-specific behavior belongs behind `LanguageProvider` hooks such as implicit receiver inference and dispatch selection.

### Registry-Primary Scope Resolution

```text
ParsedFile[]
     |
     v
finalize scope model
     |
     v
ScopeResolutionIndexes
     |
     v
resolve reference sites
     |
     v
ReferenceIndex
     |
     +--> emit receiver-bound calls
     +--> emit free-call fallback
     +--> emit property/reference edges
     +--> emit import edges
     |
     v
KnowledgeGraph
```

The goal is that both paths emit the same node identities, edge vocabulary, and confidence tiers so downstream tools do not care which path produced a relationship.

## Storage Model

```text
<repo>/.gitnexus/
|
+-- lbug          LadybugDB database
+-- lbug.wal      write-ahead log
+-- lbug.lock     single-writer lock
+-- meta.json     commit, timestamp, stats, capabilities

~/.gitnexus/
|
+-- registry.json global repo discovery for MCP/server
```

The graph schema uses separate node tables per node kind and one `CodeRelation` relationship table with a `type` property. This keeps Cypher queries predictable while preserving a unified edge vocabulary.

## Read/Query Surfaces

All major read surfaces converge on the local graph index.

```text
direct CLI tools
  gitnexus query/context/impact/cypher/detect-changes
        |
        v
LocalBackend
        |
        v
LadybugDB

MCP stdio
  editor/agent -> gitnexus mcp -> MCP server -> LocalBackend -> LadybugDB

HTTP bridge
  browser/curl -> gitnexus serve -> Express API -> LocalBackend -> LadybugDB

MCP over HTTP
  remote MCP client -> /api/mcp -> MCP server session -> LocalBackend -> LadybugDB
```

## Web UI Architecture

The web UI is a browser client that connects to a running backend.

```text
React App
  |
  +--> DropZone / server connection
  +--> Header / repo switching / analyze trigger
  +--> FileTreePanel
  +--> GraphCanvas
  +--> CodeReferencesPanel
  +--> RightPanel
  +--> SettingsPanel
  +--> StatusBar
          |
          v
services/backend-client.ts
          |
          v
HTTP endpoints on `gitnexus serve`
```

The client fetches repo info, streams graph data as NDJSON when requested, runs Cypher/search/grep/file reads, watches heartbeat via SSE, and streams analyze/embed job progress.

## HTTP API Shape

Important endpoint groups:

| Endpoint Group | Purpose |
| --- | --- |
| `/api/health`, `/api/heartbeat`, `/api/info` | Server health, liveness, and version context. |
| `/api/repos`, `/api/repo` | Repo registry and current repo metadata. |
| `/api/graph` | Graph download, including streaming NDJSON mode. |
| `/api/query`, `/api/search`, `/api/grep`, `/api/file` | Query, search, text grep, and file reads. |
| `/api/processes`, `/api/process`, `/api/clusters`, `/api/cluster` | Process and community views. |
| `/api/analyze/*`, `/api/embed/*` | Background analysis and embedding jobs. |
| `/api/mcp` | MCP over StreamableHTTP. |

## Search and Embeddings

```text
indexed graph nodes
      |
      +--> FTS indexes for BM25
      |
      +--> optional embeddings table
               |
               v
hybridSearch = BM25 + vector/exact semantic search
               |
               v
Reciprocal Rank Fusion
```

Embeddings are preserved across routine analyze runs unless explicitly dropped. If vector indexing is unavailable on a platform, GitNexus can fall back to bounded exact scan when embeddings exist.

## Multi-Repo Group Architecture

```text
member repos + group.yaml
        |
        v
group config parser
        |
        v
contract extractors
        |
        v
contracts.json
        |
        v
bridge DB / cross-impact
        |
        v
group-aware query/context/impact
```

Group mode does not add separate `group_query` or `group_impact` tools. Instead, existing `query`, `context`, and `impact` accept `repo="@group"` or `repo="@group/memberPath"`.

## Architectural Constraints

- Shared ingestion code must not special-case languages. Use `LanguageProvider`, import resolver configs, or `ScopeResolver`.
- The graph schema and relationship vocabulary are shared contracts for CLI, MCP, HTTP, web, embeddings, and group bridge consumers.
- Analyze owns writes to the graph index; query surfaces are read-oriented.
- LadybugDB expects single-writer ownership. Avoid overlapping analyze and direct DB mutation.
- Staleness matters. Tools compare indexed commits with `HEAD` and should prompt re-analysis when stale.
- Impact analysis is required before editing shared symbols when graph tooling is available.
