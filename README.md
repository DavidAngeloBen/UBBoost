# UBBoost

UBBoost is a private fork of official `ggml-org/llama.cpp` focused on faster prompt processing in `llama-server`.

This fork is based on official llama.cpp and is intended to be used together with the MTP work/pull for Qwen3.6 MTP models. It also documents the test setup used with `llama-swap`.

## What UBBoost Adds

UBBoost adds a prompt-processing ubatch boost runtime for `llama-server`:

- prompt processing can run with a large temporary ubatch
- token generation returns to the normal/default runtime
- the default runtime can use different GPU layer and ubatch settings
- MTP/spec draft can be disabled during prompt processing to free VRAM
- the connection is kept alive during runtime reload
- prompt cache reuse is protected so continuations do not full re-evaluate the prompt

## Current UBB Flags

```text
--promptprocessing-ubatchboost-size 6144
--promptprocessing-ubatchboost-gpu-layers 1
--promptprocessing-ubatchboost-spec-draft-off
--promptprocessing-ubatchboost-keep
--tokengeneration-no-warmup
```

Normal llama-server arguments still describe the default token-generation runtime. The `--promptprocessing-ubatchboost-*` arguments describe only the temporary prompt-processing boost runtime.

## Branch And Changed Files

| Item | Value |
|------|-------|
| Fork name | `UBBoost` |
| Base project | official `ggml-org/llama.cpp` |
| Branch used locally | `master` |
| Build target | `llama-server` |
| Runtime manager used for tests | `llama-swap` |
| Main config file | `config.yaml` |

| Changed file | Purpose |
|--------------|---------|
| `common/common.h` | Added UBB/default-runtime parameter fields. |
| `common/arg.cpp` | Added UBB CLI flags and token-generation warmup flag. |
| `tools/server/server-context.cpp` | Implemented UBB runtime loading, default runtime restore, no-re-eval prompt cache handling, target-only checkpoint fallback, UBB tail checkpoints, and MTP/spec disabling when draft KV is missing. |
| `tools/server/server-queue.cpp` | Added smoother streaming keep-alive behavior while runtime reload is happening. |
| `tools/server/server-queue.h` | Added queue/reader support needed by the keep-alive path. |

## llama-swap

This repo includes a root `config.yaml` for `llama-swap`.

The YAML is intentionally only the test matrix from this README. It does not include model files, build outputs, logs, or local backup files.

Start `llama-swap` from a directory where `llama-server.exe`, `llama-swap.exe`, and the model file exist, or adjust paths in the YAML for your local layout.

## Model Used In Testing

Primary model in the benchmark chart:

```text
Qwen3.6-35BA3B-MTP-Q8_0.gguf
```

Other local test models existed, but model files are not part of this Git upload. `*.gguf` stays ignored.

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

| # | Configuration | MTP | UBB | Prefill Speed | Prefill Time | Token Gen Speed | Key Flags |
|---|---------------|-----|-----|---------------|--------------|-----------------|-----------|
| 1 | UBB High Boost | OFF | ON | 1288.8 T/s | 10s/batch | ~17-18.5 T/s | `--promptprocessing-ubatchboost-size 6144`, `-b 12888` |
| 2 | UBB Standard | OFF | ON | ~400 T/s | reload accounted | ~17-18.5 T/s | 20-30K context |
| 3 | UBB Alt Config | OFF | ON | 230 T/s | 50s/batch | ~22.5 T/s | `--promptprocessing-ubatchboost-size 2816`, `-b 11264` |
| 4 | Reference Best | OFF | OFF | 375 T/s | 30s/batch | 17-19.5 T/s | `-ub 2816`, `-b 11264` |
| 5 | MTP ON, UBB OFF | ON | OFF | 120 T/s | 100s/batch | ~24 T/s | `-ub 512`, `-b 12288` |
| 6 | MTP ON, UBB OFF Slow | ON | OFF | 80 T/s | 132-141s/batch | ~24 T/s | VRAM spill close |

## Detailed Test Configurations

### Test 1: UBB High Boost, MTP OFF

| Parameter | Value |
|-----------|-------|
| MTP | OFF |
| UBB | ON |
| Prefill Speed | 1288.8 T/s |
| Prefill Time | 10s per batch, pure prefill, model reload unaccounted |
| Token Generation | ~17-18.5 T/s |
| Context | 20-30K |

Flags:

```text
-b 12888
--promptprocessing-ubatchboost-size 6144
--promptprocessing-ubatchboost-gpu-layers 1
--promptprocessing-ubatchboost-spec-draft-off
--tokengeneration-no-warmup
```

Real-world prefill at 20-30K context was about 400 T/s with model reload penalty accounted. Larger contexts benefit more from UBB.

### Test 2: UBB Standard, MTP OFF

| Parameter | Value |
|-----------|-------|
| MTP | OFF |
| UBB | ON |
| Prefill Speed | 230 T/s |
| Prefill Time | 50s per batch, pure prefill |
| Token Generation | ~22.5 T/s |

Flags:

```text
-b 11264
--promptprocessing-ubatchboost-size 2816
--promptprocessing-ubatchboost-gpu-layers 1
--tokengeneration-no-warmup
```

### Test 3: Reference, MTP OFF, UBB OFF

| Parameter | Value |
|-----------|-------|
| MTP | OFF |
| UBB | OFF |
| Prefill Speed | 375 T/s |
| Prefill Time | 30s per batch |
| Token Generation | 17-19.5 T/s |
| Notes | Slows down when batch split at continuing prompts |

Flags:

```text
-ub 2816
-b 11264
```

`-ub 3072` may be too much for 8GB VRAM.

### Test 4: MTP ON, UBB OFF

| Parameter | Value 1 | Value 2 |
|-----------|---------|---------|
| MTP | ON | ON |
| UBB | OFF | OFF |
| Prefill Speed | 120 T/s | 80 T/s |
| Prefill Time | 100s/batch | 132-141s/batch |
| Token Generation | ~24 T/s | ~24 T/s |
| ubatch | 768 | 512 |
| Batch Size | 12288 | 12288 |
| VRAM Usage | Non-spilling | ~7.4GB, close to spill |

Flags:

```text
--spec-type mtp
-ub 512
-b 12288
```

### Test 5: MTP ON, UBB ON

| Parameter | Value |
|-----------|-------|
| MTP | ON |
| UBB | ON |
| Prefill Speed | 230 T/s |
| Prefill Time | 50s per batch |
| Token Generation | ~22.5 T/s |

Flags:

```text
-m Qwen3.6-35BA3B-MTP-Q8_0.gguf
-b 11264
--promptprocessing-ubatchboost-size 2816
--promptprocessing-ubatchboost-gpu-layers 1
--spec-type mtp
--spec-draft-n-max 2
--tokengeneration-no-warmup
```

## Default Test Settings

| Flag | Value |
|------|-------|
| `--parallel` | 1 |
| `--top-k` | 20 |
| `--min-p` | 0.0 |
| `--temperature` | 0.6 |
| `--log-prefix` | enabled |
| `--log-timestamps` | enabled |
| `--threads` | 23 |
| `--gpu-layers` / `-ngl` | 99, tweaked during UBB |
| `--n-cpu-moe` | 40 |
| `--no-mmap` | enabled |
| `-c` | 131072 |
| `-ub` | 512 default runtime unless a test says otherwise |
| `-ctk` | q4_0 |
| `-ctv` | q4_0 |

VRAM usage was about 7-7.5GB on an RTX 2080 8GB.

## Key Observations

| Observation | Detail |
|-------------|--------|
| UBB benefit | Larger context makes UBB more useful. |
| MTP advantage | MTP can keep token-generation speed higher, but it costs VRAM and prompt speed. |
| VRAM constraint | The 8GB RTX 2080 VRAM limit is the main tuning limit. |
| Model reload | Reload cost matters less for very large prompts. |
| Optimal ubatch | Boost during prompt processing, default lower ubatch during token generation. |
| Best TG speed | About 24 T/s with MTP ON in the tested setup. |
| Best prefill | 1288.8 T/s burst with UBB high boost. |

## Known Issues And Notes

| Issue | Detail |
|-------|--------|
| Connection handling | For model reload testing, the connection was kept active to measure real-world prefill speed combined with reload time. |
| MTP and UBB | If `--promptprocessing-ubatchboost-spec-draft-off` is used, UBB cache restored into the default runtime is target-only. Speculative generation is disabled for that restored continuation instead of full prompt re-evaluating. |
| Git upload | Build outputs, model files, logs, and local backups are intentionally excluded. |
