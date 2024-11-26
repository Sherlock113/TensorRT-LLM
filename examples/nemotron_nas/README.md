# Nemotron-NAS

This document shows how to convert and build a model generated by Nemotron-NAS, such as Llama-3_1-Nemotron-51B-Instruct, in TensorRT-LLM.

- [NemotronNas](#nemotron-nas)
  - [Overview](#overview)
  - [Support Matrix](#support-matrix---verify-with-omer--nave)
  - [Custom Layers](#custom-layers)
  - [Usage](#usage)
    - [Build TensorRT engine(s)](#build-tensorrt-engines)
  - [Runtime](#runtime)

## Overview

The TensorRT-LLM Nemotron-NAS implementation can be found in [tensorrt_llm/models/nemotron_nas/model.py](../../tensorrt_llm/models/nemotron_nas/model.py). The TensorRT-LLM Nemotron-NAS example code is located in [`examples/nemotron_nas`](./). There is one main file:

* [`convert_checkpoint.py`](./convert_checkpoint.py) to convert the model into tensorrt-llm checkpoint format.

## Support Matrix

  * FP16
  * BF16
  * FP8
  * Tensor parallelism
  * Pipeline parallelism

## Custom Layers

Nemotron-NAS offers the ability to replace both `attention` and `FFN` layers with either `Linear` or `NoOp` layers.
The `attention` layers can be replaced with `LinearAttention` (which eventually calls `tensorrt_llm/layers/Linear`).
Additionally, `attention` layers can also be replaced with `NoOpAttention` (which essentially returns 0, thus implementing a no-op operation).
The `LinearAttention` and `NoOpAttention` layers require no KV-cache.
Likewise, `FFN` layers can be replaced with either `LinearFFN` or `NoOpFFN`.

Different attention layers of the model can have a different number of key-value attention heads and different MLP layers can have different hidden sizes.

## About Pipeline Parallelism

Due the non-uniform architecture of the model, the different pipeline parallelism ranks might run different types of layers, resulting in a possibly unbalanced load between GPUs during inference.

## Usage

The TensorRT-LLM example code is located at [examples/nemotron_nas](./).
The `convert_checkpoint.py` script accepts Hugging Face weights as input, and builds the corresponding TensorRT engines.
The number of TensorRT engines depends on the number of GPUs used to run inference.

### Build TensorRT Engines

To build a TensorRT engine, you first need to obtain a Nemotron-NAS checkpoint in Hugging Face format. For example, [Llama-3_1-Nemotron-51B-Instruct](https://huggingface.co/nvidia/Llama-3_1-Nemotron-51B-Instruct).

The `trtllm-build` command builds TensorRT engines from a TensorRT-LLM checkpoint.
If no checkpoint directory is specified, TensorRT-LLM builds the engines with dummy weights.

The `trtllm-build` command has a variety of options.
In particular, the plugin-related options have two categories:

* Plugin options that require a data type, such as `gpt_attention_plugin`, you can:
    * explicitly specify `float16`, `bfloat16`, or `float32`, so that the plugins are enabled with the specified precision
    * implicitly specify `auto`, so that the plugins are enabled with the precision that is automatically inferred from the model dtype, the dtype specified in weight conversion


0. Optional: prepare environment variables
    ```bash
    export MODEL_DIR="~/models/huggingface/nemotron-nas"
    export TRT_CHECKPOINT_DIR="~/models/intermediate/nemotron-nas"
    export TRT_ENGINE_DIR="~/models/engines/nemotron-nas"
    export TP_SIZE=1
    export PP_SIZE=1
    ```
1. Clone the HF model:
    ```bash
    git clone https://huggingface.co/nvidia/Llama-3_1-Nemotron-51B-Instruct $MODEL_DIR
    ```

2. Convert the checkpoint:
  * For BF16/FP16:
    ```bash
    # Note, currently must provide --trust_remote_code flag
    python ./convert_checkpoint.py \
                  --model_dir $MODEL_DIR \
                  --output_dir $TRT_CHECKPOINT_DIR \
                  --dtype bfloat16 \
                  --tp_size $TP_SIZE \
                  --pp_size $PP_SIZE \
                  --trust_remote_code
    ```

  * **Alternatively**, Quantize to FP8:
    ```bash
    # Below is how to optionally consume the recommended dataset for optimal accuracy.
    # You may use any dataset you prefer, however.

    export DATASET_DIR="~/datasets/nemotron-nas"
    python ./calibration_utils.py $DATASET_DIR # download and transform the recommended dataset.

    python ../quantization/quantize.py \
                  --model_dir $MODEL_DIR \
                  --output_dir $TRT_CHECKPOINT_DIR \
                  --dtype bfloat16 \
                  --kv_cache_dtype fp8 \
                  --qformat fp8 \
                  --calib_dataset $DATASET_DIR
    ```

3. Build the engine:
    ```bash
    # Build the model engine using a single GPU and FP16
    trtllm-build --checkpoint_dir $TRT_CHECKPOINT_DIR \
                --output_dir $TRT_ENGINE_DIR \
                --gemm_plugin auto \
                --kv_cache_type paged
    ```

The conversion script supports additional models with variable GQA, such as [DeciLM-7B](https://huggingface.co/Deci/DeciLM-7B).

## Runtime

After you build the engine, you can use the engine with any TensorRT-LLM entrypoint or API.
For example, you can run inference with [examples/run.py](../run.py):

```bash
export MODEL_DIR="~/models/huggingface/nemotron-nas"
export TRT_ENGINE_DIR="~/models/engines/nemotron-nas"

python run.py --engine_dir $TRT_ENGINE_DIR --tokenizer_dir $MODEL_DIR --max_output_len 1024 ...

# for multi-GPU inference (engine must be built with either tp_size>1, pp_size>1, or both)
export NUM_GPUS=2
mpirun -n $NUM_GPUS --allow-run-as-root python run.py ...
```