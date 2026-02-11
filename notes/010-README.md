# Project Notes Index

## What is NeMo-SDP?

NeMo Speech Data Processor (SDP) is an NVIDIA-developed Python toolkit (v0.1.0, Apache 2.0) for processing speech datasets into NeMo-compatible formats. It's a pipeline system where configurable "processors" are chained together via YAML configs.

- Python 3.10+ officially supported
- Part of the NVIDIA NeMo ecosystem
- Repo: github.com/NVIDIA/NeMo-speech-data-processor

## Core Concept

Everything revolves around **JSONL manifests** -- each line is a JSON object like:
```json
{"audio_filepath": "path/to/audio.wav", "text": "transcription", "duration": 3.45}
```

Processors read a manifest, transform it, and write a new one. They chain together:
```
Raw Data -> Processor 0 -> manifest_00.json -> Processor 1 -> manifest_01.json -> ... -> final_manifest.json
```

## Setup & Commands

```bash
# Install
pip install -r requirements/main.txt

# Run a pipeline
python main.py \
  --config-path="dataset_configs/<lang>/<dataset>/" \
  --config-name="config.yaml" \
  processors_to_run="all" \
  data_split="train" \
  workspace_dir="<output_dir>"

# Tests
pip install -r requirements/tests.txt
python -m pytest tests/                          # all tests
python -m pytest tests/test_cfg_runtime_tests.py # no-NeMo tests
python -m pytest tests/test_data_to_data.py -v   # single test file

# Formatting
pip install pre-commit && pre-commit install
pre-commit run --all-files
black sdp/ --skip-string-normalization --line-length=119
isort sdp/ --profile black
```

## Key Dependencies

**Core**: hydra-core, omegaconf, joblib, dask/distributed, librosa, pandas, tqdm, jiwer
**Audio**: ffmpeg, sox, pydub, soundfile
**Optional**: nemo-toolkit (ASR), faster-whisper, vllm, fasttext, pymarian

Notably numpy is pinned to `>=1.26, <2.0` (1.x API assumptions).

## Supported Datasets

Configs exist for 10+ languages: Arabic, Armenian, English, Georgian, Italian, Kazakh, Portuguese, Spanish, Uzbek, plus multilingual (Granary pipeline for 25 EU languages).

Dataset sources include: LibriSpeech, MLS, Mozilla Common Voice (MCV), VoxPopuli, FLEURS, CORAAL, Earnings, HiFiTTS2, KSC2, MASC, MediaSpeech, MTEDX, YouTube Commons (via Granary).

## Existing Configs to Study

- `dataset_configs/english/librispeech/config.yaml` -- simple 4-step pipeline (download, convert, duration, lowercase)
- `dataset_configs/multilingual/granary/config.yaml` -- complex 47-step pipeline (ASR, LID, hallucination detection, PnC restoration, translation, quality estimation, tarring)
