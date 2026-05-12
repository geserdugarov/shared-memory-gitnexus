# GitNexus Project Overview

## What GitNexus Is

GitNexus is a graph-powered code intelligence system for software agents and developers. It indexes a repository into a persistent knowledge graph, then exposes that graph through CLI commands, MCP tools, an HTTP API, and a browser UI.

The central idea is:

```text
source repo -> static analysis -> knowledge graph -> query surfaces -> safer agent work
```

Instead of relying only on text search, GitNexus records code relationships: files, symbols, imports, calls, inheritance, routes, tools, communities, and execution flows. AI coding tools can then ask questions such as "what calls this?", "what breaks if this changes?", or "which process does this symbol participate in?"

## Main Audiences

| Audience | Uses GitNexus For |
| --- | --- |
| AI agents | Context, impact analysis, change review, symbol navigation, safer edits. |
| Developers | Local codebase exploration, graph search, route/tool maps, web visualization. |
| Repo maintainers | Guardrails, stale-index detection, generated agent instructions, wiki output. |
| Multi-repo teams | Contract extraction, group sync, cross-repo impact through a bridge graph. |

## Monorepo Layout

| Path | Package / Area | Role |
| --- | --- | --- |
| `gitnexus/` | CLI/Core package | CLI commands, MCP server, HTTP API, ingestion, graph persistence, search, embeddings. |
| `gitnexus-web/` | Web UI | Vite + React browser client for graph exploration, repo connection, chat, and panels. |
| `gitnexus-shared/` | Shared types | Graph types, schema constants, language detection, scope-resolution contracts, resilient fetch. |
| `eval/` | Evaluation harness | Python-based evaluation and benchmark tooling. |
| `.github/` | CI | Workflows and setup actions for CLI/core and web packages. |
| `.claude/`, plugin packages | Agent integration | Skills and marketplace/plugin metadata for supported editors and agents. |
| `docs/` | Project docs | Guides, plans, and language-specific indexing docs. |

## Core Capabilities

- Repository indexing with tree-sitter based parsing and language-specific providers.
- Unified knowledge graph with node types such as File, Function, Class, Method, Route, Tool, Community, Process, and language-specific types.
- Relationship tracking for CONTAINS, DEFINES, CALLS, IMPORTS, EXTENDS, IMPLEMENTS, ACCESSES, route/tool handling, process steps, and more.
- Local persistence in LadybugDB under `.gitnexus/`.
- Full-text search and optional local embeddings for hybrid search.
- MCP tools for `query`, `context`, `impact`, `detect_changes`, `rename`, route maps, API impact, shape checks, and group operations.
- HTTP API and browser UI through `gitnexus serve`.
- Multi-repo group support through a Contract Registry and cross-impact bridge.

## Main Workflows

### 1. Index a Repository

```text
developer runs `gitnexus analyze`
        |
        v
scan files -> parse symbols -> resolve relationships -> build graph
        |
        v
persist `.gitnexus/lbug` + `.gitnexus/meta.json`
        |
        v
register repo in `~/.gitnexus/registry.json`
```

### 2. Use an Agent With MCP

```text
agent/editor -> MCP stdio server -> LocalBackend -> LadybugDB index
                                        |
                                        v
                              query/context/impact results
```

This is the primary reliability workflow. Agents use graph-aware tools before changing code, especially impact analysis for shared symbols and detect-changes before commit.

### 3. Use the Browser UI

```text
browser UI -> `gitnexus serve` -> HTTP API -> LocalBackend -> LadybugDB
```

The web UI is a thin client. It fetches repo metadata, streams graph data, reads files, performs search, displays graph/call/process views, and can start analyze/embed jobs through the local server.

### 4. Analyze Cross-Repo Groups

```text
group.yaml + member repos
        |
        v
group_sync -> contracts.json + bridge graph
        |
        v
query/context/impact with repo="@group"
```

Group mode lets GitNexus bridge local impact walks across service boundaries when provider/consumer contracts are known.

## Runtime Storage

| Location | Purpose |
| --- | --- |
| `<repo>/.gitnexus/lbug` | LadybugDB graph database. |
| `<repo>/.gitnexus/lbug.wal` | Write-ahead log. |
| `<repo>/.gitnexus/lbug.lock` | Single-writer lock. |
| `<repo>/.gitnexus/meta.json` | Last indexed commit, timestamp, stats, capabilities. |
| `~/.gitnexus/registry.json` | Global registry for MCP and server discovery. |

## Operating Rules That Matter

- Do not commit secrets or real environment values.
- Keep writes minimal and scoped to the task.
- Run impact analysis before changing shared symbols when graph tools are available.
- Run detect-changes before committing when graph tools are available.
- If the graph is stale, refresh with `npx gitnexus analyze`.
- Do not add language-specific behavior to shared ingestion code; add language hooks/providers instead.
- Preserve embeddings unless an explicit drop is intended.

## Quick Development Commands

| Area | Command |
| --- | --- |
| CLI/Core dev | `cd gitnexus && npm run dev` |
| CLI/Core tests | `cd gitnexus && npm test` |
| CLI/Core typecheck | `cd gitnexus && npx tsc --noEmit` |
| Web UI dev | `cd gitnexus-web && npm run dev` |
| Web UI tests | `cd gitnexus-web && npm test` |
| Web UI typecheck | `cd gitnexus-web && npx tsc -b --noEmit` |
| Local HTTP bridge | `npx gitnexus serve` |
