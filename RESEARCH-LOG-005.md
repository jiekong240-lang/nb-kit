# RESEARCH-LOG-005｜Blackwell 笔记本专项：量化格式选型（NVFP4/FP8/int8_convrot/GGUF）+ Windows 显存溢出策略 + 笔记本功耗墙

> **页眉（目标机 / 阶段）**
> - **目标机**：RTX 5070 Laptop **8GB**（Blackwell，计算能力 **sm_120**）+ **32GB RAM** + **Win11**（本目录研究约束，见 CODEBUDDY.md）。
> - **阶段**：在 LOG-001（社区基线/卸载/VAE）、LOG-002（32GB RAM）、LOG-003（注意力后端）、LOG-004（蒸馏+缓存）之后，专答 DOSSIER 与 LOG-001~004 仍未清楚的 **权重量化格式选型** 与 **Windows 显存溢出 / 功耗墙** 问题。
> - **证据等级约定**：A=本机已复现（**本任务不可能出现 A**）；B=外部可复现（官方文档/已合并 PR/多源一致）；C=单源实测或论文自报；D=只见摘要/无法核实。**每条均标等级与来源链接**。
> - **LOG-001~004 已覆盖条目不重抄**，仅在新数字/新版本时引用编号（如 LOG-003 L3-xx、LOG-004 L4-xx）。
> - **24GB+ / 5090 / RTX PRO 6000 卡的数字一律标明硬件，禁止当作 8GB 数字写入推荐表**（违反即 D 级无效外推）。
> - **访问日期**：本轮所有网页直读于 **2026-11-10 JST** 完成。

---

## 一、量化格式选型（问题 a：NVFP4 / MXFP4 / FP8 / int8_convrot / GGUF）

### L5-01｜NVIDIA 官方 RTX 5070 Laptop 规格 + 第五代 Tensor Core（FP4/FP8 原生能力）
- **来源完整 URL**：https://www.nvidia.com/pt-br/geforce/laptops/compare （NVIDIA 官方 50 系列规格对比页；Blackwell / sm_120 由 comparigon 交叉确认：https://comparigon.com/gpu-comparison/geforce-rtx-5070-laptop-vs-geforce-rtx-5090-laptop 该页明确写 “CUDA: 12.0”）。
- **发布/提交日期**：规格页为 NVIDIA 当前在售页面（2025-01-07 发布 RTX 50 系列）；本轮直读 2026-11-10。
- **原文测试条件**：官方规格表，无第三方实测。
- **报告效果（规格）**：RTX 5070 Laptop GPU：**4608 CUDA cores、8GB GDDR7（另有 12GB 变体）、128-bit、384 GB/s 带宽、Boost 1425–2347 MHz、GPU Subsystem Power (W) 50–100 W、798 AI TOPS、第五代 Tensor Cores、Blackwell 架构**。NVIDIA 50 系列（Blackwell）全系标注 “Tensor Cores Fifth Generation”，FP4/NVFP4 与 FP8 为 Blackwell 新引入的低精度数据类型（NVFP4 原生 fp4 tensor core 路径由 MATLOWAI 卡页与 nunchaku 仓库在 sm_120 上一致确认，见 L5-02/L5-05）。
- **显存/内存**：8GB GDDR7（本机形态）。
- **画质代价**：无（纯硬件规格）。
- **证据等级**：**B**（NVIDIA 官方规格页直读；FP4 能力由卡页/仓库多源交叉确认）。
- **对 5070 Laptop 8GB+32GB 适用性与不确定点**：确认本机是 **Blackwell / sm_120 / 第五代 Tensor Core**，具备 NVFP4 原生硬件路径（H1 的硬件前提成立）。**TGP 50–100W 是官方区间**（第三方 comparigon/techpowerup 记为 50–125W，见 L5-10）；显存仅 8GB，任何 >8GB 权重文件都必须走 LOG-002/003 的 RAM 卸载策略。本页无耗时数字。

### L5-02｜MATLOWAI/minimax-h3-nvfp4 · H3 NVFP4 量化卡（sm120 原生 fp4 TC，回落慢于 int8_convrot）
- **来源完整 URL**：https://huggingface.co/MATLOWAI/minimax-h3-nvfp4 （README 直读）。
- **发布/提交日期**：卡页写明量化修改日期 **2026-08-19**（“post-training weight quantisation … performed 2026-08-19”）。
- **原文测试条件**：**RTX PRO 6000（Blackwell，96GB——非 8GB 卡！）**；stock ComfyUI（per-layer comfy_quant 元数据，无需自定义节点）；与 W4A8 文件同 seed 同 graph 对照。
- **报告效果（耗时，RTX PRO 6000）**：NVFP4 vs W4A8：**0.84× wall @ 29k tokens、0.90× @ 45k**（更快）；12-step turbo：**210 s vs W4A8 239 s（0.88×）**；文件 **12.5 GB**，与 W4A8 同尺寸（均 4.54 bits/param）→ 这是**速度打法而非省显存打法**。
- **显存/内存**：文件 **12.5 GB**（>8GB，8GB 卡必须卸载到 RAM）。
- **画质代价（量化误差，卡页自报）**：相对 bf16 的 **median weight error 9.4% rms**；W4A8 文件 7.3%；**int8_convrot 仅 1.0%**。同 seed 渲染是「干净的姊妹镜头」，场景/台词相同、演绎略不同。
- **关键事实**：「On Blackwell (sm120) the GEMMs run natively on the fp4 tensor cores」；**非 Blackwell 卡 ComfyUI 回落到 dequantised matmuls，比 int8_convrot 还慢**，官方建议改用 int8_convrot 文件。
- **证据等级**：**B**（HF 卡页直读，格式/机制/数字可核对）；但**所有速度数字均在 RTX PRO 6000（96GB）上得出，禁止当作 8GB 数字**（违反即 D）；8GB 侧仅能确认「原生路径存在 + 文件 12.5GB 需卸载」。
- **对 5070 Laptop 8GB+32GB 适用性与不确定点**：**H1 部分核实的关键依据**——Blackwell 上 NVFP4 有原生 fp4 TC 路径且比 W4A8/int8 更快；但**画质代价（9.4%）高于 int8_convrot（1.0%）与 W4A8（7.3%）**，与 H1「画质代价比 int8/FP8 明显」一致。12.5GB 文件 > 8GB，需依赖 32GB RAM 卸载（与 LOG-002 交叉）。**无 8GB 本机耗时，不可外推**。

### L5-03｜comfyui-wiki FastH3 Trim · 8GB 向剪枝 8 步 H3 的格式→GPU 映射与文件体积
- **来源完整 URL**：https://comfyui-wiki.com/zh/news/2026-10-06-fastvideo-fasth3-trim （ComfyUI 官方 wiki 新闻页）。
- **发布/提交日期**：**2026-10-06**。
- **原文测试条件**：ComfyUI **0.36.0+** 自带 FastH3 T2V 模板；采样器 8 步 res_multistep / simple / MiniMaxH3SigmaShift（video 10 / audio 3）；294 个已校准层（NVFP4 激活缩放在 1000 条提示词上校准）。
- **报告效果（格式→GPU 与体积）**：
  - `fastvideo_fasth3_trim_8step_nvfp4.safetensors` **11.9 GB** → NVIDIA Blackwell（RTX 50 系列 / RTX PRO 6000 / DGX Spark）
  - `…_fp8.safetensors` **19.7 GB** → NVIDIA Ada 及更新（RTX 40 系列）
  - `…_int8_convrot.safetensors` **18.9 GB** → 任意较新 NVIDIA GPU（与模板默认值一致）
  - `…_bf16.safetensors` **37.5 GB** → 参考质量 / 转其它格式
- **显存/内存**：**NVFP4 11.9GB、int8_convrot 18.9GB 均 > 8GB** → 8GB 卡必须卸载。
- **画质代价**：未量化（仅称「8 步剪枝 + 校准」）；NVFP4 用激活缩放校准以期控误差。
- **证据等级**：**B**（ComfyUI 官方 wiki 直读：格式/体积/GPU 映射/版本可核对）；无 8GB 耗时数字（D 级外推禁止）。
- **对 5070 Laptop 8GB+32GB 适用性与不确定点**：给出本机最直接可用的 **NVFP4 11.9GB（Blackwell 专用）** 与 **int8_convrot 18.9GB（通用）** 两条路线，二者都 >8GB 需 RAM 卸载。NVFP4 是 Blackwell 原生最快路径（呼应 L5-02），int8_convrot 是通用最低画质代价兜底（呼应 L5-02 的 1.0%）。**无 8GB 实测秒数**。

### L5-04｜civitaiarchives · ComfyUI 原生格式 native-vs-emulated 对照表（「全比 GGUF 快」）
- **来源完整 URL**：https://civitaiarchives.com/seaart/models/fbc18b5f8dcc729efabaa00d07237c60/versions/6983d621317a819fbdf2a9b20656c4ff （ComfyUI-native w4a8/nvfp4/int8/mxfp8 卡页，镜像归档）。
- **发布/提交日期**：页面未标日期（镜像站，本轮直读 2026-11-10）；内容对应 ComfyUI 0.32+ 原生格式。
- **原文测试条件**：声称「Measured on two real machines」——5090/ComfyUI 0.32 与 3090/ComfyUI 0.33.1、0.30；同场景同 seed 同 shape（544×960 / 243 帧 / 2 shots / 10 steps）。
- **报告效果（native/emulated 表）**：
  - **5090 / 0.32**：int8_tensorwise、asym_w4a8_int8、nvfp4、mxfp8、float8_e4m3fn **全部 native**
  - **3090 / 0.33.1**：int8、asym_w4a8 **native**；nvfp4、mxfp8、fp8 **emulated**
  - **3090 / 0.30**：int8 native；w4a8 **absent（无法加载）**
  - 结论：「50-series (Blackwell) → nvfp4 12.53 GB，最快；40-series → w4a8；30-series 需更新 ComfyUI 才能 native w4a8」；「every one of them is faster than the GGUF ladder」；「A 30-series owner who grabs the 12.5 GB file because it fits will get a worse result than one who takes the 21 GB file that runs native」（emulated 比 native 慢）。
- **显存/内存**：nvfp4 12.53GB、int8 20.97GB、mxfp8 21.56GB、fp8 20.96GB（均为扩散模型权重，>8GB）。
- **画质代价**：未量化。
- **证据等级**：**C**（第三方归档镜像、单来源自称双机实测、无独立复现；与 L5-02/L5-03 机制方向一致但数字不可直信）。**不得高于 C**。
- **对 5070 Laptop 8GB+32GB 适用性与不确定点**：机制层面**支持 H1**（Blackwell 上 nvfp4 native 最快；emulated 格式更慢）与 **H2**（原生格式整体快于 GGUF）。但文件均 >8GB 需卸载；**无 8GB 计时**；镜像站可信度低于官方 wiki/HF 卡，仅作 C 级旁证。

### L5-05｜nunchaku (SVDQuant) NVFP4 · Blackwell sm_120 Win 轮子 + 8GB 实测（图像，非 H3 视频）
- **来源完整 URL**：https://github.com/nunchaku-tech/nunchaku （引擎仓库，含 SVDQuant 论文与性能段）+ https://github.com/nunchaku-tech/ComfyUI-nunchaku/issues/742 （RTX 5060 sm_120 8GB Windows 实测）+ SVDQuant 论文 https://arxiv.org/abs/2411.05007。
- **发布/提交日期**：引擎仓库 News：2025-02-20「Support NVFP4 precision on NVIDIA RTX 5090! ~3× speedup over BF16」；Issue #742 为 2026 年 nunchaku 1.1.0 时段；SVDQuant 论文 2024-11-07 公开。
- **原文测试条件**：
  - Issue #742：**RTX 5060（sm_120，8GB）**、Windows 11、CUDA 12.8、PyTorch **2.11 nightly**、nunchaku **1.1.0+torch2.11** 预编译 wheel（win_amd64 存在）；测试 **FLUX 图像**（非 H3 视频）。
  - 引擎仓库性能段：**RTX 5090（桌面）** 与 **RTX 4090 Laptop**。
- **报告效果（耗时）**：
  - Issue #742：RTX 5060 8GB 上「fully supported」；FP4 模型加载 **~1.27 GB VRAM**，首图推理 **~109 s**；「Our Actual Testing Journey」确认可跑 z-image/Sana/Flux。
  - 引擎仓库：SVDQuant 将 12B FLUX.1 模型体积 **3.6×**、显存 **3.5×**；INT4 模型在桌面/笔记本 **RTX 4090 上 3.0× 快于 NF4 W4A16**；**NVFP4 在 RTX 5090 上比 BF16 与 NF4 快 3.1×**；笔记本 4090 靠消除 CPU 卸载达 10.1×。
- **显存/内存**：FP4 量化模型显存大幅下降（FLUX 最低 ~4 GiB，见 v0.1.4 新闻）；5060 实测加载 1.27GB。
- **画质代价**：SVDQuant 论文称在 PixArt-Σ 上视觉质量显著优于其它 W4A4/W4A8 baseline；FLUX 4-bit 质量接近 BF16（社区共识）。
- **证据等级**：**B**（引擎仓库与 arXiv 直读，机制/数字可核对）；但 **5060 8GB 数字是图像生成、单报告 → 对 H3 视频只算 C 级旁证**；RTX 5090/4090 数字为 24GB+/16GB 卡 → 禁入 8GB 推荐。
- **对 5070 Laptop 8GB+32GB 适用性与不确定点**：证明 **NVFP4 Windows 预编译轮子已可用于 8GB Blackwell（5060 sm_120）**，且需 **PyTorch 2.11 nightly + cu128**（Junlong 若走 nunchaku 路线必须此依赖）。但 H3 视频走 **ComfyUI 原生 NVFP4（L5-02/03）** 更贴本机；nunchaku 当前公开视频 DiT（HunyuanVideo/Wan）支持见 ComfyUI-nunchaku News（v1.0+ 主打开 Qwen-Image/FLUX，H3 视频需确认）。**本条目不直接给 H3 8GB 视频秒数**。

### L5-06｜city96/ComfyUI-GGUF · GGUF 走反量化后 bf16 计算（无原生低比特 kernel）
- **来源完整 URL**：https://github.com/city96/ComfyUI-GGUF （README 直读）。
- **发布/提交日期**：仓库初始 2024-08-15；README 最近提交 2024-10-23；loader.py 2026-01-12（本轮直读 2026-11-10，仓库仍活跃）。
- **原文测试条件**：原生 ComfyUI 自定义节点；支持 flux/sd3.5/t5 GGUF；`Unet Loader (GGUF)` 节点；量化粒度可变比特（Q2–Q8）。
- **报告效果（机制）**：README 明确「**transformer/DiT models such as flux seem less affected by quantization**」，故低比特可跑在低端 GPU；这是 **WIP**；权重以 GGUF 存储，**加载时反量化**（dequant）到更高精度计算。仓库无原生 FP4/fp4 tensor core kernel——与 NVFP4 的原生 fp4 GEMM（L5-02）路径不同。
- **显存/内存**：GGUF Q4/Q5 体积小（盘/内存友好），但**计算在 bf16/fp16**，不省计算显存峰值。
- **画质代价**：DiT 对量化相对不敏感（README 定性）；无 H3 具体指标。
- **证据等级**：**B**（仓库 README 直读：机制/支持列表可核对）；**但 city96 列出的预量化模型为 flux/sd3.5，未列 MiniMax H3 → H3 GGUF 是否存在属未核实（D/C）**。
- **对 5070 Laptop 8GB+32GB 适用性与不确定点**：**支持 H2**——GGUF 在 ComfyUI 中是「反量化后 bf16 计算」，体积小但**速度不如原生 FP8/NVFP4**；在 8GB 上的主要价值是**压盘读/内存占用以便卸载**，而非提速。H3 是否有官方/社区 GGUF 量化**本轮未确认**（city96 未列），须自行核实后再用。

### L5-07｜int8_convrot · 8GB 安全兜底原生格式（最低画质代价，但需卸载）
- **来源完整 URL**：https://huggingface.co/MATLOWAI/minimax-h3-nvfp4 （MATLOWAI 卡页：「int8_convrot 1.0% weight error」「non-Blackwell 用 int8_convrot」）+ https://comfyui-wiki.com/zh/news/2026-10-06-fastvideo-fasth3-trim （FastH3 Trim `int8_convrot 18.9GB`、「任意较新 NVIDIA GPU」）。另见 L5-02/L5-03 条目。
- **发布/提交日期**：同上（2026-08-19 / 2026-10-06）。
- **原文测试条件**：ComfyUI 0.32+ 原生 `int8_tensorwise` 算子（L5-04 表：5090/3090 均 native）；FastH3 Trim int8_convrot 18.9GB。
- **报告效果**：int8_convrot 权重误差 **1.0% rms（L5-02，全格式最低）**；在 Blackwell 上**慢于原生 NVFP4**（L5-02：「native nvfp4 beats；emulated 慢于 int8_convrot」即非 Blackwell 时 int8 是原生最快）；通用性强（任意较新 NVIDIA GPU native）。
- **显存/内存**：18.9GB > 8GB → 必须 RAM 卸载。
- **画质代价**：1.0% 权重误差，全量化格式中最低（L5-02）。
- **证据等级**：**B**（卡页/wiki 直读交叉确认）。
- **对 5070 Laptop 8GB+32GB 适用性与不确定点**：本机 **NVFP4 不可用时（如 ComfyUI 过旧 / PyTorch 无 cu128）的兜底原生格式**；画质代价最低（1.0%）但 18.9GB 需 32GB RAM 卸载。与 NVFP4 组合进 V1.1/V1.2 推荐表（见 §六）。

---

## 二、Windows 显存溢出策略（问题 b：Sysmem Fallback）

### L5-08｜NVIDIA CUDA – Sysmem Fallback Policy（官方页无法直读，行为多源一致）
- **来源完整 URL**：https://nvidia.custhelp.com/app/answers/detail/a_id/5490 （NVIDIA 官方支持文；本轮直读 **返回 Oracle 技术故障页「Technical Difficulties / Incident Number 0.e9ba1402…」，无法访问**）。行为由社区转述：runaihome（http://runaihome.com/blog/shared-gpu-memory-slow-local-ai-sysmem-fallback-fix-2026）、freakyjolly（https://freakyjolly.com/cuda-out-of-memory-fix）。
- **发布/提交日期**：官方文无法访问，日期未核实；社区转述 2026 年。
- **原文测试条件**：NVIDIA Control Panel → Manage 3D Settings → CUDA – Sysmem Fallback Policy。
- **报告效果（三档，社区转述官方）**：① **Driver Default**；② **Prefer Sysmem Fallback**（溢出时写入系统 RAM）；③ **Prefer No Sysmem Fallback**（溢出即报错而非静默溢出）。官方文章「calls the restart out explicitly」（策略在 CUDA context 创建时读取，需重启应用）。
- **显存/内存**：开启 Prefer Sysmem Fallback 时，VRAM 溢出部分进入「Shared GPU memory」（系统 RAM，经 PCIe），带宽骤降。
- **画质代价**：无（机制问题）。
- **证据等级**：**D（官方页无法访问，不采信其原文）** + **C（社区多源对三档与「溢出→共享内存→变慢」行为描述一致，runaihome/freakyjolly 均引用 NVIDIA article 5490）**。
- **对 5070 Laptop 8GB+32GB 适用性与不确定点**：本机 8GB 极易触发溢出（见 L5-02/03 文件均 >8GB）。**官方说明页本轮未能直读**，仅能依赖社区转述；结论与 H3 一致：溢出到共享显存会让视频 DiT 采样**慢一个数量级**（见 L5-09）。

### L5-09｜社区/ComfyUI 推荐：对 python.exe 设 Prefer No Sysmem Fallback + 自身卸载
- **来源完整 URL**：https://freakyjolly.com/cuda-out-of-memory-fix （+ https://freakyjolly.com/how-much-vram-for-comfyui-16gb-test/）+ http://runaihome.com/blog/shared-gpu-memory-slow-local-ai-sysmem-fallback-fix-2026 + ComfyUI 文档（grokipedia 提及「disabling the CUDA system memory fallback feature through the NVIDIA Control Panel」）。
- **发布/提交日期**：2026 年（文章标注 2026）；本轮直读 2026-11-10。
- **原文测试条件**：Windows + ComfyUI（python.exe，常位于 `ComfyUI\venv\Scripts\python.exe` 或 portable `python_embeded\python.exe`）；Program Settings 针对该 exe 设置。
- **报告效果**：
  - freakyjolly：「Windows 上 NVIDIA 驱动会把显存溢出到系统 RAM（Task Manager 的 Shared GPU memory），**作业继续跑但可慢 5–20×**」；建议设 Prefer No Sysmem Fallback 以拿到清晰 OOM 而非「等一小时爬行的渲染」。
  - freakyjolly 16GB 实测：「1.64 GB 的差异决定了 16 秒 vs 26 分钟」。
  - runaihome：指 python 类工具（如 ComfyUI）应指向**实际运行解释器** `venv\Scripts\python.exe`，重启后策略才生效；「Prefer」是偏好非硬保底，但实践中溢出分配会快速失败报错。
  - ComfyUI 文档：用户可通过 NVIDIA Control Panel 关闭 CUDA system memory fallback 防静默溢出。
- **显存/内存**：溢出→共享内存→PCIe 瓶颈。
- **画质代价**：无。
- **证据等级**：**C**（多社区来源一致，但均为经验性建议、非官方基准；无 H3 专属 A/B 计时）。
- **对 5070 Laptop 8GB+32GB 适用性与不确定点**：**支持 H3**——8GB 卡溢出风险极高，社区普遍建议对 ComfyUI 的 python.exe 设 **Prefer No Sysmem Fallback**，并**依赖 ComfyUI 自身卸载（LOG-002/003 的 RAM 卸载策略）**而非靠驱动静默溢出。代价是可能更早 OOM 崩溃，故必须配 `--reserve-vram` / 低步数 / 卸载兜底。**本机实验须把 Sysmem Fallback 开/关 A/B 作为必做项（见 §七 实验 2）**。

---

## 三、笔记本功耗墙 / 温控（问题 c：TGP、Dynamic Boost、MUX、电源模式）

### L5-10｜comparedb · RTX 5070 Laptop 50–125W TGP「性能陷阱」（同芯片 >78% 算力差）
- **来源完整 URL**：https://www.comparedb.com/gpu/rtx-5070-laptop-tgp-specs-performance-brackets-1808313-3 （第三方零售清单审计）。
- **发布/提交日期**：页面未标精确日期（本轮直读 2026-11-10）；内容基于零售清单审计。
- **原文测试条件**：审计 NVIDIA 文档与 Micro Center/Newegg/Amazon 零售清单的 TGP 披露。
- **报告效果**：
  - **TGP 区间 50–125W**；FP32 算力从 **13 TFLOPS（低功耗）到 23.2 TFLOPS（满功耗），78% 摆动**；Boost clock 1410→2520 MHz（78% 摆幅）；Pixel Fill 68→121 GPixel/s。
  - 三档：sub-70W（ thin-and-light 税，1440p 需妥协）；80–100W（主流，合理散热下 boost 维持 2000–2200 MHz，**持续 GPU 重负载 10–15 分钟后热节流**）；115W+（逼近 23.2 TFLOPS，但需 2.2kg+ 机身与 280W 适配器、风扇吵）。
  - OEM 常只标「50–125W」区间，GPU-only TGP 难找；130W 适配器是 sub-70W 硬上限。
- **显存/内存**：无。
- **画质代价**：无。
- **证据等级**：**C**（第三方分析、单来源、无本机 H3 计时；与 L5-01 官方 50–100W 区间方向一致，但上限更高）。
- **对 5070 Laptop 8GB+32GB 适用性与不确定点**：**支持 H4**——同「RTX 5070 Laptop」标称，不同机型 TGP 50–125W，算力差可达 **>78%（远超 H4 假设的 30%）**；**长时视频生成（数分钟）会触及 10–15 分钟热节流**。Junlong 须先查清本机 OEM 实际 TGP（看电源适配器瓦数 / 厂商规格表），再评估耗时基线。

### L5-11｜dazhe · ROG 幻14 Air RTX 5070 实测（90W 标称 → 双烤 62W、单烤 85W<90s 后 −15.3% 频率）
- **来源完整 URL**：https://www.dazhe.com/article/36947.html （硬件评测组，恒温 25℃ 连续 30 分钟满载）。
- **发布/提交日期**：2026 年（页面标「2026 年 ROG 幻14 Air 发布后」）；本轮直读 2026-11-10。
- **原文测试条件**：ROG 幻14 Air，整机 1.58kg，标称 RTX 5070（90W TGP）；双烤（CPU+GPU）/单烤 GPU；3DMark Time Spy、UE5.3、赛博朋克 2077 光追全高 60 帧压力；30 分钟连续记录。
- **报告效果**：
  - 双烤：GPU 峰值锁 **78W**，持续 **12 分钟后回落至 62W（较标称 90W 低 31%）**；C 面 69.3℃ 触发智能降频。
  - 单烤：GPU 短暂冲 **85W** 但维持 **<90 秒**，热管饱和后核心频率 **2175→1840 MHz（−15.3%）**；噪音 48.7dB(A)。
  - 「狂暴模式」锁 2025MHz 但 CPU 限 35W，帧生成延迟 +17.6ms，微卡顿。
- **显存/内存**：无。
- **画质代价**：无（游戏帧率/温度，非生成画质）。
- **证据等级**：**C**（单机型实验室测试、单来源；薄机身极端案例，方向具代表性）。
- **对 5070 Laptop 8GB+32GB 适用性与不确定点**：**支持 H4 的「长时降频可观测」**——薄机身 5070 在持续负载下功率/频率显著跌落（−15~31%）。对**数分钟视频生成**意味着后半段明显变慢；Junlong 若用轻薄本须预期此衰减，厚机型（L5-12 的 115W+ 档）更稳。

### L5-12｜laptopjudge · ASUS TUF A16 RTX 5070（140W 目标 → 暖机后 ~115W）+ 持续监测方法论
- **来源完整 URL**：https://laptopjudge.com/asus-tuf-gaming-a16-ryzen-9-rtx-value （+ tsight.io/articles/17719159「持续负载 ≈ 桌面 5060 或更低」）。
- **发布/提交日期**：2026 年；本轮直读 2026-11-10。
- **原文测试条件**：ASUS TUF A16，Ryzen 9 7945HX + RTX 5070，**140W GPU 功率目标**；建议 30 分钟游戏循环 + HWiNFO 记录 GPU power/temp/clock/frame time；并建议 nvidia-smi 查询 `power.draw,clocks.gr,temperature.gpu`。
- **报告效果**：140W 目标「不保证永远 140W」；**暖机机箱内持续 GPU 功率趋向 ~115W**（sustained cooling limit，非驱动故障）；比较应以「sustained」结果而非单次 Time Spy。tsight 结论：理论等效桌面 5060 Ti，但**持续负载等效桌面 5060 或更低**（30 分钟以上渲染/4K 游戏因 TDP 限频下滑）。
- **显存/内存**：无。
- **画质代价**：无。
- **证据等级**：**C**（单来源评测 + 方法论建议；与 L5-10/11 方向一致）。
- **对 5070 Laptop 8GB+32GB 适用性与不确定点**：提供**可操作的长时生成监测方案**（30 分钟循环、HWiNFO/nvidia-smi 记 power/clock/temp、平衡电源模式、垫高后部改善风道）；印证 H4「同配置耗时差 >30%」与「长时降频可观测」。Junlong 本机实验应把 TGP/电源模式作为变量（见 §七 实验 3）。

---

## 四、近 3 个月 arXiv 量化论文（问题 §4(4)，仅写方法要点与自报数字，标 C）

### L5-13｜PulseQuant · 4-bit 视频 DiT PTQ（含 MiniMax H3 评测，2026-09）
- **来源完整 URL**：https://arxiv.org/abs/2609.33384 （v1 2026-09-27，v2 2026-09-29）。
- **发布/提交日期**：**2026-09-27**（v1）；修订 2026-09-29。属近 3 个月窗口。
- **原文测试条件**：4-bit 后训练量化（PTQ）；在 **Wan 模型、Self Forcing、MiniMax H3** 上评估；方法=结合轨迹敏感度与激活几何的离线校准 + 响应子空间修正（保留原始 4-bit 权重表示）。
- **报告效果（论文自报）**：在 Wan / Self Forcing / MiniMax H3 上「improvements in key consistency and dense-reference metrics while remaining competitive on other attributes across model scales and generation paradigms」。无具体 Blackwell/8GB 秒数。
- **显存/内存**：4-bit 权重（省显存，论文定位）。
- **画质代价**：以 key consistency / dense-reference metrics 衡量，称优于基线。
- **证据等级**：**C**（arXiv 自报、无第三方复现、无 8GB 计时；近 3 月）。
- **对 5070 Laptop 8GB+32GB 适用性与不确定点**：证明 **H3 已被纳入 4-bit 视频 DiT 量化研究**，方向支持「NVFP4/4-bit 是 H3 可行路线」；但无本机可跑的实现/秒数，仅作研究线索（呼应 L5-02/03 的 NVFP4 实装）。

### L5-14｜TriTile/Triage · NVFP4 注意力分块（Wan2.2/HunyuanVideo/LTX-2.5 最高 2.32×，2026-10）
- **来源完整 URL**：https://arxiv.org/abs/2610.35706 （v1 2026-10-26，v2 2026-11-03，标题由 Triage 改 TriTile）。
- **发布/提交日期**：**2026-10-26**（v1）；修订 2026-11-03。属近 3 个月窗口。
- **原文测试条件**：训练无关的注意力方法；三层近似——skip（块均值替换）/ skim（**NVFP4**）/ refine（NVFP4 残差校正）；轻量 router 按块统计预测误差；**融合 kernel 在 RTX Blackwell 的 FP4 tensor cores 上单趟执行**；评测 Wan2.2 / HunyuanVideo / LTX-2.5。
- **报告效果（论文自报）**：相对 dense FlashAttention **端到端加速最高 2.32×**（Wan2.2 1.38–1.83×、HunyuanVideo 2.28–2.32×、LTX-2.5 1.84×），且每档 PSNR 高于最高 PSNR 基线 0.3–3.0 dB。
- **显存/内存**：注意力近似（省注意力计算）。
- **画质代价**：PSNR 高于基线（论文称质量不损）。
- **证据等级**：**C**（arXiv 自报、无 8GB 计时、近 3 月）。
- **对 5070 Laptop 8GB+32GB 适用性与不确定点**：显示 **NVFP4 不仅在权重、也在注意力层有 Blackwell 原生加速路径**，与 L5-02 的「fp4 tensor cores」互补；对视频 DiT（Wan/HunyuanVideo/LTX，与 H3 同族）有 2.32× 潜力。无 H3 实测、无 8GB 秒数，仅作技术方向（H1 的 NVFP4 原生路径在注意力侧也成立）。

---

## 五、假设 H1–H4 检验表

| 假设 | 内容 | 检验结论 | 依据 |
|---|---|---|---|
| **H1** | Blackwell 消费卡 NVFP4 有原生 fp4 TC 路径，NVFP4 DiT 比 int8_convrot 快且显存更省；画质代价比 int8/FP8 明显；8GB 数字很少 | **部分核实** | 原生路径：B（L5-02 MATLOWAI「sm120 fp4 tensor cores」、L5-03 FastH3「Blackwell 专用」、L5-04 civitai「5090 nvfp4 native 最快」、L5-05 nunchaku 5060 sm_120 轮子）。比 int8 快：B（L5-02「native nvfp4 快；emulated 慢于 int8_convrot」）。画质代价更高：B（L5-02 9.4% vs int8 1.0% / W4A8 7.3%）。**8GB 数字稀缺：已核实**——仅 nunchaku 5060 8GB（图像，C）与无 H3 8GB 计时。NVFP4 文件 11.9–12.5GB >8GB，省的是「同尺寸下更快」而非「显存更省」（L5-02「same file size as W4A8 → speed play not memory play」）→「显存更省」子命题**被否定/修正**。 |
| **H2** | GGUF Q4/Q5 在 ComfyUI 反量化后 bf16 计算，体积小但速度不如原生 FP8/NVFP4；8GB 上主要价值是压内存/盘读 | **已核实** | 反量化机制：B（L5-06 city96 README「dequant」、WIP、无原生 fp4 kernel）。速度不如原生：B/C（L5-04 civitai「every one faster than GGUF ladder」+ L5-02 原生 nvfp4 比 emulated 快）。内存价值：C（无 H3 GGUF 计时，city96 未列 H3 → H3 GGUF 存在性未核实）。 |
| **H3** | Windows Sysmem Fallback 开启时溢出到共享显存让视频 DiT 采样慢一个数量级；社区建议对 python.exe 设 Prefer No Sysmem Fallback 并依赖自身卸载 | **已核实（社区级）** | 慢 5–20×：C（L5-09 freakyjolly 多源一致；L5-08 官方页无法访问→D，行为靠社区转述）。推荐设 python.exe + 自卸载：C（L5-09 freakyjolly/runaihome/ComfyUI 文档）。无 H3 专属 A/B 计时（本机实验补，见 §七 实验 2）。 |
| **H4** | 5070 Laptop TGP 差异（50–125W）致同配置耗时差 >30%；长时生成温控降频可观测 | **已核实** | TGP 区间：B（L5-01 NVIDIA 官方 50–100W）+ C（L5-10 comparedb 50–125W、13→23.2 TFLOPS=78% 摆动）。>30%：已核实（78% > 30%）。长时降频：C（L5-10「10–15 分钟节流」、L5-11 幻14 −15.3% 频率/−31% 功率、L5-12 TUF 140→115W）。 |

---

## 六、V1.1 / V1.2 量化格式推荐表（仅由 B 或诚实标注 C 拼出；24GB+ 数字禁入）

> 面向 **5070 Laptop 8GB + 32GB RAM + Win11 + ComfyUI 0.36+（含 comfy-kitchen）**。所有权重文件均 >8GB，必须 RAM 卸载（见 LOG-002/003）。**不写入任何 5090/RTX PRO 6000/4090 秒数**。

| # | 格式 | 文件体积 | 原生路径（sm_120） | 画质代价 | 推荐级别 | 依据 | 备注 / 禁止项 |
|---|---|---|---|---|---|---|---|
| 1 | **NVFP4**（FastH3 Trim / MATLOWAI） | 11.9 / 12.5 GB | **B 原生 fp4 TC**（L5-02/03/04） | 9.4% 权重误差（L5-02，高于 int8） | **V1.1 首选（Blackwell 原生最快）** | B（L5-02/03）+ C（L5-04） | 需 PyTorch cu128 + 当前 ComfyUI；检查加载日志「Native ops: nvfp4」，否则在 dequant 回落（L5-02/04）。>8GB 必卸载。 |
| 2 | **int8_convrot** | 18.9 GB | B 原生 int8_tensorwise（L5-04/07） | **1.0% 权重误差（最低）** | **V1.2 兜底（NVFP4 不可用时）** | B（L5-02/03/07） | 通用、画质最佳；慢于原生 NVFP4（L5-02）。>8GB 必卸载。 |
| 3 | **FP8（e4m3）** | 19.7 GB（FastH3 fp8） | Ada+ 原生（L5-03）；Blackwell 亦支持 | 未量化（L5-03） | 备选（若 NVFP4 文件缺失） | B（L5-03） | 体积最大一档；8GB 卡需卸载。 |
| 4 | **GGUF Q4/Q5/Q6/Q8** | 依量化 | **C 反量化后 bf16 计算，无原生 fp4 kernel**（L5-06） | DiT 较不敏感（L5-06 定性） | **仅作内存/盘读兜底，不提速** | B（机制 L5-06）+ C（无 H3 GGUF 计时） | **H2 成立**：速度不如原生 NVFP4/FP8；H3 GGUF 是否存在未核实（D/C），须先确认。 |
| 5 | MXFP4 | — | Blackwell 微缩放 FP4（NVFP4 同族） | — | 与 NVFP4 同路径，未单独发布 H3 | — | 任务所列格式，本机以 NVFP4 代表。 |

> **说明**：NVFP4 的「快」来自 sm_120 原生 fp4 GEMM（L5-02/04/05），**不是省显存**（文件与 W4A8 同尺寸，L5-02）；8GB 卡省的是「靠 32GB RAM 卸载后仍能跑原生快速路径」。24GB+ 卡的 0.84–0.90×、210/239s、3.1× 等数字（L5-02/05）**仅作机制参考，禁入本表耗时栏**。

---

## 七、下一轮本机最小实验 3 项（具体可执行）

> 目标机：RTX 5070 Laptop 8GB（sm_120）+ 32GB RAM + Win11 + ComfyUI 0.36+（comfy-kitchen / PyTorch cu128）。所有实验固定 warm session（首跑丢弃）、同 seed、同 prompt/参考图/分辨率/帧数/采样器。

**实验 1 · NVFP4 原生路径验证 + 8GB 卸载计时（确认 H1 本机形态）**
- **命令/节点**：ComfyUI FastH3 T2V 模板 → UNETLoader 指向 `fastvideo_fasth3_trim_8step_nvfp4.safetensors`（11.9GB）；8 步 res_multistep / simple / SigmaShift video 10 audio 3；分辨率 544×960（或 832×480）、帧数 124。若 NVFP4 文件缺失，用 MATLOWAI 12.5GB 同法。
- **观测指标**：加载日志是否含「Native ops: nvfp4」（确认非 dequant 回落）；`wall_sec`（warm，2 次中位数）、`peak_vram_mib`、`peak_ram_used_gb`、`peak_pagefile_gb`、`board_power_w`、`gpu_util_pct`、`gpu_clock_mhz`、`gpu_temp_c`。
- **停止条件**：两次 warm 差异 <5% 即采样；否则重启 ComfyUI 消 pagefile 压力（LOG-004 L4-05/20）后重测。
- **回滚**：若 OOM/崩 → 开 `--offload-adaln`（LOG-004 L4-19）或降帧至 96；仍崩退回 int8_convrot 18.9GB（V1.2）。

**实验 2 · Sysmem Fallback 开/关 A/B（必做，确认 H3）**
- **命令/节点**：同实验 1 的 NVFP4 8 步配置（制造 8GB 溢出/临界负载）；仅变量 = NVIDIA Control Panel → Manage 3D Settings → Program Settings → `ComfyUI\venv\Scripts\python.exe` 的 **CUDA – Sysmem Fallback Policy**：A=**Prefer No Sysmem Fallback**，B=**Driver Default**；每组同 seed 生成。
- **观测指标**：`wall_sec` 开关对比；Task Manager「Shared GPU memory」是否攀升；是否出现 OOM 崩溃（A 档）或 5–20× 静默变慢（B 档，L5-09）；`peak_ram_used_gb` / `peak_pagefile_gb`。
- **停止条件**：若 B 档出现「显存满 + 共享内存涨 + 每步数十秒」即确认 H3（慢一个数量级）；A 档若直接 OOM，记录崩溃并回退。
- **回滚**：A 档 OOM → 降分辨率/帧数或切 int8_convrot；恢复 Driver Default 仅作对照。**结论倾向：8GB 卡设 Prefer No Sysmem Fallback + 靠 ComfyUI 自身卸载**。

**实验 3 · TGP / 电源模式 / 温控 A/B（确认 H4）**
- **命令/节点**：同实验 1；变量 = Windows 电源模式（平衡 vs 最佳性能）+ 机器是否垫高后部；用 HWiNFO 或 `nvidia-smi --query-gpu=power.draw,clocks.gr,temperature.gpu` 记录。
- **观测指标**：30 分钟连续生成的 `gpu_power_w`、`gpu_clock_mhz`、`gpu_temp_c` 轨迹；前半段 vs 后半段 `wall_sec/step` 衰减百分比；`board_power_w` 是否触达本机 OEM TGP 上限。
- **停止条件**：若后半段时钟跌落 >15%（呼应 L5-11 的 −15.3%）或功率跌落 >20%（呼应 L5-10/12）即确认长时降频；先查清本机 OEM TGP（电源适配器瓦数）。
- **回滚**：降频严重 → 切平衡模式 / 改善风道 / 降帧数；记录为「本机持续 TGP 实测值」供耗时基线。

---

## 八、链接核实表（本轮 2026-11-10 JST 直读）

| 来源 | 直读结果 | 等级 | 处置 |
|---|---|---|---|
| https://www.nvidia.com/pt-br/geforce/laptops/compare | 可访问，规格表直读 | B | L5-01 |
| https://huggingface.co/MATLOWAI/minimax-h3-nvfp4 | 可访问，README 直读 | B | L5-02 |
| https://comfyui-wiki.com/zh/news/2026-10-06-fastvideo-fasth3-trim | 可访问，新闻页直读 | B | L5-03 |
| https://civitaiarchives.com/.../6983d621317a819fbdf2a9b20656c4ff | 可访问（镜像归档） | C | L5-04 |
| https://github.com/nunchaku-tech/nunchaku + /ComfyUI-nunchaku/issues/742 | 仓库与 issue 均可访问 | B/C | L5-05 |
| https://github.com/city96/ComfyUI-GGUF | 可访问，README 直读 | B | L5-06 |
| https://nvidia.custhelp.com/app/answers/detail/a_id/5490 | **无法访问（Oracle 技术故障页）** | D | L5-08（行为靠社区转述） |
| https://freakyjolly.com/cuda-out-of-memory-fix + /how-much-vram-for-comfyui-16gb-test/ | 可访问 | C | L5-09 |
| http://runaihome.com/blog/shared-gpu-memory-slow-local-ai-sysmem-fallback-fix-2026 | 可访问 | C | L5-09 |
| https://www.comparedb.com/gpu/rtx-5070-laptop-tgp-specs-performance-brackets-1808313-3 | 可访问 | C | L5-10 |
| https://www.dazhe.com/article/36947.html | 可访问 | C | L5-11 |
| https://laptopjudge.com/asus-tuf-gaming-a16-ryzen-9-rtx-value + tsight.io/articles/17719159 | 可访问 | C | L5-12 |
| https://arxiv.org/abs/2609.33384 (PulseQuant) | 可访问（arXiv abs） | C | L5-13 |
| https://arxiv.org/abs/2610.35706 (TriTile) | 可访问（arXiv abs） | C | L5-14 |

> **未直读/仅摘要**：NVIDIA article 5490 官方页（D，已说明）；city96 未列 H3 GGUF（H3 GGUF 存在性 D/C，待核实）；nunchaku H3 视频原生支持需查 ComfyUI-nunchaku 版本说明（C，未深挖）。

---

## 九、事实 / 推测 / 建议 汇总

- **事实（B/C 已标）**：5070 Laptop = Blackwell / sm_120 / 第五代 Tensor Core / 8GB / TGP 50–100W（官方，L5-01）；NVFP4 在 sm_120 有原生 fp4 TC 路径，比 W4A8/int8 快但画质代价 9.4% > int8 1.0%（L5-02）；FastH3 Trim 给出 NVFP4 11.9GB / int8 18.9GB 的 Blackwell/通用映射（L5-03）；GGUF 在 ComfyUI 反量化后 bf16 计算、无原生 fp4 kernel（L5-06）；Sysmem Fallback 溢出到共享内存可慢 5–20×（L5-09，社区一致）；5070 Laptop TGP 50–125W、同芯片算力差 78%、长时 10–15 分钟热节流（L5-10/11/12）。
- **推测（C，未核实，不写结论）**：把 RTX PRO 6000 / 5090 / 4090 的 NVFP4 秒数搬到此 8GB 卡；H3 已有官方/社区 GGUF 量化；nunchaku 直接支持 H3 视频 NVFP4；本机具体 NVFP4 8GB 计时与卸载开销。
- **建议（非结论）**：V1.1 首选 NVFP4（FastH3 Trim 11.9GB）走原生 fp4 路径 + 32GB RAM 卸载；NVFP4 不可用时退 int8_convrot 18.9GB；GGUF 仅作内存兜底不提速；对 ComfyUI 的 python.exe 设 **Prefer No Sysmem Fallback** 并依赖自身卸载（不靠驱动静默溢出）；先查清本机 OEM TGP 再估耗时；所有秒数须本机 warm session 复测，禁引他档硬件。

---

*本日志由 R5（2026-11-10 JST）编写：亲开 14 个来源（含 NVIDIA 规格页、MATLOWAI/HF 卡、comfyui-wiki、civitaiarchives、nunchaku 仓库+issue、city96、Sysmem Fallback 社区文、comparedb/dazhe/laptopjudge 功耗文、PulseQuant/TriTile arXiv），无 A 级（本任务无法本机复现）。NVIDIA article 5490 官方页未能直读，相关行为以社区多源转述并标 D/C。所有 24GB+ 卡数字未写入 8GB 推荐表。*

---

## 修订记录 r1（R6 审核，2026-11-11 JST）

> R6 轮（2026-11-11）回看 LOG-005，按「汇总结论不得高于最弱依据」「来源须直读标注」原则做以下 5 项修订。**不删原文**，仅在下方追加。本轮新直读来源见 sources/R6-03.md（NVIDIA 5490）、sources/R6-04.md（NVIDIA en-us 规格页）；comparedb/dazhe/laptopjudge 已在 R5-08/09/10，本轮补到精确日期。

### r1.1｜L5-01 TGP 冲突：官方 en-us 直读 = 50–100W，仍与第三方 50–125W 冲突（冲突未解决）
- 本轮直读 NVIDIA 英文官方对比页 https://www.nvidia.com/en-us/geforce/laptops/compare/ （访问 2026-11-11）：RTX 5070 Laptop **GPU Subsystem Power (W) 50 - 100 W**、Boost Clock 1425–2347 MHz、CUDA Cores 4608、AI TOPS 798、Standard Memory Configuration 列显示为 **12 GB GDDR7**（注：官方对比表把 5070 Laptop 列为 12GB；本任务目标机是 8GB 变体，属 OEM 定制，官方表未单列 8GB 行）。页面脚注仅关于 PCIe/HDMI/DisplayPort，**未找到 Dynamic Boost 额外功率的脚注说明**。
- 第三方 comparedb（L5-10）仍写 50–125W（FP32 13→23.2 TFLOPS）。**官方 50–100W vs 第三方 50–125W 仍冲突，按规则不自行取舍，记为「冲突未解决」**。

### r1.2｜L5-08 NVIDIA Sysmem Fallback 官方文（article 5490）本轮可直读 → 升 B
- 本轮直读成功（此前 R5 轮返回 Oracle 故障页）：https://nvidia.custhelp.com/app/answers/detail/a_id/5490/~/system-memory-fallback-for-stable-diffusion ，**Updated 10/31/2023**。
- 原文要点（逐字）：「In driver 536.40, we implemented a new method to allow an application to use shared memory in cases that exhausted the GPU memory... albeit at lower speeds... Stable Diffusion happens to require close to 6 GB of GPU memory often... reducing the application speed. In driver 546.01 and above we added a setting to disable the shared memory fallback」；步骤：Program Settings → python.exe → **CUDA - Sysmem Fallback Policy → Prefer No Sysmem Fallback**；「Stable Diffusion must be restarted following these steps」。
- 处置：L5-08 证据等级由「D + C」升级为 **B（官方页直读）**；行为（溢出→共享内存→变慢、546.01+ 可关闭、针对 python.exe、需重启生效）现在有官方原文背书。

### r1.3｜L5-09 来源标注错误：「grokipedia」不是 ComfyUI 文档 → 改标官方 NVIDIA（B）
- 原文把「对 python.exe 设 Prefer No Sysmem Fallback」归到「ComfyUI 文档（grokipedia 提及）」，这是**错误标注**：grokipedia 是第三方百科，不是 ComfyUI 官方文档，不得标 B。
- 该建议的真实出处即上方 r1.2 的 **NVIDIA 官方 article 5490（B）**（其步骤本就是针对 python.exe 设 Prefer No Sysmem Fallback，对 ComfyUI 的 python_embeded/venv 同样适用）。L5-09 中「ComfyUI 文档（grokipedia…）」一句改标为 **「NVIDIA 官方支持文 5490（B，直读）」**；freakyjolly/runaihome 仍维持 C（社区实测）。L5-09 整体等级维持 C（社区多源一致），但官方机制来源升为 B。

### r1.4｜L5-10 / L5-11 / L5-12 日期含糊 → 补精确日期
- **L5-10 comparedb**：本轮直读补到 **「By comparedb · Jun 3, 2026 · Updated Jul 30, 2026」**（此前仅写「2026 年」）。
- **L5-11 dazhe**：本轮直读补到 **发布时间 2026-04-20 20:18:02**（此前仅写「2026 年」）。
- **L5-12 laptopjudge**：本轮直读补到 **「Updated on September 29, 2026」**（发布日期页未给，仅能确认更新日 2026-09-29，记为「更新 2026-09-29，首发日无法确认」）。

### r1.5｜H3 / H4 结论等级过高 → 降为「部分核实（C 级，无生成耗时实测）」
按「汇总结论不得高于最弱依据」：
- **H3（Sysmem Fallback）**：机制与「对 python.exe 设 Prefer No Sysmem Fallback」现已有官方 5490 背书（B，见 r1.2/r1.3），但「溢出让采样慢 **5–20×**」的乘数仍是社区自报（freakyjolly，C），且**仍无 H3 视频在 8GB 卡上的专属 A/B 计时**。修订：H3 结论由「已核实（社区级）」改为 **「部分核实：机制与设置建议已升 B（官方 5490 直读），但『慢 5–20×』乘数为社区 C，且缺 H3 视频 8GB 专属生成耗时实测」**。本机实验补计时见 LOG-006 §七 实验 2。
- **H4（TGP 致耗时差 >30%）**：TGP 区间 50–100/125W、算力 78% 摆幅来自规格/第三方分析（B/C），但「**同配置生成耗时差 >30%**」是由规格 TFLOPS 摆幅**外推**，并非 H3 视频实际生成耗时实测。修订：H4 结论由「已核实」改为 **「部分核实：TGP/算力摆幅为 B/C（规格+第三方），但『生成耗时差 >30%』是规格外推、无 H3 视频本机/社区计时证据；缺本机 OEM TGP 实测与 H3 生成耗时对照」**。

*R6 审核修订完成（2026-11-11 JST）。L5-08 升 B、L5-09 官方来源更正、L5-01 TGP 冲突未解决、L5-10/11/12 补精确日期、H3/H4 降为部分核实。*
