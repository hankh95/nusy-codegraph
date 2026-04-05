# nusy-codegraph

**Code as a live Arrow object graph** -- parse source code into navigable, queryable Arrow RecordBatches using tree-sitter parsers.

## What It Does

`nusy-codegraph` parses Python and Rust source files into two Arrow tables:

- **CodeNodes**: functions, classes, modules, files -- every object in the code graph
- **CodeEdges**: calls, imports, inheritance, containment -- relationships between objects

All data is stored as Apache Arrow RecordBatches for zero-copy querying and versioned with `nusy-arrow-git` (graph-native git on Arrow).

## Quick Start

```rust
use nusy_codegraph::{parse_python_file, schema::{CodeNode, CodeEdge}, ingest_pipeline::verify_graph};

// Parse a Python file into CodeNodes + CodeEdges
let source = std::fs::read_to_string("my_module.py")?;
let result = parse_python_file(std::path::Path::new("my_module.py"), &source)?;

println!("{} nodes extracted", result.nodes.len());
for node in &result.nodes {
    println!("  {:?}: {}", node.kind, node.name);
}

// Parse Rust the same way
use nusy_codegraph::rust_parser::parse_rust_file;

let rust_source = std::fs::read_to_string("src/lib.rs")?;
let rust_result = parse_rust_file(std::path::Path::new("src/lib.rs"), &rust_source)?;
```

## Binaries

### `nusy-codegraph-ingest`

Ingest a Cargo workspace or single crate into Parquet files.

```bash
# Ingest all crates in a Cargo workspace
nusy-codegraph-ingest ingest --workspace /path/to/nusy-product-team \
                              --output ./self-graph/

# Verify graph coherence
nusy-codegraph-ingest verify --graph ./self-graph/
```

Outputs two Parquet files:
- `nodes.parquet` -- all CodeNodes (functions, structs, modules, etc.)
- `edges.parquet` -- all CodeEdges (calls, imports, containment)

### `nusy-codegraph-query`

Run demonstration queries against a self-graph Parquet snapshot. Answers questions like "what does nusy-being call?" and "what crates have edges into nusy-arrow-core?"

```bash
nusy-codegraph-query research/shared/self-graph/
```

### `nusy-codegraph-service`

Starts a NATS service on `codegraph.cmd.*` exposing 8 code graph tools. Loads the graph from Parquet on startup and persists mutations back.

```bash
nusy-codegraph-service --graph-dir .codegraph \
                       --workspace . \
                       --nats-url nats://192.168.8.110:4222
```

### `nusy-mcp-bridge`

Exposes `nusy-codegraph-service` as a [Model Context Protocol](https://modelcontextprotocol.io/) (MCP) server so Claude Code and other MCP clients can query the code graph via JSON-RPC over stdio.

```bash
nusy-mcp-bridge --nats nats://192.168.8.110:4222 --services codegraph
```

In Claude Code `settings.json`:

```json
{
  "mcpServers": {
    "nusy-code": {
      "command": "nusy-mcp-bridge",
      "args": ["--nats", "nats://192.168.8.110:4222", "--services", "codegraph"]
    }
  }
}
```

## Architecture

```
Python/Rust source
       │
       ▼
tree-sitter CST  (Concrete Syntax Tree)
       │
       ▼
CodeNodes + CodeEdges  (Arrow RecordBatches)
       │
       ▼
nusy-arrow-git  ──► versioned Parquet snapshots
       │
       ▼
nusy-graph-query  ──► hybrid semantic + graph search
```

### Node Kinds

| Kind | Language | Description |
|------|----------|-------------|
| `File`, `Module`, `Class`, `Function` | Both | Generic code objects |
| `RustFn`, `RustStruct`, `RustTrait`, `RustImpl`, `RustMod` | Rust | Rust-specific constructs |
| `RustUse`, `RustMacro`, `RustConst` | Rust | Imports and declarations |
| `PythonFunction`, `PythonClass`, `PythonDecorator` | Python | Python-specific constructs |

### Edge Predicates

| Predicate | Meaning |
|-----------|---------|
| `calls` | Function/method invocation |
| `imports` | Module/crate import |
| `contains` | Parent-child containment |
| `inherits` | Class inheritance |
| `param` | Function parameter |
| `raises` | Exception raised |
| `decorates` | Python decorator applied |

## Dependencies

- `nusy-arrow-core` -- Arrow-native graph store
- `nusy-arrow-git` -- Graph-native git primitives
- `nusy-graph-query` -- Hybrid semantic + graph search
- `noesis-ship` -- NATS communication
- `tree-sitter` + `tree-sitter-python` + `tree-sitter-rust` -- Parsing

## License

MIT
