
## setup env

```bash
sudo apt-get update && sudo apt-get install -y sox libsox-fmt-all

uv venv --python 3.12

source .venv/bin/activate

uv pip install -r requirements/main.txt

uv pip install pytorch-lightning \
            "nvidia-cublas-cu12" \
            "nvidia-cudnn-cu12==9.*" \
            faster_whisper

export LD_LIBRARY_PATH=$(python -c "import nvidia.cublas.lib, nvidia.cudnn.lib; print(nvidia.cublas.lib.__path__[0] + ':' + nvidia.cudnn.lib.__path__[0])")

uv pip install "optree>=0.13.0" vllm
```

## process using granary config

```bash
SDP_DIR=/path/to/NeMo-speech-data-processor

python ${SDP_DIR}/main.py \
    --config-path ${SDP_DIR}/dataset_configs/multilingual/granary/ \
    --config-name config.yaml \
    input_manifest_file=/path/to/input_manifest.json \
    output_dir=/path/to/output/dir \
    sdp_dir=${SDP_DIR}
```    
