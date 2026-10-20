# AMD Radeon 890M (48GB) Public Benchmark Evidence & Candidate Evaluation Dossier

**Target Card**: AMD Radeon 890M (48 GB Unified Memory, RDNA 3.5 / gfx1150)
**Primary Candidates**:
- Candidate A: `lfm2.5-2.6b` (`LiquidAI/LFM2.5-2.6B-GGUF@84022ce711b28455e8c4fc364ce68c00cf995875`)
- Candidate B: `qwen3.5-9b` (`unsloth/Qwen3.5-9B-GGUF@3885219b6810b007914f3a7950a8d1b469d598a5`)
**Launch Profiles**:
- `registry/launches/llama.cpp-vulkan-lfm2.5-2.6b-qad-q4-0-128k.json`
- `registry/launches/llama.cpp-vulkan-qwen3.5-9b-q4-k-m-128k.json`
**Card Definition**: `registry/cards/amd/radeon-890m-48gb.json`
**Evaluation Status**: Pre-Execution Technical Evidence & Candidate Evaluation Dossier

---

## 1. Executive Summary & Purpose

This technical document compiles verified public benchmark data, hardware architectural characteristics, exact memory calculations, and reproduction commands for deploying registered candidate models at 128K context (`131,072` tokens) on the AMD Radeon 890M integrated GPU (48 GB unified memory configuration).

This dossier supports Pull Request [#294](https://github.com/sybil-solutions/local-ai-registry/pull/294) as a complete, transparent engineering submission:
- **No physical six-gate run has been performed by this submission**. The candidate profiles and card definition are presented as an infrastructure proposal.
- **No synthetic or unmeasured run records** have been created in `data/lab-runs/`.
- **No unverified recipe file** has been committed to `registry/recipes/` without passing all six official gates (`load`, `chat`, `reasoning`, `tools`, `context`, `speed`).
- The bounty speed acceptance threshold specified in Issue #223 is **$\ge 15.0\text{ tok/s}$**. The speed gate is classified as **NOT VERIFIED**. Public estimates below 15 tok/s represent analytical projections and do not establish an actual measured failure.

---

## 2. Acceptance Criteria & Speed Gate Status

Issue #223 defines the completion condition:
> "Done when: all six checks pass: loads, answers, thinks, calls a tool, recalls at a nearly full window, 15 tok/s or more"

### Speed Gate Classification: NOT VERIFIED
- **Acceptance Requirement**: $\ge 15.0\text{ tok/s}$ sustained decode throughput.
- **Analytical Projections by Candidate**:
  - **Qwen3.5-9B (5.68 GB weights)**: Projected at $\approx 12.27\text{--}13.05\text{ tok/s}$ (stock clocks) to $\approx 14.21\text{ tok/s}$ (clock-locked 2.9 GHz). Due to memory bandwidth limits on the 128-bit bus, reaching $\ge 15.0\text{ tok/s}$ is a severe bottleneck.
  - **LFM2.5-2.6B (1.59 GB weights)**: Projected at $\approx \mathbf{43.8\text{ tok/s}}$ under the same memory bus efficiency ($\frac{69.7\text{ GB/s}}{1.59\text{ GB}}$), easily surpassing the $15.0\text{ tok/s}$ threshold with a $+192\%$ safety margin.
- **Methodological Distinction**:
  - These values are mathematical throughput projections based on model weight streaming across an estimated effective memory bus.
  - They **do not prove an actual pass**, nor do they **establish an actual measured failure**.
  - Runtime factors such as Vulkan shader optimizations, memory channel saturation, driver caching, and kernel scheduling on RDNA 3.5 can only be determined by a physical hardware benchmark.
  - Under `local-ai-registry` criteria, the speed gate remains **NOT VERIFIED**.

---

## 3. Hardware Architecture & System Specifications

The AMD Radeon 890M is the integrated graphics processor of the AMD Ryzen AI 300 series (Strix Point APU family, e.g., Ryzen AI 9 HX 370 / Ryzen AI 9 365).

### Verified Hardware Specifications
- **Compute Architecture**: AMD RDNA 3.5 (`gfx1150`)
- **Compute Units (CUs)**: 16 CUs (1024 Stream Processors), 32 Ray Accelerators
- **Base / Boost GPU Clock**: 2.0 GHz base / 2.9 GHz boost
- **Package TDP**: 28W–54W (dynamically allocated between CPU and GPU)
- **Memory Subsystem**: LPDDR5X-7500 on a 128-bit dual-channel memory bus
  $$\text{Theoretical Peak Bandwidth} = 7500 \times 10^6\text{ transfers/s} \times 16\text{ bytes} = 120.0\text{ GB/s}$$
- **VRAM / Unified Memory Allocation**: Configurable via system BIOS UMA frame buffer up to 48 GB from host system memory.
- **Serving Backend Compatibility**:
  - Official AMD ROCm runtime support for `gfx1150` remains experimental and is not officially enabled in standard ROCm releases without `HSA_OVERRIDE_GFX_VERSION=11.5.0` workarounds.
  - The Vulkan runtime (`RADV` / Mesa or AMD proprietary Vulkan ICD) provides stable, production-grade compute support for RDNA 3.5 compute shaders via `llama.cpp`'s native Vulkan backend (precedent: PR #313).

---

## 4. Candidate Model Comparison & Analysis

| Evaluation Metric | Candidate A: `lfm2.5-2.6b` (Recommended) | Candidate B: `qwen3.5-9b` |
| :--- | :--- | :--- |
| **Exact Registered Revision** | `LiquidAI/LFM2.5-2.6B-GGUF@84022ce711b28455e8c4fc364ce68c00cf995875` | `unsloth/Qwen3.5-9B-GGUF@3885219b6810b007914f3a7950a8d1b469d598a5` |
| **GGUF File & Quantization** | `LFM2.5-2.6B-QAD-Q4_0.gguf` (`QAD-Q4_0`) | `Qwen3.5-9B-Q4_K_M.gguf` (`Q4_K_M`) |
| **Verified File Size** | **$1,593,894,944\text{ bytes}$** ($1.59\text{ GB}$) | **$5,680,522,464\text{ bytes}$** ($5.68\text{ GB}$) |
| **LFS SHA-256 Digest** | `a247afd6414918eac8e520a9e6137dc271235461ecbe1180462221d5b8d40b03` | `03b74727a860a56338e042c4420bb3f04b2fec5734175f4cb9fa853daf52b7e8` |
| **Native Context Length** | **$131,072\text{ tokens}$** ($128\text{K}$) | **$131,072\text{ tokens}$** ($128\text{K}$) |
| **KV Floor Compliance** | **PASS** ($131,072 \ge 131,072\text{ floor}$) | **PASS** ($131,072 \ge 131,072\text{ floor}$) |
| **Reasoning Gate (Gate 3)** | **Compatible**: Built-in `<think>` tokens parsed via `--reasoning-format deepseek`. Passed in `apple-m1-8gb` recipe. | **Compatible**: Built-in `<think>` tokens parsed via `--reasoning-format deepseek`. |
| **Tools Gate (Gate 4)** | **Compatible**: Standard non-streaming single-function call (`get_weather(city="Paris")`). Immune to streaming quote bug. Passed in `apple-m1-8gb` recipe. | **Compatible**: Standard JSON function calling. |
| **Process Memory at 128K** | **$\approx 3.19\text{ GB}$ (Q4_0 KV) / $\approx 4.74\text{ GB}$ (FP16 KV)** | **$\approx 8.14\text{ GB}$ (Q4_0 KV) / $\approx 11.23\text{ GB}$ (FP16 KV)** |
| **Headroom on 48 GB APU** | **$>43\text{ GB free}$** | **$>36\text{ GB free}$** |
| **Projected Decode Speed** | **$\approx 43.8\text{ tok/s}$** ($+192\%$ over 15 tok/s threshold) | **$\approx 12.3\text{--}13.1\text{ tok/s}$** (Falls below 15 tok/s threshold) |
| **Primary Technical Risk** | Vulkan kernel execution of LFM2 short convolutions on `gfx1150`. | Inherent physical bandwidth bottleneck failing the $\ge 15\text{ tok/s}$ speed gate. |

### Why LFM2.5-2.6B is Technically Superior for Bounty #223
1. **Speed Gate Feasibility**: Qwen3.5-9B requires streaming $5.68\text{ GB}$ per token, which requires $85.2\text{ GB/s}$ of effective memory bandwidth to hit $15\text{ tok/s}$. The stock Radeon 890M only achieves $\approx 69.7\text{ GB/s}$ ($58.1\%$ efficiency). Conversely, LFM2.5-2.6B only requires $23.9\text{ GB/s}$ to achieve $15\text{ tok/s}$, projecting to $\approx 43.8\text{ tok/s}$.
2. **Context and Floor Qualification**: Unlike other sub-9B models in the registry (e.g. Gemma 4 12B or Bonsai 2 9B, which have context limits capped at 32K or 64K), LFM2.5-2.6B has a native $131,072\text{-token}$ context window, satisfying `KV_FLOOR_TOKENS = 131072`.
3. **Repository Precedent**: `lfm2.5-2.6b` already has an accepted modern six-gate recipe in the registry on an 8 GB device (`registry/recipes/apple/apple-m1-8gb/lfm2.5-2.6b.llama.cpp.8k.json`), confirming that its reasoning, tools, and context output formats cleanly pass `lab.py`'s verification suite.

---

## 5. Memory Footprint Analysis: Calculations, Assumptions & Unmeasured Quantities

### 5.1 Candidate A (`lfm2.5-2.6b`) Memory Breakdown at 128K
- **Weights**: Exactly **$1,593,894,944\text{ bytes}$** ($1.59\text{ GB}$).
- **Full-Attention KV Cache (8 layers, 8 KV heads, head dim 64, 131,072 tokens)**:
  $$\text{Elements} = 8\text{ layers} \times 2\text{ (K + V)} \times 8\text{ heads} \times 64\text{ head\_dim} \times 131,072 = 1,073,741,824\text{ elements}$$
  - FP16 KV Cache: $1,073,741,824 \times 2\text{ B} = \mathbf{2.000\text{ GiB}}\;(2.147\text{ GB})$
  - Q4_0 KV Cache (`block_q4_0`, 18 bytes / 32 elements): $\mathbf{0.5625\text{ GiB}}\;(0.604\text{ GB})$
- **Short Convolution State (22 layers, conv dim 2048, kernel 4, FP32)**:
  $$22 \times 2048 \times 4 \times 4\text{ bytes} = 720,896\text{ bytes} \approx \mathbf{0.69\text{ MiB}}\;(0.0007\text{ GB})$$
- **Total Calculated Process Memory**:
  - FP16 KV: $1.59 + 2.15 + 0.001 + 1.0\text{ (workspace)} \approx \mathbf{4.74\text{ GB}}$
  - Q4_0 KV: $1.59 + 0.60 + 0.001 + 1.0\text{ (workspace)} \approx \mathbf{3.19\text{ GB}}$
  - Available Headroom on 48 GB Card: $\mathbf{>43\text{ GB free (89.9\% margin)}}$.

### 5.2 Candidate B (`qwen3.5-9b`) Memory Breakdown at 128K
- **Weights**: Exactly **$5,680,522,464\text{ bytes}$** ($5.68\text{ GB}$).
- **Full-Attention KV Cache (8 layers, 4 KV heads, head dim 256, 131,072 tokens)**:
  - FP16 KV: $\mathbf{4.000\text{ GiB}}\;(4.295\text{ GB})$
  - Q4_0 KV: $\mathbf{1.125\text{ GiB}}\;(1.208\text{ GB})$
- **Recurrent State (24 layers, 32 value heads, head dim 128, FP32 SSM)**: $\mathbf{49.50\text{ MiB}}\;(0.052\text{ GB})$.
- **Total Calculated Process Memory**: $\mathbf{\approx 8.14\text{ GB}}$ (Q4_0 KV) to $\mathbf{\approx 11.23\text{ GB}}$ (FP16 KV), leaving $>36\text{ GB}$ headroom.

---

## 6. Public Benchmark Analysis & Hardware Attribution

### 6.1 Direct Stock-Clock Radeon 890M Benchmark
- **Source**: [Randomblock1 Gist](https://gist.github.com/Randomblock1/bf9b15926d7f6e96defb839f63c175a3)
- **Hardware**: AMD Ryzen AI 9 HX 370 (Radeon 890M, 16 CUs, LPDDR5X-7500, stock clocks/TDP).
- **Benchmark Run**: LLaMA-2-7B Q4_0 (3.825 GB model file) achieved **18.22 tok/s**.
- **Effective Weight Bandwidth**:
  $$\text{BW}_{\text{eff}} = 18.22\text{ tok/s} \times 3.825\text{ GB} = 69.70\text{ GB/s}\;(58.1\%\text{ of 120.0 GB/s peak})$$
- **Analytical Projections**:
  - `lfm2.5-2.6b` (1.59 GB): $\frac{69.70\text{ GB/s}}{1.59\text{ GB}} \approx \mathbf{43.8\text{ tok/s}}$ ($\ge 15.0\text{ tok/s}$ gate satisfied).
  - `qwen3.5-9b` (5.68 GB): $\frac{69.70\text{ GB/s}}{5.68\text{ GB}} \approx \mathbf{12.27\text{ tok/s}}$ (Falls short of 15.0 tok/s gate).

### 6.2 Direct Clock-Locked Radeon 890M Benchmark
- **Source**: [chris-youngblut-solutions/gfx1150-bench](https://github.com/chris-youngblut-solutions/gfx1150-bench)
- **Hardware**: Radeon 890M, clock-forced (`power_dpm_force_performance_level=high`, 2.9 GHz lock, 33W package limit).
- **Benchmark Runs**:
  - Qwen3-4B Q4_K_M (2.49 GB): 32.41 tok/s $\implies 80.70\text{ GB/s}$ effective bandwidth ($67.2\%$ bus efficiency).
  - Qwen2.5-Coder-14B Q4_K_M (8.99 GB): 9.30 tok/s $\implies 83.60\text{ GB/s}$ effective bandwidth.
- **Analytical Projections**:
  - `lfm2.5-2.6b` (1.59 GB): $\frac{80.70\text{ GB/s}}{1.59\text{ GB}} \approx \mathbf{50.75\text{ tok/s}}$.
  - `qwen3.5-9b` (5.68 GB): $\frac{80.70\text{ GB/s}}{5.68\text{ GB}} \approx \mathbf{14.21\text{ tok/s}}$.

---

## 7. Official Hardware Reproduction Protocol

> [!WARNING]
> **UNEXECUTED ON PHYSICAL HARDWARE**: The following commands are verified against `lab.py`'s argument parser, URL concatenation logic, and container entrypoints, but have **NOT** yet been executed on a physical AMD Radeon 890M machine.

### Option 1: Primary Candidate — LFM2.5-2.6B (High Speed Feasibility)

#### Step 1 — Download Candidate Weights & Verify Integrity
```bash
python3 -c "
import urllib.request, hashlib
url = 'https://huggingface.co/LiquidAI/LFM2.5-2.6B-GGUF/resolve/84022ce711b28455e8c4fc364ce68c00cf995875/LFM2.5-2.6B-QAD-Q4_0.gguf'
target = 'data/models/LFM2.5-2.6B-QAD-Q4_0.gguf'
print('Downloading LFM2.5-2.6B weights...')
urllib.request.urlretrieve(url, target)
sha = hashlib.sha256(open(target, 'rb').read()).hexdigest()
assert sha == 'a247afd6414918eac8e520a9e6137dc271235461ecbe1180462221d5b8d40b03'
print('LFM2.5-2.6B SHA256 verified successfully.')
"
```

#### Step 2 — Start the Vulkan Container Engine
```bash
docker run -d --name lair-890m-server \
  --device=/dev/kfd --device=/dev/dri --group-add video \
  --shm-size=4g \
  -v $(pwd)/data/models:/models \
  -p 8080:8080 \
  ghcr.io/ggml-org/llama.cpp@sha256:1086b8815273e566bc0f549bac1d6d2dc469e6409a952cca3f502e53d72877e7 \
  -m /models/LFM2.5-2.6B-QAD-Q4_0.gguf \
  --alias LFM2.5-2.6B-GGUF \
  --host 0.0.0.0 --port 8080 \
  --ctx-size 131072 \
  --n-gpu-layers 999 \
  -b 512 -ub 128 -t 4 \
  --parallel 1 --cont-batching --poll 0 \
  --flash-attn on \
  --jinja --reasoning-format deepseek --metrics
```

#### Step 3 — Execute the Official `lab.py try` Harness
*(Note: `--endpoint` expects the root URL `http://127.0.0.1:8080` without `/v1`)*:
```bash
python3 lab/lab.py try \
  LiquidAI/LFM2.5-2.6B-GGUF@84022ce711b28455e8c4fc364ce68c00cf995875 \
  --model lfm2.5-2.6b \
  --engine llama.cpp-vulkan-lfm2.5-2.6b-qad-q4-0-128k \
  --card radeon-890m-48gb \
  --on endpoint \
  --endpoint http://127.0.0.1:8080
```

---

### Option 2: Fallback Candidate — Qwen3.5-9B

#### Step 1 — Download Candidate Weights & Verify Integrity
```bash
python3 -c "
import urllib.request, hashlib
url = 'https://huggingface.co/unsloth/Qwen3.5-9B-GGUF/resolve/3885219b6810b007914f3a7950a8d1b469d598a5/Qwen3.5-9B-Q4_K_M.gguf'
target = 'data/models/Qwen3.5-9B-Q4_K_M.gguf'
urllib.request.urlretrieve(url, target)
sha = hashlib.sha256(open(target, 'rb').read()).hexdigest()
assert sha == '03b74727a860a56338e042c4420bb3f04b2fec5734175f4cb9fa853daf52b7e8'
print('Qwen3.5-9B SHA256 verified successfully.')
"
```

#### Step 2 — Start the Vulkan Container Engine
```bash
docker run -d --name lair-890m-server \
  --device=/dev/kfd --device=/dev/dri --group-add video \
  --shm-size=8g \
  -v $(pwd)/data/models:/models \
  -p 8080:8080 \
  ghcr.io/ggml-org/llama.cpp@sha256:1086b8815273e566bc0f549bac1d6d2dc469e6409a952cca3f502e53d72877e7 \
  -m /models/Qwen3.5-9B-Q4_K_M.gguf \
  --host 0.0.0.0 --port 8080 \
  --ctx-size 131072 \
  --n-gpu-layers 999 \
  -b 512 -ub 512 -t 4 \
  --parallel 1 --cont-batching --poll 0 \
  --flash-attn on --kv-unified \
  --cache-type-k q4_0 --cache-type-v q4_0 \
  --jinja --reasoning-format deepseek --metrics
```

#### Step 3 — Execute the Official `lab.py try` Harness
```bash
python3 lab/lab.py try \
  unsloth/Qwen3.5-9B-GGUF@3885219b6810b007914f3a7950a8d1b469d598a5 \
  --model qwen3.5-9b \
  --engine llama.cpp-vulkan-qwen3.5-9b-q4-k-m-128k \
  --card radeon-890m-48gb \
  --on endpoint \
  --endpoint http://127.0.0.1:8080
```
