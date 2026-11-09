# R4-08｜Spectrum Apply MiniMax H3 当前默认参数 + 上游 README（RAM 副作用 / 与 Turbo 叠加）

- URL（本轮实际打开，两个）：
  - https://minimax3.org/spectrum-apply-minimax-h3
  - https://github.com/xmarre/ComfyUI-Spectrum-MiniMax-H3
- 读取时间：2026-11-08 JST
- 读取方式：WebFetch 直连打开（GitHub README 返回前 15000/55590 字符，仍是直读；已取到所需段落）。
- 类别：LOG-004 (3) Spectrum 参数；(4) 与 Turbo/缓存叠加；(6) RAM/VRAM 副作用。

## 1. minimax3.org（逐字，页面标注 `Updated Aug 26, 2026`）

Verified facts 表（搜索索引片段逐字，本页直读亦确认）：

> | Release checked | **v0.2.20** |
> | Package | ComfyUI-Spectrum-MiniMax-H3 ｜ Maintainer xmarre ｜ Node `Spectrum Apply MiniMax H3` |
> | Default blend_weight | **0.50** ｜ Default audio_blend_weight **0.00** ｜ Default degree **1** |
> | Default warmup_steps | **1** ｜ Default tail_actual_steps **1** |
> | Default history storage | **system_ram** ｜ offline smoothing replay **Enabled** |
> | T2VA / FL2VA / Ref2VA | Supported |
> | **EasyCache / LazyCache on same branch** | **Do not combine** |
> | Multi-GPU Spectrum forecasting | Not currently validated; native path is used |
> | Output identical to native H3? | No guarantee |

> With the current default degree-1 scheduling, a typical **20-step** Euler or ER-SDE single-pass run can resolve to approximately: `A F A F A F A F A F A F A F A F A F A A` — **11 actual H3 transformer evaluations and 9 forecasts**.
> **9 forecasts out of 20 steps does not mean every MiniMax H3 workflow becomes 45% faster end-to-end.** Total render time also depends on the output head, references, model offloading, CPU transfers, sampler behavior, replay, latent size, attention backend and hardware.

默认配置全表（逐字）：`enabled=true`、`blend_weight=0.50`、`audio_blend_weight=0.00`、`degree=1`、`ridge_lambda=0.10`、`window_size=2.0`、`flex_window=0.75`、`warmup_steps=1`、`tail_actual_steps=1`、**`max_history=8`**、**`history_storage=system_ram`**、`bootstrap_first_forecast=true`、`offline_smoothing_replay=true`、**`offline_archive_storage=system_ram`**、`model_aware_mode=off`、`model_aware_risk_threshold=0.65`。

**内存段（逐字，本条最重要）**：

> ## Does Spectrum Use More VRAM or RAM?
> Spectrum stores feature history for forecasting and, when offline smoothing replay is enabled, stores replay anchors as well. The current defaults keep both `history_storage` and `offline_archive_storage` in **system RAM**.
> For most users, system RAM is the safer default. **Large H3 hidden histories can consume multiple GiB at higher resolution or duration, while moving the history to VRAM can increase GPU memory pressure.**

> **Important: older Spectrum guides may show different settings** — Early releases used more conservative configurations … If your node shows values such as `degree=4` or `warmup_steps=5`, do not assume they are the current defaults.

## 2. 上游 GitHub README（逐字）

> A native ComfyUI MiniMax H3 implementation of **Spectrum** … introduced by Jiaqi Han, Juntong Shi, Puheng Li, Haotian Ye, Qiushan Guo, and Stefano Ermon in *Adaptive Spectral Feature Forecasting for Diffusion Sampling Acceleration* (arXiv:2603.01623).
> Spectrum is an **approximate accelerator**. Forecasted steps change the denoising trajectory, so output can differ from native H3 even with the same seed and workflow.

Recent releases（逐字）：`v0.2.29 — Core BSA/Flow/Untwist runtime compatibility`（最新）、`v0.2.23`、`v0.2.22`、`v0.2.18`、`v0.2.16`、`v0.2.15`、`v0.2.14`、`v0.2.12`（"Preserved the normal **11-actual / 9-forecast** schedule in validated 20-step native ER-SDE tests"）、`v0.2.11`。

> ## Step counts vs. H3 model-call opportunities
> The ComfyUI **steps** value is the number of outer sigma intervals. It is **not** necessarily the number of logical MiniMax-H3 model-call opportunities … SEEDS-2 `2N-1`, SEEDS-3 `3N-2`, Active SA-Solver PECE `2N-1` … a 10-outer-step SA-Solver PECE run has 19 logical H3 calls: `balanced` is nominally **11 actual / 8 forecast**, `max_speed` **10 / 9**.

> ### Few-step / acceleration LoRAs
> **Turbo/LightX2V-style LoRAs can be combined with Spectrum.** They can reduce sampling time further, but **maintainer testing found noticeably larger changes in composition, motion and fine-detail quality than the normal 20-step path. Treat this as a separate quality/speed tradeoff rather than a free additional acceleration.**

> Recommended model chain: `MiniMax H3 model loader -> LoRA / model patches -> MiniMax H3 Sigma Shift -> Spectrum Apply MiniMax H3 -> guider / sampler`
> `offline_smoothing_replay=true` uses a compute-heavy causal capture pass followed by a transformer-free replay. It was introduced to preserve audio fidelity after matched H3 testing showed **speech/stutter regressions** from direct audio spectral mixing … The default keeps `audio_blend_weight=0.0`.
> A normal 20-step run commonly resolves to **11 actual H3 transformer evaluations + 9 forecasts**.
> For quality-critical work, compare Spectrum enabled/disabled with the exact same seed, prompt, checkpoint, references, sampler, schedule, resolution, duration and LoRAs.

（注：本页直读片段中**未见 "EasyCache / LazyCache on same branch: Do not combine" 这一行**；该行来自 WebSearch 返回的本页索引片段，故在 LOG-004 中标注为「片段读取」，不冒充直读。）

## 3. 事实 / 推测 / 建议

- **事实（B，直读）**：当前默认参数全表（degree=1 / warmup=1 / max_history=8 / history_storage=system_ram / offline_archive_storage=system_ram / audio_blend_weight=0）；20 步 → 11 实际 + 9 预测；**"9 forecasts ≠ 45% 端到端提速"** 为作者明确声明；**大分辨率/长时长下 H3 hidden history 可吃掉「multiple GiB」系统 RAM，改放 VRAM 会抬 GPU 压力**（(6) 核心证据）；`audio_blend_weight=0` 是为避免语音/卡顿回归；上游明确「Turbo 可与 Spectrum 组合，但构图/运动/细节变化明显更大，不是免费加速」。
- **推测（C）**：把 "multiple GiB RAM" 换算到本机 32GB RAM 的具体峰值——**未给数字，未核实**；`history_storage=vram` 在 8GB 卡上的后果——**未核实**。
- **建议（非结论）**：本机 8GB + 32GB RAM 应保留 `history_storage=system_ram`，并在长片时监控进程 RAM（与 LOG-002 的 32GB 约束交叉）；Turbo + Spectrum 属"单独质量/速度取舍"，须同 seed A/B，不能当叠加收益写进速度栈。

## 4. 等级

**B（节点维护方参数页 + 上游 README 直读，可公开复核）**；其中 "EasyCache/LazyCache 不叠加" 一行 = **B（片段读取，非直读）**；8GB 具体 RAM 峰值 = **C/未核实**。
