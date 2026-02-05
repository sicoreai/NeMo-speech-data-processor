# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

NeMo Speech Data Processor (SDP) is a Python toolkit for processing speech datasets. It provides a modular, manifest-based pipeline where each "processor" reads a NeMo-style JSON manifest (JSONL format), applies transformations, and outputs a new manifest.

## Common Commands

### Installation
```bash
pip install -r requirements/main.txt
# Optional features:
pip install -r requirements/ipl.txt         # Iterative Pseudo-Labeling
pip install -r requirements/huggingface.txt # HuggingFace integration
pip install -r requirements/tts.txt         # TTS features
```

### Running Tests
```bash
pip install -r requirements/tests.txt
python -m pytest tests/                                    # All tests
python -m pytest tests/test_cfg_runtime_tests.py           # Tests without NeMo
python -m pytest tests/test_cfg_end_to_end_tests.py        # End-to-end tests
python -m pytest tests/ -v                                 # Verbose
python -m pytest tests/ --cov=sdp --cov-report=term-missing  # With coverage
```

### Code Formatting
```bash
pip install pre-commit && pre-commit install
pre-commit run --all-files
black sdp/ --skip-string-normalization --line-length=119
isort sdp/ --profile black
```

### Running SDP
```bash
python main.py \
  --config-path="dataset_configs/<lang>/<dataset>/" \
  --config-name="config.yaml" \
  processors_to_run="all" \
  data_split="train" \
  workspace_dir="<output_directory>"
```

## Architecture

### Pipeline Model
```
Input Data → Processor 1 → Processor 2 → ... → Processor N → Output
                ↓              ↓                    ↓
           Manifest 1    Manifest 2           Manifest N
```

Each processor reads a JSONL manifest, transforms entries, and outputs a new manifest.

### Key Components

- **`main.py`**: Entry point using Hydra for configuration management
- **`sdp/run_processors.py`**: Orchestrates processor execution, manages Dask client, links manifest files
- **`sdp/processors/base_processor.py`**:
  - `BaseProcessor`: Abstract base for all processors
  - `BaseParallelProcessor`: Adds parallel processing via Dask or multiprocessing
- **`sdp/utils/import_manager.py`**: Dynamically loads processor classes

### Processor Categories
- `sdp/processors/datasets/`: Create manifests from raw datasets (LibriSpeech, MLS, MCV, etc.)
- `sdp/processors/inference/`: Run ASR/NLP models
- `sdp/processors/modify_manifest/`: Filter, transform, split, combine manifests
- `sdp/processors/langs/`: Language-specific text processing
- `sdp/processors/ipl/`: Iterative pseudo-labeling
- `sdp/processors/huggingface/`: HuggingFace integration
- `sdp/processors/tts/`: TTS processing

### Manifest Format
```json
{"audio_filepath": "path/to/audio.wav", "text": "transcription", "duration": 3.45, ...}
```

### Configuration
- Hydra + OmegaConf for YAML-based configuration
- Custom resolvers: `${subfield:node,field}`, `${not(value)}`, `${equal(field,value)}`
- Configs in `dataset_configs/` organized by language/dataset

## Creating Custom Processors

Extend `BaseParallelProcessor` for most use cases:

```python
from sdp.processors.base_processor import BaseParallelProcessor, DataEntry

class CustomProcessor(BaseParallelProcessor):
    def process_dataset_entry(self, entry):
        # Transform entry, return list of DataEntry objects
        return [DataEntry(data=transformed_data)]

    def finalize(self, metrics):
        # Optional post-processing
        pass
```

Register in `sdp/processors/__init__.py` or use full module path in config `_target_`.

## Parallelization

Processors support two backends:
- **Dask** (default): Distributed computing with web dashboard
- **Multiprocessing**: Fallback using joblib

Control via `use_dask: true/false` in config (global or per-processor).

## Code Style

- Black formatting: line length 119, skip string normalization
- isort with Black profile
- Pre-commit hooks enforce YAML validation, case conflicts, private key detection
