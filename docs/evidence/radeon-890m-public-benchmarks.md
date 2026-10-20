# AMD Radeon 890M (48GB) Public Benchmark Evidence & Technical Dossier

**Target Card**: AMD Radeon 890M (48 GB Unified Memory, RDNA 3.5 / gfx1150)
**Candidate Model**: `qwen3.5-9b` (`unsloth/Qwen3.5-9B-GGUF@3885219b6810b007914f3a7950a8d1b469d598a5`)
**Launch Profile**: `registry/launches/llama.cpp-vulkan-qwen3.5-9b-q4-k-m-128k.json`
**Card Definition**: `registry/cards/amd/radeon-890m-48gb.json`
**Evaluation Status**: Pre-Execution Technical Evidence Dossier (Outcome B: Infrastructure & Candidate Profile Proposal)

---

## 1. Executive Summary & Purpose

This technical document compiles verified public benchmark data, hardware architectural characteristics, exact memory calculations, and reproduction commands for deploying `qwen3.5-9b` at 128K context (`131,072` tokens) on the AMD Radeon 890M integrated GPU (48 GB unified memory configuration).

This dossier supports Pull Request [#294](https://github.com/sybil-solutions/local-ai-registry/pull/294) as a complete, transparent engineering submission:
- **No physical six-gate run has been performed by this submission**. The candidate profile and card definition are presented as an experimental infrastructure proposal.
- **No synthetic or unmeasured run records** have been created in `data/lab-runs/`.
- **No unverified recipe file** has been committed to `registry/recipes/` without passing all six official gates (`load`, `chat`, `reasoning`, `tools`, `context`, `speed`).
- The bounty speed acceptance threshold specified in Issue #223 is **$\ge 15.0\text{ tok/s}$**. The speed gate is classified as **NOT VERIFIED**. Public estimates below 15 tok/s represent analytical projections and do not establish an actual measured failure.

---

## 2. Acceptance Criteria & Speed Gate Status

Issue #223 defines the completion condition:
> "Done when: all six checks pass: loads, answers, thinks, calls a tool, recalls at a nearly full window, 15 tok/s or more"

### Speed Gate Classification: NOT VERIFIED
- **Acceptance Requirement**: $\ge 15.0\text{ tok/s}$ sustained decode throughput.
- **Analytical Projections**:
  - Baseline memory-bound proxy (from stock LLaMA-2-7B): $\approx 12.27\text{ tok/s}$.
  - Adjusted proxy with FlashAttention enabled (+6.3%): $\approx 13.05\text{ tok/s}$.
  - Performance-level proxy (from clock-locked 2.9 GHz run): $\approx 14.21\text{ tok/s}$.
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

## 4. Model Artifact Integrity & Architecture

### Verified Model Metadata
- **Hugging Face Repository**: `unsloth/Qwen3.5-9B-GGUF`
- **Pinned Commit**: `3885219b6810b007914f3a7950a8d1b469d598a5`
- **GGUF File**: `Qwen3.5-9B-Q4_K_M.gguf`
- **Exact File Size**: **$5,680,522,464\text{ bytes}$** ($5.2904\text{ GiB} / 5.6805\text{ GB}$)
- **LFS SHA-256 Digest**: `03b74727a860a56338e042c4420bb3f04b2fec5734175f4cb9fa853daf52b7e8`
- **Registered Capabilities**: Reasoning (`"reasoning": true`), Tool-use (`"tools": true`), Context limit $131,072\text{ tokens}$ ($128\text{K}$).

### Official Architecture Configuration (`text_config`)
From official `Qwen/Qwen3.5-9B` configuration:
- `num_hidden_layers`: 32
- `layer_types`: 8 `full_attention` layers and 24 `linear_attention` layers (`full_attention_interval: 4`)
- Full attention parameters: 16 Query heads, 4 Key/Value heads (GQA factor 4), `head_dim: 256`
- Linear attention parameters: 16 Query heads, 16 Key heads, 32 Value heads (`linear_num_value_heads: 32`), `linear_key_head_dim: 128`, `linear_value_head_dim: 128`, `linear_conv_kernel_dim: 4`, `mamba_ssm_dtype: "float32"`.

---

## 5. Memory Footprint Analysis: Calculations, Assumptions & Unmeasured Quantities

### 5.1 Calculated Memory Components
1. **Model Weights**:
   - Exactly **$5,680,522,464\text{ bytes}$** ($5.2904\text{ GiB} / 5.6805\text{ GB}$) in memory-mapped storage.
2. **Full-Attention KV Cache (at 131,072 Tokens across 8 Layers)**:
   $$\text{Elements} = 8\text{ layers} \times 2\text{ (K + V)} \times 4\text{ heads} \times 256\text{ head\_dim} \times 131,072\text{ tokens} = 2,147,483,648\text{ elements}$$
   - **FP16 KV Cache**:
     $$2,147,483,648 \times 2\text{ bytes} = 4,294,967,296\text{ bytes} = \mathbf{4.000\text{ GiB}}\;(4.295\text{ GB})$$
   - **Q4_0 Quantized KV Cache (`-ctk q4_0 -ctv q4_0`)**:
     Under `llama.cpp`'s `block_q4_0` structure (18 bytes per 32 elements = 4.5 bits/element = 0.5625 bytes/element):
     $$2,147,483,648 \times \frac{18}{32}\text{ bytes} = 1,207,959,552\text{ bytes} = \mathbf{1.125\text{ GiB}}\;(1.208\text{ GB})$$
3. **Recurrent SSM & Convolution State (Constant across Context Length)**:
   - Recurrent State Matrix ($S$, 24 layers, 32 value heads, FP32):
     $$24\text{ layers} \times 32\text{ value heads} \times 128 \times 128 \times 4\text{ bytes} = 50,331,648\text{ bytes} = \mathbf{48.00\text{ MiB}}\;(0.0503\text{ GB})$$
   - Short 1D Convolution State (24 layers, 4096 channels, kernel 4, FP32):
     $$24\text{ layers} \times 4096\text{ channels} \times 4\text{ width} \times 4\text{ bytes} = 1,572,864\text{ bytes} = \mathbf{1.50\text{ MiB}}\;(0.0016\text{ GB})$$
   - Total Recurrent State: $\mathbf{49.50\text{ MiB}}\;(\approx 0.052\text{ GB})$.

### 5.2 Runtime Assumptions
- **Vulkan Driver & Compute Workspace**: Estimated at $\mathbf{\approx 1.0\text{ to } 1.5\text{ GB}}$ for driver memory heaps, command buffers, activation scratch buffers, and compute graphs.
- **Total Calculated Process Memory**:
  - With FP16 KV Cache: $5.68\text{ GB} + 4.30\text{ GB} + 0.05\text{ GB} + 1.20\text{ GB} \approx \mathbf{11.23\text{ GB}}$.
  - With Q4_0 KV Cache: $5.68\text{ GB} + 1.21\text{ GB} + 0.05\text{ GB} + 1.20\text{ GB} \approx \mathbf{8.14\text{ GB}}$.
  - Calculated Headroom on 48 GB Hardware: $\mathbf{>36.7\text{ GB free (76.6\% margin)}}$.

### 5.3 Quantities That Still Need Physical Measurement
The following quantities cannot be verified without physical hardware execution:
- **Actual Process VRAM / GTT Allocation**: The split between dedicated UMA VRAM and GTT system RAM managed by the `amdgpu` driver under Vulkan.
- **Peak Memory Under Context Ingestion**: Peak allocation during 128K prefill and KV-cache expansion.
- **Fragmentation & Allocator Overhead**: Measured via `/sys/class/drm/card*/device/mem_info_*_used` and `probe.py`.

---

## 6. Public Benchmark Analysis & Hardware Attribution

### 6.1 Direct Stock-Clock Radeon 890M Benchmark
- **Source**: [Randomblock1 Gist](https://gist.github.com/Randomblock1/bf9b15926d7f6e96defb839f63c175a3)
- **Hardware**: AMD Ryzen AI 9 HX 370 (Radeon 890M, 16 CUs, LPDDR5X-7500, stock clocks/TDP).
- **Benchmark Run**: LLaMA-2-7B Q4_0 (3.825 GB model file) achieved **18.22 tok/s**.
- **Effective Weight Bandwidth**:
  $$\text{BW}_{\text{eff}} = 18.22\text{ tok/s} \times 3.825\text{ GB} = 69.70\text{ GB/s}\;(58.1\%\text{ of 120.0 GB/s peak})$$
- **Analytical Projection for Qwen3.5-9B (5.68 GB)**:
  $$\text{Baseline Proxy} = \frac{69.70\text{ GB/s}}{5.68\text{ GB}} \approx 12.27\text{ tok/s}$$
  $$\text{Adjusted Proxy (with FA=1, +6.3\%)} = 12.27 \times 1.063 \approx 13.05\text{ tok/s}$$
- **Limitations**: Measures LLaMA-2-7B (pure transformer), not Qwen3.5-9B (hybrid attention with linear recurrent layers). The effective throughput for hybrid architectures may differ.

### 6.2 Direct Clock-Locked Radeon 890M Benchmark
- **Source**: [chris-youngblut-solutions/gfx1150-bench](https://github.com/chris-youngblut-solutions/gfx1150-bench)
- **Hardware**: Radeon 890M, clock-forced (`power_dpm_force_performance_level=high`, 2.9 GHz lock, 33W package limit).
- **Benchmark Runs**:
  - Qwen3-4B Q4_K_M (2.49 GB): 32.41 tok/s $\implies 80.70\text{ GB/s}$ effective bandwidth ($67.2\%$ bus efficiency).
  - Qwen2.5-Coder-14B Q4_K_M (8.99 GB): 9.30 tok/s $\implies 83.60\text{ GB/s}$ effective bandwidth.
- **Analytical Projection for Qwen3.5-9B**:
  $$\text{Performance-Level Proxy} = \frac{80.70\text{ GB/s}}{5.68\text{ GB}} \approx 14.21\text{ tok/s}$$
- **Limitations**: Forced-clock profiles are noncompliant with standard stock-clock registry evaluations.

### 6.3 External GPU Reference Benchmarks (For Context Only)
The following benchmarks are conducted on different hardware and **do not confirm or validate Radeon 890M performance**:
- **Discrete AMD GPU Reference**:
  - Source: [llama.cpp Discussion #26785](https://github.com/ggml-org/llama.cpp/discussions/26785)
  - Hardware: AMD Radeon RX 7800 XT (624 GB/s dedicated GDDR6 VRAM, RDNA 3).
  - Result: Vulkan Qwen3.5-9B Q4_K_M decode = 71.95 tok/s (scaling to ~86 tok/s with MTP $n=3$), confirming Vulkan shader compute efficiency on RDNA architectures.
- **Strix Halo Architecture Benchmark**:
  - Source: [AMD-Pro-Bench](https://amd-pro-bench.com/models/qwen/qwen3-5-9b)
  - Hardware: AMD Ryzen AI Max+ 395 (Strix Halo): 40 CUs, 256-bit bus (256.0 GB/s peak), 120W+ package TDP.
  - Result: Qwen3.5-9B Q4_K_M decode = 37.5–38.6 tok/s (83.2% bus efficiency).
  - **No Confirmation Implied**: Strix Halo's 40-CU compute engine, 256-bit bus, and unconstrained power budget do not translate directly to a 16-CU 28W APU. While naive bandwidth scaling ($37.5 \times \frac{120}{256} = 17.58\text{ tok/s}$) might theoretically exceed 15 tok/s, Strix Point's lower bus efficiency (58%–67%) and thermal throttling realistically project to 12.3–14.2 tok/s. This calculation is purely theoretical and does not validate 890M throughput.

---

## 7. Official Hardware Reproduction Protocol

> [!WARNING]
> **UNEXECUTED ON PHYSICAL HARDWARE**: The following commands are verified against `lab.py`'s argument parser, URL concatenation logic, and container entrypoints, but have **NOT** yet been executed on a physical AMD Radeon 890M machine.

### Step 1 — Download Candidate Weights & Verify Integrity
```bash
python3 -c "
import urllib.request, hashlib
url = 'https://huggingface.co/unsloth/Qwen3.5-9B-GGUF/resolve/3885219b6810b007914f3a7950a8d1b469d598a5/Qwen3.5-9B-Q4_K_M.gguf'
target = 'data/models/Qwen3.5-9B-Q4_K_M.gguf'
print('Downloading candidate weights...')
urllib.request.urlretrieve(url, target)
sha = hashlib.sha256(open(target, 'rb').read()).hexdigest()
assert sha == '03b74727a860a56338e042c4420bb3f04b2fec5734175f4cb9fa853daf52b7e8'
print('SHA256 verified successfully.')
"
```

### Step 2 — Start the Pinned Vulkan Container
*(The container image `ghcr.io/ggml-org/llama.cpp` has `ENTRYPOINT ["llama-server"]`, so server arguments are passed directly)*:
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

### Step 3 — Execute the Official `lab.py try` Harness
*(Note: `--endpoint` expects the root URL `http://127.0.0.1:8080` without `/v1`, as `lab.py`'s internal client functions concatenate paths such as `endpoint + "/v1/chat/completions"` and `endpoint + "/tokenize"`)*:
```bash
python3 lab/lab.py try \
  unsloth/Qwen3.5-9B-GGUF@3885219b6810b007914f3a7950a8d1b469d598a5 \
  --model qwen3.5-9b \
  --engine llama.cpp-vulkan-qwen3.5-9b-q4-k-m-128k \
  --card radeon-890m-48gb \
  --on endpoint \
  --endpoint http://127.0.0.1:8080
```
