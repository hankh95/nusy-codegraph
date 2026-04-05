# Changelog

## [0.14.3] — 2026-04-05

### Fixed
- README: Document 6-step ingestion pipeline with NameResolver/RustModuleResolver distinction
- README: Add performance characteristics (ingestion time, memory, bottlenecks)
- ingest: Prefer `extract_scip_call_edges` over text-scanning when rust-analyzer is available

### Changed
- `extract_call_edges` now falls back to text scanning only when no SCIP data — improved call edge recall for Rust

## [0.14.2] — 2026-04-01

Initial FOSS release.
