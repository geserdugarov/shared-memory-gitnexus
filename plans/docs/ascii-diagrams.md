# GitNexus ASCII Diagrams

## 1. One-Line Mental Model

```text
repo files -> static analysis -> knowledge graph -> CLI/MCP/API/UI -> better code decisions
```

## 2. User-Facing Surfaces

```text
                 +------------------+
                 |   GitNexus User  |
                 +---------+--------+
                           |
          +----------------+----------------+
          |                |                |
          v                v                v
   CLI commands       AI editor/agent     Browser UI
          |                |                |
          v                v                v
   direct tools       MCP stdio        HTTP requests
          |                |                |
          +----------------+----------------+
                           |
                           v
                    LocalBackend
                           |
                           v
                    LadybugDB index
```

## 3. Monorepo Package Map

```text
GitNexus
|
+-- gitnexus/                 CLI, MCP, HTTP API, ingestion, DB, search
|
+-- gitnexus-web/             React/Vite graph explorer and chat client
|
+-- gitnexus-shared/          shared graph, schema, scope, and integration types
|
+-- eval/                     evaluation harness
|
+-- docs/                     guides, design docs, language notes
|
+-- plugin/integration dirs   marketplace and editor integration metadata
```

## 4. Analyze Command Flow

```text
gitnexus analyze
      |
      v
src/cli/analyze.ts
      |
      v
ensure heap, parse flags, show progress
      |
      v
runFullAnalysis()
      |
      +--> check current commit and prior metadata
      +--> cache existing embeddings if needed
      |
      v
runPipelineFromRepo()
      |
      v
KnowledgeGraph in memory
      |
      v
loadGraphToLbug()
      |
      +--> create full-text indexes
      +--> restore/generate embeddings
      +--> save `.gitnexus/meta.json`
      +--> register in `~/.gitnexus/registry.json`
      +--> update agent context files
```

## 5. Ingestion Phase DAG

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
    +------+------+------+
    |      |      |      |
    v      v      v      |
 routes  tools  orm     |
    |      |      |      |
    +------+------+------+
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

## 6. Parse and Language Abstraction

```text
file path
   |
   v
language detection
   |
   v
tree-sitter parser
   |
   v
language query captures
   |
   v
unified capture tags
   |
   v
shared extractors
   |
   v
SemanticModel + KnowledgeGraph nodes
```

## 7. Resolution Paths

```text
                 parsed symbols and call/reference sites
                                  |
                  +---------------+---------------+
                  |                               |
                  v                               v
        legacy call-resolution DAG       registry-primary resolver
        for unmigrated languages         for migrated languages
                  |                               |
                  +---------------+---------------+
                                  |
                                  v
                  unified CALLS / IMPORTS / ACCESSES / USES edges
```

## 8. Legacy Call-Resolution DAG

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
emit-edge
```

## 9. Scope-Resolution Pipeline

```text
ParsedFile[]
    |
    v
finalizeScopeModel
    |
    v
ScopeResolutionIndexes
    |
    v
resolveReferenceSites
    |
    v
ReferenceIndex
    |
    +--> emitReceiverBoundCalls
    +--> emitFreeCallFallback
    +--> emitReferencesViaLookup
    +--> emitImportEdges
    |
    v
KnowledgeGraph
```

## 10. Graph Persistence

```text
KnowledgeGraph
     |
     v
CSV / batch load
     |
     v
LadybugDB
     |
     +-- node tables:
     |     File, Folder, Function, Class, Method, Route, Tool, Process, ...
     |
     +-- relation table:
     |     CodeRelation(type, confidence, reason, step, ...)
     |
     +-- embedding table:
           Embedding(nodeId, contentHash, vector, ...)
```

## 11. Query Surfaces to the Same Backend

```text
gitnexus query/context/impact
          |
          v
   CLI tool command
          |
          v
     LocalBackend
          |
          v
     LadybugDB

editor agent
          |
          v
     MCP stdio
          |
          v
     MCP server
          |
          v
     LocalBackend
          |
          v
     LadybugDB

browser UI
          |
          v
      HTTP API
          |
          v
     LocalBackend
          |
          v
     LadybugDB
```

## 12. HTTP Bridge and Web UI

```text
gitnexus serve
      |
      v
Express server
      |
      +--> /api/repos
      +--> /api/repo
      +--> /api/graph
      +--> /api/query
      +--> /api/search
      +--> /api/file
      +--> /api/processes
      +--> /api/analyze/*
      +--> /api/embed/*
      +--> /api/mcp
      |
      v
React UI
      |
      +--> GraphCanvas
      +--> FileTreePanel
      +--> CodeReferencesPanel
      +--> RightPanel
      +--> SettingsPanel
      +--> StatusBar
```

## 13. Search Flow

```text
query text
   |
   +--> BM25 / FTS search
   |
   +--> semantic embedding search
   |
   v
Reciprocal Rank Fusion
   |
   v
ranked graph-aware results
```

## 14. Impact Analysis

```text
target symbol
     |
     v
resolve matching graph node
     |
     v
walk relationships upstream or downstream
     |
     +--> depth 1: direct dependents or dependencies
     +--> depth 2: indirect affected symbols
     +--> depth 3+: transitive risk
     |
     v
risk summary + affected processes + affected modules
```

## 15. Multi-Repo Group Bridge

```text
group.yaml
   |
   v
member repo indexes
   |
   v
contract extraction
   |
   v
contracts.json
   |
   v
bridge graph
   |
   v
repo="@group" query/context/impact
```

## 16. Storage Locations

```text
repo root
|
+-- .gitnexus/
    |
    +-- lbug
    +-- lbug.wal
    +-- lbug.lock
    +-- meta.json

home directory
|
+-- ~/.gitnexus/
    |
    +-- registry.json
```
