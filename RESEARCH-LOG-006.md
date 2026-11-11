# RESEARCH-LOG-006｜8GB Windows 显存管理内核（异步卸载/pinned memory/权重流式/block swap）+ torch.compile·triton-windows 在 sm_120 的可用性 + H3 GGUF 存在性

> **页眉（目标机 / 阶段）**
> - **目标机**：RTX 5070 Laptop **8GB**（Blackwell，计算能力 **sm_120**）+ **32GB RAM** + **Win11**（见 CODEBUDDY.md）。
> - **阶段**：在 LOG-001~005 之后，专答它们未覆盖的 **ComfyUI 显存管理内核（异步卸载/pinned memory/权重流式/--reserve-vram/--lowvram/--novram/dynamic VRAM）**、**torch.compile·triton-windows 在 sm_120 + Windows 可用性**、**H3 GGUF 是否存在**。
> - **证据等级**：A=本机已复现（**本任务不可能出现 A**）；B=外部可复现（官方文档/已合并 PR/多源一致）；C=单源实测或论文自报；D=只见摘要/无法核实。每条标等级与链接。
> - **LOG-001~005 已覆盖不重抄**，新数字才引用（如 LOG-005 L5-03 FastH3 体积、L5-02 MATLOWAI）。
> - **24GB+ / 5090 / RTX PRO 6000 数字一律标明，禁入 8GB 推荐表**。
> - **访问日期**：本轮网页直读于 **2026-11-11 JST** 完成。

---

## 一、ComfyUI 显存管理内核（问题 a）

### L6-01｜ComfyUI 官方启动参数文档 · 异步卸载与动态 VRAM 默认开启、pinned memory / async / fast-disk 开关
- **来源完整 URL**：https://docs.comfy.org/development/comfyui-server/startup-flags （ComfyUI 官方文档，基于 `comfy/cli_args.py`；本轮直读 2026-11-11）
- **发布/提交日期**：文档随 ComfyUI 版本更新；本轮直读 2026-11-11。
- **原文测试条件**：官方对 `comfy/cli_args.py` 全部启动参数的说明。
- **报告效果（逐字要点）**：「Async weight offloading is on by default on Nvidia and AMD GPUs, and dynamic VRAM is enabled automatically where it is supported.」`--async-offload [NUM_STREAMS]`：**enabled on Nvidia and AMD**，Optional stream count（**default: 2**）；`--disable-async-offload`：Disable async weight offloading。`--disable-pinned-memory`：Disable pinned memory use。`--fast-disk`：Force disk-backed dynamic loading and offload over unpinned RAM；「When neither this flag nor --disable-fast-disk is set, disk-backed loading and offload are enabled automatically if every model file is on storage detected as fast.」`--disable-smart-memory`：Aggressively offload to RAM。`--disable-dynamic-vram` / `--enable-dynamic-vram`（auto on Nvidia, ROCm 7.14+）。
- **显存/内存**：异步卸载默认开（Nvidia/AMD）；动态 VRAM 在支持系统自动开；--fast-disk 把「未 pin 的 RAM」换成「NVMe 盘」作卸载层。
- **画质代价**：无（机制/开关）。
- **证据等级**：**B**（官方文档直读，参数名/默认值可核对）。
- **对 5070 Laptop 8GB+32GB 适用性与不确定点**：**本机默认即异步卸载 + 动态 VRAM**，无需手动开；`--async-offload` 默认 2 流。对 8GB 卡跑 >8GB 权重（L5-02/03 的 11.9–18.9GB）这是默认路径。**pinned memory 默认开启**意味着预分配锁页 RAM 作中转——这正是 H1 担心的「吃 RAM 触发 pagefile」点，排查 OOM 时可用 `--disable-pinned-memory`（见 L6-04）。`--fast-disk` 对本机 NVMe 可能自动启用，把 RAM 压力转移到盘。

### L6-02｜ComfyUI 官方 · --reserve-vram / --vram-headroom / --lowvram / --novram / --disable-smart-memory 行为（含 Windows pagefile 提示）
- **来源完整 URL**：同 L6-01（https://docs.comfy.org/development/comfyui-server/startup-flags）
- **发布/提交日期**：同 L6-01。
- **原文测试条件**：同上。
- **报告效果（逐字/要点）**：`--reserve-vram GB`：VRAM in GB to reserve for the OS and other software。`--vram-headroom GB`（default 0）：Extra VRAM for dynamic VRAM to keep free above default headroom；ComfyUI tries to keep this much VRAM completely free, even counting VRAM used by other apps。`--lowvram`：**No effect when dynamic VRAM is enabled. Otherwise, runs text encoders on CPU.** `--novram`：Minimal VRAM usage when --lowvram is not enough。`--highvram`：Keep models in GPU memory。`--disable-smart-memory`：Aggressively offload to RAM instead of keeping models in VRAM（useful when several model families share one card）。
- **显存/内存**：`--reserve-vram` 直接决定留给 Windows 桌面/浏览器的显存；`--vram-headroom` 给动态 VRAM 额外余量。
- **画质代价**：无。
- **证据等级**：**B**（官方文档直读）。
- **对 5070 Laptop 8GB+32GB 适用性与不确定点**：Windows 笔记本常开浏览器/桌面占用 GPU，**`--reserve-vram 1.0~2.0` 几乎必开**（社区 freakyjolly 也强调，见 L6-05）。关键点：**`--lowvram` 在动态 VRAM 默认开启时无效**（B 明确），不要指望它省显存——若要强制旧式低显存模式须先 `--normalvram` 关动态 VRAM。`--disable-smart-memory` 在多模型共享一卡时更有用，但可能更慢。本机 8GB 首要靠动态 VRAM + 异步卸载 + 权重流式，而非 --lowvram。

### L6-03｜comfy-aimdo（AI Model Dynamic Offloader）· 异步权重流式卸载 PyPI 包（host_buffer/vram_buffer + 多 CUDA stream）
- **来源完整 URL**：社区博文引述 https://alishangtian.com/article/user-9065771665 （述 comfy-aimdo v0.5.6）；PyPI 包名 `comfy-aimdo`（AI Model Dynamic Offloader）。
- **发布/提交日期**：博文述「2026 年 ComfyUI 做的一次最重要的架构手术」；comfy-aimdo 版本 0.5.6。本轮直读 2026-11-11。
- **原文测试条件**：ComfyUI v0.40 之后的异步权重流式卸载架构；抽成独立 PyPI 包 comfy-aimdo。
- **报告效果（博文引述）**：提供 **host_buffer 与 vram_buffer 两个缓冲池**，配合多 CUDA stream（源码 `NUM_STREAMS` 控制流数量）；一条流消费第 N 层权重时另一条流已在把第 N+1 层从主机内存搬入显存、把冷权重搬回，计算与传输重叠；日志提示「**Using async weight offloading with N streams**」。README 说法：**4GB VRAM + 8GB RAM 即可相对快速地跑最大的开源模型且打满 GPU 算力**。另：v0.42.0 有「CPU device doesn't support fp8, nvfp4, mxfp8 or int8」提交——量化格式与搬运路径深度耦合。
- **显存/内存**：声称 4GB VRAM + 8GB RAM 可跑大模型（**注意：这是 vendor 宣称，未在本机验证；且 8GB RAM 比本机 32GB 还小，本机余量更大**）。
- **画质代价**：无（搬运机制，不量化）。
- **证据等级**：**C**（第三方博文引述 PyPI 包，未直读源码/基准脚本；机制方向与 L6-01 官方「async offload 默认开」一致，但 4GB+8GB 跑最大模型为厂商宣称，标 C）。
- **对 5070 Laptop 8GB+32GB 适用性与不确定点**：印证 H1 的「异步权重流式」机制（与 L6-01 官方默认一致）；多 stream 重叠搬运正是让小显存卡「流式」跑大模型的核心。但 **>20% 提速是图像生成场景的社区报告（3070/4070ti 10-30%），无 H3 视频 8GB 实测** → H1 提速子命题仍缺本机证据。comfy-aimdo 是底层实现，普通用户用官方 `--async-offload` 即可。

### L6-04｜instasd · pinned memory OOM 案例 + --fast-disk 磁盘层级 + 「RAM 才是决定性规格」
- **来源完整 URL**：https://www.instasd.com/post/comfyui-vram-offloading-guide （社区实测汇总；本轮直读 2026-11-11）
- **发布/提交日期**：页面标注 2026 年（具体日未标）；本轮直读 2026-11-11。
- **原文测试条件**：汇总 Reddit/社区多机型 H3 与各模型 OOM 报告；含 5060 Ti（128GB RAM）、V100、4070 等。
- **报告效果（要点）**：
  - pinned memory 与意外 OOM：ComfyUI 用 `cudaHostRegister` 把等待权重放在锁页 RAM，并维持已 pin 总量；**「A user with a 5060 Ti and 128 GB of RAM... was hitting out-of-memory errors on the default H3 template. The fix was... --disable-pinned-memory. In their words: 'It was that stupid pinned memory. I disabled it and the OOM's stopped.'」** → 当前排查模板 OOM 的顺序：**先试 `--disable-pinned-memory`**。
  - `--fast-disk`：described as "Prefer disk-backed dynamic loading and offload over unpinned RAM. Can be faster for users with fast NVME disks." 层级：VRAM → pinned RAM → 普通 RAM → NVMe，按 tensor 管理。
  - **「System RAM is the spec that actually decides」**：12GB/16GB 成功案例几乎都同时说 64GB RAM；OOM 报告集中在 **32GB**；V100 用户测 5 秒 0.2MP H3 用了 28GB VRAM + 54GB 系统 RAM。
  - 软失败模式：跑完但极慢——4070 在 608x352 用 167 秒，864x480 用 11 分钟（4× 慢，因模型 spill、每步付传输成本）。
- **显存/内存**：系统 RAM 是瓶颈；32GB 是 OOM 聚集点，64GB 体验好；pinned memory 会额外占 RAM。
- **画质代价**：无。
- **证据等级**：**C**（社区多源汇总，无独立复现；与 L6-01 官方 `--disable-pinned-memory`/`--fast-disk` 存在性一致）。
- **对 5070 Laptop 8GB+32GB 适用性与不确定点**：**直接支持 H1 的「pinned memory 吃 RAM，32GB 跑 12–19GB 权重可能触发 pagefile/OOM」**——本机恰是 32GB，跑 int8_convrot 18.9GB（L5-03/07）或 NVFP4 11.9GB（L5-03）这类 >8GB 权重时，pinned memory 预占锁页 RAM 可能把 32GB 推到 OOM/pagefile 边缘；**本机排查首选项 = `--disable-pinned-memory`**（与 L6-01 官方 flag 对应）。`--fast-disk` 对本机 NVMe 值得一试（转移 RAM 压力到盘）。**关键结论：32GB RAM 是本机跑 H3 的硬约束，加 RAM 比任何 flag 更有效（社区共识）**。

### L6-05｜freakyjolly · MiniMax H3 在 16GB 卡上的 OOM 修复（PR #15446 / #15456）+ 启动参数表
- **来源完整 URL**：https://freakyjolly.com/minimax-h3-out-of-memory-comfyui-16gb-fix （社区 H3 实测；本轮直读 2026-11-11）
- **发布/提交日期**：页面 2026 年（具体日未标）；本轮直读 2026-11-11。对象为 H3（MiniMax-H3）。
- **原文测试条件**：**RTX 16GB 卡**（非 8GB，但 H3 同模型族，可借机制）；模型 `minimax_h3_fl2va_pruned_int8_convrot`（T2V/I2V）、`minimax_h3_ref2va_pruned-w4a8_convrot_pruned`（R2V）；Native int8 kernels；Loader = UNETLoader；Megapixels 0.3；Duration 4–6s（124 帧约 5s）；Steps 6 + turbo LoRA。
- **报告效果（要点）**：
  - **PR #15446**：使 H3 decode 显存不再随 clip 长度增长；**PR #15456**：在 tiled-decode 重试前释放其它模型显存（此前重试时 ~14GB 流式权重仍驻留导致再失败；旧版可在采样与 decode 间调 `unload_all_models()` 解决）。
  - 启动参数表（均真实存在于 `cli_args.py`）：`--reserve-vram 1.0`（Windows 开浏览器必开）、`--cache-none`、`--disable-pinned-memory`、`--disable-smart-memory`、`--disable-async-offload`（更稳但更慢）、`--lowvram`/`--novram`（旧式）。
  - **RAM 建议 64GB**；其 worker 单任务峰值约 27GB RAM；**pagefile 重要**：VRAM 溢出时 Windows  spill 到 shared GPU memory（即系统 RAM），曾出现 120–145% commit 致加载失败；建议 pagefile 系统管理或固定大值。
  - 低显存节点：`MiniMaxLowVRAMAttention` + `MiniMaxChunkFeedForward(4096)`。
- **显存/内存**：16GB 卡 + 27GB RAM 峰值；H3 权重经流式卸载；pagefile 在溢出时顶上。
- **画质代价**：turbo LoRA + 6 步 + int8_convrot 量化（画质代价见 LOG-005 L5-02/L5-07）。
- **证据等级**：**C**（单社区作者 H3 实测，16GB 卡；参数名与 L6-01/02 官方一致，但 16GB 数字禁入 8GB 推荐表；机制可迁移到 8GB）。
- **对 5070 Laptop 8GB+32GB 适用性与不确定点**：**这是少数直接针对 H3 的显存管理实测**，PR #15446/#15456 是 H3 解码显存的关键修复（本机须用含这些 PR 的新版 ComfyUI）。启动参数表与 L6-01/02 官方一一对应，确认 `--reserve-vram`/`--disable-pinned-memory`/`--disable-smart-memory`/`--cache-none` 在 H3 上的实用价值。**其 64GB RAM 建议印证 L6-04：本机仅 32GB 是 H3 跑通的硬约束**；pagefile 建议（系统管理 NVMe）对本机直接适用。

### L6-06｜kijai ComfyUI-WanVideoWrapper · Block Swap 节点（blocks_to_swap / prefetch_blocks / use_non_blocking）与 8GB 推荐
- **来源完整 URL**：https://deepwiki.com/kijai/ComfyUI-WanVideoWrapper/14-performance-optimization-guide （DeepWiki 基于仓库源码；本轮直读 2026-11-11）+ https://comfy.icu/node/WanVideoBlockSwap （节点说明）
- **发布/提交日期**：基于 kijai/ComfyUI-WanVideoWrapper 仓库；本轮直读 2026-11-11。
- **原文测试条件**：WanVideoWrapper 性能优化；Block Swap 节点参数：blocks_to_swap（INT 0-48，14B 共 40 块、1.3B/5B 30 块、LongCat 48）、vace_blocks_to_swap（0-15）、prefetch_blocks（0-40，1-2 通常抵消速度损失）、use_non_blocking（BOOLEAN，更快但更占 RAM）、offload_img_emb / offload_txt_emb（省 1-2GB VRAM）、block_swap_debug。
- **报告效果（要点）**：
  - Block Swap 机制：只让工作子集驻留 VRAM，其余在 CPU RAM，forward 时按需搬回；更多 swap = 更少 VRAM、更慢。
  - **按 VRAM 推荐**：24GB+（blocks_to_swap 0-10, prefetch 0-1, non_blocking false）；16GB（15-20, 1-2, true）；**12GB（25-30, 1-2, true）；8GB（35-40, 0-1, true）**。
  - **对比**：Block Swap vs Model Offloading——Block Swap「Faster (selective offload)」，Model Offloading「Slower (more aggressive)」；Block Swap 粒度 per-block，Model Offloading 按参数百分比。
- **显存/内存**：8GB 档 blocks_to_swap 35-40（14B 模型）；prefetch 0-1（省 RAM）；use_non_blocking 占更多 RAM。
- **画质代价**：无（仅搬运，不量化）。
- **证据等级**：**B/C**（DeepWiki 基于仓库源码 `nodes_model_loading.py` 直读，参数与默认可核对 → 偏 B；8GB 推荐档为文档建议，标 C 子级）。
- **对 5070 Laptop 8GB+32GB 适用性与不确定点**：kijai block swap 是 **WanVideo** 的逐块卸载，与本机 H3 走 **原生 ComfyUI 动态 VRAM + 异步卸载**（L6-01/05）是两套路径。对 H3 而言，**H2 的「原生路径优先、无需 wrapper」更成立**——H3 由 Comfy-Org 官方模板（FastH3，LOG-005 L5-03）与 comfy-kitchen 原生支持，不走 kijai。block swap 的「逐块 + prefetch + non_blocking」思路可类比理解原生异步卸载（L6-03 的 NUM_STREAMS 同理）。**若未来跑 Wan2.x 同类视频 DiT 才需 kijai wrapper**。

---

## 二、torch.compile / triton-windows 在 sm_120 + Windows（问题 b）

### L6-07｜triton-windows 官方 README · Blackwell sm_120 官方支持（Triton≥3.3 / PyTorch≥2.7 / CUDA≥12.8）+ Windows 已知问题
- **来源完整 URL**：https://github.com/woct0rdho/triton-windows （README；本轮直读 2026-11-11）
- **发布/提交日期**：仓库活跃；最新引用 release line **triton-windows 3.6**（对应 PyTorch 2.10）；本轮直读 2026-11-11。
- **原文测试条件**：Triton Windows fork 安装与 GPU 支持矩阵。
- **报告效果（逐字要点）**：
  - 「**RTX 50xx (Blackwell architecture, sm120) This is officially supported by Triton. It only works with Triton >= 3.3, PyTorch >= 2.7, and CUDA >= 12.8.**」
  - 「triton.jit and torch.compile just work」「All unit tests passed」「It's as fast as on WSL on the same machine」「Windows 10 and 11 are supported」。
  - PyTorch↔Triton 映射：2.7→3.3, 2.8→3.4, 2.9→3.5, 2.10→3.6；CUDA 工具链自 triton-windows 3.2 起打包进 wheel。
  - **Windows 已知问题**：「Windows file path length limit (260) causes compilation failure」——`torch.compile` 可能生成超长临时文件名致错，需启用 Windows 长路径支持并重启；vcredist（msvcp140/vcruntime140）版本需 ≥14.42。
- **显存/内存**：未涉及显存；编译产生临时文件。
- **画质代价**：无。
- **证据等级**：**B**（官方仓库 README 直读，支持矩阵/版本要求/已知问题可核对）。
- **对 5070 Laptop 8GB+32GB 适用性与不确定点**：**确认 H3 支持前提：triton-windows 官方支持 sm_120**，但须 **Triton≥3.3 + PyTorch≥2.7 + CUDA≥12.8**（与 LOG-005 L5-05 nunchaku 的 PyTorch 2.11/cu128 一致）。本机若走 torch.compile 加速，须装 `triton-windows<3.7`（embeded Python 用 `python_embeded\python.exe -m pip install`）。**Windows 长路径限制是 torch.compile 的首坑**（B 明列），本机须先开长路径。

### L6-08｜PyTorch sm_120 Windows 兼容性 · PTX JIT fallback 致 20–30% 性能损失 + Triton 编译失败
- **来源完整 URL**：https://github.com/kentstone84/PyTorch-2.10.0a0/blob/main/docs/REDDIT_RESPONSE.md （社区对「PyTorch cu128 已支持 sm_120」说法的核实文档；本轮直读 2026-11-11）
- **发布/提交日期**：文档述 PyTorch 2.7.0 发布于 2025-04-23 首个稳定 sm_120 支持；本轮直读 2026-11-11。
- **原文测试条件**：PyTorch 2.7.0+ 在 Windows 的 Blackwell 支持状况。
- **报告效果（要点）**：
  - PyTorch 2.7.0+ 确实支持 Blackwell，但：**「Windows support - While CUDA 12.8 binaries exist for Windows in PyTorch 2.7, there are ongoing compatibility issues reported by users」**；**「Performance - PTX JIT compilation fallback still occurs in some scenarios, reducing performance by 20-30%」**；**「Triton support - Custom kernel compilation for sm_120 on Windows has reported issues」**。
  - 用户报告：`RuntimeError: CUDA error: no kernel image is available for execution on the device`、DLL 加载错误、Triton 编译失败；GitHub Issue #159207 持续讨论。
- **显存/内存**：PTX fallback 影响性能非显存。
- **画质代价**：无。
- **证据等级**：**C**（社区核实文档，引述用户报告与 issue；非官方定论，但多源一致）。
- **对 5070 Laptop 8GB+32GB 适用性与不确定点**：**支撑 H3 的「与卸载/GGUF 组合常出问题 / 首编译很久」**——Windows 上 sm_120 即便「支持」也可能 PTX fallback 丢 20-30% 性能，且 Triton 编译在 Windows 有坑。本机若开 torch.compile，须预期首次编译慢 + 可能的 fallback 性能损失；**不作为 8GB 首选项，列为实验项**（见 LOG-006 §七 实验 2）。

### L6-09｜ComfyUI · TorchCompileModel 节点 + comfy-kitchen --enable-triton-backend + Windows 兼容性说明
- **来源完整 URL**：https://docs.comfy.org/development/comfyui-server/startup-flags （官方，含 `--enable-triton-backend`/`--disable-triton-backend`）+ ComfyUI 节点 TorchCompileModel（社区节点，未直读官方页，标 C）
- **发布/提交日期**：官方 flag 文档随版本；本轮直读 2026-11-11。
- **原文测试条件**：ComfyUI 启动参数与 comfy-kitchen Triton 后端。
- **报告效果（要点）**：官方 flag：`--enable-triton-backend`（Disabled by default at launch，启用 comfy-kitchen 的 Triton 后端）、`--disable-triton-backend`（强制禁用，覆盖 --enable）。另有 `--disable-comfy-compiler`（Disable the Comfy model compiler, including its CUDA graph subfeature）、`--assert-graph-breaks`、`--disable-cuda-graphs`。TorchCompileModel 是社区常用编译节点（对采样器/模型做 torch.compile）。
- **显存/内存**：torch.compile + CUDA graphs 可能增加显存峰值（社区共识，C）。
- **画质代价**：编译通常不影响画质（数值一致）；但 graph break / fallback 可能。
- **证据等级**：**B**（官方 flag 文档）+ **C**（TorchCompileModel 节点行为社区经验）。
- **对 5070 Laptop 8GB+32GB 适用性与不确定点**：comfy-kitchen 的 Triton 后端默认**关闭**，本机 8GB 显存紧张时开启编译恐增峰值显存、且遇 Windows 长路径/PTX fallback（L6-07/08）问题。**H3 视频 DiT 上未找到 8GB 实测提速数字** → H3「10–30% 提速」缺本机证据。建议本机作为实验项而非默认。

### L6-10｜smeltcore · VoxCPM2 在 RTX 5070 Ti（Blackwell sm_120）上 torch.compile 的 Windows 具体坑
- **来源完整 URL**：https://smeltcore.com/recipes/voxcpm2-on-rtx-5070-ti-30-language-48khz-voice-cloning-in-8-gb-vram （同族 Blackwell 8/16GB 实战；本轮直读 2026-11-11）
- **发布/提交日期**：页面 2026 年（具体日未标）；本轮直读 2026-11-11。
- **原文测试条件**：**RTX 5070 Ti（16GB，Blackwell sm_120）** 跑 VoxCPM2 语音模型，约 8GB VRAM 目标；PyTorch 与 Triton 版本匹配。
- **报告效果（要点）**：
  - Blackwell（sm_120）kernel 可用性缺口：若 torch 安装时缺 sm_120 内核会报「not compatible with the current PyTorch installation」或静默 fallback/崩溃；修复 = 从 CUDA 12.8+ index 重装 torch；**Issue #258 协作者建议升级 CUDA 与 torch**；**PR #250 记录 CUDA 13.0 是原生 sm_120 支持最干净的 toolchain**。
  - **torch.compile / Triton 报错（warm-up）**：`torch._dynamo.exc.Unsupported` 或 Triton import 失败，源于 PyTorch/Triton 版本不匹配；快速修复 = 跳过 torch.compile（`optimize=False`）。**Windows 特有问题**：`optimize=True`（torch.compile + CUDA Graphs）在 Gradio 后台线程跑几代后会命中 thread-local-storage 断言；**Windows Issue #258 报告 optimize=True 在后台线程的 TLS 断言**；最大稳定性建议 optimize=False。PyTorch↔Triton 兼容表：2.7→3.3, 2.8→3.4 等。
- **显存/内存**：目标 ~8GB VRAM。
- **画质代价**：语音模型，未涉视频画质。
- **证据等级**：**C**（单社区 recipe，RTX 5070 Ti 16GB 非视频、非 8GB；与 L6-07/08 同族问题交叉印证，可迁移）。
- **对 5070 Laptop 8GB+32GB 适用性与不确定点**：**直接印证 H3 的「torch.compile 在 Windows Blackwell 组合常出问题 / 首编译很久」**——同族 sm_120 上 `optimize=True`（torch.compile+CUDA Graph）在 Windows 后台线程会 TLS 断言崩溃，且 **CUDA 13.0 才是 sm_120 最干净工具链**（提示本机若走编译，优先 cu130 而非 cu128）。本机 8GB 跑 H3 时若开编译，须预期此类 Windows 专属崩溃，列为实验项并备 `--disable-comfy-compiler` 回滚。

---

## 三、H3 GGUF 是否存在（问题 c）

### L6-11｜Abiray/MiniMax-H3-Pruned-GGUF · 真实存在 HF 仓库（Q3_K_M 8.9GB … Q8_0 21.6GB，2026-08-07 发布）
- **来源完整 URL**：https://huggingface.co/Abiray/MiniMax-H3-Pruned-GGUF （HF 卡页；本轮直读 2026-11-11；另见 https://savrn.com/models/minimax-h3-pruned-gguf 镜像含精确文件大小与 SHA）
- **发布/提交日期**：**First published 2026-08-07**（savrn 镜像给出 revision 76061e7，2026-08-07；last updated 2026-08-07）。
- **原文测试条件**：HF 模型卡，GGUF 量化；Requirement：ComfyUI **v0.30.0+**（原生 MiniMax-H3 架构支持）、安装 **ComfyUI-GGUF** 自定义节点；用 **UnetLoaderGGUF** 加载。
- **报告效果（文件大小，卡页/镜像逐字）**：
  - FL2VA（First-and-last-frame）：**Q3_K_M 8.9 GB、Q4_K_S 11.6 GB、Q4_K_M 11.6 GB、Q5_K_S 14.1 GB、Q5_K_M 14.1 GB、Q6_K 16.7 GB、Q8_0 21.6 GB**。
  - Ref2VA（Omni-reference）：Q3_K_M 8.9 GB、Q4_K_M 11.6 GB、Q5_K_M 14.1 GB、Q6_K 16.7 GB、Q8_0 21.6 GB 等。
  - 说明：**Pruned 架构压缩到 8.9–21.6 GB，把 H3 直接带到消费级硬件**；Q4_K_M 标「Recommended Balance (16 GB GPUs)」、Q3_K_M 标「Low VRAM (~12 GB GPUs)」。
- **显存/内存**：Q3_K_M 8.9GB < 8GB? 否，仍 >8GB（8.9GB 需卸载）；但比原生 int8 18.9GB（L5-03/07）小很多。
- **画质代价**：GGUF Q3–Q8 量化（K-Quant），越低越损；卡页未给 H3 专属画质指标，仅按比特位标档。
- **证据等级**：**B**（HF 官方卡页 + savrn 镜像（含 SHA/大小）直读，仓库真实存在、文件大小可核对）。
- **对 5070 Laptop 8GB+32GB 适用性与不确定点**：**H3 GGUF 真实存在且较完整**（pruned 版覆盖 8.9–21.6GB）。注意：**Q3_K_M 8.9GB 仍 > 本机 8GB，须 RAM 卸载**（与 LOG-002/003 一致）；但其体积显著小于原生 int8 18.9GB，RAM 压力更低。**加载需 ComfyUI v0.30+ 与 ComfyUI-GGUF 节点**（见 L6-13 架构支持 caveat）。

### L6-12｜freakyjolly · H3 GGUF 工作流实测（Q4_K_M 11.4GB UNet + CLIPLoaderGGUF，city96 需加 minimax_h3 架构 / 改用 leejet fork）
- **来源完整 URL**：https://freakyjolly.com/best-minimax-h3-gguf-workflow-comfyui/ （社区 H3 GGUF 工作流；本轮直读 2026-11-11）
- **发布/提交日期**：页面 2026 年（具体日未标）；本轮直读 2026-11-11。
- **原文测试条件**：**Windows 16GB 卡**（非 8GB，但 H3 GGUF 同工作流）；MiniMax H3，含原生立体声道音频。
- **报告效果（要点）**：
  - 文件：UnetLoaderGGUF 加载 `minimax_h3_fl2va_pruned-Q4_K_M.gguf` **11.4 GB**；CLIPLoaderGGUF（typeminimax）加载 `qwen3vl-32B-MiniMax-H3-Q2_K.gguf` **8.49 GB**；VAE 用 Comfy-Org/MiniMax-H3 的 fp16/fp32 safetensors（~5.2GB / ~0.6GB）。
  - **架构支持坑**：「On my install it threw 'Unknown model architecture!' until I added a minimax_h3 entry to the pack's architecture list in loader.py. leejet's model card recommends leejet/ComfyUI-GGUF instead, because the city96 repository appears to no longer be actively maintained.」→ **推荐新手用 leejet/ComfyUI-GGUF fork**。
  - 其它 H3 GGUF 仓库：**unsloth/MiniMax-H3-GGUF**（pruned FL2VA/Ref2VA 从 Q2_K 6.26GiB 到 Q8_0，含两个 Qwen3-VL 编码器）；**Abiray/MiniMax-H3-Pruned-GGUF**（16GB 用 Q4_K_M、24GB 用 Q5_K_M）；**molbal/MiniMax-H3-GGUF**（U16G 混合量化，需自有 ComfyUI-GGUF fork）。
  - 收益：切换 GGUF 后单 ComfyUI 实例在 UNet 侧省约 7.4GB 提交 RAM（11.4GB vs 18.8GB int8）、编码器侧省约 6GB。
- **显存/内存**：GGUF 体积更小 → 提交 RAM 更低（对 32GB 本机直接利好）。
- **画质代价**：Q4_K_M UNet + Q2_K 编码器（低比特，画质代价未给 H3 专属数字）。
- **证据等级**：**C**（单社区作者 H3 GGUF 工作流实测，16GB 卡；仓库真实性由 L6-11/HF 背书）。
- **对 5070 Laptop 8GB+32GB 适用性与不确定点**：**证实 H3 GGUF 已可被社区工作流实际使用**，且 **Q4_K_M 11.4GB + Q2_K 8.49GB 编码器**体积优于原生 int8（L5-03/07 的 18.9GB），对 32GB 本机 RAM 压力更小。**但关键 caveat：city96/ComfyUI-GGUF 未内置 minimax_h3 架构，须手动加 arch 或换 leejet fork**——这削弱「GGUF 开箱即用」程度。本机若试 GGUF，须先解决架构支持（见 L6-13）。

### L6-13｜unsloth/MiniMax-H3-GGUF + city96/ComfyUI-GGUF H3 架构支持状态
- **来源完整 URL**：https://huggingface.co/unsloth/MiniMax-H3-GGUF （HF；本轮直读 2026-11-11）+ https://github.com/city96/ComfyUI-GGUF （README，LOG-005 L5-06 已直读，本轮沿用）
- **发布/提交日期**：unsloth 仓库 2026 年（具体日未标，与 Abiray 同期）；city96 README 见 L5-06（loader.py 2026-01-12）。
- **原文测试条件**：unsloth H3 GGUF 卡页（列 pruned FL2VA/Ref2VA 从 Q2_K 6.26GiB 到 Q8_0，加两个 Qwen3-VL 编码器 Q2_K_M 12.20GiB / Q4_K_M 16.97GiB）；city96 ComfyUI-GGUF 支持 flux/sd3.5/t5 GGUF，**未列 MiniMax H3 架构**（L5-06 已记）。
- **报告效果（要点）**：unsloth 同社区转换（非官方 MiniMax）；city96 README 的 loader 支持列表**不含 H3**——故 H3 GGUF 加载需 leejet fork 或手动加 minimax_h3 架构（L6-12）。
- **显存/内存**：unsloth 量化级同 Abiray 量级（6–22GB）。
- **画质代价**：同 GGUF 量化档。
- **证据等级**：**B**（unsloth HF 卡页直读，仓库真实）+ **B/C**（city96 架构列表不含 H3 为 L5-06 已核实事实）。
- **对 5070 Laptop 8GB+32GB 适用性与不确定点**：结论——**H3 GGUF 在 HF 上确实存在（Abiray、unsloth 等社区仓库，B），但 city96 原包未内置 H3 架构支持，须 leejet fork/手动 arch（C 级操作门槛）**。因此对 H4「无广泛使用 H3 GGUF」须**否定**（见 §五 H4）。GGUF 仍是「反量化后 bf16 计算、无原生 fp4 kernel」（L5-06），速度不如原生 NVFP4/int8（LOG-005 H2），但其**体积优势**对 32GB 本机 RAM 友好——可作备选而非首选。

---

## 四、近 3 个月 arXiv（仅方法要点与自报数字，标 C）

### L6-14｜arXiv 2609.21849「The Weight Is Over — Interactive Diffusion on Consumer GPUs」· 低比特(FP8/NVFP4)+ 多模型管线 GPU 卸载方案
- **来源完整 URL**：https://arxiv.org/html/2609.21849v1 （SIGGRAPH Asia 2026 Technical Communications；本轮直读 2026-11-11）
- **发布/提交日期**：v1，2026-09（arXiv ID 2609.21849）；属近 3 月窗口。
- **原文测试条件**：消费级 GPU 上的交互式图像扩散部署；低比特量化（FP8/NVFP4）+ 多模型（embedder/transformer/decoder/辅助 conditioning）GPU 显存卸载调度。
- **报告效果（论文自报）**：低比特量化有两个可分离效果——**权重量化显著减少内存占用**，**激活量化在计算受限阶段加速**；对权重超 VRAM 的多模型管线，需要**精心设计的 GPU 显存卸载方案**（跨 text encoder、VAE、辅助 conditioning）；结果：在 recent GPUs 上 time-to-first-image-iteration（TTFI）< 1 秒，并在更广硬件上维持 QoS。贡献含 embedding translator（小编码器对齐大编码器以减权重/延迟）、speed/quality/memory 三角的可复现 sweep recipe、亚秒 TTFI 交互式编辑器。
- **显存/内存**：低比特 + 卸载使多模型 pipeline 适配消费级显存。
- **画质代价**：量化 + translator，论文称维持 QoS。
- **证据等级**：**C**（arXiv 自报、无第三方复现、无 H3/8GB 计时；近 3 月）。
- **对 5070 Laptop 8GB+32GB 适用性与不确定点**：**方法层面支持 H1 的「权重流式/卸载 + 低比特」方向**——NVFP4 既是权重压缩也是激活加速（与 LOG-005 L5-02/05 一致）；多模型（H3 含 Qwen3-VL 32B 编码器 + VAE + UNet）卸载调度正是本机 32GB RAM 要解决的问题。无 H3 视频计时，仅作方法线索。

### L6-15｜arXiv 2611.08023「Precision-Preserving Subtensor Inference (PSI)」· 固定 GPU 工作集的 subtensor 权重流式（LTX-2.5 22B，RTX A6000）
- **来源完整 URL**：https://arxiv.org/html/2611.08023v1 （CONSTRAINT 2026 Workshop @ IEEE BigData 2026；本轮直读 2026-11-11）
- **发布/提交日期**：v1，2026-11（arXiv ID 2611.08023）；属近 3 月窗口。
- **原文测试条件**：BF16 LTX-2.5 **22B** 联合音视频 transformer；固定 GPU 工作集（fixed-arena）runtime；跨 9 个 matched full-prompt cases on **RTX A6000**；固定 3 GiB available-memory cap。
- **报告效果（论文自报）**：相对官方 MATH，modified PSI 把 sampled transformer-process **peak memory 从 8,578 MiB 降到 360 MiB**；managed arena 80 MiB；但 11 次 transformer 调用的 mean time 从 **247.11 s 升到 349.00 s**（约 +41%，因流式搬运）；在固定 3 GiB 显存上限下完整生成成功（27/27 解码成功）。FFN reuse 降 mean block time 4.0%。
- **显存/内存**：峰值 360MiB / arena 80MiB under 3GiB cap（A6000，非 8GB 卡，但证明固定工作集流式可行）。
- **画质代价**：BF16 不变（Precision-Preserving），端点误差略增但保持 BF16 输出。
- **证据等级**：**C**（arXiv 自报、单 GPU 实测、无 H3/8GB 计时；近 3 月；与视频 DiT 同族 LTX 相关）。
- **对 5070 Laptop 8GB+32GB 适用性与不确定点**：**证明「subtensor 权重流式 + 固定 GPU 工作集」能在极小显存上限跑 22B 视频 DiT**，但代价是时间 +41%——印证 H1「流式换速度」的权衡方向；对 H3（同族视频 DiT）的 8GB 适配有方法论参考价值。无本机数字。

---

## 五、假设 H1–H4 检验表（LOG-006 定义）

| 假设 | 内容 | 检验结论 | 依据 |
|---|---|---|---|
| **H1** | ComfyUI 近期异步卸载+pinned memory 在 Windows 上让 8GB 卡跑 >8GB 权重比旧同步卸载明显更快（>20%）；但 pinned memory 吃 RAM，32GB 跑 12–19GB 权重可能触发 pagefile | **部分核实** | 机制：异步卸载默认开（B，L6-01）、pinned memory 默认开（B，L6-01）、comfy-aimdo 多 stream 流式（C，L6-03）。提速 >20%：**仅图像生成社区报告（C，L6-03 称 3070/4070ti 10-30%），无 H3 视频 8GB 实测**。pinned memory 吃 RAM 致 OOM/pagefile：**C，L6-04（5060 Ti 128GB 用户 --disable-pinned-memory 解决 OOM）；本机 32GB 跑 12–19GB（L5-03）未测但风险成立**。 |
| **H2** | kijai block swap 与 ComfyUI 原生卸载性能差距已很小；H3 原生 ComfyUI 路径优先，无需 wrapper | **部分核实** | kijai block swap 比整体模型卸载快（B/C，L6-06 deepwiki 对比）；但与 ComfyUI 原生 dynamic VRAM + async offload 的直接速度对比**无数据**；H3 由官方 FastH3 模板 + comfy-kitchen 原生支持（L5-03），走 kijai wrapper 非必需（C，L6-06）。「原生优先」成立，「差距很小」无直接对比。 |
| **H3** | triton-windows 已支持 sm_120；torch.compile 在 Windows Blackwell 上对视频 DiT 有 10–30% 提速，但与卸载/GGUF 组合常出问题或首编译很久 | **部分核实** | sm_120 支持：**B（L6-07 triton-windows README，要求 Triton≥3.3/PyTorch≥2.7/CUDA≥12.8）**。10–30% 提速：**无 H3 视频 8GB 实测，缺证据**。已知问题：**B（L6-07 Windows 长路径致 torch.compile 失败）+ C（L6-08 PTX fallback 20-30% 损失、L6-10 optimize=True TLS 断言）**。 |
| **H4** | 目前无广泛使用 H3 GGUF；8GB 社区主流是 NVFP4 / int8_convrot / W4A8 原生格式 | **被否定（部分）** | 「无 H3 GGUF」：**被否定**——H3 GGUF 已存在且较广泛（Abiray/MiniMax-H3-Pruned-GGUF 2026-08-07，B L6-11；unsloth/MiniMax-H3-GGUF，B L6-13；freakyjolly 工作流，C L6-12）。但 city96 原包未内置 H3 架构，须 leejet fork/手动 arch（C，L6-12/13）。「8GB 主流是原生格式」：**仍可能成立**（GGUF 需额外节点/fork，原生 NVFP4/int8 更直接），维持观察。 |

---

## 六、8GB 本机推荐启动参数组合表（仅由 B 或诚实标注 C 拼出；24GB+ 数字禁入；每参数写来源编号）

> 目标机：RTX 5070 Laptop 8GB（sm_120）+ 32GB RAM + Win11 + 含 H3 修复 PR（#15446/#15456）的新版 ComfyUI（≥ v0.30，见 L6-05/11）。**每次只列本机适用项，不引 16GB+/24GB+ 数字**。

| 参数 / 组合 | 作用 | 来源编号 | 备注 / 禁止项 |
|---|---|---|---|
| `--reserve-vram 1.0~2.0` | 留显存给 Windows 桌面/浏览器 | B（L6-01/02）+ C（L6-05） | Windows 笔记本必开；值随其它 GPU 应用占用调。 |
| 异步卸载保持默认开（勿加 `--disable-async-offload`） | 权重流式重叠搬运，8GB 跑 >8GB 权重的基础 | B（L6-01） | 仅排查问题时才 `--disable-async-offload`（更稳但更慢，C L6-05）。 |
| 动态 VRAM 保持默认开 | 按 tensor 实时调度 VRAM/RAM | B（L6-01） | **`--lowvram` 在动态 VRAM 下无效**（B L6-02），勿指望它省显存。 |
| `--disable-pinned-memory`（当 32GB RAM 吃紧/OOM 时） | 关锁页 RAM 中转，释放 RAM 防 OOM | C（L6-04 instasd 案例）+ B flag 存在（L6-01） | 本机 32GB 跑 12–19GB 权重的 OOM 首选排查项；正常时不必关（会略降传输速度）。 |
| `--fast-disk`（若本机 NVMe 为快盘） | 把未 pin 的 RAM 卸载层转移到 NVMe | C（L6-04）+ B flag 自动检测快盘（L6-01） | 转移 RAM 压力到盘，对 32GB 友好；SATA/外接盘别开。 |
| `--cache-none`（连续多任务间清 RAM） | 不缓存节点输出，省 RAM | B（L6-01/02）+ C（L6-05） | 代价：每次重算，单任务更慢；任务间清 RAM 有用。 |
| `--disable-smart-memory`（多模型共享一卡时） | 激进卸载到 RAM | B（L6-01/02）+ C（L6-05） | 更稳但可能更慢；本机单机单模型一般不首要。 |
| `UnetLoaderGGUF` + leejet/ComfyUI-GGUF（若试 H3 GGUF） | 加载 H3 GGUF（Q4_K_M 11.6GB 等） | B（L6-11/13）+ C（L6-12 架构坑） | **禁入「原生 8GB 首选」**：GGUF 需 fork/手动 arch，且反量化后 bf16 计算、速度不如原生 NVFP4/int8（L5-06/02）；仅作 RAM 压力备选。 |
| 模型权重文件：NVFP4 11.9GB / int8_convrot 18.9GB（原生） | 本机 H3 首选格式 | B（LOG-005 L5-03） | 均 >8GB 须 RAM 卸载（LOG-002/003）；GGUF Q3_K_M 8.9GB 仍 >8GB 亦须卸载（L6-11）。 |

> **说明**：本表无 16GB/24GB/5090/RTX PRO 6000 数字（L6-05 的 16GB 27GB RAM 峰值仅作机制参考，不进耗时栏）。核心约束 = **32GB RAM 是本机跑 H3 的硬上限**（C L6-04/05），加 RAM 比任何 flag 更有效。

---

## 七、下一轮本机最小实验 3 项

> 目标机：RTX 5070 Laptop 8GB（sm_120）+ 32GB RAM + Win11 + 新版 ComfyUI（含 H3 PR #15446/#15456）。固定 warm session（首跑丢弃）、同 seed、同 prompt/分辨率/帧数/采样器。

**实验 1 · 异步卸载 / pinned memory 开 vs 关 A/B（必做，确认 H1）**
- **命令/节点**：FastH3 T2V 模板 → UNETLoader 指向 `fastvideo_fasth3_trim_8step_nvfp4.safetensors`（11.9GB，LOG-005 L5-03）；8 步 res_multistep；544×960、124 帧。A=默认（异步卸载开 + pinned memory 开）；B=加 `--disable-pinned-memory`（其余默认）。
- **观测指标**：`wall_sec`（warm 2 次中位）、`peak_vram_mib`、`peak_ram_used_gb`、`peak_pagefile_gb`、`gpu_util_pct`。
- **停止条件**：若 B 档 OOM/崩 → 确认 pinned memory 是元凶（呼应 L6-04）；若 A/B 速度差 <5% 或 B 更稳，记录结论。
- **回滚**：B 崩 → 退回 A 并降帧至 96；仍崩 → 切 int8_convrot 18.9GB 或 GGUF Q4_K_M 11.6GB（L6-11）。

**实验 2 · torch.compile / triton 开 vs 关（确认 H3 本机形态）**
- **命令/节点**：同实验 1 配置；变量 = comfy-kitchen `--enable-triton-backend` 或在采样链加 TorchCompileModel 节点（A=开，B=关/`--disable-comfy-compiler`）。前置：Windows 启用长路径（L6-07）、装 `triton-windows<3.7`（Triton≥3.3/PyTorch≥2.7/CUDA≥12.8，L6-07/08）。
- **观测指标**：首次编译时长（min）、`wall_sec`（含/不含编译）、`peak_vram_mib`、是否崩（TLS 断言 L6-10）、是否有 PTX fallback（性能损失 L6-08）。
- **停止条件**：若首编译 >10 min 或触发 Windows 专属崩溃 → 确认 H3「首编译久/常出问题」，回退关编译。
- **回滚**：崩 → `--disable-comfy-compiler` + 关 Triton 后端，回到纯异步卸载路径。

**实验 3 · H3 GGUF（Abiray Q4_K_M 11.6GB）vs 原生 int8_convrot 18.9GB 卸载计时（确认 H4 反转）**
- **命令/节点**：同实验 1 分辨率/帧数；A=原生 `minimax_h3_fl2va_pruned_int8_convrot`（18.9GB，LOG-005 L5-03/07）经 UnetLoader；B=GGUF `MiniMax-H3-FL2VA-Pruned-Q4_K_M.gguf`（11.6GB，L6-11）经 `UnetLoaderGGUF` + leejet/ComfyUI-GGUF（解决 minimax_h3 架构，L6-12/13）。
- **观测指标**：`wall_sec`、`peak_vram_mib`、`peak_ram_used_gb`、`peak_pagefile_gb`、是否报「Unknown model architecture」（L6-12 坑）。
- **停止条件**：若 B（GGUF）RAM 峰值显著低于 A 且可跑通 → 确认 H3 GGUF 在本机可行（否定 H4 的「无 GGUF」）；若 B 因架构/节点问题崩 → 记录操作门槛。
- **回滚**：B 崩 → 退回原生 int8_convrot；必要时 `--disable-pinned-memory`（实验 1 结论）。

---

## 八、链接核实表（本轮 2026-11-11 JST 直读）

| 来源 | 直读结果 | 等级 | 处置 |
|---|---|---|---|
| https://docs.comfy.org/development/comfyui-server/startup-flags | 可访问，官方文档直读 | B | L6-01/02/09 |
| https://github.com/woct0rdho/triton-windows | 可访问，README 直读 | B | L6-07 |
| https://github.com/kentstone84/PyTorch-2.10.0a0/blob/main/docs/REDDIT_RESPONSE.md | 可访问 | C | L6-08 |
| https://smeltcore.com/recipes/voxcpm2-on-rtx-5070-ti-30-language-48khz-voice-cloning-in-8-gb-vram | 可访问 | C | L6-10 |
| https://www.instasd.com/post/comfyui-vram-offloading-guide | 可访问 | C | L6-04 |
| https://freakyjolly.com/minimax-h3-out-of-memory-comfyui-16gb-fix | 可访问 | C | L6-05 |
| https://deepwiki.com/kijai/ComfyUI-WanVideoWrapper/14-performance-optimization-guide | 可访问（基于仓库源码） | B/C | L6-06 |
| https://comfy.icu/node/WanVideoBlockSwap | 可访问 | C | L6-06 |
| https://huggingface.co/Abiray/MiniMax-H3-Pruned-GGUF | 可访问，HF 卡页直读 | B | L6-11 |
| https://savrn.com/models/minimax-h3-pruned-gguf | 可访问（镜像，含大小/SHA） | B | L6-11 |
| https://freakyjolly.com/best-minimax-h3-gguf-workflow-comfyui/ | 可访问 | C | L6-12 |
| https://huggingface.co/unsloth/MiniMax-H3-GGUF | 可访问，HF 卡页直读 | B | L6-13 |
| https://alishangtian.com/article/user-9065771665 | 可访问（引述 comfy-aimdo） | C | L6-03 |
| https://arxiv.org/html/2609.21849v1 | 可访问（arXiv abs/html） | C | L6-14 |
| https://arxiv.org/html/2611.08023v1 | 可访问（arXiv abs/html） | C | L6-15 |
| https://www.nvidia.com/en-us/geforce/laptops/compare/ | 可访问（LOG-005 r1 修正 L5-01） | B | LOG-005 r1.1 |
| https://nvidia.custhelp.com/app/answers/detail/a_id/5490 | 可访问（LOG-005 r1 升 B 修正 L5-08/09） | B | LOG-005 r1.2/r1.3 |

> **未直读/仅摘要**：TorchCompileModel 节点的官方独立页未直读（L6-09 的节点行为标 C）；city96 README 沿用 LOG-005 L5-06（未重读）。

---

## 九、事实 / 推测 / 建议 汇总

- **事实（B/C 已标）**：
  - ComfyUI **异步权重卸载默认开（Nvidia/AMD）、动态 VRAM 在支持系统自动开**；相关 flag：`--async-offload [NUM_STREAMS]`（默认 2）、`--disable-async-offload`、`--disable-pinned-memory`、`--fast-disk`、`--reserve-vram`、`--vram-headroom`、`--lowvram`（**动态 VRAM 下无效**）、`--disable-smart-memory`（B，L6-01/02）。
  - **pinned memory 默认开会吃 RAM**，32GB 跑 >8GB 权重可能 OOM/pagefile；社区案例 `--disable-pinned-memory` 解决模板 OOM（C，L6-04）；**32GB RAM 是 H3 跑通的硬约束**（C，L6-04/05）。
  - H3 解码显存修复 **PR #15446（decode 不随 clip 增长）/ #15456（tiled-decode 前释放其它模型）**（C，L6-05）。
  - **triton-windows 官方支持 sm_120**，要求 **Triton≥3.3 + PyTorch≥2.7 + CUDA≥12.8**；Windows 长路径限制致 torch.compile 失败（B，L6-07）；Windows 上 PTX fallback 可丢 20-30%、`optimize=True` 后台线程 TLS 断言（C，L6-08/10）。
  - **H3 GGUF 真实存在且较广泛**：Abiray/MiniMax-H3-Pruned-GGUF（2026-08-07，Q3_K_M 8.9GB…Q8_0 21.6GB，B L6-11）、unsloth/MiniMax-H3-GGUF（B L6-13）；但 **city96/ComfyUI-GGUF 未内置 H3 架构**，须 leejet fork 或手动加 arch（C，L6-12/13）。
  - 近 3 月 arXiv：低比特(FP8/NVFP4)+多模型 GPU 卸载方案（C L6-14）；PSI 固定工作集 subtensor 流式在 A6000 上把 22B 视频 DiT 峰值从 8578→360MiB（代价时间 +41%，C L6-15）。
- **推测（C，未核实，不写结论）**：异步卸载在 H3 视频 8GB 上比旧同步卸载 >20% 提速（仅图像场景有社区数字）；torch.compile 在 H3 视频 8GB 上 10–30% 提速；GGUF 在本机速度与画质。
- **建议（非结论）**：本机 8GB 跑 H3 首选 **原生 NVFP4 11.9GB / int8_convrot 18.9GB + 默认异步卸载 + 动态 VRAM + `--reserve-vram 1~2`**；RAM 吃紧首试 `--disable-pinned-memory`，NVMe 快则试 `--fast-disk`；torch.compile 仅作实验项（先开 Windows 长路径、装 triton-windows<3.7、备 `--disable-comfy-compiler` 回滚）；GGUF 可作 RAM 压力备选但需解决架构支持，不列为 8GB 首选；务必用含 H3 PR #15446/#15456 的新版 ComfyUI；**32GB RAM 是硬约束，优先确认本机 OEM TGP 与 pagefile 策略**。

---

*本日志由 R6（2026-11-11 JST）编写。亲开 17 个来源（ComfyUI 官方启动参数文档、triton-windows README、instasd/freakyjolly 社区 H3 实测、deepwiki/comfy.icu kijai block swap、Abiray/unsloth HF H3 GGUF、两篇近 3 月 arXiv、NVIDIA 5490 与 en-us 规格页用于 LOG-005 修订）。无 A 级（本任务无法本机复现）。LOG-005 修订记录 r1 已追加（L5-08 升 B、L5-09 官方来源更正、L5-01 TGP 冲突未解决、L5-10/11/12 补精确日期、H3/H4 降为部分核实）。所有 16GB+/24GB+ 数字未写入 8GB 推荐表。*
