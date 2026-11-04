# RESEARCH-LOG-001：8GB 显存 Blackwell 笔记本部署大尺寸模型的低显存技术调研

- 实验性质：文献/社区实测调研（证据等级见下），**非本机复现**（A 级不可能，本任务纯调研）。
- 目标硬件：RTX 5070 Laptop 8GB（Blackwell, sm_120）、32GB RAM、Win11（参考同代 Blackwell 笔记本实测：RTX PRO 2000 Laptop 8,151 MiB / RTX 5090）。
- 调研日期：2026-11-04。所有数字均来自下方“来源摘录”文件（`./sources/R1-*.md`），未凭记忆书写；SOURCES.md 侦察笔记中的数字若未独立打开核实，已在各条与核实表标注。

## 证据等级约定
- **A** = 本机已复现（本任务无）。
- **B** = 外部可复现（公开源码+完整方法+第三方/作者实机数字）。
- **C** = 研究候选（有真实代码/论文/官方文档，但本任务未逐字核实关键数字，或示例非 H3 直接实测）。
- **D** = 未经证实（宣称/榜单/未打开）。

## 技术假设（用来源检验，不预设）
> 显存不足时瓶颈转移到 PCIe 传输与系统内存/磁盘；有效手段 = “降低权重体积（量化/剪枝）+ 降低步数（蒸馏）+ 分块卸载重叠计算 + 控制激活峰值（VAE 分块、分辨率）”。
- 来源检验结论：**假设基本成立**，但出现两个反例/陷阱——(1) 稀疏注意力（VSA/SLA）在 H3 上**增加**显存并 OOM（R1-11），并非总是省；(2) Windows 上“系统内存回退”会把显存瓶颈转成整机崩溃而非变慢（R1-04），说明“转移到系统内存”在笔记本上不是免费的安全网。

---

## 一、卸载 / block swap / 动态 VRAM（≥2 条）

**R1-01｜ComfyUI DynamicVRAM（Blackwell 笔记本实测）**
- 来源：R1-04（zenn cosara，2026-08-15，B）
- 原文测试条件：RTX PRO 2000 Blackwell Laptop 8,151 MiB / 95.5GB RAM / Win11 / ComfyUI 2a68ce33 / PyTorch 2.13.0+cu130 / DiT NVFP4 11.66GB / `--reserve-vram 1.0`。
- 报告效果：DynamicVRAM 以 32MB 页错误粒度把权重流式搬入 VRAM；800×448 / 15.083s 成功（908.1s）。**关闭“CUDA 系统内存回退”后，可生成长度从 209 帧降到 124 帧**，但机崩 8 次→0 次；124 帧耗时 248.6s vs 248.7s（速度本身不变）。
- 画质代价：无（仅限制长度/分辨率）。
- 证据等级：B。
- 对 8GB 适用性：直接适用于 5070 Laptop（同为 Blackwell 笔记本）。不确定点：DynamicVRAM 依赖系统内存回退作为安全网；关闭回退后容量骤降，需配合更低分辨率/更短时长，或改用 W4A8/GGUF 减小常驻。

**R1-02｜comfy-kitchen 异步 2-stream 分块卸载重叠（Ampere 实测）**
- 来源：R1-02（qiita 3070Ti，2026-08-13，B）
- 原文测试条件：RTX 3070 Ti 8GB / 16GB RAM / ComfyUI 0.32.0 / comfy-kitchen 0.2.30 / 864×480 / 124 帧。
- 报告效果：DynamicVRAM + mmap + fast-disk + pinned 6456MiB + async offload：async OFF 92.34s / 1 stream 96.85s / **2 streams 89.56s**（block 传输与计算重叠，比 1-stream 快 8.1%）。
- 画质代价：无（仅调度）。
- 证据等级：B（Ampere，非 Blackwell）。
- 对 8GB 适用性：comfy-kitchen 的 block 级异步卸载在 Blackwell 上同样可用（需 cu130 启用 cuda backend，R1-01）。不确定点：2-stream 最优是否随分辨率/PCIe 带宽变化；笔记本 PCIe 多为 x8 甚至更窄，重叠收益可能更小。

**R1-03｜DiffSynth 动态 VRAM + 磁盘卸载（机制文档）**
- 来源：R1-05（DiffSynth-Studio VRAM_management.md，B，示例 Qwen-Image）
- 原文测试条件：通用；`vram_limit` 参数驱动层级卸载，例 56G→40G(CPU offload)→21G(FP8)+disk。
- 报告效果：Layer-level Offload（`vram_limit=0` 尽量压 RAM，按需搬入 VRAM）；Disk Offload 仅 `.safetensors`，逐层懒加载，建议 NVMe。文档明确“VRAM 不足时框架会尝试超过 vram_limit 以保推理”，不能绝对保证上限。
- 画质代价：FP8 量化有轻微画质下降（文档自述）。
- 证据等级：B（机制）；对 H3 为 C（DiffSynth 有 MiniMax-H3-NF4 变体但未独立打开）。
- 对 8GB 适用性：可作为 ComfyUI 之外的另一条“层卸载+磁盘”路线；与“量化让更多权重驻留 VRAM 以避免卸载速度损失”的洞察一致。不确定点：DiffSynth 对 H3 的 dynamic VRAM 支持完整度与 Windows 性能未验证。

**R1-04｜SGLang 层级卸载 layerwise offload（官方配方，Linux）**
- 来源：R1-06（SGLang cookbook，C，页面超时仅确认结构）
- 原文测试条件：消费级最小配方 `--performance-mode memory --layerwise-offload-components dit,text_encoder,vae --layerwise-resident-layers video_vae=36`，864×480/124帧/20 NFE，要求 12GB VRAM+32GB host+NVMe。
- 报告效果（SOURCES 侦察，未逐字核实）：Recipe A 16.8–18.7 s/it（实验室模拟）；4090 24GB convrot_int8+layerwise offload 峰值约 18GB。
- 画质代价：无（resident-layers 调优）。
- 证据等级：C（无 8GB 配方；SGLang 仅 Linux，Windows 需 WSL2）。
- 对 8GB 适用性：低。官方无 8GB 配方；WSL2+layerwise 在笔记本上需自行降配，风险高。可作“层卸载”原理参考。

**R1-05｜diffusers group offloading（通用机制）**
- 来源：R1-12（diffusers 官方 memory 文档，B）
- 原文测试条件：通用；`enable_group_offloading`（block_level / leaf_level），`use_stream=True` 时需 **2× 模型大小 CPU 内存**。
- 报告效果：比 model offload 省显存、比 sequential CPU offload 快；可 `offload_to_disk_path` 落盘。
- 画质代价：无。
- 证据等级：B（机制）；H3 非 diffusers 默认管线（H3 走 ComfyUI/SGLang/DiffSynth）。
- 对 8GB 适用性：若走 diffusers 路线，32GB RAM 且 stream 模式需 2× 模型大小内存——对 H3 数十 GB 权重是硬约束，需配合量化+落盘。

## 二、量化（≥2 条）

**R1-06｜int8 ConvRot（Comfy-Org 官方变体，Ampere 实测）**
- 来源：R1-01（qiita 3070，B）+ SOURCES（Comfy-Org 文件清单）
- 原文测试条件：RTX 3070 8GB / DiT `pruned_int8_convrot` 21GB（pruned 19.5GB）/ TE NVFP4 15.7GB / ComfyUI 0.31.0 / cu130。
- 报告效果：480×832 8步 345.2s / 7,322 MiB；cu130 使 int8 路径 R2V 313.4→200.9s（**1.56×**）。int8 走 comfy-kitchen cuda backend（sm_86 支持）。
- 画质代价：int8 相对 bf16 轻微（社区公认可接受）。
- 证据等级：B。
- 对 8GB 适用性：首选权重格式。5070 Laptop 同为 Blackwell，int8 张量核加速可用；DiT 19.5GB 需全量卸载+流式，靠 DynamicVRAM/GGUF 路线。不确定点：本机 PCIe 带宽与 32GB RAM 下实际峰值需自测。

**R1-07｜NVFP4（Blackwell 原生）**
- 来源：R1-04（cosara Blackwell，B）+ R1-01（Ampere 仅 TE 可用）+ R1-06（SGLang 需 CC≥10.0）
- 原文测试条件：Blackwell 笔记本 DiT NVFP4 11.66GB（cosara）；Ampere 3070 上仅 TE `nvfp4_awq` 15.7GB（因 TE 只跑一次，占比小，可用但不加速）。
- 报告效果：cosara 用 NVFP4 DiT 在 8,151MiB 上跑通 800×448/15s；SGLang 注明 NVFP4 需 CC≥10.0（sm_120=12.0 满足，**Blackwell 原生加速**）。
- 画质代价：AWQ 量化轻微。
- 证据等级：B（Blackwell 可用）；Ampere 上无加速为 B 实测。
- 对 8GB 适用性：**关键**。5070 Laptop 是 Blackwell，NVFP4 应有硬件加速且 DiT 体积极小（11.66GB），是 8GB 本机最值得优先验证的量化路线。不确定点：NVFP4 DiT 在 5070 Laptop 实际峰值与是否需 comfy-kitchen/特殊内核；驱动 cu130 支持（侦察推测 r580+，C）。

**R1-08｜W4A8（Ampere 实测，mixed ConvRot）**
- 来源：R1-02（3070Ti，B）+ R1-01（3070，B）
- 原文测试条件：3070 Ti：DiT+TE `w4a8_mixed` + int8 video VAE + 4-step Turbo；864×480/124帧。3070：非 pruned Ref2VA w4a8 22.8GiB。
- 报告效果：3070 Ti 4-step 79.74s（warm）/ 89.56s（cold），比 baseline 20-step 快 3.41×；但 3070 上 w4a8（cu128）1001.2s 比 int8 313.4s 慢 3.2×，且文件比 pruned int8 更大（22.8 vs 19.5GiB），cu130 后追平 int8（194.7s）。
- 画质代价：3070Ti 称输出正常无伪影；近似误差在低步数下相对大。
- 证据等级：B。
- 对 8GB 适用性：仅在 cu130 + comfy-kitchen cuda backend 启用后才有意义（否则 eager 回退极慢）。5070 Laptop 应默认 cu130，W4A8 mixed 可作 int8 之外的备选。不确定点：H3 是否有 pruned W4A8（非 pruned 体积无优势）；Blackwell 上 kernel 支持度。

**R1-09｜GGUF（unsloth，stable-diffusion.cpp 路线）**
- 来源：R1-08（unsloth HF 卡，B）
- 原文测试条件：DiT `fl2va_pruned-Q2_K` 6.26GiB … `Q8_0` 19.97GiB；TE `Q2_K_M` 12.20GiB / `Q4_K_M` 16.97GiB；命令 `sd-cli --mode vid_gen --backend te=cpu --diffusion-fa --offload-to-cpu`；示例 960×544/124帧/8步/seed11（**未给显存/耗时数字**）。
- 报告效果：权重体积极小（DiT Q2_K 6.26GiB）；`--backend te=cpu` 把 12GB TE 移出显卡；`--cfg-scale 1.0` 必选（H3 蒸馏无 CFG）。
- 画质代价：GGUF 低比特（Q2_K）有可见降级，需权衡。
- 证据等级：B（文件与命令真实）；效果数字缺失为 D。
- 对 8GB 适用性：是 ComfyUI 之外的兜底路线，权重最小。不确定点：sd.cpp 在 Windows + 5070 Laptop 的实际显存占用/速度/画质未实测；H3 GGUF 是否需架构补丁（社区称原版 ComfyUI-GGUF 不认 H3，D，待核实）。

**R1-10｜FP8 量化（DiffSynth layerwise casting，机制）**
- 来源：R1-05（DiffSynth 文档，B）
- 原文测试条件：offload/onload dtype=float8_e4m3fn，计算 upcast BF16；例 Qwen-Image 56G→21G。
- 报告效果：FP8 **只减显存不提速**（Native FP8 计算仅 Hopper 支持，框架不启用 FP8 计算）。
- 画质代价：轻微（文档自述；归一化/调制层跳过 fp8）。
- 证据等级：B（机制，Hopper 限制明确）；H3 为 C。
- 对 8GB 适用性：Blackwell 支持 FP8 计算（不同于 Hopper-only 的限制描述，需确认 DiffSynth 是否在 Blackwell 启用），可作为显存压缩手段，但加速需依赖 Blackwell FP8 张量核。不确定点：DiffSynth 对 Blackwell FP8 计算的实际启用。

**R1-11｜NF4（bitsandbytes，DiffSynth 变体）**
- 来源：R1b-12（sources/12-hf-diffsynth-nf4.md，B）+ R1b-14（sources/14-zenn-colab-t4.md，B）
- 原文测试条件：DiffSynth-Studio/MiniMax-H3-NF4 卡（apache-2.0）：pruned DiT NF4 约 **9.8GB**；默认 `vram_limit`=总显存-4GB，张量逐张量从磁盘流式载入；卡面宣称“as little as 8GB”。zenn Colab T4（约 14.56GiB 显存）实测：`VRAM_LIMIT_GB=9.0` + `OFFLOAD_MODE=disk` 下仅 **288×160/22帧/6步** 完成，更高分辨率（448×256/56帧/12步、320×192/39帧/8步 等）均 OOM。
- 报告效果：NF4 变体真实存在、机制可行（B）；但实测可用性远低于“8GB”宣称——即便 14.56GiB 的 T4 也只能跑 288×160 极低分辨率，且无任何真实 8GB 卡成功记录。未给生成耗时数字（zenn 未给）。
- 画质代价：NF4 相对 bf16 有降级（bitsandbytes 通用认知）。
- 证据等级：B（变体存在+机制+文件真实）；**8GB 本机可行性为 C**（无真实 8GB 卡实测，T4 14.56GiB 仅极低分辨率成功）。
- 对 8GB 适用性：权重介于 NVFP4(11.66) 与 int8(19.5) 之间，但实测可用性存疑；5070 Laptop 需以 disk offload 自测极低分辨率。所有数字（9.8GB / 288×160 / 22帧 / 6步 / 14.56GiB）均来自 sources/12、14。

## 三、步数蒸馏 LoRA（≥2 条）

**R1-12｜Turbo LoRA（Abiray / lightx2v，多卡实测）**
- 来源：R1-01（3070，B）、R1-02（3070Ti，B）、R1-09（LightX2V，B）
- 原文测试条件：3070：DiT+TE pruned + `minimax_h3_turbo_4step_ckpt600_ema_V4`，steps 8–12，sigma shift video 12 / audio 6，res_multistep+simple；3070Ti：4-step preview 79.74s；LightX2V 4/8-step 768p LoRA。
- 报告效果：3070 20→8 步约 1.7×；ema_V4 + steps 12 最适（8 步手臂融化，16 步无改善）；3070Ti 4-step 比 20-step 快 3.41×。
- 画质代价：步数越低（4 步）运动/细节越易损；ckpt850_V1 出环状伪影（3070）。
- 证据等级：B。
- 对 8GB 适用性：必备。蒸馏 LoRA 是“降步数”主手段，直接缩短 wall-clock。不确定点：4 步 vs 8/12 步在 5070 Laptop 的画质-速度取舍；需本机选稳定 ckpt。

**R1-13｜LightX2V 4/8-step 蒸馏 + DMD2 支持**
- 来源：R1-09（LightX2V，B）
- 原文测试条件：MiniMax-H3 Turbo 4-/8-step 768p distilled LoRA（2026-08-11 / 08-27 发布）；Few-Step Inference 含 **DMD2** 与 Wan2.1 MeanFlow；Wan2.2 NVFP4 4-step 在 RTX5090 达 50× over 40 NFE。
- 报告效果：发布记录明确支持 H3 4/8-step 768p 同步音视频；具体 H3 加速倍数本页未逐字给。
- 画质代价：蒸馏固有（低步数近似）。
- 证据等级：B（支持真实）；H3 具体倍数 C。
- 对 8GB 适用性：与 Turbo LoRA 同类替代；DMD2 是更通用的少步蒸馏范式。不确定点：LightX2V 的 H3 4-step LoRA 在 5070 Laptop 显存/画质需自测；走 LightX2V 框架 vs ComfyUI 节点。

**R1-14｜PDD 加速 LoRA / FastH3 8-step（SGLang 官方）【部分核实】**
- 来源：R1-06（SGLang cookbook，C，仅确认章节存在：PDD acceleration LoRAs、FastH3 8-step distilled、FastH3 Omni Ref2VA 8-step PDD）
- 原文测试条件：SGLang 官方 LoRA recipes（无 8GB 配方）。
- 报告效果：未逐字抽取；SOURCES 侦察：4090 上 Spectrum 跳步 3.53×。
- 画质代价：PDD 为步数蒸馏，低步数近似。
- 证据等级：C（章节存在已确认，数字未核实）。
- 对 8GB 适用性：PDD/FastH3 是官方少步配方，理论上可大幅降步；但需 SGLang（Linux/WSL2）或对应 ComfyUI 适配。不确定点：8GB 下能否加载 PDD 权重并配合卸载。

## 四、VAE 分块 / 分时解码（≥2 条）

**R1-15｜ComfyUI PR #15456 handles_tiling 解码 OOM 修复**
- 来源：R1-10（ComfyUI PR #15456，B）
- 原文测试条件：H3 video VAE 与 SeedVR2 设 `handles_tiling=True`（模型内部已分块）；H3 长片 dynamic VRAM 下。
- 报告效果：原“regular 解码 OOM→重试 tiled”对这类 VAE 原样重跑同样 OOM（~14GB 流式 DiT/TE 常驻，解码只需 ~1.2GB）。修复：重试前 `free_memory(1e30, device, keep_loaded=[VAE])` 释放其它模型；新增单测 11 passed、全量 54 passed。
- 画质代价：无（仅释放内存时机）。
- 证据等级：B。
- 对 8GB 适用性：直接受益——长尺（>124 帧）解码 OOM 概率下降，是 15s 生成的关键修复。需本机 ComfyUI 含该 PR（≥对应 commit）。

**R1-16｜diffusers VAE slicing / tiling（通用机制）**
- 来源：R1-12（diffusers 文档，B）
- 原文测试条件：通用；`enable_slicing`（batch 拆分）、`enable_tiling`（tile 拆分，低于阈值默认关如 512×512）。
- 报告效果：slicing 例“4 张同生成解码峰值激活 4×→1×”；tiling 逐 tile 处理降峰值。
- 画质代价：tiling 偶有边界接缝（通用认知）。
- 证据等级：B（机制）；H3 走 ComfyUI 内部 tiling（见 R1-15）。
- 对 8GB 适用性：H3 的 Video VAE 自带 tiling（handles_tiling），激活峰值已由模型内部控制；关键是 R1-15 的“解码前释放其它模型”。

## 五、注意力后端（≥2 条；含 Blackwell/Windows 可用性）

**R1-17｜SDPA（默认，实测基准）**
- 来源：R1-01（3070，B）、R1-04（cosara，B）
- 原文测试条件：两文均全程 SDPA（ComfyUI 默认）。
- 报告效果：3070 全 SDPA 测得各分辨率耗时/显存（见 R1-01、R1-06）；cosara 称“SageAttention 有纯噪声 bug，先用 SDPA 确认再加速”。
- 画质代价：无（参考实现）。
- 证据等级：B。
- 对 8GB 适用性：安全基线。5070 Laptop 先用 SDPA 跑通，再尝试更快后端。不确定点：SDPA 在 Blackwell 上的绝对速度。

**R1-18｜SageAttention（Blackwell 支持，但 H3 有噪声现象，分两类）**
- 来源：R1-07（SageAttention 仓库，B）+ R1-02（3070Ti 可用且更快，B）+ R1-01（3070 纯噪声，B）+ R1b-28（ComfyUI #15263，B）
- 原文测试条件：
  - 3070Ti（Ampere）实测 4096tok/16head/dim128：SDPA 5.616ms vs SageAttention 2.310ms（2.432×），端到端快 14%。
  - **#15263（sm_120 / Blackwell）**：SageAttention FP8 PV kernel 在约 160k token 以上出纯噪声（约 154k 以下干净）；作者 2026-08-09 更新称 ComfyUI 0.31.1 上可用（KJNodes 节点设 `auto`，1920×1088/15s 54min vs sage off 94min）。**此为 sm_120 现象，非 3070（Ampere）。**
  - 3070（Ampere）作者单独报告 H3 + SageAttention 输出纯噪声（R1-01，全文用 SDPA）——其归因 #15263，但 #15263 实为 sm_120，二者可能根因不同，**单列为本任务独立现象**（见下方“3070 噪声单列”）。
  - SageAttention3 有 `sageattention3_blackwell`，需 CUDA>=12.8。
- 报告效果：5090（同代 Blackwell 桌面）SageAttention 560T，比 FA2 快 2.7×；但 H3 上曾报告噪声（#15263：sm_120 FP8 PV kernel 长序列，0.31.1 起报告修复）。
- 画质代价：正常时无损（论文）；H3 上异常为灾难性（纯噪声）。
- 证据等级：B（3070Ti 加速、#15263 0.31.1 修复报告均为单一作者实测/报告）。
- 对 8GB 适用性：潜力大（2.4–2.7× 注意力加速），但 5070 Laptop（sm_120）若试 SageAttention3，须在 ComfyUI 0.31.1+ 用 KJNodes `auto`，并在 SDPA 基准上逐帧画质比对确认非纯噪声再加。不确定点：Blackwell Windows 编译门槛（triton/CUDA 编译）。
- **3070 噪声单列**：3070 作者（R1-01，Ampere）报告 H3+SageAttention 纯噪声、故全文用 SDPA；该现象被其归因到 #15263，但 #15263 实指 sm_120，根因未必相同，故单列，不与 #15263 合并。

**R1-19｜稀疏注意力 VSA/SLA（H3 上反而 OOM）**
- 来源：R1-11（ComfyUI #16175，B）
- 原文测试条件：RTX 3090 Ti 24GB / 64GB RAM / Win11 / ComfyUI 0.34–0.35 / `pruned ref2va int8_convrot` + `nvfp4_awq` / 1MP / 15s / 6 步 / 开 Model Sparse Attention（sol-attn/SLA 默认）。
- 报告效果：开启后约 4/6 步 OOM（Currently allocated 4.89 / Requested 6.24 / limit 23.99 GiB）；临时方案 `--disable-comfy-compiler` 实测 6/6 步成功（580.42s vs 失败 375.60s）；开发者称已修但 v0.35.0 仍复现。
- 画质代价：稀疏近似（未量化，因 OOM 阻断）。
- 证据等级：B。
- 对 8GB 适用性：**不建议作为首要加速**。即便 24GB 卡都 OOM，8GB 笔记本风险更高；且依赖 Comfy Compiler（Blackwell 上更易出问题）。先排除，待确认稳定后再试。

## 六、缓存 / 跳步（≥2 条）

**R1-20｜FirstBlockCache（Ampere 实测）**
- 来源：R1-02（3070Ti，B）
- 原文测试条件：864×480/124帧/20步；FirstBlockCache Safe 复用 5/20 step。
- 报告效果：20-step 271.84s → 211.52s（Safe）；但 **4/6/8-step Turbo 关闭 cache**（低步数下近似误差相对大）。
- 画质代价：Safe 模式轻微；低步数叠加会损质。
- 证据等级：B。
- 对 8GB 适用性：仅在“高步数（≥20）”基线时有用；本机若用 4/8-step Turbo，则 cache 价值有限甚至有害。可作“若退回高步数保画质”时的加速。

**R1-21｜Spectrum 跳步（Blackwell 笔记本实测 + SGLang）**
- 来源：R1-04（cosara 用 ComfyUI-Spectrum-MiniMax-H3 v0.2.3，B）+ R1-06（SGLang Spectrum 3.53× @4090，C）
- 原文测试条件：cosara Blackwell 笔记本跑通（节点 v0.2.3，具体跳步数未给）；SGLang 4090 上 Spectrum 3.53×。
- 报告效果：cosara 15s 成功依赖该节点（未给单独加速比）；SGLang 称 Spectrum 跳步 3.53×。
- 画质代价：跳步近似，需调跳步比例。
- 证据等级：B（cosara 跑通）；C（3.53× 数字未独立核实）。
- 对 8GB 适用性：与蒸馏 LoRA 互补的“跳步”手段，cosara 已在 Blackwell 8GB 笔记本验证可用。不确定点：与 Turbo LoRA 同用时是否冲突/叠加收益。

**R1-22｜TeaCache / MagCache / TaylorSeer（训练无关缓存）**
- 来源：R1-13（TeaCache 仓库，B）+ R1-09（LightX2V 支持，B）
- 原文测试条件：TeaCache（CVPR2025）training-free，估计 timestep 间输出波动来缓存；支持 Wan/HunyuanVideo/CogVideoX/LTX/FLUX 等，**未直接列 H3**；DiffSynth 与 LightX2V 已接入。
- 报告效果：通用视频扩散加速（无 H3 数字）。
- 画质代价：缓存近似（可调阈值）。
- 证据等级：B（方法真实）；H3 适用性 C。
- 对 8GB 适用性：与 FirstBlockCache 同类；H3 上需确认 timestep 波动特性。低步数（4/8-step）下近似误差相对大（同 R1-20 经验）。

## 七、Windows 专属内存坑（≥2 条）

**R1-23｜CUDA 系统内存回退导致整机崩溃（Blackwell 笔记本）**
- 来源：R1-04（cosara，B）
- 原文测试条件：ThinkPad P16v / RTX PRO 2000 Blackwell / VBS+DMA 保护（IOMMU 强制）；长尺（209/362 帧）触发 VRAM 溢出处 RAM 退避。
- 报告效果：**8 次整机重启**（HYPERVISOR_ERROR 0x20001，IOMMU 内部错误）；设置“优先无系统内存回退”后 **0 次**，但可生成长度 209→124 帧。124 帧耗时差 0.04%（速度不变）。
- 画质代价：无；代价是容量上限下降。
- 证据等级：B。
- 对 8GB 适用性：**高优先警告**。5070 Laptop 同为 Blackwell 笔记本，若开 VBS/DMA 保护 + 系统内存回退，长尺生成可能整机崩而非 OOM。建议：短尺/低分辨率，或关闭回退（但容量更紧）。不确定点：本机是否启用 VBS/DMA 保护（笔记本默认常开）。

**R1-24｜pagefile 增长（Ampere 实测）**
- 来源：R1-02（3070Ti，B）
- 原文测试条件：16GB RAM / cold vs warm run。
- 报告效果：cold run 进程 RAM 9.74GiB，pagefile 增约 2GiB（TE 与模型初始化）；warm run pagefile 降至 0.02–0.15GiB。
- 画质代价：无。
- 证据等级：B。
- 对 8GB 适用性：32GB RAM 比 3070Ti 的 16GB 宽裕，但**仍需足够 pagefile + NVMe**；cold 首跑会触及 pagefile，建议固定大 pagefile 避免抖动。侦察提及“32GB RAM 加载卡死”传闻（D），需本机验证。

**R1-25｜笔记本供电 / 散热降频（Blackwell 笔记本，2.76× 差异）**
- 来源：R1-04（cosara，B）
- 原文测试条件：USB-C 85W 坞使电力框限 25W（默认 45W/最大 60W，SW Power Cap Active）；换 140W AC 直插 → 60W。
- 报告效果：同生成 **655.8s → 237.2s（2.76×）**；`SW Power Cap = Active` 是电力触顶信号（`nvidia-smi -q -d PERFORMANCE`）。
- 画质代价：无（纯吞吐）。
- 证据等级：B。
- 对 8GB 适用性：**极易被忽视的瓶颈**。5070 Laptop 插电模式/散热直接影响 2–3× 速度；务必满血供电、观察温度与 Power Cap。不确定点：5070 Laptop 的 TGP 与散热余量（8GB 版 TGP 可能较低）。

**R1-26｜pinned memory / --disable-pinned-memory（社区修复 flags）**
- 来源：R1b-25（sources/25-comfyui-issue-15663.md，B）
- 原文测试条件：ComfyUI 0.33.0 / Win11 / RTX 4090 24GB + RTX 3090 / `--disable-dynamic-vram` / `pruned int8_convrot` + 8-step turbo + VAE fp16。短片段 VRAM 估计偏低→OOM，且首次 OOM 后同工作流每次都在同点 OOM（插入任意节点或改任一参数可解除）。
- 报告效果：根因 = `comfy-aimdo` 初始化失败使 DynamicVRAM 未激活 → VRAM 估计不准 → 异常 OOM（`aimdo_setup_hooks: funchook_prepare(cuMemAlloc_v2) failed: 8 Failed to allocate memory`）。社区临时 flags：`--disable-pinned-memory --reserve-vram 1 --disable-async-offload`；另一用户在 ComfyUI v0.37.2 升级后开始出现此问题。实测数字（2s→514MB reserved / 峰值 23339 MiB）出自 4090，非本机。
- 画质代价：无。
- 证据等级：B（issue 真实、根因明确、flags 有用户报告；现象出在 24GB 卡 + `--disable-dynamic-vram`，非本机 8GB 默认配置）。
- 对 8GB 适用性：若本机 DynamicVRAM 未生效/莫名 OOM，可试这组 flag。注：Tomiigo 8GB 模拟（R1-29）也用 `--disable-pinned-memory --reserve-vram 1.5`。

**R1-27｜note JUNI：RTX 3060 Ti 8GB / 64GB RAM 实测（E4 新增）**
- 来源：R1b-24（sources/24-note-juni-3060ti.md，B）
- 原文测试条件：RTX 3060 Ti 8GB（Ampere/SM86）/ VRAM 8,192MiB / RAM 64GB / 驱动 591.86 / ComfyUI 0.31.0 portable / PyTorch 2.13.0+cu130 / comfy-kitchen 0.2.28 / triton-windows 3.7.1。模型合计 39.55GB（DiT pruned int8_convrot 19.53GB + TE nvfp4_awq 14.61GB + VAE）。
- 报告效果（逐字数字）：
  - 5s（124帧）raw 25步 cold 625.8s / warm 595.7s；5s turbo 6步 warm **185.5s**，峰值 **6,304 MiB**。
  - 10s（243帧）turbo lightx2v 8步 605.7s；15s（362帧）turbo 8步 2113.2s，峰值 7,677 MiB（余 515 MiB）。
  - **15s 成功依赖 PR #15446（Kijai）流式 VAE 解码**，使增长速率降为 1/2.5。
  - 结论：实际瓶颈是系统 RAM 而非 VRAM；EasyCache 对 turbo 无意义（740.8 vs 735.9s，-0.7%）。
- 画质代价：turbo 质量：QrusherZA "Rough"、lightx2v "awkward"、Tutu 降质、joyfox 4步 460.8s 但音频宽带噪声。
- 证据等级：B（作者实机逐字数字）。
- 对 8GB 适用性：与 5070 Laptop 同为 8GB 笔记本（但 RAM 64GB 比本机 32GB 宽裕）。关键经验：15s 长片靠 #15446 流式 VAE 解码；系统 RAM 是隐藏瓶颈。

**R1-28｜ezlaunch 双击安装器（宣传性，D）**
- 来源：R1b-32（sources/32-github-ezlaunch.md，D）
- 原文测试条件：支持 NVIDIA 8GB+（Pascal/20/30/40/50）；profiles：8GB Pascal ~55min/5s@480p；8GB modern(3070/5060) ~5–15min；10–12GB(3060/4070/5070) Turbo 8-step。要求 ~32GB RAM、~100GB 磁盘。
- 报告效果：无法验证时间是否实测（页面 docs/HARDWARE-EVIDENCE 未展开 MEASURED/REPORTED），按**宣传性 D 级**处理。
- 画质代价：无说明。
- 证据等级：D（宣传性，未验证实测）。
- 对 8GB 适用性：作安装入口参考；其“5070 归 10–12GB 档、RAM+PCIe 卸载是限制”与本任务一致，但具体数字勿采信。

**R1-29｜Tomiigo：模拟 8GB 上限实测（RTX 5070 Ti 16GB，C）**
- 来源：R1b-23（sources/23-github-tomiigo-sim8gb.md，C）
- 原文测试条件：RTX 5070 Ti 16GB 用脚本锁 8,188MiB 上限（**非实体 8GB 卡**），RAM 32GB；`--disable-pinned-memory --reserve-vram 1.5`；依赖 EasyCache + KJNodes SageAttention patch。四变体峰值 7,130–7,669 MiB / 8,188 上限，无 swap，0.02–0.04GiB 写盘。
- 报告效果：165→175→190→190s（单次 190s 时 7,444MiB / 8,188）；明确“Timings will not transfer to a real 8GB card”；Windows 未运行。16GB 不受限对比 243帧全分辨率 1,027s vs 降采样 170s。
- 画质代价：无。
- 证据等级：C（模拟上限，非真实 8GB 卡；结论对真实卡不可直接外推）。
- 对 8GB 适用性：提供 5070 系（Blackwell）在 ~8GB 约束下的**参考**峰值与 flags；但须以本机 5070 Laptop 实测为准。

---

## 八、8GB Windows 社区实测基线小结（B 级，供本机对照）
| 硬件 | 配置要点 | 可跑上限 | 参考 |
|---|---|---|---|
| RTX 3070 8GB (Ampere, 48GB RAM) | int8 DiT + NVFP4 TE + Turbo LoRA 12步 SDPA cu130 | T2V 480×832 / I2V 544×800 / 124帧 | R1-01 |
| RTX 3070 Ti 8GB (16GB RAM) | W4A8 mixed + 4-step Turbo + SageAttn + 2-stream | 864×480 / 124帧 / 4步 79.74s | R1-02 |
| RTX 4060 Laptop 8GB (32GB RAM, **Ada Lovelace sm_89，非 Blackwell**) | int8 DiT + NVFP4 TE（**无 Turbo LoRA**） | 0.3MP(416×736) / 5s 8:53→kitchen attn 6:30 | R1-03 |
| RTX PRO 2000 Blackwell 8,151MiB (95.5GB RAM) | NVFP4 DiT + Spectrum + 关系统内存回退 | 800×448 / 15s 908.1s | R1-04 |
| RTX 3060 Ti 8GB (Ampere, 64GB RAM) | int8 DiT + NVFP4 TE + Turbo LoRA 6–8步 + comfy-kitchen | 5s 124帧 turbo 6步 185.5s / 6,304MiB；15s 362帧 2113.2s（依赖 PR#15446 流式 VAE） | R1-27 |

> 共同点：全部依赖量化变体（int8/NVFP4/W4A8）+ 卸载/流式 + 低分辨率（≤0.5MP）+ 少步数（4–12）。5070 Laptop 同为 Blackwell，最接近 R1-04（RTX PRO 2000 Blackwell Laptop）；R1-03（RTX 4060 Laptop）仅是同为 8GB 笔记本 + 32GB RAM（Ada Lovelace，**非 Blackwell**）。本机 RAM 为 32GB（非 R1-04 的 95.5GB），长尺更受系统内存约束。

---

## 九、链接核实表（逐条）
| 编号 | URL | 读取方式 | 核实状态 | 等级 |
|---|---|---|---|---|
| R1-01 | qiita.com/Kiyoaki_SUgiyama/items/bff51bdf389a732dc87c | WebFetch 全文 | 已读，数字逐字 | B |
| R1-02 | qiita.com/makuragadekasugiru810114514/items/471ecc2c041e431359c6 | WebFetch 全文 | 已读，数字逐字 | B |
| R1-03 | note.com/rono_ai/n/n982a96de3812 | WebFetch 全文 | 已读（峰值显存缺失） | B |
| R1-04 | zenn.dev/cosara/articles/vram8gb-minimax-h3-15s | WebFetch 全文 | 已读，数字逐字 | B |
| R1-05 | github.com/modelscope/DiffSynth-Studio/.../VRAM_management.md | WebFetch 全文 | 已读（机制） | B |
| R1-06 | docs.sglang.io/cookbook/diffusion/MiniMax/MiniMax-H3 | WebFetch（超时截断，读前15k+目录） | 结构确认，数字未逐字 | C |
| R1-07 | github.com/thu-ml/SageAttention | WebFetch 全文 | 已读 | B |
| R1-08 | huggingface.co/unsloth/MiniMax-H3-GGUF | WebFetch 全文 | 已读（效果数字缺失） | B |
| R1-09 | github.com/ModelTC/LightX2V | WebFetch 全文 | 已读（H3 倍数未逐字） | B |
| R1-10 | github.com/Comfy-Org/ComfyUI/pull/15456 | WebFetch 全文 | 已读 | B |
| R1-11 | github.com/Comfy-Org/ComfyUI/issues/16175 | WebFetch 全文 | 已读（经摘要） | B |
| R1-12 | huggingface.co/docs/diffusers/main/en/optimization/memory | WebFetch 全文 | 已读（机制） | B |
| R1-13 | github.com/ali-vilab/TeaCache | WebFetch 全文 | 已读 | B |
| NF4(R1-11条) | huggingface.co/DiffSynth-Studio/MiniMax-H3-NF4 ; zenn.dev/nakanishitf（Colab T4） | R1b-12 / R1b-14 已打开 | 已读，补数字，重定级 B/C | B/C |
| pinned(R1-26条) | github.com/Comfy-Org/ComfyUI/issues/15663 | R1b-25 已打开 | 已读，根因明确，重定级 B | B |

## 十、只看了标题/摘要未细读（不写结论）
- SOURCES.md 列出的第三方“榜单”站（whichh3.com、genvidkit.com、openclawdc.com、localrigcheck.com 等）——D 级线索，未打开，仅作后续搜索入口。
- note JUNI（note.com/j_alpha_lab/...）、ezlaunch（github.com/yume-arasaki/ezlaunch-minimax-h3）、Tomiigo 8GB 指南（github.com/Tomiigo/minimax-h3-16gb）——**已由 T1 打开并留底（sources/24、32、23 → R1b-24、R1b-32、R1b-23），移出本节**，分别见 R1-27、R1-28、R1-29。
- SGLang cookbook 的 RTX 5090 / 4090 / H200 Cache-DiT 具体基准数字——页面超时未抽取，未独立核实。

## 十一、事实 / 推测 / 建议 区分
- **事实（B 级实测）**：8GB 卡可跑 H3（需量化+卸载+低分辨率）；Blackwell NVFP4 DiT 11.66GB 跑通 15s；关系统内存回退消除机崩但缩容量；供电 25W→60W 带来 2.76×；SageAttention 在 3070Ti（Ampere）可用更快；#15263 为 sm_120 FP8 PV kernel 长序列（>~160k token）纯噪声、ComfyUI 0.31.1 起 KJNodes `auto` 报告修复（B，单一作者）；3070（Ampere）作者另报 H3+SageAttention 纯噪声（单列现象，根因或与 #15263 不同）；稀疏注意力在 24GB 卡 H3 上也 OOM。
- **推测（C/D）**：5070 Laptop 的 NVFP4 硬件加速与驱动 cu130 支持（基于 sm_120 与侦察）；SGLang 无 8GB 配方故需 WSL2 降配；DiffSynth NF4 9.8GB“最低 8GB”宣称（未核实）；32GB RAM 加载卡死传闻（D）。
- **建议**：先用 SDPA + NVFP4 DiT（或 int8）+ Turbo LoRA（8/12步）+ DynamicVRAM 跑通；满血供电并监控 Power Cap；关闭或谨慎使用系统内存回退；SageAttention/稀疏注意力待画质比对后再启用；长尺依赖 PR #15456 的 VAE 解码修复。

## 十二、下一轮最值得验证的 3 项（附最小实验与停止条件）
> 仅列有可靠依据、可在本机（5070 Laptop 8GB/32GB/Win11）实做的项；无可靠把握的不凑数。

1. **NVFP4 DiT 在 5070 Laptop 的可用性与峰值**（最高优先，因 Blackwell 原生且体积最小 11.66GB）
   - 最小实验：ComfyUI（含 PR#15456）+ DiT `fl2va NVFP4` + TE `nvfp4_awq` + Turbo LoRA 8步 + SDPA + `--reserve-vram 1.0`；测 416×736/124帧 与 800×448/124帧 的峰值显存与是否 OOM。
   - 停止条件：若 800×448 峰值 >7.8GiB 或 OOM → 退回 640×352；若 NVFP4 内核报错/极慢（疑似无加速）→ 改 int8 ConvRot。记录耗时与峰值。
2. **系统内存回退策略的取舍**（Blackwell 笔记本特有崩溃风险）
   - 最小实验：同配置下分别测“默认回退”与“优先无系统内存回退”，跑 124 帧与 209 帧各 3 次，记录是否机崩 / OOM / 最长可生成帧数。
   - 停止条件：若默认回退在 >124 帧整机崩 → 永久关闭回退并锁定 ≤124 帧或降分辨率；若关闭后 124 帧也 OOM → 启用 GGUF/更小量化。
3. **SageAttention3 在 H3 的画质安全性**（潜在 2.4–2.7× 注意力加速，受 E2 修正影响）
   - 背景：#15263 = sm_120 FP8 PV kernel 在 ~160k token 以上出纯噪声，ComfyUI 0.31.1 起 KJNodes 设 `auto` 报告可用（B，单一作者）。3070（Ampere）另有独立纯噪声报告，不与 #15263 合并。
   - 最小实验：SDPA 基准生成 1 段 480×832/8步；切 SageAttention3（Blackwell, CUDA>=12.8，KJNodes `auto`）生成同 seed 同参数；逐帧目视+差异图比对是否纯噪声/伪影（尤其长序列 >154k token）。
   - 停止条件：若任何帧出现纯噪声/结构性伪影 → 禁用 SageAttention（回退 SDPA）；若画质一致且更快（尤其长序列）→ 才纳入常规。

---

## 十三、审核修正记录（R1b-fix，2026-11-04）

本日志经审核退回，逐条修正 E1–E4。原始版本另存为 `RESEARCH-LOG-001.v1.md`。所有数字均来自 `./sources/R1b-*.md`（复制自 T1 已打开留底 sources/12、14、24、25、32、23），无凭记忆书写。

| E编号 | 原文（错误） | 改后 | 依据文件 |
|---|---|---|---|
| E1 | 第八节表格把 RTX 4060 Laptop 写成 `(32GB RAM, Blackwell)`；小结称“5070 Laptop 最接近 R1-03/R1-04（Blackwell + 8GB）” | 4060 Laptop = **Ada Lovelace sm_89，非 Blackwell**；5070 Laptop 同为 Blackwell 仅接近 R1-04（RTX PRO 2000 Blackwell Laptop）；4060 Laptop 仅同 8GB 笔记本 + 32GB RAM | 架构事实（RTX 40 系=Ada Lovelace sm_89；Ampere 是 RTX 30 系 sm_86）；sources/21、R1-03-note 均未提 Blackwell；E2 同据 |
| E2 | R1-18 把 #15263 写成“3070 上 H3 + SageAttention 纯噪声”；“#15263 是否已修”标为不确定 | #15263 实为 **sm_120 FP8 PV kernel 在 ~160k token 以上出噪声（<~154k 干净）**，ComfyUI **0.31.1 起 KJNodes 设 `auto` 报告可用（B，单一作者）**；3070（Ampere）噪声单列，不与 #15263 合并 | sources/28-comfyui-issue-15263.md（R1b-28）；sources/R1-01、R1-02 |
| E3 | R1-11 NF4、R1-26 #15663 原标 D“未打开” | 已读留底并重定级：R1-11 → B（变体真实）/ 8GB 可行性 C（无真实 8GB 卡实测，T4 14.56GiB 仅 288×160 成功）；R1-26 → B（issue 真实、根因明确、flags 有报告） | sources/12、14（R1b-12、R1b-14）；sources/25（R1b-25） |
| E4 | 第十节未细读列 note JUNI、ezlaunch、Tomiigo | 已读 sources/24、32、23，移出未细读区；JUNI 补为 **R1-27**；ezlaunch 补为 **R1-28（D 宣传）**；Tomiigo 补为 **R1-29（C 模拟）** | sources/24（R1b-24）、32（R1b-32）、23（R1b-23） |

“下一轮最值得验证的 3 项” #3 已随 E2 同步更新（#15263 现标“0.31.1 起报告可用”，验证时仍须逐帧比对长序列噪声）。

F1（R2 审核，2026-11-04）：更正 E1 依据栏的架构名。

---
*结束。来源摘录见 ./sources/R1-*.md 及 R1b-*.md。本日志无 A 级（本机复现）结论。*
