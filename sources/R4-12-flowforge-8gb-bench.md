# R4-12｜FlowForgeLabAi 8GB 笔记本实测数据集（4/6/8/20 步耗时，R4 复核）

- URL：https://huggingface.co/datasets/FlowForgeLabAi/minimax-h3-8gb-bench/raw/main/README.md
- 读取时间：2026-11-08 JST
- 读取方式：WebFetch 直连打开（返回 5928 字符全文）。
- 对应：LOG-003 L3-07 的来源；本轮 R4 **重新直读复核**，用于 LOG-004 (1) 步数-耗时与 (6) RAM 约束。

## 逐字摘录

> license: mit ｜ pretty_name: **MiniMax-H3 on an 8 GB laptop GPU - measured benchmark**
> # MiniMax-H3 on an 8 GB laptop GPU — measured benchmark
> First-hand measurements of running **MiniMax-H3** video generation on a single consumer laptop: **RTX 5060 Laptop (8 GB VRAM) + 15.26 GiB system RAM, Windows, ComfyUI 0.35.0**.
> This is not a leaderboard. It is a record of what actually happens on this class of hardware, **including the runs that failed** …
> > **Draft.** Single machine, single seed (12345), single operating system. Read *Limitations* before quoting any number.

RAM 是瓶颈（逐字）：

> 1. **The binding constraint is system RAM, not VRAM.** The working set is roughly **19.5 GiB (main model) + 15.0 GiB (NVFP4 text encoder) ≈ 34.5 GiB against 15.26 GiB of RAM.** The process lives in the pagefile.
> 2. **That failure is invisible in the usual metrics.** `utilization.gpu` keeps reading 99–100 % the whole time, because the GPU is not idle — it is starved between kernels. The signal that does move is **board power**.

`results.csv` — **18 runs × 33 columns**：`steps (4/6/8/20)`、sampler、**turbo-LoRA on/off**、TE-Speed on/off、resolution/aspect、video-encoding path；列含 `peak_vram_mib`、`peak_ram_used_gb`、`peak_pagefile_gb`、`peak_power_w`、`peak_temp_c`、`wall_sec`、`sec_load/sec_denoise/sec_post`、`seconds_per_block`、`completed/crashed/process_crash`、`quality_note`。

`h3_monitor_log.csv` — 534 samples × 13 cols，10s 间隔；`power_w` min 0.0 / **median 32.8** / max 106.6，而 `gpu_util_pct` 99–100%；"A healthy run on this machine draws 64–98 W. A flat ~33 W at 99 % utilisation is the pagefile-starvation signature."

**耗时与步数（逐字，(1) 核心）**：

> - **Measured wall clock, 768×1024, fresh session**: 5 s clip — **4 / 6 / 8 steps = 245 / 297 / 378 s**; 10 s clip — **8 steps 858 s**, 20 steps **2029–2613 s**.
> - **After ~1.7 h of accumulated page-file pressure** the same 8-step 10 s graph goes from **~1.9 s/block to 212 s/block**. Restart ComfyUI before a run you intend to time.
> - Fresh session: **0.9–6 s/block**. After ~1.7 h of accumulated pagefile pressure: **212 s/block** — same graph, same resolution.
> - A 259-module LoRA applied **208 modules (80.3 %)**; all 51 `adaln_proj.linear` silently skipped.
> - At H3's real attention geometry (56 heads × 128 head_dim, **S = 8192**, bf16): **SDPA 143.1 ms, comfy-kitchen INT8 22.8 ms, SageAttention 29.0 ms**.
> - `SaveVideo`'s H.264 re-encode fails when the frame's width or height is odd …

Limitations（逐字）：

> - **One machine, one seed.** No run-to-run variance study was done; treat single digits as indicative, not reproducible to the second.
> - **Peaks, not means.** `peak_*` columns are maxima.
> - **`h3_monitor_log.csv` has no date column** — only `HH:MM:SS` …
> - **Windows only.** Pagefile behaviour is not portable to Linux swap.
> - **`quality_note` is Chinese and informal** … Quality was judged by eye on a handful of clips; **no reference metric is included**.
> - No model weights are redistributed here.

Provenance：作者自采，用带埋点的自定义 sampler 节点记录 per-block 时序、峰值内存与板卡功率；`ComfyUI-GPU-Stall-Meter` 扩展；companion Space `FlowForgeLabAi/minimax-h3-8gb-advisor`。**Scripts are not yet included in this repository.**

## 事实 / 推测 / 建议

- **事实（C，第一手 8GB 笔记本实测，但单机单 seed、无第三方复现、脚本未公开）**：RTX 5060 Laptop 8GB（**同代 Blackwell 笔记本，最接近目标机形态**）/ 15.26 GiB RAM / Win11 / ComfyUI 0.35.0；768×1024 5s：4/6/8 步 = 245/297/378s；10s 8 步 858s、20 步 2029–2613s；pagefile 压力累积后 1.9→212 s/block；`gpu_util` 99–100% 但板卡功率中位数仅 32.8W 是"pagefile 饥饿"信号；SDPA 143.1ms / kitchen INT8 22.8ms / Sage 29.0ms（S=8192）。
- **推测（C/D）**：本机是 **32GB RAM**（比该机 15.26GiB 宽裕一倍），因此"34.5GiB 工作集 vs 15.26GiB RAM"导致的 pagefile 崩塌**不应直接照搬**；但"8GB 卡固定开销高、步数收益非线性"方向一致。S=8192 远小于 H3 实际 2.7万–9.2万 token，长序列未测。
- **建议（非结论）**：本数据集是 LOG-004 中**唯一同架构 + 同显存档 + 同 OS** 的步数-耗时曲线，可作本机基线的**定性形状**参考（245/297/378s 的 4→8 步只增 ~54%，说明步数不是线性耗时）；但任何秒数都标 C、不外推；本机实验须固定 warm session、记录 peak RAM/pagefile 与板卡功率。

## 等级

**C**（第一手但单机器单 seed、脚本未公开、无第三方复现；按 LOG-001 r-grade 标准）。与 LOG-003 L3-07 的原评级一致。
