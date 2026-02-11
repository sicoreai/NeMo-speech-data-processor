# Decisions

Key design decisions observed in the codebase and their rationale.

## 1. Hydra for Configuration

**Decision**: Use Hydra + OmegaConf instead of argparse or custom config parsing.

**Why**: Hydra provides structured YAML configs with variable interpolation, composition, and CLI overrides. This is critical because SDP pipelines can have 40+ processors each with many parameters. The `_target_` pattern lets Hydra instantiate processor objects directly from config.

**Trade-off**: Hydra's default behavior creates output directories and logs; `main.py` hacks `sys.argv` to suppress this (`hydra.run.dir=.`, `hydra.output_subdir=null`).

## 2. Sequential Pipeline with Intermediate Manifests

**Decision**: Processors run sequentially and write intermediate manifest files between each step.

**Why**:
- Enables resuming from any point via `processors_to_run: "5:"`
- Each intermediate `manifest_XX.json` serves as an audit trail
- Easy to debug by inspecting any intermediate file
- No complex DAG scheduler needed

**Trade-off**: More disk I/O and storage for intermediates. Mitigated by temp directories for processors without explicit output paths.

## 3. Dual Parallelization: Dask + Multiprocessing

**Decision**: Support both Dask (distributed) and joblib multiprocessing backends.

**Why**: Dask provides distributed computing with a monitoring dashboard, good for large-scale production runs. Multiprocessing via joblib is simpler and has fewer dependencies. Auto-fallback to multiprocessing if Dask isn't installed.

**How it works**: One shared Dask client is created in `run_processors()` and passed to all processors that need it. Per-processor `use_dask` flag allows mixing backends in a single pipeline.

**Note**: The Granary config sets `use_dask: False` -- suggesting multiprocessing is more reliable for GPU-heavy inference workloads.

## 4. DataEntry Pattern (Not In-Place Mutation)

**Decision**: `process_dataset_entry()` returns `List[DataEntry]` rather than modifying entries in place.

**Why**:
- Supports 1-to-many mappings (splitting utterances into segments)
- `DataEntry(data=None)` cleanly drops entries without special return codes
- `DataEntry.metrics` allows collecting per-entry statistics for `finalize()`
- Compatible with both Dask and multiprocessing (pure function, no shared state)

**Constraint**: Cannot modify processor instance attributes inside `process_dataset_entry()` -- would cause undefined behavior in parallel contexts. Counters/accumulators go in `metrics`.

## 5. Runtime Test Cases in Config

**Decision**: Processors support inline `test_cases` in YAML config that are checked before any processing begins.

**Why**: Fail-fast validation. If a regex substitution doesn't produce expected output, the user finds out immediately rather than after hours of processing. Tests run via `processor.test()` before `processor.process()`.

**Format**:
```yaml
test_cases:
  - input: {text: "hello!"}
    output: {text: "hello."}
  - input: {text: "remove_me"}
    output: null  # null means entry should be dropped
```

## 6. Static Imports in __init__.py (with ImportManager Opt-In)

**Decision**: `sdp/processors/__init__.py` statically imports all ~70 processor classes. ImportManager is an opt-in alternative.

**Why**: Static imports allow using short names in `_target_` (e.g., `sdp.processors.SubRegex` instead of `sdp.processors.modify_manifest.data_to_data.SubRegex`). ImportManager exists because some processors have heavy dependencies (NeMo, vLLM, faster-whisper, fasttext) that users shouldn't need to install if they're not using those processors.

**Trade-off**: Static imports mean all dependencies must be installed; ImportManager solves this but rewrites `__init__.py` which is fragile.

## 7. LambdaExpression as Universal Glue

**Decision**: The `LambdaExpression` processor evaluates arbitrary Python expressions per manifest entry.

**Why**: Avoids creating one-off processors for simple field computations, boolean filters, and derived fields. Used heavily in the Granary pipeline for things like:
- `expression: entry.end - entry.start` (compute duration)
- `expression: (entry.language == "en") & (entry.language_probability >= 0.7)` (filter)
- `filter: True` parameter to drop entries where expression evaluates to False

**Risk**: Executes arbitrary Python. Should only be used with trusted configs.

## 8. Processor `should_run` Flag

**Decision**: Any processor can include `should_run: False` to be skipped at runtime.

**Why**: Allows conditional pipeline stages without duplicating entire config files. Combined with Hydra overrides, a single config can serve multiple use cases (e.g., `convert_to_audio_tarred_dataset.should_run: True/False`).

## 9. No In-Memory Pipeline (Disk-Based)

**Decision**: All manifest data passes through disk (JSONL files) between processors.

**Why**: Simplifies memory management for large datasets (millions of entries). Each processor can process data in chunks (`in_memory_chunksize`) without loading everything into RAM. Works naturally with Dask's distributed model.

**Trade-off**: Slower than in-memory for small datasets, but scales to production speech corpora.

## Open Questions / Areas to Investigate

- Why does `LegacyParallelProcessor` still exist alongside `BaseParallelProcessor`? It seems to be "for reference" per comments.
- The Dask implementation computes `bag.count()` before processing, which requires a full pass over the data. Is this necessary?
- `read_manifest()` in `BaseParallelProcessor` strips lines but the legacy version doesn't -- any edge case differences?
- ImportManager's `setup_import_hooks()` monkey-patches `yaml.safe_load` globally -- is this used anywhere?
