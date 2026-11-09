# RESEARCH-LOG-004｜V1.1 少步蒸馏 + 步级缓存提速调研

> **页眉（目标机 / 阶段）**
> - **目标机**：RTX 5070 Laptop **8GB**（Blackwell，计算能力 **sm_120**）+ **32GB RAM** + **Win11**（本目录研究约束，见 CODEBUDDY.md）。
> - **阶段**：V1.0 基线（社区基线 / 32GB RAM / 注意力后端）之后的 **V1.1 提速调研**：在 MiniMax H3（及同类视频 DiT Wan2.x / HunyuanVideo / LTX）上评估 **少步蒸馏（Turbo LoRA / DMD / consistency）** 与 **步级缓存（FirstBlockCache / TeaCache / EasyCache / Spectrum / AdaptiveCache）** 的加速比、显存/内存代价、画质代价，以及 **两者叠加是否冲突**。
> - **不跳到 V2.0**（蒸馏范式演进仅作线索，见 §5）。
> - **证据等级约定**：A=本机已复现（**本任务不可能出现 A**，无本机实测）；B=外部可复现（公开源码+完整方法+第三方复现）；C=研究候选（真实代码/论文但无直接本机复现）；D=未经证实。**每条均标等级与来源链接**。
> - **社区基线 / 32GB RAM / 注意力后端数字见 LOG-001 / LOG-002 / LOG-003，本文只引用编号，不重抄。**
> - **5090 / 4090 / 24GB+ 卡的秒数一律标明硬件，禁止当作 8GB 数字**（违反即视为 D 级无效外推）。

---

## 一、Turbo / 蒸馏 LoRA（少步蒸馏）

### L4-01｜lightx2v / ModelTC · FL2VA Turbo 规格表（H1 甜点基准、H4 检验）
- **来源**：https://github.com/ModelTC/Minimax-H3-Turbo（README，`Model specs`）+ https://comfyui-wiki.com/en/news/2026-08-11-minimax-h3-turbo-lightx2v-v1 + https://huggingface.co/lightx2v/Minimax-h3-Turbo（**本轮 2026-11-09 JST 已重开 GitHub，与 R4-01 摘录逐字一致**）。
- **发布/评论日期**：v0.1 预览 2026-08-07；v1.0 发布 2026-08-11；GitHub README 末次更新 2026-08-27（本轮重开可见）。
- **原文测试条件**：官方仓库，无 GPU/显存/耗时数字（纯规格表）。
- **报告效果（规格，非耗时）**：
  - FL2VA Turbo **4-step v0.1**：544p 混合比例，shift 12/3，蒸馏 NFE=4，推荐推理 4。
  - FL2VA Turbo **8-step v1.0**：544p，shift 12/3，蒸馏 NFE=8，推荐推理 **8 / 4**。
  - FL2VA Turbo **4-step v1.0 768p**：1344×768，shift 6/3，蒸馏 4，推荐 4。
  - FL2VA Turbo **8-step v1.0 768p**：1344×768，shift 6/3，蒸馏 8，推荐 8。
  - Ref2VA Turbo 4-step v0.1：544p，shift 12/3，蒸馏 4，推荐 4。
  - Studio 现用 FL2VA Turbo 8-step v1.0 768p，推荐 8 NFE / video shift 6 / audio shift 3。
- **显存**：未给出（卡页无参数）。
- **画质代价**：未给出。
- **证据等级**：**B**（官方仓库+发布页直读，本轮已重开复核）。
- **对 5070 Laptop 8GB+32GB 适用性与不确定点**：8-step v1.0 的「推荐 8 / 4」给出 **H1 的 8 步甜点默认**直接依据；768p 版需更多显存/内存，本机 8GB 建议先用 544p 版。卡页（HF）**无 alpha 8 / 无步数参数** → **H4 保持 D**（见 §6 检验表）。无任何 8GB 耗时数字，不可外推。

### L4-02｜larryvrh · MiniMax-H3 Turbo LoRA（4/6/8 步与画质代价，H1 核心）
- **来源**：https://huggingface.co/larryvrh/MiniMax-H3-Turbo-Lora/raw/main/README.md（R4-02 直读）。
- **发布/评论日期**：卡页未注明日期（R4 记为 2026-11-08 直读）。
- **原文测试条件**：无具体 GPU/显存/耗时表（作者自述「80 GB GPU 才舒适」为容量描述，非实测）。
- **报告效果（耗时）**：作者称「**~5× sampling speedup**」——**C 级**，明确为采样步数之比（~20→4），非端到端 wall-clock，无 8GB 实测秒数。
- **显存**：base model ~33B；`--offload-adaln` 用 **~13 GB 系统 RAM 换 ~13 GB VRAM**；`low_vram=on` 合并权重降峰值 VRAM（量化基座上偏软）。
- **画质代价（明确定性）**：4 步为推荐**下限**、4–8 有效区间、>8 步无益且过锐；**v4 在 4 步 + 大幅快速运动下有 motion-smear / trailing ghosting**，6–8 步基本消除；音频与快速运动仍是官方承认的未完善项。LoRA ~744 MB，`alpha = rank`（**非 alpha 8**），`strength=1.0`、`scheduler=simple`。
- **证据等级**：**B**（官方 HF 卡直读，参数与画质描述可公开复核）；「~5× 加速」= **C**（步数比，非实测）；8GB 外推 = **C**。
- **对 5070 Laptop 8GB+32GB 适用性与不确定点**：**H1 的「8 步甜点 / 4 步再快但运动退化」在公开报告中可定性确认**；退化为目视描述、非量化指标（无 VBench/PSNR），「可量化」证据不足。本机 8GB 建议 `low_vram=on` 作显存兜底但需同 seed A/B 比对。

### L4-03｜alibaba-pai · MiniMax-H3-Acc-LoRAs（PDD 8-step 蒸馏，另一条路线）
- **来源**：https://huggingface.co/alibaba-pai/MiniMax-H3-Acc-LoRAs/raw/main/README.md（R4-03 直读）。
- **发布/评论日期**：R4 记为 2026-11-08 直读。
- **原文测试条件**：需 `diffusers>=0.40.0` + ModularPipeline（与 ComfyUI 路线不同）；Demo 视频仅在 4 与 8 NFE 出过。
- **报告效果（耗时）**：**卡页无任何耗时/显存/GPU/分辨率数字**，也**无画质指标** → 效果/D 级。
- **显存**：未给出。
- **画质代价**：未给出（仅有「LoRA weight 1.0 在 4/8 NFE」的 demo 说明）。
- **关键事实**：PDD 蒸馏；两个 8-step Acc LoRA（FL2VA / Ref2VA），`rank=64`、`network_alpha=64`、BF16。**这是唯一明确写出 network_alpha 数值的 H3 卡（=64，不是 8）**；方法论文 arXiv:2607.26004。
- **证据等级**：**B**（官方卡直读：方法/权重/rank/alpha 可核对）；效果/耗时/显存 = **D**（卡页无数字）；对 8GB 适用性 = **D**。
- **对 5070 Laptop 8GB+32GB 适用性与不确定点**：V1.1 **暂不纳入**（需 diffusers 0.40+ 与 ModularPipeline，非本机 ComfyUI 路线）。引用 alpha 时写 `network_alpha=64`（alibaba-pai），**不写 alpha 8**。

### L4-04｜forgebun / Stable Yogi · Turbo 20/8/4 步 warm 耗时（**24GB+ 卡，禁止冒充 8GB**）
- **来源**：https://forgebun.com/go/civit_h3_turbo_ref（R4-13 直读）。
- **发布/评论日期**：Stable Yogi · 2026-08-25。
- **原文测试条件**：**960×544 / 124 帧 / 模型已加载（warm）**；页面明确要求 **「a card with 24 GB or more」**；LoRA `minimax_h3_fl2v_turbo_8step_v1.0_comfyui_bf16.safetensors`（1.9 GB）；ComfyUI 0.30+。
- **报告效果（耗时，warm）**：
  - 20 步 no turbo：**113 s**；8 步 turbo：**53 s**（2.13×）；4 步 turbo：**34 s**（3.32×）。
  - Ref2V 960×544 / 73 帧：20 步 69 s / 8 步 31 s / 4 步 21 s。
- **显存**：未给具体数字（仅要求 24GB+）。
- **画质代价（作者目视）**：8 与 4 步「faces held、无 smear/melt」；4 步与 8 步在机位等小细节有差异；建议「8 步为常规，4 步用于筛 prompt」。
- **证据等级**：**B**（页面直读，条件/数字/显卡要求可核对）；**但对 5070 Laptop 8GB 不适用（24GB+ 卡）**，仅作方法论参考，**不进入 V1.1 速度栈 8GB 依据栏**。
- **对 5070 Laptop 8GB+32GB 适用性与不确定点**：**禁止把 113/53/34s 当本机数字**（违反 R4 §8）。仅借用其方法论：warm-to-warm 对照、忽略首跑加载、8 步为常规/4 步筛 prompt。

### L4-05｜FlowForgeLabAi · 8GB 笔记本实测步数-耗时（**同架构 Blackwell 8GB，唯一同形态 8GB 曲线**）
- **来源**：https://huggingface.co/datasets/FlowForgeLabAi/minimax-h3-8gb-bench/raw/main/README.md（R4-12 直读，本轮沿用）。
- **发布/评论日期**：数据集标注 `Draft`，R4 直读 2026-11-08。
- **原文测试条件**：**RTX 5060 Laptop 8GB VRAM + 15.26 GiB 系统 RAM + Win11 + ComfyUI 0.35.0**；768×1024；fresh session。
- **报告效果（耗时）**：
  - 5 s clip：**4 / 6 / 8 步 = 245 / 297 / 378 s**（4→8 步仅 +54%，说明步数非线性耗时）。
  - 10 s clip：8 步 858 s；20 步 2029–2613 s。
  - pagefile 压力累积后 8-step 10s 图从 ~1.9 s/block → **212 s/block**（同图同分辨率）。
  - SDPA 143.1ms / comfy-kitchen INT8 22.8ms / SageAttention 29.0ms（S=8192，非 H3 真实长序列）。
- **显存/内存**：工作集 ~19.5 GiB（主模型）+ 15.0 GiB（NVFP4 TE）≈ 34.5 GiB **vs 仅 15.26 GiB RAM** → 进程常驻 pagefile；`gpu_util` 99–100% 但板卡功率中位数仅 32.8W（pagefile 饥饿信号）。
- **画质代价**：`quality_note` 为中文非正式目视，无参考指标。
- **证据等级**：**C**（第一手 8GB 笔记本实测，但单机单 seed、脚本未公开、无第三方复现；与 LOG-003 L3-07 评级一致）。
- **对 5070 Laptop 8GB+32GB 适用性与不确定点**：**本机 32GB RAM 比该机 15.26 GiB 宽裕约一倍**，故「34.5GiB vs 15.26GiB」的 pagefile 崩塌**不应直接照搬**；但「8GB 卡固定开销高、步数收益非线性」方向一致，可作本机基线**定性形状**参考。任何秒数标 C、不外推；本机实验须固定 warm session、记录 peak RAM/pagefile 与板卡功率。

---

## 二、FirstBlockCache / FBCache（步级缓存）

### L4-06｜ParaAttention FBCache · FLUX.1-dev 单卡 L20（阈值-缓存加速比）
- **来源**：https://github.com/chengzeyi/ParaAttention（R4-04 直读，本轮沿用）。
- **发布/评论日期**：R4 直读 2026-11-08。
- **原文测试条件**：单卡 **NVIDIA L20**（48GB）；FLUX.1-dev；Wall Time (s)。
- **报告效果（耗时）**：26.36 → 21.83（rdt 0.06）/ 17.01（0.08）/ 16.00（0.10）/ **13.78 s（0.12）** = 最高 **1.91×**（rdt 越大跳步越多）。
- **显存**：未单列（L20 48GB）。
- **画质代价**：作者称「up to 2x while maintaining high accuracy」；`residual_diff_threshold` 越高跳步越多、加速越高但精度下降越大。
- **证据等级**：**B**（官方仓库直读，含完整方法与作者实测数字）。
- **对 5070 Laptop 8GB+32GB 适用性与不确定点**：FLUX 为图生图、非视频 DiT，且 L20 48GB ≠ 8GB；**机制可参考**（残差差作缓存指示、阈值控速质），数字**不可外推**。FBCache 收益来自「多次 ≥20–30 步迭代里跳步」，步数越少可跳越少。

### L4-07｜ParaAttention FBCache · HunyuanVideo 720p / 129 帧 / 单 L20 48GB（视频 DiT 实测）
- **来源**：https://github.com/chengzeyi/ParaAttention/blob/main/doc/fastest_hunyuan_video.md（R4-04 直读）。
- **原文测试条件**：单卡 **NVIDIA L20（48GB）**；HunyuanVideo 720p / **129 帧 / 30 推理步**；`residual_diff_threshold=0.06`（HunyuanVideo 默认）。
- **报告效果（耗时）**：baseline（rdt=0.0）**3675.71 s** → FBCache **2271.06 s** = **1.62×**（作者自述「up to 2x」）。
- **显存/内存**：作者明确建议 HunyuanVideo 需 **≥48GB**，否则 OOM；需切 text conditions 避免 `attn_mask` 随序列平方增长；`enable_model_cpu_offload` / `vae.enable_tiling()` 应急。
- **画质代价**：「very good quality」；wall-time 加速低于迭代加速（含 TE/VAE 开销）。
- **证据等级**：**B**（官方专文直读，含完整方法+作者实测数字）。
- **对 5070 Laptop 8GB+32GB 适用性与不确定点**：**30 步 / 48GB / 720p 视频 DiT 的 1.62× 是 H2「≥20 步有稳定加速」的 B 级证据**；但硬件档位与本机 8GB 完全不同，**数字不可搬**。本机若退回 20 步原生基线可试 FBCache（rdt 0.06 起步）；若用 4/8 步 Turbo，按 §四 证据应关闭。

---

## 三、Spectrum / TeaCache / EasyCache（视频 DiT 缓存）

### L4-08｜TeaCache · Wan2.1 单 A800 实测（H3 不在支持列表）
- **来源**：https://github.com/ali-vilab/TeaCache + https://github.com/ali-vilab/TeaCache/blob/main/TeaCache4Wan2.1/README.md（R4-05 直读，**本轮 2026-11-09 JST 已重开 Wan2.1 子页，与 R4-05 摘录一致**）。
- **发布/评论日期**：CVPR 2025 Highlight；News 2025 年多条（2025/03/13 优化 Wan2.1 等）。
- **原文测试条件**：**单 A800（>40GB）**；Wan2.1 各规格；`rel_l1_thresh` 控速质。
- **报告效果（耗时）**：
  - t2v 1.3B：~175 → 88 s（0.08，~1.99×）。
  - t2v 14B：~55 → 27 min（0.2，~2.04×）。
  - i2v 480P：~735 → 300 s（0.26，2.45×）。
  - i2v 720P：~29 → 12 min（0.3，2.42×）。
- **显存**：未给 peak VRAM 数字；命令含 `--offload_model True --t5_cpu`。
- **画质代价**：作者称「2x without much visual quality degradation」；需按模型重新拟合多项式系数。
- **证据等级**：**B**（官方仓库+官方 benchmark 直读）。
- **对 5070 Laptop 8GB+32GB 适用性与不确定点**：**H3 不在 TeaCache 支持模型列表**（本轮直读确认）；社区由 kijai WanVideoWrapper 接入。**无 8GB、无少步蒸馏叠加、无 peak VRAM 数字**。本页数字**禁止外推到 8GB**。要用于 H3 需自行移植并按 H3 timestep 波动重新标定系数。

### L4-09｜EasyCache 论文 + 官方仓库 · HunyuanVideo / Wan2.1（A800 / H20，与 SVG 叠加）
- **来源**：https://arxiv.org/abs/2507.02860 + https://github.com/H-EmbodVis/EasyCache（R4-06 直读）。
- **发布/评论日期**：arXiv submitted 2025-07-03（**非近 3 个月**）；GitHub News 2025-07~08。
- **原文测试条件**：**NVIDIA A800**；HunyuanVideo / Wan2.1-1.3B。
- **报告效果（耗时）**：
  - HunyuanVideo：1124.30 → **507.97 s（2.21×）**，PSNR 32.66。
  - Wan2.1-1.3B：175.35 → **69.11 s（2.54×）**，PSNR 25.24。
  - 与稀疏注意力 **SVG 叠加（H20）**：6594 → **1981 s（3.33×）**，PSNR 27.26；作者称「orthogonal」。
- **显存/内存**：未给。
- **画质代价**：PSNR/SSIM/LPIPS 定量给出；与 SVG 叠加仍保持质量。
- **证据等级**：**B**（arXiv+官方仓库 benchmark 直读）。
- **对 5070 Laptop 8GB+32GB 适用性与不确定点**：**「正交叠加」是与稀疏注意力 SVG 叠加，非与少步蒸馏 Turbo 叠加**——勿混为一谈。无 H3、无 8GB、无 peak VRAM 数字。把 A800/H20 的 2.2–3.3× 搬到 8GB+Turbo **不成立**。To Do 列表无 MiniMax H3。

### L4-10｜Spectrum（H3 专属）· comfyui-wiki 20 步 ~30% 加速与画质 caveat
- **来源**：https://comfyui-wiki.com/en/news/2026-08-03-comfyui-spectrum-minimax-h3（R4-07 直读）。
- **发布/评论日期**：2026-08-03。
- **原文测试条件**：ComfyUI **0.30.0+**（原生 H3 合并 2026-08-03）；20 步保守预设；采样器 Euler / RES multistep / CFG++。
- **报告效果（耗时）**：
  - 20 步保守预设下：Euler 实际 13 / 预测 7（**35%** transformer-step 削减）；RES 实际 14 / 预测 6（**30%**）。
  - **AMD ROCm 用户**（Radeon AI PRO R9700，32GB 级，非 8GB）：20 步 RES multistep / 864×480 / 107 帧，212.7 → 161.0 s（**1.32×**）。
- **显存/内存**：未单列（R9700 为 32GB 卡）。
- **画质代价（明确）**：近似加速非无损；快速/短暂动作中**运动、姿态、时机、视线可能偏离**；快速移动/短暂可见细节（**眼睛、手指、指甲**）偶尔畸形或不稳；同 prompt/seed 也可能不同。
- **证据等级**：**B**（发布页直读：机制/跳步表/AMD 用户数字/版本可核对）；但「~30% 端到端更快」= **C（社区口径 "community tests"）**；AMD 1.32× = **C（第一手但单用户、非 8GB、无第三方复现）**。
- **对 5070 Laptop 8GB+32GB 适用性与不确定点**：Spectrum 是 **H3 专属节点**（Chebyshev 岭回归预测 hidden feature），比通用 TeaCache 更接近 H3；但 30% 为 transformer-step 削减比例，**非端到端耗时保证**；R9700 非 8GB，**1.32× 不可外推**。本机试 Spectrum 须同 seed 开/关各跑一次逐帧比对；20 步以上才可能有效，低步数 Turbo 按 §四 关闭。

### L4-11｜Spectrum（H3）· 当前默认参数 + 内存副作用 + 与 Turbo 叠加（维护方 README）
- **来源**：https://minimax3.org/spectrum-apply-minimax-h3（Updated 2026-08-26）+ https://github.com/xmarre/ComfyUI-Spectrum-MiniMax-H3（R4-08 直读）。
- **发布/评论日期**：minimax3.org 2026-08-26；GitHub 最新 v0.2.29。
- **原文测试条件**：节点 `Spectrum Apply MiniMax H3`；默认 `degree=1 / warmup_steps=1 / tail_actual_steps=1 / max_history=8 / history_storage=system_ram / offline_archive_storage=system_ram / audio_blend_weight=0.0 / blend_weight=0.50`。
- **报告效果（步数，非耗时）**：典型 20 步 Euler/ER-SDE 解析为 **11 actual + 9 forecasts**；作者明确「**9 forecasts ≠ 45% 端到端提速**」，总耗时还受 output head / 参考 / 卸载 / CPU 传输 / 采样器 / 注意力后端 / 硬件影响。
- **显存/内存（本条核心）**：`history_storage` 与 `offline_archive_storage` 默认均存 **system RAM**；**大分辨率/长时长下 H3 hidden histories 可消耗「multiple GiB」系统 RAM，改放 VRAM 会抬高 GPU 显存压力**。
- **画质代价**：output 不保证与 native H3 一致；`audio_blend_weight=0` 为规避语音/卡顿回归。
- **与 Turbo 叠加**：上游明确「**Turbo/LightX2V 式 LoRA 可与 Spectrum 组合**，但维护方测试发现**构图/运动/细节质量变化明显更大**，应视为单独质量/速度取舍，而非免费额外加速」。
- **证据等级**：**B**（维护方参数页+上游 README 直读）；「9 forecasts≠45% 端到端」「multiple GiB RAM」均为作者声明 B；8GB 具体 RAM 峰值 = **C（未给数字）**。
- **对 5070 Laptop 8GB+32GB 适用性与不确定点**：本机 **32GB RAM** 应保留 `history_storage=system_ram`，长片时监控进程 RAM（与 LOG-002 的 32GB 约束交叉）；Turbo+Spectrum 属「单独质量/速度取舍」，须同 seed A/B，不能当叠加收益写进速度栈。

### L4-12｜WanVideo EasyCache 节点 · 默认参数 + 「不要叠在 speed LoRA 上」（冲突反面）
- **来源**：https://comfy.icu/node/WanVideoEasyCache（R4-09 直读）。
- **发布/评论日期**：kijai；Created 2 years ago / Updated 6 months ago。
- **原文测试条件**：节点 `WanVideo EasyCache`；默认 `easycache_thresh=0.015`、`start_step=10`、`end_step=-1`、`cache_device=offload_device`。
- **报告效果（机制，非耗时）**：EasyCache 与 TeaCache/MagCache 同族（步跳过缓存）；`start_step=10` 意味着 **4/6/8 步 Turbo 采样根本不会进入缓存窗口**。
- **显存/内存**：`cache_device=offload_device`（默认）把缓存张量放离 GPU（**省 VRAM**）；改 `main_device` 才占显存。
- **画质代价**：社区口径「收益微弱 / 人脸一致性漂移」。
- **冲突段（核心）**：维护方明确「**Don't stack it on a speed LoRA. Caches and step-distillation accelerators (LightX2V, CausVid) don't play together** —— 社区指导明确不应将这类缓存与加速器同用；speed LoRA 已把你压到少数几步，缓存几乎没有可跳的步。Pick one acceleration path.」
- **证据等级**：**B**（节点文档直读：默认参数与「不要叠加」指导可公开复核）；「不兼容」的量化验证 = **C（社区指导）**，另有 R4-10 / R4-11 一手实测佐证。
- **对 5070 Laptop 8GB+32GB 适用性与不确定点**：本机若用 4/8 步 Turbo，**默认关闭 EasyCache/TeaCache/MagCache**；若退回 20+ 步原生基线再单独测缓存，并把 `cache_device` 留在 `offload_device`。该页为文档页，无 8GB 实测耗时数字。

---

## 四、Turbo + 缓存 叠加（正反面，H2 核心）

### L4-13｜ComfyUI-MiniMaxH3-AdaptiveCache · 5090 W4A8 18 组受控矩阵（叠加正面 + EasyCache 反面）
- **来源**：https://github.com/FFFFFFpy/ComfyUI-MiniMaxH3-AdaptiveCache（R4-10 直读，**本轮 2026-11-09 JST 已重开，与 R4-10 摘录逐字一致**）。
- **发布/评论日期**：Last commit 2026-09-23；README 版本段写 `当前版本：0.1.1`（与 commit 时间不一致，照录）。
- **原文测试条件**：**RTX 5090 32GB**；PyTorch 2.12.1+cu130；ComfyUI 0.30.0；MiniMax H3 DiT **W4A8 混合量化**；1376×768 / 124 帧 / 24fps（~5.17s）；18 种组合各生成 2 次取平均；AdaptiveCache `Balanced`（residual 落 GPU）；EasyCache `threshold=0.2, start_percent=0.15, end_percent=0.95`。
- **报告效果（耗时，相对 20 步原版 297.32s）**：
  - Adaptive Cache（20 步）：186.24 s（**1.60×**）。
  - Sol-Attn + Adaptive Cache（20 步）：121.89 s（**2.44×**）。
  - Turbo-8：131.71 s（2.26×）；Sol-Attn + Adaptive Cache + Turbo-8：**57.07 s（5.21×）**。
  - Turbo-4：73.19 s（4.06×）；Sol-Attn + Adaptive Cache + Turbo-4：**39.40 s（7.55×）**。
  - **EasyCache + Turbo-4：73.33 s（vs Turbo-4 73.19 s）**；**EasyCache + Turbo-8：131.72 s（vs 131.71 s）**；日志分别 **`skipped 0/4 steps` 与 `skipped 0/8 steps`**。
- **显存/内存**：`cache_device=gpu` 时 residual 可占**数百 MB 显存**；`auto` 仅在 residual+安全余量能进空闲显存时选 GPU，否则 CPU；**16GB 卡建议 `auto`**。
- **画质代价**：W4A8 + Turbo-4 部分样本明显画质异常；作者称数据不足以证明 Adaptive Cache 与 Turbo 不兼容，更可能是 **Turbo 少步 / LoRA 与 W4A8 量化近似之间的兼容性或误差放大**；Turbo 到极限后约 **15–16 s 为 VAE/封装固定开销**（sampler ~23.8s vs 总 ~39.4s）。
- **证据等级**：**B**（公开仓库 + 18 组×2 次受控表 + 完整方法/版本/环境，可公开复核；单仓库单作者，无第三方复现故不外推）。「Turbo + 缓存无损兼容」= **C/证据不足（作者自述未验证）**；对 5070 Laptop 8GB 的秒数 = **C（禁止搬用，5090 32GB+W4A8）**。
- **对 5070 Laptop 8GB+32GB 适用性与不确定点**：**关键区分**——(a) **block 级残差缓存（AdaptiveCache）在 5090 上对 Turbo-4/8 仍叠加**（机制上跳尾部 Block，与「阈值跳步」不同）；(b) **阈值缓存（EasyCache）对 Turbo 为 skipped 0/4 与 0/8，确认无效**。两者结论不同，勿混为一谈（H2 注记一致）。本机 8GB 若试 AdaptiveCache+Turbo，须在非 W4A8 / BF16 同 seed A/B，且秒数**不可引用 5090 数字**。

### L4-14｜note JUNI · 8GB 上 Turbo + EasyCache A/B（-0.7%，无收益，反面一手）
- **来源**：https://note.com/j_alpha_lab/n/n2661aa9a31cd?hl=en（R4-11 直读，**本轮 2026-11-09 JST 已重开，与 R4-11 摘录一致**）。
- **发布/评论日期**：JUNI · 2026-08-11 21:21。
- **原文测试条件**：**RTX 3060 Ti 8GB（Ampere SM86）/ 64GB 系统 RAM / ComfyUI 0.31.0 / cu130**；832×480 / 24fps / 带音频 / i2v / 同图同 prompt 同 seed；EasyCache A/B 条件完全相同（832×480 / 243 帧 / 8 步 / er_sde / turbo lightx2v），仅 EasyCache 开关为变量。
- **报告效果（耗时）**：
  - 10s/243 帧 turbo lightx2v 8 步 er_sde：**605.7 s**（两次 605.7 / 610.7，方差 0.8%）。
  - 5s/124 帧 turbo 6 步 warm：**185.5 s**；20→25 步仅 1.12×。
  - **EasyCache 关：740.8 s；EasyCache 开：735.9 s；差 -0.7%（误差内）**。
- **显存/内存**：模型合计 39.55GB（DiT pruned int8_convrot 19.53 + TE nvfp4_awq 14.61 + video VAE 4.85 + audio VAE 0.56）；`--disable-pinned-memory` 为关键启动 flag；作者强调「**实际约束是系统 RAM 而非 VRAM**，64GB 才宽裕」。
- **画质代价**：Turbo LoRA「always cuts something」；walking 场景 awkward；但某些题材 turbo 反而更好（主观目视，无指标）。
- **证据等级**：**C**（第一手 8GB 实机数字、方法清楚，但**单作者单机、无第三方复现**；EasyCache A/B 段为**片段读取**非完整直读，不冒充直读）。
- **对 5070 Laptop 8GB+32GB 适用性与不确定点**：**本机 32GB RAM 比 JUNI 的 64GB 紧**，但比 5060 的 15.26GiB 宽；`-0.7%` 在误差内，作者归因「只有 8 步、没触发 reuse_threshold 0.2」。**8GB 上 Turbo+阈值缓存无效 = 与 R4-09/R4-10(EasyCache skipped 0) 三方一致**。605.7s 等耗时**不可直接搬到本机**（RAM/卡不同）。

### L4-15｜「缓存与 step-distillation 不兼容」社区指导（冲突定性，正面反面汇总）
- **来源**：综合 R4-09（comfy.icu WanVideoEasyCache「don't stack on speed LoRA」）+ R4-10（EasyCache skipped 0/4、0/8）+ R4-11（JUNI -0.7%）+ R4-08（Spectrum 维护方「Turbo 组合质量变化更大」）。
- **发布/评论日期**：2026-08 区间多篇。
- **原文测试条件**：见各来源条目。
- **报告效果（结论）**：
  - **阈值/步跳过缓存（TeaCache / EasyCache / MagCache / FBCache）**：在 ≥20 步有稳定加速（B，L4-06/07/08/09）；在 4/6/8 步 Turbo 下收益≈0 或负（B/C，L4-09 start_step=10、L4-10 skipped 0、L4-14 -0.7%）。
  - **block 级残差缓存（AdaptiveCache）**：5090 W4A8 上对 Turbo-4/8 仍叠加（B，L4-13），但画质兼容性作者自述未验证（C）。
  - **Spectrum（H3 专属预测式）**：可与 Turbo 组合但质量变化更大（B，L4-11）。
- **显存/内存**：见各条目（L4-10 residual 数百 MB、L4-11 multiple GiB RAM、L4-12 offload_device 省 VRAM）。
- **画质代价**：低步数下缓存近似放大误差（与 R4-14 Radian「少步早期误差沿 rollout 传播」机制一致）。
- **证据等级**：**B（机制/文档/受控表）+ C（单源画质未验证）混合**；结论按各条目分别定级。
- **对 5070 Laptop 8GB+32GB 适用性与不确定点**：**H2 部分核实**——「阈值缓存低步数无效」多源 B/C 成立；「block 级残差缓存可叠加」仅单源 B(5090) 且画质未验证，本机需自测（见 §七 实验 3）。

---

## 五、近 3 个月 arXiv / 新 release（蒸馏范式演进，V1.1 不据此改配置）

### L4-16｜arXiv 近 3 月：DMAD / Leta / Radian / Spectrum v2（蒸馏与缓存方法进展）
- **来源**：https://arxiv.org/abs/2609.40037（Radian）、https://arxiv.org/abs/2610.34092（Leta）、https://arxiv.org/abs/2610.02188（DMAD）、https://arxiv.org/abs/2603.01623（Spectrum v2，修订 2026-09-30）（R4-14 直读，本轮沿用）。
- **发布/评论日期**：Radian 2026-09-30；Leta 2026-10-25；DMAD 2026-10-01(v1)/10-13(v2)；Spectrum v2 修订 2026-09-30（均在近 3 个月窗口）。
- **原文测试条件**：训练侧方法论文；Wan2.1-1.3B / 14B、ImageNet-64、COCO-10K、H3-33B（DMAD）。
- **报告效果（数字，非 8GB 推理耗时）**：
  - **DMAD**：四步 Wan2.1-T2V-14B VBench 85.15；**MiniMax-H3-33B 四步学生 vs DMD2 人类偏好 79.1%、vs rCM 84.6%**（联合音视频）；代码/模型/demo 可得（yzmblog.github.io/projects/DMAD）。
  - **Leta**：较 DMD 在 VBench +0.9、VideoAlign +0.66（G:E=1:1 时 +1.6 / +0.35）。
  - **Radian**：Wan2.1-1.3B 四步 VBench 0.8444、VideoAlign 0.8033；**明确指出少步早期误差沿 rollout 传播→细节退化/结构漂移/运动不稳**（误差放大机制佐证）。
  - **Spectrum v2**：FLUX.1 最高 4.79×、Wan2.1-14B 4.67×；原论文指出「传统缓存依赖局部近似、大跳步误差迅速增长、高倍速下画质下降」（H2 理论依据）。
- **显存/内存**：均为训练侧或论文摘要，**无 8GB / 显存 / 耗时数字**。
- **画质代价**：以 VBench/VideoAlign/人类偏好等量化指标给出（优于旧方法），非 H3 本机画质。
- **证据等级**：**B**（arXiv 官方 abs 页直读：方法/日期/数字/代码线索可复核）；「移到 H3 / 8GB 上的效果」= **C/未核实**；二手中文宣传数字（aiboss88/cloudsai「8GB 可跑」）= **D（未直读、不采信）**。
- **对 5070 Laptop 8GB+32GB 适用性与不确定点**：这些是**训练侧方法论文**，非本机 8GB 推理实测；V1.1 **不据此改配置**，仅记为蒸馏范式演进线索（DMD→PDD/DMAD/Leta/Radian；缓存 Spectrum 大跳步误差理论）供 V2.0 参考。DMAD 的 H3-33B 四步人类偏好是「蒸馏路线进展」信号，但无 8GB 可行性数据。

---

## 六、显存 / 内存副作用专项（与 LOG-002 交叉仅写新证据）

### L4-17｜AdaptiveCache residual 显存占用 + cache_device 策略
- **来源**：L4-13 同页（R4-10）。
- **原文测试条件**：16GB / 5090 32GB；`cache_device` gpu/cpu/auto。
- **报告效果（显存，非耗时）**：`gpu` 模式较高分辨率下 residual 可占**数百 MB 显存**；`cpu` 存 pinned host memory 异步搬回；`auto` 仅在能放下时选 GPU；**16GB 卡建议 `auto`**。
- **画质代价**：无。
- **证据等级**：**B**（维护方文档直读）。
- **对 5070 Laptop 8GB+32GB 适用性与不确定点**：本机 8GB 应优先 `auto` 或 `cpu`，避免 residual 占满显存触发 OOM；与 LOG-002 的 8GB VRAM 约束交叉（新证据：缓存 residual 额外显存项）。

### L4-18｜Spectrum hidden-history 系统 RAM 消耗（multiple GiB）
- **来源**：L4-11 同页（R4-08）；另 R4-15 登记 modelscope 435669 仅拿到标题（**正文无法访问，不引用其数字**）。
- **原文测试条件**：大分辨率/长时长 H3。
- **报告效果（内存，非耗时）**：feature history 默认存 **system RAM**；**大分辨率/长时长下可消耗「multiple GiB」系统 RAM**，改放 VRAM 抬高 GPU 压力。
- **画质代价**：无。
- **证据等级**：**B**（维护方参数页直读）。
- **对 5070 Laptop 8GB+32GB 适用性与不确定点**：本机 **32GB RAM** 比 5060 的 15.26GiB 宽裕，但长片（15s+）时 history 增长需监控进程 RAM（与 LOG-002 的 32GB 约束交叉）；具体峰值未给数字 → **C**。

### L4-19｜larryvrh `--offload-adaln` 用 ~13GB RAM 换 ~13GB VRAM
- **来源**：L4-02 同卡（R4-02）。
- **原文测试条件**：standalone 脚本。
- **报告效果（显存/内存交换）**：`--offload-adaln` **用 ~13 GB 系统 RAM 换 ~13 GB VRAM**；`low_vram=on` 合并 LoRA 进权重降峰值 VRAM（量化基座上偏软）。
- **画质代价**：`low_vram=on` 在量化基座上「a bit softer」。
- **证据等级**：**B**（官方卡直读）。
- **对 5070 Laptop 8GB+32GB 适用性与不确定点**：本机 32GB RAM 足以支撑 `--offload-adaln` 的 13GB RAM 交换；作为显存兜底手段（与 LOG-002 的卸载策略交叉，新证据：adaln 专项卸载）。

### L4-20｜5060 Laptop 8GB pagefile 饥饿（RAM 约束实证）
- **来源**：L4-05 同数据集（R4-12）。
- **原文测试条件**：RTX 5060 Laptop 8GB / **15.26 GiB RAM** / Win11；工作集 ~34.5 GiB vs 15.26 GiB。
- **报告效果（内存，非耗时）**：进程常驻 pagefile；`gpu_util` 99–100% 但板卡功率中位数仅 **32.8W** 是「pagefile 饥饿」签名；pagefile 压力累积后同图同分辨率 1.9→212 s/block。
- **画质代价**：无。
- **证据等级**：**C**（第一手 8GB 笔记本实测，单 seed/单机器/脚本未公开）。
- **对 5070 Laptop 8GB+32GB 适用性与不确定点**：本机 **32GB RAM 约为该机 2 倍**，pagefile 崩塌风险显著降低，但工作集 ~34.5GiB 仍可能触及 32GB 边界（尤其 15s+ 长片 + Spectrum history）→ 须监控 peak RAM/pagefile。与 LOG-002 的 32GB RAM 约束交叉（新证据：pagefile 饥饿的板卡功率信号）。

---

## 七、假设 H1–H4 检验表

| 假设 | 内容 | 检验结论 | 依据 |
|---|---|---|---|
| **H1** | 8GB 上 Turbo 8 步是速度/质量默认甜点；4 步再快但肢体/手指/音频同步退化在公开报告里可量化 | **部分核实** | 8 步甜点：L4-01（B 规格推荐 8）、L4-02（B 6–8 步最佳）、L4-04（B 24GB+ 8 步常规）；4 步退化：L4-02（B 4 步大运动 smear）、L4-14（C 步行场景 awkward）——但均为**定性目视，无量化指标分数**，「可量化」证据不足。 |
| **H2** | FirstBlockCache/Spectrum 类在 ≥12–20 步稳定加速；Turbo 4/6/8 步下收益≈0 或负；不同缓存实现结论不同 | **部分核实** | 高步数加速：L4-06/07（B FBCache 1.62–1.91×）、L4-08（B TeaCache 2×）、L4-09（B EasyCache 2.2–2.5×）、L4-10（B Spectrum 20 步 30%）、L4-13（B AdaptiveCache 20 步 1.60×）。低步数无效：L4-09（start_step=10）、L4-10（EasyCache skipped 0/4、0/8）、L4-14（-0.7%）。**但 block 级残差缓存 AdaptiveCache 对 Turbo 仍叠加（L4-13 B，5090）且画质未验证**——故「收益≈0/负」对阈值缓存成立，对 block 级残差缓存不成立。 |
| **H3** | TeaCache/EasyCache 等视频 DiT 公开 8GB 数字稀缺；搬到 H3 条件比图生图更严 | **已核实** | L4-08/09 均无 8GB 数字（A800/H20）；TeaCache **H3 不在支持列表**（L4-08 B）；EasyCache 对 Turbo skipped 0（L4-10/14）；Spectrum 是 H3 专属但 ~30% 为社区口径（L4-10 C）、AMD 用户非 8GB（L4-10 C）；仅 JUNI 给了 8GB 但仅 Turbo+EasyCache（L4-14 C）。视频 DiT 缓存公开 8GB 数字确实稀缺。 |
| **H4** | lightx2v「4 步 / alpha 8」参数组合在 HF 卡页无法核实，保持 D | **已核实（保持 D）** | L4-01（B 本轮重开 HF 卡：无 alpha / alpha 8 / 步数参数，已变 Studio 引流页）；SOURCES.md 第 34 行「4 步，alpha 8」为二手笔记，DOSSIER R7 亦要求标 D。本轮**未升格**，维持 D。 |

---

## 八、V1.1 候选速度栈表（仅由已有 B 或诚实标注 C 拼出；24GB+ 数字不写入默认栈）

> 默认栈面向 **5070 Laptop 8GB + 32GB RAM + Win11**。每行标注依据等级；「依据不足」处显式写出。

| # | 组件 | 推荐配置 | 依据 | 等级 | 备注 / 禁止项 |
|---|---|---|---|---|---|
| 1 | 基座量化 | pruned `int8_convrot` DiT + NVFP4 TE（+ fp16 VAE） | LOG-003（引用，不重抄） | B(机制)/C(8GB数字) | 见 LOG-003；本机 8GB 容量基线。 |
| 2 | Turbo LoRA | **lightx2v FL2VA Turbo 8-step v1.0**（544p），8 NFE | L4-01（B 规格）/ L4-02（B 画质）/ L4-14（C 8步605.7s@3060Ti） | B + C | **默认甜点**；768p 版显存更紧，暂缓。 |
| 3 | 步数 | **8 NFE**；4 步仅用于筛 prompt | L4-02（B）/ L4-04（B 但 24GB+，仅方法论） | B(8步甜点)/C(8GB耗时) | 4 步画质退化定性确认（L4-02/14），**不用于成品**。 |
| 4 | 注意力后端 | SageAttention2 / comfy-kitchen INT8 | LOG-003（引用） | B(机制) | 不重抄 LOG-003 数字。 |
| 5 | 缓存策略（8 步栈） | **关闭 EasyCache / TeaCache / MagCache** | L4-09（B start_step=10）/ L4-10（B skipped 0）/ L4-14（C -0.7%） | B + C | **禁止在 4/8 步 Turbo 上叠加阈值缓存**。 |
| 6 | 缓存策略（20 步栈，可选） | FBCache（rdt 0.06）/ Spectrum / AdaptiveCache | L4-07（B 1.62×@L20）/ L4-10（B 20步30%）/ L4-13（B 5090不可搬） | B(高步数机制)/C(8GB自测) | **仅当退回 ≥20 步原生基线时启用**；AdaptiveCache 本机须自测（见实验 3）。 |
| 7 | 内存兜底 | `history_storage=system_ram`；`--offload-adaln` 应急；`cache_device=auto/cpu` | L4-11（B）/ L4-19（B）/ L4-17（B） | B | 长片监控进程 RAM（L4-18/20）。 |
| 8 | 固定开销 | warm session、VAE tiling、记录 peak RAM/pagefile/板卡功率 | L4-05（C）/ L4-20（C） | C | 避免 pagefile 饥饿（L4-20 32.8W 信号）。 |
| 9 | 速度栈总加速（8GB 端到端） | **依据不足** | — | — | 无 8GB 端到端实测数字；5090/24GB+ 秒数（L4-04/13）**禁止写入本栈**。 |

> **说明**：24GB+ 卡（forgebun L4-04 的 53/34s、FFFFFFpy L4-13 的 57/39s）均为**他档硬件**，仅作机制/方法论参考，**不进入 8GB 默认栈**。

---

## 九、下一轮本机最小实验 3 项（具体可执行）

> 目标机：RTX 5070 Laptop 8GB（sm_120）+ 32GB RAM + Win11 + ComfyUI 0.30+。所有实验固定 warm session（首跑丢弃）、同 seed、同 prompt/参考图/分辨率/帧数/采样器。

**实验 1 · Turbo 步数基线（确认 H1 甜点与固定开销）**
- **命令/节点**：ComfyUI「MiniMax H3」原生流程 + `MiniMax H3 Turbo LoRA`（lightx2v FL2VA Turbo 8-step v1.0）+ `MiniMax H3 Sigma Shift`（video shift 6 / audio shift 3）+ Sampler `simple`/`er_sde`；分辨率 832×480、帧数 124（5s）；steps 依次 4 / 6 / 8 / 20。
- **观测指标**：`wall_sec`（warm，取 2 次中位数）、`peak_vram_mib`、`peak_ram_used_gb`、`peak_pagefile_gb`、`board_power_w`、`gpu_util_pct`。
- **停止条件**：同 steps 两次 warm 差异 < 5% 即采样本；否则重启 ComfyUI 消 pagefile 压力（L4-05/20）后重测。
- **回滚**：若 OOM/崩，先降帧数至 96 或开 `--offload-adaln`（L4-19）；仍崩则退回非 Turbo 20 步基线。

**实验 2 · EasyCache + Turbo 本机验证（确认 L4-14 的 -0.7% 是否复现）**
- **命令/节点**：同实验 1 的 8 步 turbo 配置，叠加 `WanVideo EasyCache`（或 H3 对应节点）`easycache_thresh=0.2 / start_step=10 / cache_device=offload_device`，仅开关为变量；同 seed 生成。
- **观测指标**：`wall_sec` 开关对比；逐帧对齐同 seed 输出比对画质（人脸/手部/运动）。
- **停止条件**：若耗时差在误差内（≈L4-14 的 -0.7%）且无画质差异 → 确认本机 8 步 Turbo 下**关闭 EasyCache**。
- **回滚**：若画质异常或耗时反升 → 永久关闭 EasyCache，回到纯 Turbo。

**实验 3 · AdaptiveCache + Turbo 本机验证（确认 L4-13 的「可叠加」在 8GB/BF16 是否成立）**
- **命令/节点**：`ComfyUI-MiniMaxH3-AdaptiveCache`（FFFFFFpy）`preset=balanced / cache_device=auto` + lightx2v Turbo 8 步；**先用 BF16/非 W4A8 基座**（L4-13 作者要求排除 W4A8 干扰）做同 seed A/B（开/关 cache）。
- **观测指标**：节点控制台 `block reduction` / `executed blocks` / `skipped`；`wall_sec`；画质固定项（人脸、手部、快速运动、口型、音频同步）。
- **停止条件**：若 block reduction>0 且画质在可接受范围 → 纳入 8GB 速度栈（C 级）；若画质明显退化（参考 L4-13 W4A8 未知项）→ 标记「Turbo+AdaptiveCache 画质未验证」，回退。
- **回滚**：关 cache 回到纯 Turbo 8 步（实验 1 配置）；记录 `cache_device` 改 `cpu` 的显存/耗时权衡（L4-17）。

---

## 十、链接核实表（本轮 2026-11-09 JST 重开复核）

| 来源 | 本轮重开结果 | 与 R4 摘录一致性 | 处置 |
|---|---|---|---|
| R4-01 ModelTC/Minimax-H3-Turbo（GitHub） | **已重开**，读到完整 `Model specs` 表与 v1.0 推荐 8/4 | 逐字一致 | 维持 B，L4-01 |
| R4-05 ali-vilab/TeaCache（Wan2.1 子页） | **已重开**，读到 A800 延迟表（含更大 use_ref_steps 表） | 核心数字一致 | 维持 B，L4-08 |
| R4-10 FFFFFFpy/ComfyUI-MiniMaxH3-AdaptiveCache（GitHub） | **已重开**，18 组表逐字一致 | 逐字一致 | 维持 B，L4-13 |
| R4-11 note JUNI（note.com） | **已重开**，读到 8GB 实测与 EasyCache A/B | 逐字一致（含 -0.7%） | 维持 C，L4-14 |
| R4-04 ParaAttention / FBCache | 本轮**未再重开**（沿用 R4 直读） | R4 已直读 | 维持 B，L4-06/07 |
| R4-02 larryvrh Turbo LoRA | 本轮**未再重开**（沿用 R4 直读） | R4 已直读 | 维持 B，L4-02 |
| R4-03 / 06 / 07 / 08 / 09 / 12 / 14 | 本轮**未再重开**（沿用 R4 直读） | R4 已直读 | 维持原级 |
| https://www.virse.ai/zh/blog/minimax-h3-review | **未直读**（聚合评测站，声称 4060 Ti 8GB + EasyCache 20→12 分钟，与 L4-14 方向相反） | R4 标 D 待核实 | **D，仅登记为待核实反例线索，不进结论/不进速度栈** |
| https://modelscope.cn/learn/435669 | **正文无法访问**（仅返回标题，Spectrum 内存账本节未返回） | R4-15 登记 | **D，不引用其片段数字**；相关参数已由 L4-11（minimax3.org + 上游 README）直读覆盖 |
| Reddit / 知乎 / B 站 / Qiita / Zenn 新文章 | 本轮 WebSearch **未命中可打开的 8GB + 少步/缓存开关 + 耗时**新页 | — | 如实写「未找到」，未用旧条目录数 |

### 本轮补充检索（2026-11-09 WebSearch，仅摘要/片段，未细读，标 D/C）
- 为完成 R4 §2 第三优先「近几日新提交」核查，本轮额外做一次 WebSearch（关键词 MiniMax H3 Turbo LoRA cache EasyCache Spectrum 8GB 2026）。命中多为聚合/博客/镜像站，**无 kijai/city96/arXiv 官方新提交**，也**无新的 8GB 受控基准**。其中与本日志结论同向、可作 corroboration 的：
  - **ffner.com 博客（D）**：「实测最稳加速组合 = Turbo LoRA + SageAttention；EasyCache 是给原生 20 步准备的，跟 4 步 Turbo 不叠加，实测还多 5% 监控开销」——与 L4-15 的「不要叠在 speed LoRA 上」一致（单博客、无硬件/数字，标 D）。
  - **civitai.red 低 VRAM 工作流（D/C）**：把 alibaba-pai **ACC LoRA**（即 L4-03 的 PDD 8-step）打包进 ComfyUI 面板，注明「ACC LoRA 目前只支持 8 steps」；并给出 15s 视频 8 步：**ACC LoRA 310.47s vs 普通 turbo 277.18s**（硬件未标明，疑非 8GB 卡）→ 仅佐证 L4-03「ACC=8 步」，**不进速度栈、不标 8GB**。
  - **canadiantechnologymagazine.com（D）**：泛读「8GB 可能可行（Kijai W4A8 + Int8 VAE + Spectrum + SageAttention）」，无实测数字 → D。
  - **hqwc.cn 教程（D）**：通用 `num_inference_steps=6` + cpu_offload 代码片段，无基准 → D。
- **结论**：本轮补充检索**未找到**可进入 B/C 级的新 8GB 受控基准，也未改变 H1–H4 检验与 V1.1 速度栈；以上仅作 D/C 级旁证登记，不写入编号条目与速度栈。

### 只看了标题/摘要未细读（不进编号条目，不作依据）
- 下列页面仅 WebSearch 索引片段/标题可见，**未直连细读**，登记自 R4-15，本次未变更处置：
  - virse.ai H3 review（D，反例线索）、hellogen.ai 本地部署（D）、lpqz.cn NPU 多卡（D，非 8GB）、minimax3.com turbo-steps（D 片段）、mushroom.cv 工程文（D）、aiboss88/cloudsai DMAD 宣传（D）、deepwiki FBCache 阈值（C 线索）、arganzheng 博客（C 线索）、hysenlabs Spectrum 版本（已由 L4-11 覆盖）、gitcode Spectrum 速查（未核实）、octohz 等 TeaCache 二手（D/C）。
  - 处置原则：片段数字一律不作依据；如需补 8GB 正面缓存反例，优先直读 virse.ai 原始出处与 modelscope 435669 正文，在此之前均标 D。

---

## 十一、事实 / 推测 / 建议 汇总（缩写）

- **事实（B/C 已标）**：Turbo 8 步为公开推荐甜点（L4-01/02）；阈值缓存在 ≥20 步有 1.6–2.5× 加速（L4-06~10）；阈值缓存在 4/8 步 Turbo 下无效（L4-09/10/14，多方一致）；block 级残差缓存 AdaptiveCache 在 5090 上对 Turbo 仍叠加但画质未验证（L4-13）；Spectrum 是 H3 专属、默认存 RAM 且可吃 multiple GiB（L4-11/18）；`--offload-adaln` ~13GB RAM↔VRAM（L4-19）；lightx2v HF 卡无 alpha 8（L4-01，H4 维持 D）。
- **推测（C，未核实，不写结论）**：把任何 24GB+/5090/A800/H20/L20 秒数搬到 5070 Laptop 8GB；AdaptiveCache+Turbo 在 8GB 的无损性；Spectrum 30% 端到端提速；本机具体 RAM 峰值。
- **建议（非结论）**：默认 8 步 Turbo + 关阈值缓存；长片监控 RAM/pagefile；AdaptiveCache 本机自测后再纳入；所有耗时/显存数字均须本机 warm session 复测，禁止引用他档硬件秒数。

---
*本日志由 R4b（2026-11-09 JST）据已有 R4-01…R4-15 摘录汇编，并对 4 个关键 URL 重开复核；无 A 级（本任务无法本机复现）。LOG-001 r-grade.4 已修正 R1-27 的 R4-16→R4-11 路径错误。*
