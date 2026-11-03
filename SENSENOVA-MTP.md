# SenseNova-Loop MTP fork

This branch is a fork of
[`GuideboardLabs/llama.cpp-sensenova-loop`](https://github.com/GuideboardLabs/llama.cpp-sensenova-loop)
with MTP export and runtime support for SenseNova-Loop.

## Why this fork exists

Upstream `fad174e` supports the base `sensenova_loop` architecture, but:

- its converter drops the MTP head (`no_mtp = True`);
- its runtime has the MTP tensor loader and `graph_mtp`, but does not enable
  the MTP memory context for `LLM_ARCH_SENSENOVA_LOOP`.

A GGUF with MTP therefore loads on the clean fork for normal generation, but
`--spec-type draft-mtp` crashes during MTP context creation.

## Changes

- `conversion/sensenova.py`: export the MTP block as the nextn layer after
  the virtual loop blocks; write `nextn_predict_layers` and the recurrent
  layer flags; keep `--no-mtp` available.
- `src/llama-model.cpp`: include `LLM_ARCH_SENSENOVA_LOOP` in the
  `mtp_on_hybrid_qwen` memory path.

The base model and architecture implementation are unchanged.

## Build

`LLAMA_BUILD_SERVER=ON` is required for `llama-cli` in this fork.

### CUDA

```bash
cmake -B build \
  -DGGML_CUDA=ON \
  -DGGML_VULKAN=OFF \
  -DLLAMA_BUILD_SERVER=ON \
  -DLLAMA_CURL=OFF \
  -DCMAKE_BUILD_TYPE=Release
cmake --build build --target llama-cli -j"$(nproc)"
```

### Vulkan

```bash
cmake -B build \
  -DGGML_CUDA=OFF \
  -DGGML_VULKAN=ON \
  -DLLAMA_BUILD_SERVER=ON \
  -DLLAMA_CURL=OFF \
  -DCMAKE_BUILD_TYPE=Release
cmake --build build --target llama-cli -j"$(nproc)"
```

### CPU only

```bash
cmake -B build \
  -DGGML_CUDA=OFF \
  -DGGML_VULKAN=OFF \
  -DLLAMA_BUILD_SERVER=ON \
  -DLLAMA_CURL=OFF \
  -DCMAKE_BUILD_TYPE=Release
cmake --build build --target llama-cli -j"$(nproc)"
```

## Run with MTP

```bash
llama-cli \
  -m SenseNova-Loop-9B-Preview-MERNIK-5300-SSMF32.gguf \
  -ngl 99 -c 1024 -st \
  --temp 1.0 --top-p 0.95 --top-k 20 --min-p 0.00 \
  --spec-type draft-mtp --spec-draft-n-max 2 \
  -p "Say hello in one short sentence."
```

Without `--spec-type draft-mtp`, the same file runs as a normal text model.

## Verification

- Clean upstream `fad174e`: normal generation loads and runs.
- Clean upstream `fad174e` + `--spec-type draft-mtp`: segfaults.
- This fork: MTP generation runs.
