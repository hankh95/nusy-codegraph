# nusy-codegraph

**Code as a live Arrow object graph** — parse source code into structured Arrow
RecordBatches for zero-copy querying, versioning via git-native graph operations,
semantic search, and impact analysis.

> **Note:** This crate depends on NuSy workspace crates (`nusy-arrow-core`,
> `nusy-arrow-git`, `nusy-graph-query`, `noesis-ship`) published separately on crates.io.
> Install via `cargo add nusy-codegraph` or clone from
> [github.com/hankh95/nusy-codegraph](https://github.com/hankh95/nusy-codegraph).

## For AI Developers

This crate demonstrates a complete code-as-graph pipeline:

```
Source code → tree-sitter CST → CodeNodes + CodeEdges → Arrow RecordBatches
                                  ↓
                          Semantic search + impact analysis
```

If you want to build a similar system for your own language or toolchain, the key
ideas are:

1. **Use tree-sitter** for language-accurate CST parsing (not regex)
2. **Arrow RecordBatches** for zero-copy columnar storage and Parquet persistence
3. **CodeEdges** (calls, imports, inheritance) as first-class graph citizens
4. **Semantic search** via embeddings over code nodes (fastembed or ollama)
5. **Git-native versioning** via `nusy-arrow-git` WAL + atomic rename

---

## Features

**Code Parsing**
- **Rust** — full support: functions, structs, enums, impl blocks, macros, const, static
- **Python** — partial support: functions, classes, async def

**Code Graph**
- `CodeNode` — every function, class, module as a typed node with location and signature
- `CodeEdge` — calls, imports, inheritance, containment as typed directed edges
- Cross-file edge resolution via `NameResolver`
- Arrow schema for zero-copy columnar storage

**Semantic Search**
- Embed code nodes via fastembed or ollama
- Cosine similarity search over the embedded graph
- Natural-language queries via `codegraph_query_objects`

**Git-Native Graph Operations**
- `codegraph_diff` — what changed in the graph between commits?
- `codegraph_merge` — three-way graph merge with conflict detection
- `smart_merge` — semantic merge using AST-aware conflict resolution
- Impact analysis: which nodes are affected by a change?

**MCP Tools**
- `nusy-mcp-bridge` exposes 4 tools as an MCP server:
  - `codegraph_query_objects` — search the graph
  - `codegraph_add_edge` / `codegraph_remove_edge` — modify edges
  - `codegraph_update_object` — update node metadata

**Binaries**

| Binary | Purpose |
|--------|---------|
| `nusy-codegraph-ingest` | Ingest a directory into the graph |
| `nusy-codegraph-query` | Query the self-graph |
| `nusy-codegraph-service` | Long-running NATS service mode |
| `nusy-mcp-bridge` | MCP protocol bridge |

---

## Architecture

### Data Model

```
CodeNode
  id:            String      # "module::function" path
  kind:          String      # function | class | module | method | ...
  language:      String      # rust | python
  file_path:     String
  start_line:    u32
  end_line:      u32
  name:          String
  signature:     String      # full type signature
  doc:           String     # docstring

CodeEdge
  source_id:     String
  predicate:     String      # calls | imports | inherits | contains | ...
  target_id:     String
  file_path:     String      # where the edge was observed
```

### Schema (Arrow)

```rust
code_nodes_schema()  // subject, predicate, object, graph, version columns
code_edges_schema()  // source_id, predicate, target_id, file_path columns
```

### Ingestion Pipeline

```rust
// 1. Discover files by language
// 2. Parse with tree-sitter (rust_parser or python_parser)
// 3. Extract nodes + edges from CST
// 4. Resolve names → CodeNode IDs
// 5. Write to Arrow RecordBatch + Parquet
// 6. Optionally embed nodes for semantic search
```

**NameResolver** (`edges.rs`) — resolves unqualified names to CodeNode IDs for Python
and cross-language references. Uses a HashMap for O(1) lookups of short names
(`helper`) and qualified names (`brain.utils.helper`).

**RustModuleResolver** (`module_resolver.rs`) — handles Rust-specific resolution:
- `use` statements → specific CodeNodes (not just module-level edges)
- `impl Trait for Type` → `ImplementsTrait` edge
- `pub use` → `ReExports` edge
- `#[test]` naming convention → `TestTargets` edge

---

## Quick Start

```bash
# Ingest a Rust project into the graph
nusy-codegraph-ingest ./path/to/project --language rust

# Query the self-graph (project ingests itself)
nusy-codegraph-query "query_objects: semantic_search_term"

# Start service mode (NATS)
nusy-codegraph-service --nats nats://192.168.8.110:4222

# MCP bridge for AI agent integration
nusy-mcp-bridge --nats nats://192.168.8.110:4222
```

### As a Library

```rust
use nusy_codegraph::{
    ingest::ingest_directory,
    schema::{CodeNode, CodeEdge},
    search::semantic_search,
    edges::extract_edges,
};

fn main() -> Result<(), Box<dyn std::error::Error>> {
    // Ingest a directory
    let result = ingest_directory("./src", Language::Rust)?;

    // Search semantically
    let results = semantic_search("error handling in REST API", top = 5)?;

    // Find all callers of a function
    let callers = nusy_codegraph::search::callers("my_crate::handle_request")?;

    Ok(())
}
```

### Semantic Diff

```rust
use nusy_codegraph::semantic_diff;

let diff = semantic_diff("HEAD~1", "HEAD")?;
println!("{}", diff.format_impact_report());
```

---

## Ecosystem

`nusy-codegraph` is part of the NuSy Arrow workspace:

| Crate | Role |
|-------|------|
| [nusy-arrow-core](https://crates.io/crates/arrow-graph-core) | Arrow schemas, graph store |
| [nusy-arrow-git](https://crates.io/crates/arrow-graph-git) | Graph-native git primitives |
| [nusy-graph-query](https://crates.io/crates/nusy-graph-query) | SPARQL-style graph queries |
| **nusy-codegraph** | Code-as-graph with tree-sitter parsing |
| [nusy-kanban](https://github.com/hankh95/nusy-kanban) | Arrow-native kanban + HDD research |

---

## Performance

| Operation | Typical Time | Notes |
|----------|-------------|-------|
| Ingest Rust crate (1k files) | 30–60s | tree-sitter parsing is CPU-bound |
| Ingest Python project (1k files) | 20–40s | Partial support — faster |
| Semantic search (10k nodes) | 100–300ms | Embedding lookup + cosine similarity |
| Query (100k nodes) | <10ms | Arrow columnar — O(result set) |

**Memory usage:** ~500MB for a 10k-node graph. Larger graphs scale linearly with node count.

**Known bottlenecks:**
- tree-sitter parsing is single-threaded per file — use `--jobs` flag for parallelism
- `extract_call_edges` text scanning is O(source_lines × callable_count) — prefer `scip_calls()` for production accuracy
- Semantic search requires embedding computation on first ingest (~1s/100 nodes)

---

## Troubleshooting

**`error: failed to run tree-sitter parser`**
- Ensure tree-sitter grammars are available for the language: `--language rust` or `--language python`
- Check that source files have valid UTF-8 encoding

**`error: no such crate or crate not published`**
- `nusy-codegraph` requires `nusy-arrow-core`, `nusy-arrow-git`, `nusy-graph-query`, and `noesis-ship` from crates.io
- Run `cargo update` to ensure all NuSy dependencies are resolved

**`ingest hangs or OOM on large codebase`**
- Use `--jobs N` to parallelize parsing across N files
- For codebases >50k nodes, consider ingesting incrementally by module
- Disable embedding computation with `--no-embed` if memory-constrained

**Call edge recall is low**
- `extract_call_edges` uses text scanning — misses dynamic dispatch, indirect calls, and calls via variables
- For production-accurate call graphs, pre-compute SCIP data and use `extract_scip_call_edges()`
- Documented limitation: known recall gap for indirect/dynamic calls

---

## Limitations

- **Python support is partial** — only functions and classes are extracted. Type hints,
  decorators, and complex control flow are not yet parsed.
- **Only Rust and Python** — no TypeScript, Go, or other languages.
- **Call edges use text scanning** — `extract_call_edges` is best-effort. Use `scip_calls()`
  for production-accurate call graphs.
- **No incremental parsing** — re-ingestion is a full re-parse. Large codebases pay
  the full tree-sitter cost on every update.

---

## License

MIT
