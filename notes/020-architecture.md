# Architecture Notes

## Entry Point & Orchestration

### `main.py` (52 lines)
- Decorated with `@hydra.main()` for config management
- Hacks `sys.argv` to disable Hydra's output directory/logging (keeps working dir clean)
- Defaults `use_dask: True` if not specified in config
- Also supports `mode: 'update_imports'` for the ImportManager workflow

### `sdp/run_processors.py` -- the orchestrator
Key flow in `run_processors(cfg)`:
1. Optionally runs ImportManager to sync `__init__.py` imports with config
2. Detects if Dask is available; falls back to multiprocessing if not
3. Parses `processors_to_run` (supports Python slice notation: `"0:"`, `"2:5"`, `"-1"`)
4. Filters out processors with `should_run: False`
5. **Auto-links manifests**: if a processor lacks `output_manifest_file`, creates a temp file; if next processor lacks `input_manifest_file`, links it to previous output
6. Instantiates processors via `hydra.utils.instantiate(processor_cfg)` -- this is how `_target_` works
7. Runs `.test()` on all processors BEFORE executing any (fail-fast on test cases)
8. Creates one shared Dask client if any processor needs it
9. Runs processors sequentially: `proc.process()`
10. Temp directory cleaned up after all processing

### Custom OmegaConf Resolvers (registered in run_processors.py)
- `${subfield:node,field}` -- extract nested field from a config node
- `${not(value)}` -- boolean negation
- `${equal(field,value)}` -- equality check

## Processor Hierarchy

```
BaseProcessor (ABC)
├── process()          -- abstract, must implement
├── test()             -- optional runtime tests
├── input_manifest_file, output_manifest_file
│
├── BaseParallelProcessor
│   ├── process()      -- delegates to _process_with_dask() or _process_with_multiprocessing()
│   ├── prepare()      -- hook for pre-processing setup
│   ├── read_manifest() -- returns Dask bag OR generator depending on mode
│   ├── process_dataset_entry(entry) -- abstract, per-entry transform
│   ├── finalize(metrics) -- post-processing, logs stats
│   ├── test()         -- runs test_cases through process_dataset_entry
│   └── _chunk_manifest() -- splits input into in_memory_chunksize chunks (multiprocessing only)
│
└── LegacyParallelProcessor (for reference, not actively used)
```

### DataEntry dataclass
```python
@dataclass
class DataEntry:
    data: Optional[Dict]  # None = drop entry
    metrics: Any = None
```

Every `process_dataset_entry()` returns `List[DataEntry]`:
- Return `[DataEntry(data=transformed)]` for 1-to-1 mapping
- Return multiple for 1-to-many (e.g., splitting utterances)
- Return `[DataEntry(data=None)]` to drop an entry

## Processor Categories

### Dataset Creators (`sdp/processors/datasets/`)
Create initial manifests from raw data sources. Override `read_manifest()` to read from non-manifest sources (raw transcripts, directories, etc.).

Examples: `CreateInitialManifestLibrispeech`, `CreateInitialManifestMCV`, `CreateInitialManifestMLS`

### Inference (`sdp/processors/inference/`)
Run ML models on manifest entries:
- **ASR**: NeMo ASR, FasterWhisper, HuggingFace Transformers
- **NLP**: Punctuation/Capitalization (NeMo), FastText language ID
- **LLM**: vLLM inference (e.g., Qwen for PnC restoration, EuroLLM for translation)
- **Quality**: CometoidWMT quality estimation (pymarian)
- **Utils**: Whisper hallucination detection, RTTM segmentation

### Manifest Modifiers (`sdp/processors/modify_manifest/`)
The workhorses -- organized by pattern:

**data_to_data.py** -- transform fields without dropping:
- `SubRegex`, `SubMakeLowercase`, `NormalizeText`, `GetAudioDuration`
- `LambdaExpression` -- evaluate arbitrary Python expressions per entry
- `CountNumWords`, `GetWER`, `SplitLineBySentence`
- `CharacterHistogramLangValidator`, `EstimateBandwidth`

**data_to_dropbool.py** -- decide whether to keep/drop entries:
- `DropHighLowDuration`, `DropHighWER`, `DropHighCER`
- `DropIfRegexMatch`, `DropNonAlphabet`, `DropDuplicates`
- `PreserveByValue` -- keep entries matching operator condition

**common.py** -- structural operations:
- `KeepOnlySpecifiedFields`, `DropSpecifiedFields`, `RenameFields`
- `AddConstantFields`, `DuplicateFields`
- `CombineSources`, `ApplyInnerJoin`, `SortManifest`
- `SplitOnFixedDuration`

### File Management (`sdp/processors/manage_files/`)
- `FfmpegConvert`, `SoxConvert` -- audio format conversion
- `ExtractTar` -- extract archives
- `RemoveFiles` -- clean up intermediate files
- `ConvertToTarredAudioDataset` -- shard into tar files for NeMo training

### Language-Specific (`sdp/processors/langs/`)
Text normalization rules per language.

### Toloka Integration (`sdp/processors/toloka/`)
Crowdsourcing workflow: create projects, pools, tasks, download responses, accept/reject.

## Parallelization

Two backends, chosen per-processor:

### Dask (default)
- `read_manifest()` returns a Dask bag via `db.read_text(...).map(json.loads)`
- Processing: `bag.map(process_dataset_entry).flatten().compute()`
- Single shared Dask client created in run_processors, passed to processors
- Workers = `psutil.cpu_count(logical=False)`

### Multiprocessing (fallback)
- `read_manifest()` returns a Python generator
- `_chunk_manifest()` groups entries into `in_memory_chunksize` chunks
- Each chunk processed via `joblib.Parallel(n_jobs=max_workers, backend="multiprocessing")`
- Results written incrementally per chunk

### Control
- Global: `use_dask: True/False` in config root
- Per-processor: `use_dask:` field in individual processor config overrides global
- If Dask not installed, all processors fall back to multiprocessing automatically

## Import Management

`sdp/utils/import_manager.py` provides `ImportManager`:
- Reads `_target_` fields from YAML config
- Generates minimal `__init__.py` with only required imports
- Purpose: avoid installing dependencies for unused processors
- Enabled via `use_import_manager: True` in config
- The default `sdp/processors/__init__.py` imports ~70 processor classes statically

## Configuration System

Hydra + OmegaConf YAML configs in `dataset_configs/<language>/<dataset>/`:
- `_target_` field specifies processor class (either short name from `__init__.py` or full module path)
- Variable interpolation: `${workspace_dir}`, `${data_split}`, `${params.source_lang}`
- `???` means required (must be provided at runtime)
- `should_run: False` to skip a processor
- `test_cases` for inline runtime validation
- Granary config uses `partials/` subdirectories for language-specific regex and prompt configs

## File Layout

```
main.py                              # Entry point
sdp/
  run_processors.py                  # Orchestration
  logging.py                         # Logger (name: "sdp")
  processors/
    __init__.py                      # All processor imports (~70 classes)
    base_processor.py                # BaseProcessor, BaseParallelProcessor, DataEntry
    datasets/                        # Per-dataset manifest creators
    inference/                       # ML model inference
      asr/                           # ASR engines (nemo, faster_whisper, transformers)
      nlp/                           # NLP (punctuation, fasttext)
      llm/                           # LLM inference (vllm)
      quality_estimation/            # Translation quality
    modify_manifest/                 # Manifest transformations
      common.py                      # Structural ops (keep/drop fields, rename, combine)
      data_to_data.py                # Field transforms
      data_to_dropbool.py            # Filter/drop logic
    manage_files/                    # Audio conversion, extraction, tarring
    langs/                           # Language-specific text processing
    toloka/                          # Crowdsourcing integration
    huggingface/                     # HuggingFace dataset import
    tts/                             # TTS processing
    ipl/                             # Iterative pseudo-labeling
  utils/
    import_manager.py                # Dynamic import management
    apply_operators.py               # Generic operator application
    common.py                        # Shared utilities (ffmpeg_convert, etc.)
    metrics_computation.py           # WER/CER computation
    edit_spaces.py                   # Space normalization
    get_diff.py                      # Text diff utilities
dataset_configs/                     # YAML pipeline configs by language
tests/                               # pytest test suite
```
