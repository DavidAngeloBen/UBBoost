First of all: Feel free to copy this idea or help me out.
I crafted an example using Codex to modify llama.cpp and it works wonderful for RTX 2080 and AMD 5900x 64GB system memory - about 2x - 10x prefill prompt processing, the larger the context, the better!

So heres what I thought.
Using large ubatch speeds up prefill tremendously, but it needs lots of vram to do so.
We need to free vram!
My idea is this:
We split prompt processing and token generation.
For prefill or prompt processing, we load all but 1 dense layer into cpu (or system ram?)
and use the freed vram to increase ubatch as large as possible and disable MTP (didnt work out yet)
After processing/prefilling, we load all the layers and draft to gpu to start generating token.
I implemented this and it works out great and even does save the checkpoints, but, we had to split runtime.
Probaly we could do this in a single runtime improving loading offloading time vastly without relying on using split runtimes or loading one model,
then unload and load the other model, which is what I think it is doing using two runtimes. (Codex did it, not me, he said it is a big task to rework the runtime :o)
Now, maybe you can help me out doing the big task, or improve my idea. Feel free to copy this idea or help me out.

# UBBoost
UBBoost is a private fork of official `ggml-org/llama.cpp`)
UBBoost is focused on faster prompt processing in `llama-server` for vram constraint use cases.

# What UBBoost Adds

##UBBoost adds a prompt-processing high ubatch size in ubatch boost runtime for `llama-server`:

- prompt processing can run with a large temporary ubatch
- token generation returns to the normal/default runtime
- the default runtime can use different GPU layer and ubatch settings
- the connection is kept alive during runtime reload to read PP speed but this raises an issue: retry doesnt work if model sometimes fails to generate
- `--promptprocessing-ubatchboost-spec-draft-off` is intended to disable draft during prefill / not working yet |

# What it does:
Temporarily changes ubatch size and gpu layer in `INITIAL` first prompt processing,
but keeps default values for following prompts via flags:
`--promptprocessing-ubatchboost-size n`
` --promptprocessing-ubatchboost-gpu-layers`
`--promptprocessing-ubatchboost-n-cpu-moe`
`n --tokengeneration-no-warmup`

# How to use:
Use high ubatch size using  `--promptprocessing-ubatchboost-size n` for PP,
while tweaking `--promptprocessing-ubatchboost-gpu-layers n` with lower numbers, `--promptprocessing-ubatchboost-n-cpu-moe` with higher numbers
Use low ubatch - default `-ub 512` - to free vram to leave space for more gpu layers and less MoE experts offloading.
If you need different MoE CPU offload between prompt processing and token generation, keep normal `--n-cpu-moe` for the default runtime and use `--promptprocessing-ubatchboost-n-cpu-moe` for the UBB runtime.

## Example:
| Flag | Value |
|------|-------|
|`-ub`|512|
|`--promptprocessing-ubatchboost-size`|2816|
|`-ngl`|99|
|`--promptprocessing-ubatchboost-gpu-layers`|1|
|`--n-cpu-moe`|20|
|`--promptprocessing-ubatchboost-n-cpu-moe`|40|

This way, you may be able to keep your Q8 + MTP full model in gpu. After INITIAL PREFILL, it will use slower PP while reusing checkpoints. If at somepoint need to reprocess for any reason, like compacting context. It will trigger UBB again, also after the first compacting, it will need to read the context again and trigger UBB.

## Default Test Settings

| Flag | Value |
|------|-------|
|`--promptprocessing-ubatchboost-size`|2816|
|`--promptprocessing-ubatchboost-gpu-layers`|1|
|`--tokengeneration-no-warmup`|
|`--spec-type`|draft-mtp|
| `--threads` | 23 |
| `--gpu-layers` / `-ngl` | 99, temporarily changed with UBB flag  |
| `--n-cpu-moe` | 40 |
| `--promptprocessing-ubatchboost-n-cpu-moe` | 40 |
| `--no-mmap`|
| `-c` | 131072 |
|`-b`|11264|
| `-ub` | 512 |
| `-ctk` | q4_0 |
| `-ctv` | q4_0 |
|`--spec-draft-n-max`| 2 |

VRAM usage was about 7-7.5GB on an RTX 2080 8GB.

## Current UBB Flags

```text
--promptprocessing-ubatchboost-size n (as high as vram allowes by keeping dense layers in cpu ise flag below)
--promptprocessing-ubatchboost-gpu-layers n (n=1 recommended)
--promptprocessing-ubatchboost-n-cpu-moe n (UBB-only MoE CPU offload; normal --n-cpu-moe stays default runtime)
--promptprocessing-ubatchboost-spec-draft-off
--tokengeneration-no-warmup (for faster model switching)
 needs testing/improvement --promptprocessing-ubatchboost-keep // keep high ubatch model loading in a session past the first PP
```

## Branch And Changed Files

| Item | Value |
|------|-------|
| Fork name | `UBBoost` |
| Base project | official `ggml-org/llama.cpp` |
| Build target | `llama-server` |
| Runtime manager used for tests | `llama-swap` |
| Main config file | `config.yaml` |

| Changed file | Purpose |
|--------------|---------|
| `common/common.h` | Added UBB/default-runtime parameter fields. |
| `common/arg.cpp` | Added UBB CLI flags and token-generation warmup flag. |
| `tools/server/server-context.cpp` | Implemented UBB runtime loading, default runtime restore, no-re-eval prompt cache handling, target-only checkpoint fallback, UBB tail checkpoints, and MTP/spec draft handling. |
| `tools/llama-bench/llama-bench.cpp` | Added custom speculative benchmark arguments and draft-MTP/ngram acceptance reporting. |

## llama-swap
This repo includes a root `config.yaml` for `llama-swap` to use my bench setup with llama-swap
Start `llama-swap` from a directory where `llama-server.exe`, `llama-swap.exe`, and the model file exist, or adjust paths in the YAML for your local layout.

## Model Used In Testing

```text
Qwen3.6-35BA3B-MTP-Q8_0.gguf
```
## UBB Benchmark Test Chart

### System Specifications

| Component | Specification |
|-----------|---------------|
| CPU | R9 5900X |
| GPU | RTX 2080 8GB |
| RAM | 4x 3600MHz 64GB DDR4, 2 Channel |
| Mainboard | 2 Channel |
| Model | Qwen3.6-35BA3B-MTP-Q8_0.gguf, 37GB |

## Benchmark Comparison Table

| Comparison Pair | Configuration | MTP | UBB | Prefill Speed | Prefill Time | Token Gen Speed | Key Flags |
|---|---------------|-----|-----|---------------|--------------|-----------------|-----------|
| 1 | UBB enabled raw batch speed | :x: | :heavy_check_mark: | 1288.8 T/s | 10s/batch | ~18 T/s | `--promptprocessing-ubatchboost-size 6144`, `-b 12888` |
| - | UBB First Token | :x: | :heavy_check_mark: |   400 T/s | reload included | ~18.5 T/s | 20-30K context, this minimum speed at lower size context. With 4x context, the penalty is diminishingly small|
| 1 | Reference | :x: | :x: | 375 T/s | 30s/batch |   ~19.5 T/s | `-ub 2816`, `-b 11264` |
|-|
| 2 | UBB  | :heavy_check_mark: | :heavy_check_mark: | 230 T/s | 50s/batch | ~22.5 T/s | `--promptprocessing-ubatchboost-size 2816`, `-b 11264` |
| 2 | Ref. | :heavy_check_mark: | :x: | 120 T/s | 100s/batch |   ~24 T/s | `-ub 512`, `-b 12288`, `--spec-type draft-mtp` |
| 2 | Ref. higher def. ubatch | :heavy_check_mark: | :x: |   ~80 T/s | 132-141s/batch | ~24 T/s | VRAM spill close, `-ub 768 --spec-type draft-mtp` |

