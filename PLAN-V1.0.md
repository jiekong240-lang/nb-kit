# MiniMax H3 本地部署 V1.0 方案（PLAN-V1.0）

**目标机器**：HP OMEN 16，RTX 5070 Laptop 8GB（Blackwell，sm_120，驱动 616.92，显存 8151 MiB），i9-14900HX，32GB RAM，Win11 家庭版 26200，PowerShell 5.1
**ComfyUI 现状**：源码 + venv 安装，路径假定 `C:\Users\testuser\WorkBuddy\comfyui\ComfyUI`，版本未知（待步骤只读确认）
**编写日期**：2026-11-04
**证据等级约定**：A=本机已复现（本任务不可能出现 A）；B=外部可复现（公开源码/官方文档+完整方法+第三方复现）；C=研究候选；D=未经证实
**本方案性质**：**纯方案文档，未经任何真机执行，无任何 A 级结论，无“本机验证成功”表述**。最终由 Junlong 团队在真机按本方案与技术手册测试。

> 配套文件：`sources/P1-hf-api.md`（HF API 字节数）、`sources/P1-template.md`（模板 JSON 节点）、`sources/P1-releases.md`（ComfyUI 版本/PR）。DOSSIER.md 的 F2 小修与备份 `DOSSIER.v1.md` 由同一任务完成。

---

## §1 目标与“通过”的定义

**总目标**：在上述真机上，**真实、可重复**地生成 MiniMax H3 视频（FL2VA / T2V，5 秒级）。

**“通过”的严格定义**（须全部满足）：
1. 同一套配置（§2 基线表每一项不变）、同一个固定 seed；
2. **连续 3 次**提交都成功出片（无 OOM、无崩溃、无音频/视频纯噪声、无环状伪影）；
3. 每次输出文件落盘、可被播放、时长≈5 秒（124 帧 @24fps）；
4. 每次记录以下四项指标（单位明确，不得编造，缺失记“缺失”）：
   - 耗时（UI 提交→产出，秒）；
   - 显存峰值：**dedicated GPU memory 峰值** 与 **shared GPU memory 峰值**（均来自任务管理器“性能→GPU”或 `nvidia-smi`）；
   - 进程 RAM 峰值（PowerShell：`Get-Counter '\Process(*)\Working Set - Private'` 或 `Get-Process` 的 `WorkingSet64` / `PeakWorkingSet64` / `PrivateMemorySize64`，配合任务管理器“性能→内存”；**`nvidia-smi` 只报 GPU/显存，测不了系统 RAM**，勿用它测 RAM）；
   - **commit 峰值**（任务管理器“性能→内存”的“已提交”项）；
5. 三次的显存 dedicated 峰值、shared 峰值、进程 RAM 峰值、commit 峰值均记录；若三次中有任意一次失败或任一指标触发 §7 停止条件，则本轮“不通过”，按 §7/§5 处置，不得宣称成功。

**“可重复”的额外要求**：3 次之间不手动清理模型缓存、不重启 ComfyUI 进程（检验内存/显存是否泄漏，对应 R2-11 的“平坦”判定）。

---

## §2 选定基线配置表（V1.0）

> 每行给出：组件、精确文件名、字节数（HF API 实测，2026-11-04）、GB(10^9) 与 GiB(2^30) 双写、来源链接、等级。
> 字节数来源：Comfy-Org/MiniMax-H3、Abiray/MiniMax-H3-Turbo-Lora-Pruned-ComfyUI 的 HF API（`sources/P1-hf-api.md`）。分辨率/步数/shift/采样器等来自官方模板 JSON（`sources/P1-template.md`）与 DOSSIER §6 + RESEARCH-LOG-002，均标等级。

| # | 组件 | 精确文件名 | 字节数 | GB(10^9) | GiB(2^30) | SHA256(HEX, HF lfs.oid) | 来源 | 等级 |
|---|---|---|---:|---:|---:|---|---|---|
| 1 | ComfyUI 版本 | 固定 tag `v0.32.0`（git checkout） | — | — | — | — | `sources/P1-releases.md`、P1b-tag.md | B（tag 形式存在；v0.31.1 同为 tag，但 v0.32.0 含完整 H3 修复与内置 0.11.39 模板） |
| 2 | PyTorch/CUDA | `torch==2.11.0+cu130` `torchvision==0.26.0+cu130` `torchaudio==2.11.0+cu130` | — | — | — | — | Qiita3070（sources/19）；F2 修正 | B |
| 3 | DiT（FL2VA） | `minimax_h3_fl2va_pruned_int8_convrot.safetensors` | 20,970,379,616 | 20.97 | 19.53 | `e889202c41dafb67b10d67b97f0d8541508036a6090af23425a5c2615d03c47a` | Comfy-Org/MiniMax-H3（sources/07、P1-hf-api） | B |
| 4 | 文本编码器 TE | `qwen3vl_32b_minimax_h3_nvfp4_awq.safetensors` | 15,687,142,551 | 15.69 | 14.61 | `35a88d51044231fe332301d7a62aa81e3f2cba62febeb446e2c1e3e0ef76f2c6` | 同上 | B |
| 5 | Video VAE | `minimax_h3_video_vae_fp16.safetensors` | 5,207,808,496 | 5.21 | 4.85 | `7c1f131492e7eddacaac9069a61b81bdd39de5cc96561e677c5eab1cdce5e522` | 同上 | B |
| 6 | Audio VAE | `minimax_h3_audio_vae_fp32.safetensors` | 605,254,808 | 0.605 | 0.563 | `8e505d95dd1561d47abd43d4238fd40d9bb1ae9e147ed0a4cba778d76ae4db48` | 同上 | B |
| 7 | Turbo LoRA | `minimax_h3_turbo_4step_ckpt600_ema_V4.safetensors`（Abiray，Qiita 480×832/8步实测所用 LoRA，见 §3.3） | 620,285,656 | 0.620 | 0.578 | `7d68591e1aff9ace24b5138da2af02272cd9de125c5c261f5493ae8aa99c715c` | Abiray/MiniMax-H3-Turbo-Lora-Pruned-ComfyUI（sources/16、P1-hf-api） | B（Qiita 8GB 实测） |
| 8 | 分辨率 | **480×832**（32 的倍数，竖版，见 §3.4） | — | — | — | — | Qiita3070（sources/19） | B |
| 9 | 帧数 | **124**（=5.17s @24fps） | — | — | — | — | Qiita3070（sources/19）；zenn（sources/22） | B |
| 10 | 步数 | **8**（Qiita 8 步实测 + 官方 8 步 LoRA 训练区间） | — | — | — | — | Qiita3070（sources/19）；Abiray（sources/16） | B |
| 11 | sigma shift | **video 12 / audio 6**（显式覆盖，不靠模板默认） | — | — | — | — | Abiray README（sources/16）；Qiita3070（sources/19）；P1b-nodes.md | B |
| 12 | sampler/scheduler | `res_multistep` + `simple` | — | — | — | — | note4060（sources/21）；zenn（sources/22） | B（sampler）/ C（scheduler 名） |
| 13 | attention | **SDPA**（V1.0 默认，避 sm_120 Sage 噪声） | — | — | — | — | Qiita3070（sources/19）；#15263（sources/28） | B |
| 14 | 固定 seed | **556589502035082**（与 §3.5 一致，即模板 RandomNoise 默认值；三次重复同 seed） | — | — | — | — | 测试控制项 | — |
| 15 | 启动参数 | 见下方 §2.1 逐条解释 | — | — | — | — | #15663（sources/25）、R2-11/14；P1b-cliargs.md | B（来源）/ C（迁移） |
| 16 | 页面文件 | 固定大小：初始 32768 MB / 最大 65536 MB，置于 NVMe | — | — | — | — | R2-10（sources/R2-10） | C+（单源） |
| 17 | NVIDIA Sysmem Fallback | “CUDA - 系统内存回退策略 → **优先无系统内存回退**” | — | — | — | — | zenn（sources/22） | B（效果）/ **需 Junlong 批准** |

**最小下载体积（仅 FL2VA，不含 ref2va）**：19.53 + 14.61 + 4.85 + 0.563 + 0.578 ≈ **40.1 GiB**（按 GiB 相加）；约 43.1 GB（10^9）。加 ref2va（19.53 GiB）再 +19.53 GiB。需 NVMe 剩余 **≥ 70 GiB** 才允许开始下载（见 §7“下载前最少剩余空间”）。

**备选文件 SHA256（供 §9 下载校验）**：
- int8 Video VAE `minimax_h3_video_vae_int8_convrot.safetensors` = `52a2c8c73583c86e4f41cdcce3a6ad0ea562987bc0bf3d60a0cef5f5c8e60c0e`（F3 备选）
- 官方 8 步 LoRA `minimax_h3_fl2v_turbo_8step_v1.0_comfyui_bf16.safetensors` = `2339acdf19bfe123f46b971ea35d367a84adb85de43627e1eceafa5a5b2b111e`（F0 对照）

> SHA256 取自 HF API tree 的 `lfs.oid`（Comfy-Org/MiniMax-H3 与 Abiray/MiniMax-H3-Turbo-Lora-Pruned-ComfyUI），等级 B；oid 留底见 P1-hf-api.md 与 P1b-*。

### §2.1 启动参数逐条解释（引用 docs.comfy.org/development/comfyui-server/startup-flags，sources/R2-14）

| 参数 | 作用（help 原文） | 为何在 V1.0 用 | 等级 |
|---|---|---|---|
| `--disable-pinned-memory` | 禁用锁页（pinned）内存：`Disable pinned memory use.` | Tomiigo 在 8GB 模拟上关掉后 0 次 swap；省 RAM 是 32GB 机器的首要矛盾（R2-13） | B（sources/23）/ C（迁移） |
| `--reserve-vram 1` | 预留 1 GiB 显存不给模型：`Set the amount of vram in GB you want to reserve ...` | #15663 作者三旗之一；shared 溢出是 WDDM 慢路径与蓝屏根因（R2-13、5.8） | B（sources/25） |
| `--disable-async-offload` | 禁用异步卸载：`Disable async weight offloading.` | #15663 作者用此三旗解决 H3 短片 OOM（根因 comfy-aimdo 未启用 DynamicVRAM） | B（sources/25） |
| `--cache-none` | **降低 RAM/VRAM 占用，代价是每次运行都重新执行每个节点**（help 原文：`Reduced RAM/VRAM usage at the expense of executing every node for each run.`） | 省 RAM（R2-11）；注意 **不省单任务内部峰值**，故非万能 | B（sources/R2-14）/ v0.31.0 起已存在（cli_args.py 原文） |
| `--fast-disk` | **优先以磁盘后备的动态加载/卸载取代未锁页（unpinned）RAM；在快速 NVMe 上可能更快**（help 原文：`Prefer disk-backed dynamic loading and offload over unpinned RAM. Can be faster for users with fast NVME disks.`）。注意：它取代的是 unpinned RAM，**并非“权重绝不进进程 RAM”**——mmap/流式仍需少量映射 | 省 RAM 的正解（R2-02/R2-03/R2-14）；与 `--disable-pinned-memory` 协同 | B（sources/R2-14）/ v0.31.0 起已存在（cli_args.py 原文） |

> 五个启动参数自 **v0.31.0** 起均已存在于 `comfy/cli_args.py`（已用 v0.31.0 / v0.31.1 / v0.32.0 三版 raw 文件核对 help 原文，见 sources/P1b-cliargs.md）；“引入版本”不再标 D。`--disable-fast-disk` 仅 master 分支有，v0.32.0 无。真机仍以 `main.py -h` 实测为准。
> **Sysmem Fallback 设置属修改系统/驱动级配置，标“需 Junlong 批准”，本方案不自动执行**（见 DOSSIER §9.5 第 4 项）。

---

## §3 选择理由（逐条对比备选项，依据真实数据）

### 3.1 DiT 量化：int8_convrot（基线） vs W4A8 vs fp8_scaled vs w6a8 vs GGUF

| 变体 | 文件 / 字节 | GiB | 依据与判断 |
|---|---|---:|---|
| **pruned int8_convrot**（选） | `…_pruned_int8_convrot.safetensors` = 20,970,379,616 B | 19.53 | Comfy-Org 建议“prefer int8_convrot if able to use cu130”；Qiita3070 480×832/8 步/345s/7.3GB 完整 B 级实测【B，07/19】 |
| W4A8 mixed | `…_pruned_w4a8_mixed.safetensors`（AX1Y2JP）= 12,540,884,878 B | 11.68 | 3070 Ti 4 步 79.74s 实测【B，20】；但需最新 ComfyUI + comfy-kitchen，且为社区仓库（模型卡空壳，sources/36）【C】→ 作 F2 备选 |
| fp8_scaled | `…_pruned_fp8_scaled.safetensors` = 20,958,205,608 B | 19.52 | Comfy-Org 明确“fp8_scaled only if cannot use int8_convrot”【B，07】→ 不选 |
| w6a8 | `…_pruned_w6a8.safetensors` = 15,983,746,636 B | 14.88 | 无 8GB 专项实测【D】→ 不选 |
| GGUF（Q2_K 等） | unsloth/joeygambino 仓库 | 6.26–19.97 GiB | 需 ComfyUI-GGUF 补丁（架构白名单无 minimax_h3，sources/10/40）【B】；无 8GB H3 实测【D】→ 作 F4 备选 |

**结论**：基线选 int8_convrot（最强 B 级 8GB 证据 + Comfy-Org 官方首选）。

### 3.2 Video VAE：fp16（基线） vs int8_convrot

- fp16（选）：`minimax_h3_video_vae_fp16.safetensors` = 5,207,808,496 B（4.85 GiB）。Qiita3070 用 fp16 得到干净 345s/7.3GB【B，19】，是支撑“可重复”的最强证据；任务假设亦列 fp16 为首。
- int8_convrot：`minimax_h3_video_vae_int8_convrot.safetensors` = 2,811,065,184 B（2.62 GiB），是**模板默认**。更省显存/内存约 2.2 GiB，且 #15446 流式解码对其同样适用；但 int8 为近似精度，质量是否等同 fp16 无逐像素 B 级对比【C】。
- **取舍**：V1.0 主用 **fp16**（可重复证据最强）；若 32GB 内存/显存吃紧，切 int8_convrot（见 F3），代价为可能的轻微质量差异（标 C/D）。

### 3.3 LoRA 与“全链路照搬 Qiita”的自洽（关键修正）

**回到 sources/19（Qiita 3070 8GB）原文，逐字摘录 480×832 那一行**（第 32 行）：

| 分辨率 | 像素 | 步 | 耗时 | 峰值(VRAM) |
|---|---|---|---|---|
| 480×832 | 399K | 8 | 345.2s | 7,322 MiB |

- 上下文（sources/19）：该主表所用 LoRA = §模型文件所列 `minimax_h3_turbo_4step_ckpt600_ema_V4.safetensors`；采样 `res_multistep`/`simple`（§采样）；Video VAE = fp16；DiT = int8_convrot。Qiita 原文**未显式记录 shift 值**（本方案 shift 12/6 来自 Abiray README + 通用 MiniMaxH3 默认，见 P1b-nodes.md）。

- **基线 = 完整照搬 Qiita 全链路（B 级）**：DiT int8_convrot + TE nvfp4_awq + Video VAE fp16 + **LoRA = ema_V4（Abiray）** + 480×832 + 124 帧 + 8 步 + res_multistep/simple。这样 §1/§6 的参考耗时 **345.2s / 7,322 MiB** 才与基线真正可比（同 LoRA、同链路）；不再出现“用 ema_V4 测的 345s 却配官方 8 步 LoRA”的自相矛盾。
- **官方 Comfy-Org 8 步 LoRA（F0 对照，非基线）**：`minimax_h3_fl2v_turbo_8step_v1.0_comfyui_bf16.safetensors` = 1,956,193,000 B（1.822 GiB），位于 Comfy-Org/MiniMax-H3 `loras/`，是 `main` 分支模板默认加载的 LoRA（sources/P1-template）。等级 **B（来源/兼容）/ C（8GB 专项未独立复现）**。作为官方路径对照项见 §5 F0：真机先跑通 ema_V4 基线后，再用它验证官方路径兼容性。
- **Abiray ema_V4（基线，B 级 8GB 社区实测）**：`minimax_h3_turbo_4step_ckpt600_ema_V4.safetensors` = 620,285,656 B（0.578 GiB）。Qiita3070 用 ema_V4 8 步干净无伪影；ckpt850_V1 才有环状伪影（sources/19）。文件更小，回滚简单。

> 说明：旧版把“fp16 VAE 因 Qiita 选、LoRA 却用官方 8 步、而 345s 来自 ema_V4”混搭，证据不自洽；现统一为 Qiita 全链路（B），官方 8 步降为 F0 对照。DOSSIER §6.2 原写 ema_V4，与此一致。

### 3.4 分辨率 / 帧数 / 步数 / shift / attention / token

- **分辨率 480×832 = 宽 480、高 832 → 竖版（portrait，约 9:16）**（Qiita 原文行即此值，见 §3.3 摘录）。32 的倍数（B）。模板默认 1344×768（横版 16:9）过高，必须覆盖。
  - **在 v0.32.0 内置模板（0.11.39）里如何设置**：该 t2v 模板用 `MiniMaxH3ImageToVideo`，**没有 `EmptyMiniMaxH3LatentAV`**。width/height 是子图输入，由顶层 `ResolutionSelector(115)` 经连线（link 219→width、link 220→height）传入；模板自带说明表（节点 118 MarkdownNote“Size Settings Reference”）给出的 16:9 横版档位：0.2MP→608×352、0.3→736×416、0.4→864×480、0.5→960×544——**没有 480×832（竖版）这一档**。因此必须**先断开 ResolutionSelector(115) 到子图的 width/height 连线（link 219/220）**（或把子图对应输入改为手填），再在子图输入上手动填 width=480、height=832。v0.11.39 模板默认 width/height 经 ResolutionSelector 给出 1344×768（横版），需改。
- **帧数 124**（B，多源一致）；长片（>124 帧）在 32GB 风险陡增（R2-11 commit 120–145%、R2-13），V1.0 锁定 124。
- **步数 8**（官方 8 步 LoRA 训练区间 + Qiita 8 步实测；12 更优但更慢，16 无改善，sources/19）。
- **shift video12 / audio6**：Qiita 未记录 shift；本方案采用通用 MiniMaxH3 默认 video=12 与音频覆盖 audio=6（不守则音频坏/噪，Abiray README + #15263）。**v0.11.39 模板无 `MiniMaxH3SigmaShift` 节点**（见 P1b-template.md），故 audio shift 无法靠模板默认设成 6，必须手动插入该节点（见 §3.5）。
- **attention SDPA**（避开 #15263 的 sm_120 Sage 噪声）。
- **“约 12k token”计算过程（修正，依据 model.py）**：
  - VAE 空间压缩 16×（`_empty_av_latent`：`video = zeros([B,24,latent_t, height//16, width//16])`）；时间维由 `video_latent_t(frame_count)` 给出：124 帧 → `((124-5)//17)*5+2 = 37` 个时间 latent。
  - patch：`MiniMaxH3Model` 的 `patch_size=(1, 2, 2)`（model.py），即 1 token = 1 时间 latent 帧 + 2×2 空间 latent 格 → 每帧 token 数 = (H/16/2)×(W/16/2) = (H/32)×(W/32)。
  - **精确（model.py 口径）**：token = 37 × (832/32) × (480/32) = 37 × 26 × 15 = **14,430 ≈ 14.4k**。
  - **#15263 近似口径**：token = 宽/32 × 高/32 × 帧/4 = 480/32 × 832/32 × 124/4 = 15 × 26 × 31 = **12,090 ≈ 12k**（其「帧/4」把 124 帧近似成 31 段，与精确 latent_t=37 有出入）。
  - 两种口径都**远低于 #15263 的 167k 噪声阈值**，SDPA 安全区成立；原“约 12k”是 #15263 近似口径，精确约 14.4k。

### 3.5 模板导入做法（v0.32.0，修正 P1）

v0.32.0 界面 Templates → Video → MiniMax H3 实际加载的是 **workflow_templates v0.11.39** 版（对应 `comfyui-workflow-templates==0.11.39`，见 P1b-requirements.md）。实测该模板：**无 LoRA 节点、Video VAE = fp16、BasicScheduler simple/20/1、seed 556589502035082**（P1b-template.md）。`main` 分支上有 Lightning LoRA 开关 + int8 VAE + 8 步的版本在 v0.32.0 界面**看不到**。

**做法 A（选此，推荐）**：用 0.11.39 模板作底，手动插入两个节点：
1. `LoraLoaderModelOnly`：插在 **UNETLoader 的 MODEL 输出 →（新节点）→ 原 BasicScheduler.model 与 BasicGuider.model 共用的 model 通路**上。操作：先断开模板里 UNETLoader(6) 直接到 BasicScheduler(9).model（link 5）和到 BasicGuider(16).model（link 193）的两条线；把 UNETLoader 的 MODEL 输出改接到 LoraLoaderModelOnly 的 model 输入，加载 `minimax_h3_turbo_4step_ckpt600_ema_V4.safetensors`，strength 1.0，取其 MODEL 输出备用。
2. `MiniMaxH3SigmaShift`：接在 LoraLoaderModelOnly 的 MODEL 输出之后（model 流上、LoraLoader 之后），设 `shift_video=12.0`、`shift_audio=6.0`（节点默认 audio=3.0，必须改 6）。**其 MODEL 输出须同时接到两个地方**：① BasicScheduler(9) 的 model 输入（即原 link 5 的去向）；② BasicGuider(16) 的 model 输入（即原 link 193 的去向）。sigmas 由 BasicScheduler 按 model 的 model_sampling 算出，若只接 BasicGuider，shift 不会作用到 sigma 调度（此为 P1c 修正的实质错误）。
3. 分辨率/帧数/步数（详见 §3.4）：该 t2v 模板用 `MiniMaxH3ImageToVideo`，**没有 `EmptyMiniMaxH3LatentAV`**。width/height 是子图输入，由顶层 `ResolutionSelector(115)` 经连线（link 219→width、link 220→height）传入——先断开这两条连线（或改成手填）再填 480/832；length 不由手填，而是由 `ComfyMathExpression` 从 `duration`（PrimitiveFloat 节点，模板默认 2 秒，节点 111）算出（17k+5 网格：2s→56 帧、5s→124 帧，见 §3.4 公式），**必须把 duration 改为 5** 才得 124 帧（非直接覆盖 length=124）；BasicScheduler 改 steps=8（模板默认 20）。

**理由**：A 完全基于 v0.32.0 自带、源码内已存在的节点（LoraLoaderModelOnly、MiniMaxH3SigmaShift 均在 v0.32.0 中），无前端兼容性未知风险，等级 B/C 清晰，回滚只换文件。

**做法 B（备选，标 C）**：直接把 `main` 分支的 `video_minimax_h3_t2v.json`（含 Lightning LoRA 开关、int8 VAE、8 步）拖入 ComfyUI。指挥 BOT 已查：其用到的 `ComfyMathExpression`、`ComfySwitchNode`、`ResolutionSelector` 等节点在 v0.32.0 源码里都存在；但**前端兼容性未验证**（该模板非 v0.32.0 绑定版本），标 C。仅当 A 在真机出现节点缺失/加载失败时再试 B，且需 Junlong 批准。

---

## §4 风险登记表

| 编号 | 风险 | 触发信号 | 来源 | 等级 | 缓解办法 |
|---|---|---|---|---|---|
| R-01 | 32GB RAM 峰值导致假死/极慢 | 进程 RAM 峰值 > 28 GB 或 commit > 30 GB；shared GPU memory 上升 | R2-11、R2-13 | B（机制）/ D（阈值） | `--disable-pinned-memory` + `--fast-disk` + `--cache-none`；降分辨率（F1） |
| R-02 | 页面文件不足→无报错闪退 | Windows 默认页面文件过小（如 2GB）时生成中途窗口消失 | R2-10 | C+ | 固定 pagefile 初始 32768/最大 65536 MB（NVMe，需批准） |
| R-03 | TE mmap 滞留吃掉 RAM | TE 卸载后进程 RSS 不降（强引用致 13GB 映射滞留） | R2-01（#17122） | B | 编码后强制 unload + 清引用；监控 RSS 在采样后是否回落 |
| R-04 | 共享显存慢路径（WDDM） | shared GPU memory > 2 GB；步时从秒级变分钟级（>7GB 块带宽 408→12GB/s） | R2-13 | B | 降分辨率优先于加 `--vram-headroom`；必要时关 Sysmem Fallback（需批准） |
| R-05 | #15251（0.30.0 VAE 设备不一致崩） | 真机版本恰为 0.30.0 | #15251（sources/29） | B | 版本 < v0.32.0 即 checkout v0.32.0（§2 #1） |
| R-06 | #15263（sm_120 Sage 噪声） | 误用 SageAttention 且 token>167k → 视频+音频纯噪声 | #15263（sources/28） | B | V1.0 默认 SDPA；Sage 仅限 <167k token 且 `auto` 模式（V1.1 提速项） |
| R-07 | #15663（短片显存低估 OOM 且“粘住”） | 2s 片段估 514MB、保留 16GB 常驻、峰值 28.9GB；OOM 后重启仍 OOM | #15663（sources/25） | B | 启动三旗（§2.1）；OOM 后改任一节点参数或插 ShowAny 节点 |
| R-08 | VAE 解码 OOM（长片段/重试无效） | 解码峰值随帧数线性涨；重试不释放其他模型 | #15456（sources/26） | B | 锁 124 帧；确保含 #15446 的 ComfyUI（流式解码，峰值 3485→607MB） |
| R-09 | 蓝屏/重启（cosara 8 次 IOMMU） | HYPERVISOR_ERROR(0x20001)，长片段 VRAM 溢出回退系统内存 | zenn（sources/22） | B | 锁 124 帧；Sysmem Fallback=优先无回退（需批准）；原厂 140W 供电 |
| R-10 | 笔记本供电降频 | 85W USB-C 坞把功耗墙压到 25W，SW Power Cap 激活，速度差 2.76× | zenn（sources/22） | B | 必须原厂 140W 直插，禁用低功率 USB-C 坞 |
| R-11 | 磁盘空间不足 | NVMe 剩余 < 20 GB（下载 ~41GiB + 工作空间） | DOSSIER §9.2 | B/C | 下载前校验剩余；不足即停，清理需批准 |
| R-12 | ComfyUI 升级破坏现有环境 | `git pull`/同步 requirements 后节点缺失或 OOM 新变量 | DOSSIER §9.5 | D（流程） | 固定 checkout v0.32.0；任何 git 变更需批准；回滚见 §8 |
| R-13 | 许可证 | 商用门槛/地域；本地输出商用需 Comfy 许可（与许可证正文口径冲突） | DOSSIER §9.1 | B | 个人非商用合规；商用前书面确认 MiniMax + Comfy 双渠道 |

---

## §5 备选配置（按优先级排序）

| 编号 | 触发条件（何时切换） | 改了什么 | 预期代价 | 依据 | 等级 |
|---|---|---|---|---|---|
| **F0**（对照，非备选） | 基线（ema_V4）跑通后，验证官方路径兼容性 | LoRA 换 `minimax_h3_fl2v_turbo_8step_v1.0_comfyui_bf16.safetensors`（Comfy-Org 官方 8 步，见 §3.3） | 官方 LoRA 在 8GB 无独立实测（8GB 专项标 C）；文件更大 1.822 GiB | §3.3、P1b-template.md | B（来源）/ C（8GB） |
| **F1** | 基线 480×832 仍 OOM / shared>2GB / commit>30GB | 分辨率降到 **416×736**（0.3MP，note4060 稳过）或 **352×608** | 画质下降、像素更少 | Qiita3070 352×608/213s/6.7GB；note4060 416×736【B，19/21】 | B |
| **F2** | 基线 int8 DiT 在 8GB 仍卡（流式带宽不足）且 comfy-kitchen 可用 | DiT/TE 换 **W4A8 mixed**（AX1Y2JP，DiT 11.68 GiB） | 需最新 ComfyUI+comfy-kitchen；社区仓库无卡面审核【C】 | 3070 Ti W4A8 4 步 79.74s【B，20】；sources/36 | B（3070Ti）/ C（迁移） |
| **F3** | 32GB/8GB 显存吃紧，fp16 VAE 占太多 | Video VAE 换 **int8_convrot**（2.62 GiB，省 2.2 GiB） | int8 近似精度，可能轻微质量差异 | 模板默认即 int8 VAE；#15446 同样适用 | B（来源）/ C（质量） |
| **F4** | int8/W4A8 均失败，愿走非 ComfyUI 原生生态 | 改用 **GGUF / sd.cpp** 路线（TE 可放 CPU，`--backend te=cpu`） | 缺 ComfyUI 生态的 Turbo LoRA/注意力补丁；无 8GB H3 实测 | sources/09/10/34/35/40 | D（无 8GB 实测） |
| **F5** | 上述均不稳，愿引入独立推理器 | **DMAD** 路线（`github.com/Yzmblog/DMAD`）：TE 逐层流式、权重盘上、8GB 档 <7GiB | **非 ComfyUI 原生**（独立推理器）；ComfyUI 内能否复现未验证 | R2-07（sources/R2-07） | B（DMAD 自测）/ C（非 ComfyUI） |

> DMAD 明确**不是 ComfyUI 原生管线**（sources/R2-07），若采用需单独技术手册，不在本 V1.0 ComfyUI 路径内。

---

## §6 验收标准（可量化）

1. **通过率**：§1 定义的连续 3 次成功 = 100%；任一失败即“不通过”。
2. **显存**：dedicated GPU memory 峰值 < 8151 MiB（不溢出到 shared）；若 shared > 0 但 < 2 GB 且耗时未恶化，记“临界”但不判失败。
3. **内存**：进程 RAM 峰值 < 28 GB；commit 峰值 < 30 GB（阈值依据不足，标 D，待真机校准）。
4. **耗时**：480×832/124 帧/8 步，参考 Qiita3070 7.3GB/345s（3070）；5070 Laptop 无参照，仅记录，不预设上限。
5. **质量**：输出可播放、无纯噪声、无环状伪影、音频无低音缺失（EasyCache 未用，避 #15326）。
6. **可复现**：3 次同 seed 输出一致（或视觉一致）；指标波动在合理范围。
7. **数据完整性**：每次的耗时/双显存/RAM/commit 四项齐全；缺失项不得编造，记“缺失”。

---

## §7 停止条件（出现即立刻停并上报 Junlong）

**下载前最少剩余空间（强制校验，低于即停，不开始下载）** = 模型合计 + venv 备份估算 + 20 GiB 余量：
- 模型合计（基线 FL2VA，ema_V4 LoRA）：DiT 19.53 + TE 14.61 + VideoVAE 4.85 + AudioVAE 0.563 + LoRA 0.578 ≈ **40.1 GiB**（≈43.1 GB）。
- venv 备份估算：ComfyUI venv（含 torch 2.11 cu130 等）解压后约 **8 GiB**（标 C 估算，待真机 `Get-ChildItem` 实测；若用 standalone build 可能更小）。
- + 20 GiB 余量（工作空间、页面文件增长、解压临时占用）。
- 合计 ≈ **68 GiB，取整要求 NVMe 剩余 ≥ 70 GiB** 才允许开始下载；校验用 PowerShell `Get-PSDrive C | Select-Object Free` 或资源管理器属性（1 GiB≈1.074 GB）。
- 若含 ref2va（F2 备选）再 +19.53 GiB → 要求 ≥ 90 GiB。

| 停止项 | 阈值 | 依据 | 等级 |
|---|---|---|---|
| 蓝屏 / 整机重启 | 任意一次 | zenn 8 次 IOMMU 先例（sources/22） | B |
| commit 峰值 | **> 30 GB**（或 > 95% 物理 RAM 且 pagefile 活跃增长） | R2-11 27.2GB 已紧张；32GB 边界；阈值标 D 待校准 | D（阈值） |
| shared GPU memory 峰值 | **> 2 GB** → 判已进入 PCIe 慢路径，本配置“不通过” | R2-13 带宽 408→12GB/s；LOG-002 §11 建议 | B（机制）/ D（2GB 阈值） |
| 磁盘剩余 | 运行期 **< 20 GB**；下载前 **< 70 GiB**（见上方计算） | 下载 ~40GiB + venv 备份 ~8GiB + 20GiB 余量 | C |
| 需要改驱动 | 任何情况（含需更新/回滚驱动） | 规格第七节禁止擅改 | D（流程） |
| 连续 1 次 OOM 且降分辨率后仍 OOM | — | R-07/R-01 | B |
| 任意指标无法取得且 3 次均缺失 | — | 数据完整性（§6.7） | D |

> 上报内容：机器状态、触发哪一项、当时分辨率/步数、双显存/RAM/commit 实测值、日志片段。不得自行改驱动/系统配置/删除文件。

---

## §8 回滚

- **只允许重命名或复制，不允许删除**（与 DOSSIER §7 约束一致）。
- ComfyUI 回滚：`git -C $ComfyRoot checkout <备份前 commit>` 或 `git checkout v0.32.0`（需 Junlong 批准，因属 git 改动）。
- venv 回滚：失败环境 `Rename-Item .venv .venv-failed-<ts>`，解压 `.venv-backup-<ts>.zip`；原目录**保留不删**。
- 模型文件：切换 F1–F5 时旧权重**保留不删**，是否清理由 Junlong 决定。
- 任何“删除”操作（含 `.venv-failed-*`、不完整下载、旧备份）一律需 Junlong 批准，本方案不执行删除。

---

## §9 给技术手册的交接清单（手册必须覆盖）

1. **只读环境确认**：nvidia-smi（GPU 名/驱动/显存）、venv 实际路径、torch/CUDA 版本（须 cu130，否则停并报）、ComfyUI 版本（须 ≥ v0.32.0，否则 checkout）。
2. **下载与校验**：5 个权重（DiT/TE/VideoVAE/AudioVAE/LoRA）的 HF `resolve/main` 直链、目标子目录；下载后用 `Get-FileHash -Algorithm SHA256 <文件>` 计算并与 **§2 SHA256 列（HF API tree 的 lfs.oid，见 P1-hf-api.md、P1b-*）** 比对；同时 `curl -sIL` 取 `X-Linked-Size` 与本地字节数比对（§2 字节数表）。
3. **启动参数**：§2.1 五旗逐条写入启动脚本；说明 Sysmem Fallback 需 Junlong 批准。
4. **模板导入**：Templates → Video → MiniMax H3（v0.32.0 内置 0.11.39 版：无 LoRA、Video VAE=fp16，用 `MiniMaxH3ImageToVideo` 而非 `EmptyMiniMaxH3LatentAV`）；手动插入 `LoraLoaderModelOnly`（接在 UNETLoader 的 MODEL 输出之后，加载 ema_V4）与 `MiniMaxH3SigmaShift`（接在 LoraLoader 之后，shift_video=12/audio=6），**SigmaShift 的 MODEL 输出须同时接 BasicScheduler.model 与 BasicGuider.model**（否则 shift 不作用 sigma 调度）；断开 ResolutionSelector(115) 到子图的 width/height 连线（link 219/220）后手填 480/832，并把 duration（PrimitiveFloat，默认 2s）改为 5 以得 124 帧（length 由 ComfyMathExpression 算出，非直接填）；BasicScheduler steps=8、sampler res_multistep、scheduler simple、attention SDPA。详见 §3.5。
5. **每轮记录**：耗时、dedicated/shared GPU memory 峰值、进程 RAM 峰值、commit 峰值（§1）。
6. **风险对照**：R-01~R-13 触发信号与缓解；停止条件（§7）阈值。
7. **回滚步骤**：§8（重命名/复制，禁删）。
8. **合规**：许可证地域/商用（§4 R-13）。

---

## §10 来源与链接核实表

| # | 内容 | 状态 | 等级 | 留底 |
|---|---|---|---|---|
| 1 | HF API：Comfy-Org/MiniMax-H3 字节数 | 已打开可读（2026-11-04 抓取） | B | sources/P1-hf-api.md |
| 2 | HF API：Abiray/MiniMax-H3-Turbo-Lora-Pruned-ComfyUI 字节数 | 已打开可读 | B | sources/P1-hf-api.md |
| 3 | HF API：AX1Y2JP/MiniMax-H3-W4A8-ConvRot 字节数 | 已打开可读（模型卡空壳，字节数来自 API） | B | sources/P1-hf-api.md |
| 4 | 模板 JSON `video_minimax_h3_t2v.json` 节点 | 已打开可读（raw.githubusercontent） | B | sources/P1-template.md |
| 5 | ComfyUI releases / tags 日期与 PR | 已打开可读（Releases 页 + API） | B | sources/P1-releases.md |
| 6 | v0.31.1 **tag 存在**（api.github.com/git/refs/tags/v0.31.1 → commit fe4195f7，B），但**无 GitHub release 页**（releases API `/releases/tags/v0.31.1` 返回 404，仅说明未建 release 对象，不说明 tag 不存在）；v0.32.0 tag 同样存在 | 已核实（tag 存在 + release 404） | B（事实） | sources/P1b-tag.md、P1-releases.md |
| 7 | PR #15224 / #15446 / #15486 / #15479 / #15477 首次 release | 已核实（v0.32.0 release body） | B | sources/P1-releases.md |
| 8 | docs.comfy.org 启动参数页 + cli_args.py（--fast-disk/--cache-none 等五参数） | 已打开可读（sources/R2-14、P1b-cliargs） | B（存在，且 v0.31.0 起已存在于 cli_args.py 原文） | sources/R2-14、P1b-cliargs.md |
| 9 | Qiita 3070 实测（480×832/8步/345s/7.3GB、ema_V4 无伪影） | 已打开可读 | B | sources/19 |
| 10 | Qiita 3070 Ti（W4A8 4步 79.74s） | 已打开可读 | B | sources/20 |
| 11 | zenn cosara（蓝屏 8 次/IOMMU、供电 2.76×、15s） | 已打开可读 | B | sources/22 |
| 12 | #15251 / #15263 / #15663 / #15456 / #17122 | 已打开可读 | B | sources/29/28/25/26/R2-01 |
| 13 | RESEARCH-LOG-002（R2-01~R2-14，32GB 内存） | 本目录已存在 | B/C | RESEARCH-LOG-002.md |
| 14 | DMAD（非 ComfyUI 原生） | 已打开可读 | B | sources/R2-07 |

**未核实/无法访问**：Civitai（sources/33，Cloudflare 拦截）；zenn 续报（仅链接未打开）；lightx2v 4步/alpha8（HF 页无参数）。以上均不写入结论。

---

## §11 修订记录

- **r1（P1b，已审核通过的 8 条小修）**：模板版本与节点事实核对（0.11.39 无 LoRA / Video VAE=fp16 / seed / ResolutionSelector 等）、基线配置自洽（Qiita 全链路 ema_V4）、§3.4 分辨率与 token 计算、启动参数五旗版本考据、§9 交接清单、DOSSIER §7 步骤修正等 8 条，指挥 BOT 审核通过。
- **r2-1（§3.5 连线修正，实质错误）**：原“LoraLoader / SigmaShift 只插在 UNETLoader→BasicGuider 之间”错误；修正为 UNETLoader→LoraLoaderModelOnly→MiniMaxH3SigmaShift，且 **SigmaShift 的 MODEL 输出同时接 BasicScheduler.model 与 BasicGuider.model**（sigmas 由 BasicScheduler 按 model_sampling 算出，只接 BasicGuider 时 shift 不作用 sigma 调度）。
- **r2-2（模板节点与 length 来源）**：该 t2v 模板用 `MiniMaxH3ImageToVideo`，**无 `EmptyMiniMaxH3LatentAV`**；width/height 为子图输入由顶层 `ResolutionSelector(115)` 经连线传入，须先断开再手填 480/832；length 由 `ComfyMathExpression` 从 `duration`（PrimitiveFloat，模板默认 2s）算出（17k+5：2s→56 帧、5s→124 帧），须把 **duration 改为 5** 而非直接覆盖 length=124。
- **r2-3（§3.4 ResolutionSelector 说明）**：删“固定短边 768、面积上限 768×1344”无来源描述，改为引用模板自带说明表（节点 118 MarkdownNote“Size Settings Reference”）：0.2MP→608×352、0.3→736×416、0.4→864×480、0.5→960×544（均 16:9 横版）。
- **r2-4（seed 统一）**：§2 与 §3.5 统一为固定 seed **556589502035082**（模板 RandomNoise 默认值）。
- **r2-5（档案步骤，环境受阻·未完成）**：第 10 章 F2 记录末尾的注记内容已拟定，但本环境禁用 Bash（`cp` 被拒绝），`cp DOSSIER.md DOSSIER.v2.md` 无法执行，DOSSIER.v2.md 未生成；故该注记暂未写入 DOSSIER.md（避免 dangling 引用）。**待办**：在允许 Bash 的环境手动执行 `cp DOSSIER.md DOSSIER.v2.md`，再于 DOSSIER.md §10 F2 末尾追加：“注：DOSSIER.v1.md 是 P1 用 Write 分段重建的不完整副本（873 行，末尾有 __SPLIT2__ 标记），不能当备份用；F2 前的原文 = DOSSIER.v2.md 撤销 F2 那一行。”

*（本方案完。全文无 A 级证据、无“本机验证成功”表述；阈值依据不足者均标 D 并注明“待真机校准”；§11 为 P1b/P1c 修订记录。）*
