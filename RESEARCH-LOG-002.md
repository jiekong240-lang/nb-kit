# RESEARCH-LOG-002：32GB 内存约束下的 H3 整条管线（DiT + 文本编码器 + VAE）落地调研

- 性质：文献 / 源码 / 社区实测调研，**非本机复现**（本任务不可能出现 A 级）。
- 目标硬件：RTX 5070 Laptop 8GB（Blackwell sm_120）、**32GB RAM**、Win11。
- 调研日期：2026-11-04。所有数字均来自 `./sources/R2-*.md` 的逐字摘录；未打开的内容不写结论。
- 与 LOG-001 的关系：不重复其已有条目；本文只补“系统内存峰值”这一缺口，必要时引用 R1-xx 交叉验证。
- 前置修正：已完成 LOG-001 第十三节 E1 依据栏架构名更正（Ampere sm_89 → Ada Lovelace sm_89；Ampere = RTX 30 系 sm_86），并在节末追加 F1 行；修改前快照存 `RESEARCH-LOG-001.v2.md`。

## 证据等级
- **A** = 本机复现（本任务无）。
- **B** = 外部可复现：公开源码/官方文档 + 完整方法 + 作者或第三方实机数字（单一作者标“单源”）。
- **C** = 研究候选：有真实代码/论文/文档，但关键数字未独立复现，或硬件/系统与目标差异大。
- **D** = 未经证实。

---

## 零、结论先行（假设检验结果）

原假设：“32GB 下瓶颈从显存转为系统内存峰值；靠 TE 量化+卸载、DiT mmap/盘上流式、VAE 分块、`--cache-none` 压住。”

| 子假设 | 结论 | 依据 |
|---|---|---|
| 32GB 下瓶颈是系统内存而非显存 | **基本成立，但被低估的是“共享显存”这条暗渠**：显存溢出会变成 shared GPU memory（即系统 RAM），16GB 卡实测就有 12.5GB 溢到共享内存 | R2-11、R2-13 |
| 大 TE 与 DiT 同时驻留 RAM 会崩 | **成立且已定位到代码**：mmap 映射被强引用后，15.4GB TE 即使“已卸载”仍有 13GB 留在 RAM | R2-01 |
| TE 量化 + 编码后卸载有效 | **成立**：32B int8 25.28GiB → nvfp4 14.61GiB；4B int4 仅 2.62GB；卸载后采样期 RAM 可保持平坦 | R2-10、R2-11、R2-01 |
| DiT 用 mmap/盘上流式可压 RAM 峰值 | **成立且是最关键的一条**：权重可完全不进进程 RAM（mmap 未提交页），盘上流式时 host 侧只需 ~1GB + 1.4GB pinned staging | R2-02、R2-03、R2-07 |
| VAE 分块/流式能同时压显存和内存 | **成立**：H3 VAE 解码显存峰值 3485→607MB、内存峰值增量 7089→2320MB | R2-05 |
| `--cache-none` 是省内存主力 | **部分成立**：它省的是“任务之间”的缓存，**不省单任务内部峰值** | R2-11、R2-14 |

**最重要的反直觉发现**：Windows 上真正让 32GB 机器“假死”的不是 OOM，而是 WDDM 把显存溢出搬到共享内存后，每次 GEMM 都经 PCIe 重读权重——实测 >7GB 的块带宽从 408GB/s 掉到 12GB/s，步时从秒级变成分钟级（R2-13）。因此“省显存”与“省内存”必须同时做，只做一项会把问题从崩溃变成极慢。

---

## 一、32GB（或更少）内存下视频 DiT 的实测与崩溃报告（4 条）

**R2-01｜ComfyUI #17122：文本编码器 mmap 滞留吃掉 32GB 机器**
- 来源：https://github.com/Comfy-Org/ComfyUI/issues/17122 ｜ 2026-10-17 ｜ B
- 原文测试条件：RX 9070 XT 16GB / **32GB RAM** / Arch Linux + ROCm 7.2 / ComfyUI v0.41.3 / LTX-2.5 i2v，Gemma4-12B int8 TE（15.4GB）+ LTX-2.5-22B Q4_K_M；`--disable-smart-memory --cache-none --disable-pinned-memory`。
- 报告效果：**显存**：未给峰值（16GB 卡）。**内存**：15MB 的 Linear 被 `LARGEST_CASTED_WEIGHT` 强引用，导致 13GB 的 TE 文件映射留在 RAM；DiT 加载时只剩 ~200MB 可用，卡在 `copy_`/`memcpy_and_sync` 10 分钟以上，GPU 3%；手工清引用后进程 RSS 16.9GB → 3.9GB。
- 画质代价：无（纯内存生命周期问题）。
- 适用性：机制与 OS 无关（mmap + 强引用在 Windows 同样成立）；H3 的 Qwen3-VL-32B 比 Gemma4 更大，风险更高。不确定点：该 issue 在 ROCm/无 aimdo 路径复现；Windows 默认启用 aimdo 时是否同样触发未验证。

**R2-10｜网易号实测：32GB 内存 + 16GB 卡，默认 2GB 页面文件导致无报错闪退**
- 来源：https://c.m.163.com/news/a/L8AT2A240556I5F3.html ｜ 2026-10-03 ｜ C+（单源、细节自洽）
- 原文测试条件：RTX 5060 Ti 16GB / **32GB RAM** / Win11 / cu130 / H3 ref2va：DiT int8_convrot 19.53GB + TE qwen3vl_32b nvfp4_awq 14.61GB + video VAE fp16 4.85GB + audio VAE fp32 0.56GB + turbo LoRA 1.82GB（合计 41.4GB）。
- 报告效果：**显存**：生成过程稳定在 11.7GB。**内存**：文本编码阶段需把 14.6GB 编码器加载进内存；Windows 默认页面文件仅 2GB 时“跑到一半窗口直接闪退，没有任何报错”；改为初始 32768MB / 最大 65536MB（NVMe 盘）后稳定。
- 画质代价：未提。
- 适用性：与目标机同为 32GB + Win11，是最接近的“32GB 真实使用”证据；但显卡是 16GB，8GB 机器上权重驻留更少、卸载更频繁，内存压力方向相同、量级不同。不确定点：作者未给 RAM 峰值绝对值；“默认 2GB”是否为该系统实际值未截图。

**R2-11｜FreakyJolly：16GB 卡 + 共享显存溢出，单任务 RAM 峰值 27.2GB**
- 来源：https://freakyjolly.com/minimax-h3-out-of-memory-comfyui-16gb-fix/ ｜ 2026-10-20 ｜ B（单源，逐字日志）
- 原文测试条件：RTX 5080 16GB / Windows / H3 fl2va int8_convrot（T2V/I2V）与 ref2va w4a8（R2V）/ 0.3–0.4MP / 4–6s / 6 步 turbo / `--reserve-vram 1.0 --cache-none`。
- 报告效果：**显存**：采样时 Task Manager 显示 dedicated 15,823.7MB + **shared 12,578.7MB**（即 12.5GB 溢出到系统 RAM）。**内存**：worker RAM 在采样 100 秒内**平坦保持在 27.2GB**（证明 TE/VAE 未在中途释放）；commit 一度达 120–145%；作者结论“32GB is tight, 64GB is comfortable”。
- 画质代价：无（只改调度）；Q4_0 GGUF 比 int8 慢 29%（333s→431s，81 帧 I2V），但这是速度不是画质。
- 适用性：**直接命中 32GB 边界**——单任务已 27.2GB，浏览器/系统一开就进入 pagefile。8GB 卡上 shared 溢出会更早发生，故 32GB 更紧。不确定点：27.2GB 含 Windows 文件缓存口径，任务管理器“内存”列可能高估真实占用。

**R2-12｜CSDN：RTX 2070 8GB + 32GB 内存，540p/6s 实测**
- 来源：https://blog.csdn.net/weixin_32830601/article/details/165902173 ｜ 2026-09-18 ｜ C
- 原文测试条件：i7-10700 / **32GB RAM** / RTX 2070 8GB / 6s、约 540p（其余参数未给）。
- 报告效果：**显存**：峰值约 7.2GB；开“Face Refine”再 +1.5GB。**内存**：生成过程约 18GB。
- 画质代价：未量化。
- 适用性：唯一同时给出“8GB 卡 + 32GB 内存”的显存/内存双数字，可作为 32GB 有余量的弱证据。不确定点：文章声称 H3 内置“面部感知加权/Face Refine”，与公开结构不符，疑为二次创作；模型精度、步数、ComfyUI 版本均缺失，不能作为 B 级基线。

**小结（第 1 类）**：**没有找到 B 级的“8GB 显存 + 32GB 内存”完整 H3 基线**。最可靠的是 R2-11（16GB 卡、27.2GB 峰值、32GB 紧张）与 R2-10（32GB、页面文件是生死线）。8GB + 32GB 只有 C 级的 R2-12。

---

## 二、文本编码器的内存策略（3 条）

**R2-07｜DMAD：32B TE 永不整块上卡，逐层流式 + host 只留 page cache**
- 来源：https://github.com/Yzmblog/DMAD ｜ 2026-10-10（8GB 更新）｜ B（作者自测 + 开源代码）
- 原文测试条件：MiniMax-H3-33B + 4 步 LoRA / 1344×768 / 124 帧（5.2s）/ `--low-vram` / 8GB 与 12GB 档。
- 报告效果：**显存**：文本编码阶段峰值 3.4GiB（124 帧与 362 帧相同）。**内存**：文本编码 “page cache only”，权重 36GB（int8 18GB）从 checkpoint 逐层流式读，不整体进 RAM；采样阶段 host 侧 ~1GB + 1.4GB pinned staging（若加 `--weights-in-ram` 则 +36GB，int8 +18GB）。
- 画质代价：无（与原路径 bit-identical）。
- 适用性：**这是 32GB 机器最该抄的 TE 策略**——不缓存整份 TE，只保留磁盘页缓存；8GB 档已实测 5s <7GiB。不确定点：DMAD 是独立推理器（非 ComfyUI 节点），ComfyUI 内能否复现“TE 逐层流式”未验证；`--low-vram` 的 host 总量未给。

**R2-10（TE 部分，见第一节）｜TE 量化档位差 10.7GiB（25.28 → 14.61GiB）**
- 来源：同 R2-10 ｜ C+
- 原文测试条件：同 R2-10。
- 报告效果：**显存**：11.7GB 稳定（16GB 卡）。**内存**：官方 int8 TE 25.28GiB “装不下”，nvfp4_awq 14.61GiB 可装入。
- 画质代价：NVFP4/AWQ 轻微（作者未量化）。
- 适用性：32GB RAM 下 TE 体积直接决定“能否与 DiT 共存”。8GB 卡上 14.61GiB 的 TE 仍大于显存，必须靠卸载/流式。不确定点：nvfp4 在 8GB 卡上的实际编码耗时未给。

**R2-11（TE 部分，见第一节）｜TE 两个正交手段：SetReserveVRAM 3.0 + 换 4B int4**
- 来源：同 R2-11 ｜ B
- 原文测试条件：同 R2-11。
- 报告效果：**显存**：launcher `--reserve-vram 0.5` 时只剩 ~77MB 空闲、Qwen 传输卡死；在 CLIPLoader 后插 `SetReserveVRAM reserved_gb=3.0` 后文本编码通过。改用 `ClipProjLoader` + `qwen3vl_4b_int4_convrot`（streaming）后，TE 文件仅 2.62GB。
- 内存：未单独给 TE 释放前后的 RAM 数字；但 27.2GB 平坦值说明“不手动释放就不会释放”。
- 画质代价：4B 替代 32B 的语义能力下降（作者未量化，属结构性降级）。
- 适用性：`SetReserveVRAM` 是 ComfyUI 内可用的最轻量手段；4B TE 对 H3 是否保持提示词遵循度未知，属高风险高收益。

**小结（第 2 类）**：32GB 下 TE 策略优先级 = ① 编码后**强制卸载并清引用**（R2-01）→ ② 量化到 ≤15GiB（R2-10）→ ③ 有条件换 4B（R2-11）或逐层流式（R2-07）。

---

## 三、DiT 权重的 mmap / 磁盘流式 / block swap 对 RAM 峰值的影响（4 条）

**R2-02｜ComfyUI PR #11845：mmap 未提交页，权重不进进程 RAM**
- 来源：https://github.com/comfyanonymous/ComfyUI/pull/11845 ｜ 创建 2026-01-13，合并 2026-02-01 ｜ B
- 原文测试条件：RTX 5090（人为占用 8GB，等效 24GB）/ 96GB RAM / PCIe5 NVMe / Flux2 + LoRA。
- 报告效果：**显存**：原文未给峰值数字（仅截图）。**内存**：`assign=True` 加载后权重“lives in mmap permanently and never consumes any process allocated RAM”；同时用 READ 模式加载 safetensors 避免整个文件被 commit-charged；“No need to load models fully to RAM”。
- 画质代价：无。
- 适用性：**这是“省内存”而非“省显存”的关键机制**，已合并进主线，5070 Laptop 默认享有。不确定点：原文没有 RAM 峰值数字；96GB 机器无法证明 32GB 下足够。

**R2-03｜ComfyUI 官方：56GB 权重在 32GB RAM 的 RTX 5060 上跑 Wan2.2 双 14B**
- 来源：https://comfyui.org/en/dynamic-vram-in-comfyui-saving-local ｜ 2026-03-26 ｜ B
- 原文测试条件：Windows / RTX 5060 / **32GB 与 64GB RAM 两档** / Wan2.2 dual 14B fp16+fp8 / 320×320×81f / 总权重 56GB（2×28GB）。
- 报告效果：**显存**：未给数字，只称“higher GPU memory utilization”。**内存**：“Operate models beyond physical RAM without slow system page files”；safetensors 映射为未提交内存，Windows 可能显示高 RAM 但“release instantly when required”。
- 画质代价：无。
- 适用性：官方明确 32GB 可跑 56GB 权重，说明“权重不必进 RAM”在 32GB 上成立。不确定点：320×320×81f 是极小分辨率，不能外推到 H3 1344×768；未给耗时。

**R2-04｜comfy-aimdo：VBAR + 按需 fault，显存按页进入**
- 来源：https://github.com/Comfy-Org/comfy-aimdo ｜ 最近提交 2026-10-20 ｜ B
- 原文测试条件：通用（PyTorch 2.8+ / CUDA 12.8+ / Win11+）。
- 报告效果：**显存**：VBAR 只占 GPU 虚拟地址，“Creating a VBAR doesn't cost any VRAM”；张量在需要时才 `fault()` 分配物理显存。**内存**：未给数字；称该分配器“behaves a lot better on Windows systems with System Memory fallback”。
- 画质代价：无。
- 适用性：解释为什么 DynamicVRAM 模式下“模型比显存大也能跑”，但也解释 R2-13 的代价（页被放到共享内存后每次重读）。不确定点：无量化。

**R2-13（跨类）｜block swap 与 WDDM 的 RAM 代价**
- 见第五节。核心：block swap 省的是显存，被换出的块**必须留在 RAM**；当 WDDM 把页放到共享内存时，PCIe 重读把“省显存”变成“极慢”。R2-13 实测 145 帧链路 peak VRAM 7.9GB、process RSS 22.5GB（64GB 机器）。
- 结论：**mmap/盘上流式是“省内存”，block swap 是“用内存换显存”**，二者方向相反，32GB 机器应优先前者。

**小结（第 3 类）**：省内存的正确路径是 `--fast-disk`（官方语义：磁盘后备动态加载/卸载，快盘自动开启，R2-14）+ mmap（R2-02/R2-03），而不是把权重换出到 RAM。

---

## 四、VAE 分块与流式解码（3 条）

**R2-05｜ComfyUI PR #15446：H3 VAE 分块流式，显存与内存双降**
- 来源：https://github.com/comfyanonymous/ComfyUI/pull/15446 ｜ 2026-08-09 合并 ｜ B
- 原文测试条件：作者 kijai；175 帧 × 1344×768；未注明 GPU/OS（应为大显存卡，因 before 峰值 6GB 级）。
- 报告效果：
  - **显存（decode）**：peak alloc 3485MB → **607MB**；peak reserved 4150MB → **944MB**。
  - **内存（decode）**：peak 增量 7089MB → **2320MB**；decode 后残留 4936MB → 2303MB。
  - **显存（encode）**：5714MB → 2402MB；124 帧时 3803MB → 2381MB。
  - 耗时：decode 14.6s → 13.9s（基本不变）。
- 画质代价：无，输出 bitwise identical。
- 适用性：**长片（>124 帧）在 8GB 上的必需品**；显存瞬态与长度解耦。不确定点：测试是 1344×768 大卡，8GB 卡上分块尺寸与耗时需实测。

**R2-06｜ComfyUI #15484：DiT + 双 VAE 共驻 14.6GB，16GB 卡只剩 805MB**
- 来源：https://github.com/Comfy-Org/ComfyUI/issues/15484 ｜ 2026-08-11（2026-10-23 更新）｜ B
- 原文测试条件：RX 9070 XT 16GB / Win11 / ComfyUI 0.31.0 / ROCm 7.15 / DynamicVRAM 开 / `--disable-async-offload`。
- 报告效果：**显存**：MiniMaxH3 9472MB + AudioVAE 416MB + VideoVAE 4736MB = 14624MB，device free ~805MB；解码耗时在 17.88–38.76s 间抖动（均值 24.10s），#15446 后单跑 VAE 稳定 ~16.8s。**内存**：未给。
- 画质代价：无（只影响速度）。
- 适用性：8GB 卡不可能让三者共驻；必须在“编码/采样/解码”之间显式换出。R2-11 的 VRAM_Debug 就是同一思路的用户侧实现。

**R2-07（跨类）｜DMAD 的 `--vae-offload`：VAE 权重留在 host，逐层流式解码**
- 原文：“`--vae-offload` ... stream the video VAE from host memory during the decode (~9 GB less GPU memory)”；decode 峰值 3.3–3.4GiB，host 侧 1.6/2.3GiB（8GB 档）。
- 适用性：说明“VAE 不常驻显存”可再省 ~9GB 显存，代价是 host 侧 1.6–2.3GB 内存——32GB 完全可承受。

**小结（第 4 类）**：VAE 是“显存峰值 + 内存峰值”的双杀点；#15446 之后必须升级到含该 PR 的 ComfyUI，否则长片解码仍按长度线性增长。

---

## 五、Windows pagefile / WDDM / 共享显存回退（4 条）

**R2-13｜Kinoloom 实验室：WDDM 静默换页，>7GB 块带宽 408→12GB/s**
- 来源：https://github.com/harlixay7/Kinoloom/blob/main/docs/research/HARDWARE_BENCHMARKS_AND_LIMITS.md ｜ 日志时间戳 2026-10-22/23 ｜ B（单实验室，N=1 明示）
- 原文测试条件：RTX 3070 8GB / **64GB RAM** / Win11 / PCIe 3.0 x16 / 驱动 616.64 / ComfyUI + DynamicVRAM。
- 报告效果：**显存**：正常链路 peak VRAM 7.9GB。**内存**：正常 process RSS 22.5GB；故障run RSS 39GB、可用 RAM 0.55GB；修复后 peak 19.1GB、可用 ≥21.7GB。探针：≤6.75GB 的块 408GB/s，>7GB 的块 12GB/s（已在共享系统内存）。
- 其他关键数字：`--vram-headroom 1/2` **无效**（195/208 s/step）；`--disable-dynamic-vram` 首步 ~590s，而 DynamicVRAM 为 173–215s。
- 画质代价：无（但会误判为“卡死”）。
- 适用性：**5070 Laptop 会更容易触发**（笔记本 PCIe 通道更少、显存带宽更低）。不确定点：作者把“关 Sysmem Fallback”列为假设，尚未执行；“--vram-headroom 无效”与官方语义（只对自己的权重页生效）一致。

**R2-10（跨类）｜默认 2GB 页面文件 = 无报错闪退**
- 见第一节。32GB 物理内存 + H3 14.6GB TE 时，页面文件是最后的安全垫；作者设 32768/65536MB 后稳定。

**R2-11（跨类）｜commit 120–145% 是失败前兆**
- 见第一节。“When VRAM overflows, Windows spills into shared GPU memory, which is system RAM”；“Shared GPU memory is not free VRAM ... you are not OOM, you are slow”。

**R2-02（跨类）｜Windows 不允许 overcommit（除 pagefile），所以要降 commit charge**
- PR 通过 READ 模式 safetensors + 不建占位权重来降 commit；说明“页面文件够大”只是兜底，正解是减少提交量。

**小结（第 5 类）**：32GB 机器的推荐组合 = 固定大页面文件（具体 GB 数依据不足，R2-10 用 32–64GB；NVMe）+ 关闭/避免 Sysmem Fallback（或至少监控 shared GPU memory）+ 降低 commit（mmap/盘上流式）。注意 R2-13 显示 `--vram-headroom` 对“权重页被放进共享内存”无效。

---

## 六、近 3 个月新出现的省内存/省显存技巧（3 条）

**R2-07｜DMAD：H3 4 步蒸馏 + 8GB 档 <7GiB（2026-10-01/10）**
- 来源：https://arxiv.org/abs/2610.02188 ｜ 2026-10-01 ｜ B（作者自测 + 开源）
- 测试条件：H3-33B，1344×768，124 帧（5.2s），4 步，32kHz 立体声；`--low-vram 8`。
- 效果：**显存**：8GB 档 5s <7GiB、15s 12GiB；分阶段 3.4（TE）/5.9（采样）/3.3（解码）GiB。**内存**：权重 36GB 流式（int8 18GB），host 侧 ~1GB+1.4GB pinned staging。
- 画质：4 步学生相对 50 步教师，人类偏好 79.1%（vs DMD2）/84.6%（vs rCM）；4 步快速运动有涂抹（R1-12 同类经验）。输出与自身大卡路径 bit-identical。
- 适用性：**8GB 卡上目前最完整的公开方案**（含 ComfyUI 5s 工作流）。不确定点：非 ComfyUI 原生管线；32GB RAM 未明示验证；8GB 档只到 5s。

**R2-08｜DASA：压缩历史 KV cache，自回归视频 2.16×（2026-10-19）**
- 来源：https://arxiv.org/abs/2610.26262 ｜ 2026-10-19 ｜ C
- 测试条件：VBench，自回归视频 DiT（非 H3）。
- 效果：**显存**：未给绝对值，机制为“只保留生成 chunk 内的动态特征”压缩历史 KV。**内存**：未给。
- 画质：摘要称保持高保真，无逐项数字。
- 适用性：低——H3 是双向全序列去噪，不是自回归 KV 缓存范式；只能作为“KV 压缩”思路参考。不确定点：代码未在摘要中给出。

**R2-09｜HeteroVDiff：INT8 权重单份共享，权重内存 -48%（2026-10-12）**
- 来源：https://arxiv.org/abs/2610.16244 ｜ 2026-10-12 ｜ C
- 测试条件：Galaxy S25（NPU+GPU 共享 DRAM），多专家视频扩散。
- 效果：**显存/内存**：比“每个引擎各存一份”少 48% 权重内存；吞吐 1.35×。无绝对值。
- 画质：VBench-2.0 18 维总分与 FP32 持平。
- 适用性：**极低**（手机 SoC 双引擎），仅“单份量化权重 + 按 block 切分”思路可借鉴。

**小结（第 6 类）**：真正能落地到 5070 Laptop 8GB 的只有 DMAD 一类“少步 + 盘上流式”；DASA/HeteroVDiff 是方向性参考，不能写进 V1.0 配置。

---

## 七、32GB 内存候选 V1.0 配置表

> 规则：每行只由 B 级来源拼出，注明出处；没有 B 级依据的写“依据不足”。本表是“可执行的候选”，不是已验证基线。

| 组件 | 取值 | 出处 | 依据强度 |
|---|---|---|---|
| 权重格式（DiT） | `int8_convrot` safetensors + 原生 UNETLoader（非 GGUF） | R2-11（B）；R1-06 | B |
| 文本编码器 | Qwen3-VL-32B `nvfp4_awq` 14.61GB；编码后**立即 unload + 清引用** | R2-10（C+，体积）、R2-11（B，卸载）、R2-01（B，机制） | 体积 C / 卸载 B |
| TE 备选 | `qwen3vl_4b_int4_convrot`（2.62GB，streaming） | R2-11 | B（体积）/ 画质未知 |
| 权重加载方式 | DynamicVRAM + mmap + `--fast-disk`（NVMe） | R2-02、R2-03、R2-14 | B |
| 显存预留 | `--reserve-vram 1.0`；CLIPLoader 后 `SetReserveVRAM 3.0` | R2-11、R2-14 | B |
| VAE 解码 | ComfyUI ≥ 含 PR #15446（分块流式）；解码前释放 DiT/TE | R2-05、R2-11 | B |
| 分辨率 / 时长 | 0.3–0.4MP、4–6s（124 帧） | R2-11（16GB 卡） | B（但非 8GB） |
| 步数 | Turbo 6 步（4 步涂抹、>8 步无收益） | R2-11、R1-12 | B |
| 缓存 | `--cache-none`（仅省任务间缓存，不省单任务峰值） | R2-11、R2-14 | B |
| 页面文件 | 固定大小、NVMe、监控 commit 与 shared GPU memory | R2-02（B，机制）；具体 GB 数无 B 级来源 | **依据不足**（32–64GB 仅 R2-10，C+） |
| Windows Sysmem Fallback | 优先关闭（Prefer No Sysmem Fallback）以避免 PCIe 抖动 | R2-13（B，假设未执行）、R1-23（B，机崩） | B（方向）/ 未验证 |
| **8GB 卡 + 32GB 内存的完整可重复基线（含耗时/双峰值）** | — | 无 B 级来源 | **依据不足** |

**表外结论**：可以拼出“大概率能跑”的配置，但**拼不出“可重复的 V1.0 基线”**——缺的正是 8GB+32GB 的双峰值实测。这也是下一轮本机验证的全部意义。

---

## 八、链接核实表

| 编号 | URL | 读取方式 | 核实状态 | 等级 |
|---|---|---|---|---|
| R2-01 | github.com/Comfy-Org/ComfyUI/issues/17122 | WebFetch 全文 + GitHub API（日期） | 已读，数字逐字 | B |
| R2-02 | github.com/comfyanonymous/ComfyUI/pull/11845 | WebFetch 全文 + API（合并日期） | 已读（无 RAM 数字） | B |
| R2-03 | comfyui.org/en/dynamic-vram-in-comfyui-saving-local | WebFetch 全文 | 已读（无峰值数字） | B |
| R2-04 | github.com/Comfy-Org/comfy-aimdo | WebFetch README | 已读（机制） | B |
| R2-05 | github.com/comfyanonymous/ComfyUI/pull/15446 | WebFetch 全文 | 已读，数字逐字 | B |
| R2-06 | github.com/Comfy-Org/ComfyUI/issues/15484 | WebFetch 全文 + API | 已读，数字逐字 | B |
| R2-07 | github.com/Yzmblog/DMAD ; arxiv.org/abs/2610.02188 | WebFetch 两页全文 | 已读，表格逐字 | B |
| R2-08 | arxiv.org/abs/2610.26262 | WebFetch 全文 | 已读（仅摘要数字） | C |
| R2-09 | arxiv.org/abs/2610.16244 | WebFetch 全文 | 已读（仅摘要数字） | C |
| R2-10 | c.m.163.com/news/a/L8AT2A240556I5F3.html | WebFetch 全文 | 已读，数字逐字 | C+ |
| R2-11 | freakyjolly.com/minimax-h3-out-of-memory-comfyui-16gb-fix/ | WebFetch 全文 | 已读，数字逐字 | B |
| R2-12 | blog.csdn.net/weixin_32830601/article/details/165902173 | WebFetch 全文 | 已读（内容可信度存疑） | C |
| R2-13 | github.com/harlixay7/Kinoloom/.../HARDWARE_BENCHMARKS_AND_LIMITS.md | WebFetch 全文 | 已读，数字逐字 | B |
| R2-14 | docs.comfy.org/development/comfyui-server/startup-flags | WebFetch 全文 | 已读（语义） | B |

## 九、只看了标题/摘要未细读（不写结论）

- artificialguy.com “MiniMax H3 VRAM requirements”（搜索结果摘要提到 64GB RAM、`--fast-disk`）：未打开，不采信其数字。
- ai345.info、taobao 百科、sheratonhq、lpqz.cn、mwwr.cn、hqwc.cn 等聚合/SEO 站：仅出现在搜索结果，**未打开**，全部按 D 处理，不引用。
- Reddit r/StableDiffusion / r/comfyui：本轮检索未返回可用帖子（`site:reddit.com` 查询无结果），**未核实**是否有 32GB 实测。
- DMAD 的 ComfyUI 节点目录 `comfyui/ComfyUI-DMAD`：README 提及，未逐个打开源码。
- DASA 项目页（yuanyu0.github.io/DASA）：未打开，代码是否开源**未核实**。

## 十、事实 / 推测 / 建议

- **事实（B 级）**：
  1. 权重可以完全不进进程 RAM：mmap 未提交页 + 按需 fault（R2-02、R2-03、R2-04）。
  2. H3 VAE 解码的显存峰值可从 3485MB 降到 607MB、内存峰值增量从 7089MB 降到 2320MB（R2-05）。
  3. H3 单任务 worker 的 RAM 峰值实测可达 27.2GB，并出现 12.5GB 共享显存溢出；作者据此判断“32GB is tight, 64GB is comfortable”（R2-11，作者机内存容量未说明）。
  4. 文本编码器 mmap 被强引用时，13GB 映射会滞留 RAM，足以让 32GB 机器停摆（R2-01）。
  5. WDDM 下 >7GB 的块带宽从 408GB/s 掉到 12GB/s（R2-13）。
  6. DMAD 在 8GB 档把 5s H3 压到 <7GiB 显存，权重盘上流式（R2-07）。
- **推测（C/D）**：
  - 5070 Laptop 的 32GB 会因“共享显存 + 浏览器”比 R2-11 的 27.2GB 更早触顶（推测，未实测）。
  - 8GB 卡上 int8 DiT 19.5GB 全量盘上流式可行（R2-07 对 33B 已验证，但非 ComfyUI 路径）。
  - 关 Sysmem Fallback 能让 5070 Laptop 从“极慢”回到“可预测的 OOM/流式”（R2-13 假设，未执行）。
- **建议**：V1.0 先按第七节表搭；第一优先级不是画质而是**把双峰值测出来**；一旦 shared GPU memory 上升，立即降分辨率而不是加 `--vram-headroom`。

## 十一、下一轮最值得在本机验证的 3 项（附最小实验与停止条件）

1. **32GB 下的双峰值基线（最高优先，因为这是唯一“依据不足”的核心缺口）**
   - 最小实验：ComfyUI（含 #15446）+ DiT int8_convrot + TE nvfp4_awq + Turbo 6 步 + `--reserve-vram 1.0 --cache-none`；跑 416×736/124 帧 1 次；用任务管理器 + `nvidia-smi` 同时记录 **peak dedicated VRAM / peak shared GPU memory / peak process RAM / peak commit** 四个数。
   - 停止条件：若 peak commit > 30GB 或出现 pagefile 抖动 → 降到 0.2MP 或改用 4B TE；若 shared GPU memory > 2GB → 判为“已进入 PCIe 慢路径”，本配置不算通过。
2. **mmap/盘上流式对 RAM 峰值的真实收益（验证 R2-02/R2-03 在 32GB 的可复现性）**
   - 最小实验：同一图分别用 `--fast-disk`（NVMe）与 `--disable-fast-disk`（走未 pinned RAM）各跑 1 次，记录 peak process RAM 与 peak commit。
   - 停止条件：若两者 RAM 峰值差 <2GB → 说明本机瓶颈不在权重驻留，转而查 TE/VAE；若 `--fast-disk` 首步 >10 分钟 → 记录 NVMe 读带宽后放弃该路径。
3. **Sysmem Fallback 开关的取舍（验证 R2-13 的假设与 R1-23 的机崩风险）**
   - 最小实验：NVIDIA 控制面板对 python.exe 设 “CUDA – Sysmem Fallback Policy = Prefer No Sysmem Fallback”，跑 124 帧 ×3；再恢复默认跑 ×3；记录是否 OOM/机崩/步时。
   - 停止条件：任一设置下出现蓝屏/重启 → 永久关闭该设置并锁 ≤124 帧；若关闭后稳定但步时翻倍 → 保留关闭 + 降分辨率。

---

*结束。来源摘录见 `./sources/R2-01..R2-14`。本日志无 A 级结论；第 0 步 F1 修正已完成。*
