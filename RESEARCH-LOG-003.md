# RESEARCH-LOG-003：注意力后端补证（Comfy Kitchen Attention × RTX 5070 Laptop 8GB / sm_120 / Win11）

- 性质：文献 / issue / 源码 / 社区实测调研，**非本机复现**（本任务不可能出现 A 级）。
- 目标机：RTX 5070 Laptop 8GB（Blackwell sm_120）、32GB RAM、Win11。
- 调研日期：2026-11-04（R3b 重跑；R3 首轮 2026-11-04 因 429 限流 4 秒退出、未产出）。
- 主题：`comfy kitchen attention`（ComfyUI 内置 `ModelAttentionBackend` 节点 → comfy-kitchen `int8_attention`）能否作为 V1.1 提速候选；并纠正 DOSSIER 中"噪声是 sm_120 专项"的说法。

## 0. 方法说明与本次核实的硬限制（必读）

> **本轮 `WebFetch` 被权限拒绝**（工具返回 `Permission to use WebFetch has been denied …`），`Bash` 同样被拒绝。因此**没有任何一个页面是我直连打开的**。
> 本文所有"原文"均来自 `WebSearch` 返回的**搜索引擎抓取正文片段**（搜索结果里由抓取器转写出的页面正文，带 URL）。这等于"第三方代抓"，可公开复核，但**我读到的可能只是页面的一部分**，长线程的上下文可能缺失。
> 处理规则（本文一律遵守）：
> - 只读到片段却在片段内逐字可见的 → 标 **B（可公开复核）**，并在条目标注"片段读取"；
> - 只有标题/元数据、无测试条件与数字的 → 标 **D**，不写任何数字；
> - 只读到二手转述（他人文章引用原文）的 → 标 **C**；
> - **凡本文字母等级与 `sources/R3-*.md` 冲突时，以 sources 为准。**
>
> 计数：本轮共读取 **18 个来源页面**（`sources/R3-01` … `R3-18`），其中 1 个仅元数据（#15738）、1 个正文缺失（#136 本体）。未达到"25 页仍无新证据"的停止条件，但已足以判定三条假设。

> **【R4 复核说明 2026-11-08 JST】** 上述"WebFetch 被禁"是 **R3b 当时的历史限制**，R4 本轮 WebFetch 已可用。以下条目已由 R4 **直连重读**：#15263（L3-01、L3-02）、#15529（L3-04）、comfy-kitchen README + Releases（L3-05）、#15738（L3-12）、comfy-kitchen 官方 issues 列表（L3-06）、#136（H3 节 + 链接核实表第 2 行）。凡本说明、第七节链接核实表、文末「修订记录 r1」与正文原标注冲突时，**以 r1 / 链接核实表为准**（原文段落不删，采用追加/划线说明）。原标「片段读取」的上述页在本轮已升级为「直读」；**仍打不开的页面保留原标注、未编造**。摘录存 `sources/R3-136-*.md`。

## 一、假设检验结果（H1–H3）

### H1｜"Sage 长序列尾帧噪声不是 sm_120 专属，而是 FP8 PV + 长序列" → **部分核实**

| 分句 | 判定 | 依据 |
|---|---|---|
| **不是 sm_120 专属** | **已核实** | RTX 3090（sm_86, Ampere）+ SageAttention 2.2.0 + ComfyUI 0.32.0 + comfy-kitchen 0.2.31 + torch 2.10.0+cu128，1344×768/362 帧，**从第 333 帧起花屏、294 帧全干净**；同一图 PyTorch attention 无噪声 [R3-01, 2026-08-25] |
| **"FP8 PV 路径"是根因** | **未坐实** | ① 同一 issue 里另一人用 sage **1.0.6** 在 sm_86 上 28,520 / 56,120 token 全干净，且指出 Ampere 上 2.x 分派的是 **FP16 PV**；② 第三方复盘据此认为 3090 那条"作者归到 FP8 PV，但按分派它跑的是 FP16 PV" [R3-11]；③ issue 作者本人 8 月 10 日同意"FP8 PV 是原因"，但该结论的证据全部来自 **sm_120** 两张工作站卡 |
| **"长序列"是必要条件** | **已核实（分辨率相关）** | 3090：294 帧干净 / 362 帧坏，阈值在两者之间；#15263 原表：≤154k token 干净、≥167k 坏 [R3-01；`sources/28`] |

**结论（用于改文档）**：**"噪声是 sm_120 专项"这句话是错的**——它在 sm_86 上也能复现。但把它改写成"就是 FP8 PV 的锅"同样过头。**能站住的表述是：SageAttention 在长序列（约 15 万 token 量级）+ 特定 kernel 路径下会出现"尾帧块状噪声且日志报成功"，与架构不严格绑定，与序列长度强相关；具体是 FP8-PV 还是 FP16-PV 取决于 sage 版本与架构分派，目前无定论。**

### H2｜"comfy kitchen attention 同形状无噪声且更快" → **部分核实（仅 sm_86 / RTX 3090 单次实测）**

同一张表里三个后端（同 prompt、同 seed、同 pair 内同功耗墙）：

| 后端 | 首个损坏帧 | s/step | 总时长 |
|---|---|---|---|
| SageAttention（KJNodes，auto） | 333 / 362 | 267（@200 W） | 1 h 33 min |
| pytorch attention | 无 | 420（@230 W） | 2 h 22 min |
| comfy kitchen attention | **无** | **188（@230 W）** | **1 h 07 min** |

- **"无噪声"**：该次运行成立；且 294 帧的 Sage 也干净，说明这不是"每张图都坏"的普遍故障。
- **"更快"**：188 vs 420（vs Sage 267）成立，幅度 **2.23× vs pytorch、1.42× vs Sage**。
- **但**：① n=1，无重复；② 三个后端**功耗墙不同**（Sage 那条是 200 W，另两条 230 W），不完全同条件；③ 3090 是 sm_86 + 24GB，**不是 5070 Laptop 8GB**；④ 该次 kitchen 用的是 `int8_convrot` 权重却**没有**触发 #15529 的对齐错——说明 #15529 不是"所有 int8_convrot 都必崩"，而是**形状/版本相关**（见 L3-04）。
- 等级：**B（引用片段可复核）/ C（对 5070 的外推）**。

### H3｜"int8 大分辨率×长序列的 illegal memory access 来自 comfy-kitchen int8 GEMM 的 int32 溢出（#136）" → **已核实（R4 直读 #136 本体；机制三级印证 + 一手 issue）**

> 【R4 修订 2026-11-08 JST】本小节原文"#136 本体未读到"已作废，替换为下方直读结果；原段落按追加方式保留在文末「修订记录 r1」说明中，不删除。

| 证据 | 内容 | 等级 |
|---|---|---|
| #15263 评论（2026-08-25） | "the hard illegal memory access on int8 weights at large resolution x length is a different bug — an int32 overflow in comfy-kitchen's int8 GEMM output indexing, reported as Comfy-Org/comfy-kitchen#136" | B（R4 已升级为**直读**） |
| 第三方代码注释（HF 公开 commit） | "comfy-kitchen's INT8 GEMM addresses its activation/accumulator buffers with 32-bit offsets … a 15 s clip packs ~92k rows, so `fc1` would produce a **2.6e9-element** intermediate … a 5 s clip (1.1e9) runs fine, a 15 s one (2.6e9) faults" | B（机制） |
| 第三方节点（H3 Row Chunk） | "past roughly **75k rows**, a projection's intermediate exceeds CUDA's int32 element limit" | C（未直读） |
| **#136 本体（R4 直读）** | **已核实：仍开着（state=open）**。标题 `INT8 GEMM kernels index the output in int32: illegal memory access once m*n >= 2**31 (MiniMax H3 at high resolution x length)`，作者 mjungnickel18，`created_at 2026-08-24T20:47:04Z`，`closed_at=null`、`comments=0`、`updated_at=created_at`（自创建起无更新、无评论）。**修复状态：未合并、无修复版本**；作者给出本地建议修复（两个 Triton kernel 的 `offs_am/offs_bn` 改 `tl.int64`），自述本地打补丁后 `m=200000, n=21504` 可跑通且数值正确（~1% 量化误差），但**未测吞吐代价**、**未提交/未合并**。环境：RTX 3090 24GB / 驱动 595.97 / Win11 / comfy-kitchen **0.2.31**（0.2.30 同文件逐字节相同）/ ComfyUI 0.32.0 / torch 2.10.0+cu128 / triton-windows 3.6.0.post26 / Python 3.12.11 | **B（直读，GitHub issue 页 + REST API 双证）** |

**结论（R4 改写）**：机制（int32 索引溢出 → illegal memory access）**已由 #136 一手 issue 直接坐实**，且给出精确门槛 **`m*n >= 2**31`（实测边界 2.147e9）**，与"分辨率×长度"而非单纯帧数/VRAM 相关；实测 1344×768 / 243 帧(10s, 251M) 正常、294 帧(12s, 303M) 与 362 帧(15s, 374M) 均 `crash after 43 s`，作者本地 int64 补丁后分别 410 s / 620 s 跑通。**修复状态 = 仍开着、无修复版本、无评论**——因此**不能写"已修复"**，也**不能写"必现"**（作者明确排除了 `--disable-dynamic-vram`、`--disable-pinned-memory`、0.2.30/0.2.31 差异与 w4a8/GGUF 权重变体；且该 issue 走 `--enable-triton-backend` 的 Triton INT8 GEMM 路径）。R3b 中 L3-08 提到的"release 自己的 kernel 改用 int64 → 可能已修"，在本轮直读的 #136 与 comfy-kitchen Releases 页上**均未得到证实**（Releases 页为 "There aren't any releases here"），维持 **D（修复状态未核实）**。摘录见 `sources/R3-136-*.md`。

---

## 二、条目（每条：来源链接｜日期｜原文测试条件｜报告效果｜画质代价｜对 5070 Laptop 适用性/不确定点｜等级）

### L3-01｜#15263 三后端对照（H1/H2 的原始出处）

- 来源：https://github.com/Comfy-Org/ComfyUI/issues/15263 ｜ 评论 2026-08-25（作者 mjungnickel18）｜ 片段读取
- 测试条件：RTX 3090（sm_86）/ H3 T2VA / `minimax_h3_fl2va_pruned_int8_convrot` / 1344×768 / 362 帧（~15 s）/ ComfyUI 0.32.0 / comfy-kitchen 0.2.31 / torch 2.10.0+cu128 / SageAttention 2.2.0 / `--enable-triton-backend`；同 prompt、同 seed、pair 内同功耗墙
- 报告效果：见上文表；另：**帧坐标固定**（steps=1 从 334 起，steps=20 从 333 起）→ 与步数无关、与序列长度有关；mp4 码率 2.97 Mbps 正常、日志报成功
- 画质代价：Sage 尾帧整段损坏；kitchen 与 pytorch 无损坏（该次）
- 对 5070 Laptop：**方向可用，数值不可搬**。同架构（Blackwell vs Ampere 不同）、同显存档完全不同（24GB vs 8GB）；8GB 卡上 kitchen 的显存账要重算
- 等级：**B**（片段可复核）；外推 5070 → **C**

### L3-02｜#15263 反向对照：Ampere 在可达序列内干净

- 来源：同上 ｜ 评论 2026-08-10（pepikir）｜ 片段读取
- 测试条件：RTX 3090（sm_86）/ sageattention 1.0.6 / ComfyUI 0.30.1 / int8_convrot + nvfp4_awq TE / 1280×736：124 f = 28,520 token（2026-08-11 复跑）、243 f = 56,120 token（2026-08-07）；另叠加 Turbo LoRA、sigma shift、Spectrum、EasyCache
- 报告效果：全部干净；并给出 H3 在 `MAX_PIXELS`(1344×768) × 362 帧下的**理论上限 91 × 42 × 24 = 91,728 token**，"roughly half your original ~185k wall"
- 画质代价：无
- 对 5070 Laptop：**这条最重要**——它说明"在 H3 自己的画布上限内，Ampere 根本到不了 167k 阈值"。**5070 Laptop 跑 736×416×124 时只有 ~1.2 万 token，跑 1344×768×362 也只有 ~9.2 万 token**，均低于首个噪声行 167k。→ **对目标机的默认参数，Sage 的长序列噪声风险很低**
- 等级：**B**（片段）

### L3-03｜第三方机制复盘：FP8-PV vs FP16-PV 的分派争议

- 来源：https://smeltcore.com/recipes/minimax-h3-on-rtx-3090-5-second-audio-video-in-comfyui-on-pruned-int8 ｜ 成文晚于 2026-08-25
- 测试条件：无自有实测，逐条复盘 #15263 与 SageAttention 上游
- 报告效果（逐字）："1.0.6 has no architecture dispatch and no FP8 kernel at all — it is a single Triton path doing INT8 QK, FP16 PV and FP32 accumulation — and on sm_80/sm_86 a 2.x build selects an FP16 PV kernel too"；"By the dispatch just described that run was on an FP16 PV kernel; its author files it under FP8 PV all the same"；并给出 token 计数法 caveat："ComfyUI's `video_latent_t()` gives 107 temporal positions for 362 frames where ceil(frames/4) gives 91, so absolutes move about 19%"
- 画质代价：—
- 对 5070 Laptop：说明**"FP8 PV 是根因"这一条在 3090 案例上站不住**；对 sm_120 仍成立。**不要用"FP8 PV"去解释所有平台**
- 等级：**C**（第三方分析，非实测）

### L3-04｜#15529：kitchen attention × int8_convrot → 对齐报错（**阻断性**）

- 来源：https://github.com/Comfy-Org/ComfyUI/issues/15529 ｜ 日期未读到（issue 编号属 2026-08 中旬）；转述见 localaimaster 汇总表（https://localaimaster.com/blog/comfyui-attention-backend-errors）、skypray 教程（https://skypray.synology.me/blog/computer/comfyui-minimax-h3-workflow）
- 测试条件：H3 `int8_convrot` checkpoint + `ModelAttentionBackend` 设 `comfy kitchen attention`，step 0/20
- 报告效果（逐字）：`RuntimeError: quant_qk_per_thread_int8: Q/K base pointers and B/H/N strides must preserve 4-element alignment`；"Not resolved in-thread; the reporter asks for a graceful failure instead"
- 画质代价：不涉及（直接崩）
- 对 5070 Laptop：**这是把 kitchen 排除出 V1.1 默认项的第一条理由**。目标机的官方模板权重就是 `int8_convrot`。
  **不确定点（必须写清）**：L3-01 里 3090 用**同样的 int8_convrot** 却跑通了 kitchen（0.2.31）→ 说明该限制**不是权重格式的绝对禁令**，而是**形状/版本相关**（4 元素对齐取决于 Q/K 的 base pointer 与 B/H/N stride，和分辨率、头数、序列长度都有关）。所以"int8_convrot 一定不能用 kitchen"这个说法**过强**，正确表述是"**在 8GB 目标机上，用官方模板的 int8_convrot + kitchen 有已知的对齐崩溃报告，必须先单步验证**"
- 等级：**B**（报错原文可复核）/ 触发条件 → **C**

### L3-05｜comfy-kitchen 的 Windows / sm_120 可用性（README + 后端矩阵）

- 来源：https://pypi.org/project/comfy-kitchen/0.1.0/ 、https://libraries.io/pypi/comfy-kitchen 、https://www.webkkk.net/0xDELUXA/comfy-kitchen_win-rocm ｜ 2026-10-02 前后（0.2.37 发布窗口；0.1.0 版为包首发期）｜ 片段读取
- 测试条件：包元数据，非实测
- 报告效果：
  - **有官方 Windows x64 CUDA wheel**（`cp312-abi3-win_amd64`，Stable ABI 覆盖 3.12/3.13/3.14+），**不需要本地编译**；
  - 默认 `--cuda-archs` 的 **Windows 列表含 `120f`** → sm_120 在编译目标里；
  - 依赖：Python ≥3.10、PyTorch ≥2.7.0（0.1.0 期写 ≥2.5.0）、**CUDA Runtime ≥13.0**、**驱动 r580+**；
  - 后端矩阵里 `int8_attention` 只有 **cuda / triton** 两列打勾 → **CUDA 后端即可，不强制 triton**；
  - NVFP4 计算需 **SM ≥ 10.0**（sm_120 满足）。
- 画质代价：—
- 对 5070 Laptop：**可用性上没有硬阻碍**（Win x64 wheel + sm120 + cu130 + driver 580+）。目标机驱动记录值 616.92 满足 r580+。不确定点：wheel 是 `abi3` 单包多架构，**实际二进制里是否真的编进了 sm_120 kernel，只能在本机 import + 跑一次才知道**（这正是 E1 要做的）
- 等级：**B**（包元数据可直接核对）

### L3-06｜comfy-kitchen 已知 bug 清单（与长序列/Windows/int8 直接相关）

- 来源：**https://github.com/Comfy-Org/comfy-kitchen/issues** （GitHub 同源官方 issues 列表，R4 直读）｜ 列表内 issue 日期 2026-09-15 ~ 2026-11-07 ｜ **直读（列表/标题级）**
  - 【R4 修订】原 `http://ghub.com/Comfy-Org/comfy-kitchen/issues` 为第三方镜像，**不可靠、不再作为 B/C 依据**；已换成 github.com 同源官方列表并直读核实。直读结果：列表中 #178、#196、#230、#226、#241、#243、#204、#190、#188 等编号/标题与 R3b 镜像片段一致（可交叉验证），但**部分 opened 日期与镜像记录有出入**（#226 镜像记 10-01、github 记 09-30；#196 镜像记 09-25、github 记 09-24；#230 两处均 10-01）——以 github.com 为准。**#136 未出现在本页首屏（默认按时间倒序，首屏最旧为 #178）**，但 REST API 证实其 state=open（见 H3 节），故"列表首屏无 #136"不等于"#136 不存在/已关"。
  - 镜像站 `ghub.com` 本轮**未再打开**，不再作为任何结论依据；若后续无法访问 github.com 时写「无法访问」，不得回退用 ghub 充当 B/C。
- 测试条件：各 issue 自述（本轮仅直读列表标题与日期，**未逐条读正文**）
- 报告效果（标题逐字）：

| # | 日期 | 标题要点 | 对目标机的相关性 |
|---|---|---|---|
| 169 | 09-10 | Triton `rms_rope` 在 `seq_len >= 65536` 超 gridDim.z(65535) | **高**：1344×768×362 ≈ 9.2 万 token > 6.5 万；若走 triton 后端会踩线（是否触发未核实） |
| 178 | 09-15 | `int8_attention` 超 **299,520 key rows** 抛 OverflowError（int32 strides） | 中：常规 124–362 帧未到，但 Ref2VA 多参考会成倍抬升 |
| 196 | 09-25 | INT8 ConvRot 在 **Windows/NVIDIA** 上 model re-entry 卡住：100% GPU 但 ~80 W、高显存、有时 `int8_linear` OOM | **高**：目标机正是 Windows + int8_convrot |
| 230 | 10-01 | kitchen INT8 attention 在 AMD gfx1100 + int8_convrot 上**纯噪声**（>150 token 就坏） | 中：跨厂商说明该 INT8 注意力路径有静默画质故障史 |
| 226 | 10-01 | AMD RDNA3 上 CK attention 出坏图（回归） | 低（AMD） |
| 169 | — | 见上 | — |

- 画质代价：见各行（#230 是"纯噪声 + 退出码成功"，最危险）
- 对 5070 Laptop：**长序列不是免费的**——#169 的 65,536 门槛与目标机 ~9.2 万 token 的实际规模**正面冲突**；#196 的 Windows 特性与目标机平台一致
- 等级：**C**（只读标题与日期，未读正文与修复状态）

### L3-07｜8GB 笔记本 Blackwell 上的三后端 microbench（最接近目标机形态）

- 来源：https://huggingface.co/datasets/FlowForgeLabAi/minimax-h3-8gb-bench/blob/main/README.md ｜ 无日期（环境 ComfyUI 0.35.0）｜ 片段读取
- 测试条件：**RTX 5060 Laptop 8GB + 15.26 GiB RAM + Windows + ComfyUI 0.35.0**；H3 真实注意力几何（56 heads × 128 head_dim，**S = 8192**，bf16）；单机单种子
- 报告效果：**SDPA 143.1 ms / comfy-kitchen INT8 22.8 ms / SageAttention 29.0 ms**（相对 SDPA：kitchen 6.28×，Sage 4.93×）；端到端 768×1024：5 s 片段 4/6/8 步 = 245/297/378 s；10 s 片段 8 步 858 s、20 步 2029–2613 s；pagefile 压力累积后同一图从 ~1.9 s/block 劣化到 **212 s/block**，表现为 **99–100% 利用率但只有 ~33 W**
- 画质代价：作者声明"quality judged by eye on a handful of clips; no reference metric"，未量化
- 对 5070 Laptop：**这是唯一"同架构 + 同显存档 + 同 OS"的三后端对照**，且结论与 3090 一致（kitchen 最快）。但：① S=8192 远小于 H3 实际的 2.7 万–9.2 万 token，**长序列未测**；② 单机单种子；③ 权重格式未说明；④ 内存只有 15.26 GiB，pagefile 结论不能直接搬到 32GB
- 等级：**C**（第一手但无第三方复现）

### L3-08｜int32 溢出的量级门槛（第三方代码）

- 来源：https://huggingface.co/stdstu123/LynnReal-Onmi-beta-0.1/commit/5435f137e2d44959de74feff29fc2d29df0f6557 ｜ 日期未读到 ｜ 片段读取
- 测试条件：第三方 H3 fused-block 补丁的防御性检查（非 benchmark）
- 报告效果（逐字）："a 15 s clip packs ~92k rows, so `fc1` would produce a **2.6e9**-element intermediate … a 5 s clip (1.1e9) runs fine on the stock path, a 15 s one (2.6e9) faults"；`_INT32_SAFE_ELEMENTS` 取 2^31 以下留余量；"the release's own kernels switch to int64 here"
- 画质代价：不涉及
- 对 5070 Laptop：**给出可执行的安全线**：5 s（约 1.1e9）安全、15 s（约 2.6e9）越界。目标机若坚持 15 s，**必须开分块**（RowChunk 类节点或 `MiniMaxChunkFeedForward`）。注意最后一句"release 自己的 kernel 在这里改用 int64" → **可能已修，未核实**
- 等级：**B**（机制，源码级）/ 修复状态 **D**

### L3-09｜#16150：只挂 kitchen 节点的 H3 端到端 + 版本回归

- 来源：https://github.com/Comfy-Org/ComfyUI/issues/16150 ｜ 2026-09（页面出现 Sep 23）｜ 片段读取
- 测试条件：MiniMax H3 工作流，仅加 `Model Attention Backend: Comfy Kitchen`；ComfyUI 0.34.2（k 0.2.31）vs 0.35.0（k 0.2.33）；显卡/分辨率未读到
- 报告效果：0.34.2 → 147 s / 137 s，VRAM ~74%；0.35.0 → **CRASH，黑屏、风扇满转、需手动重启**（VRAM 100%）；`--vram-headroom 1` 下 146/129 s、VRAM ~90%；`--disable-comfy-compiler` 154/139 s、~77%；edit2：两者同开才稳定
- 画质代价：不涉及
- 对 5070 Laptop：**版本锁死的直接证据**。同一条链路在相邻小版本上从"能用"变成"把机器打死"，而目标机只有 8GB——显存打满的后果比大卡严重得多（参见 DOSSIER 5.x / R2-13 的共享显存结论）
- 等级：**C**（单用户、缺 GPU 与分辨率）

### L3-10｜5070 Laptop 8GB 实机：能跑，但没有后端对照

- 来源：https://juejin.cn/post/7697112171168530441（RTX 5070 Laptop 8GB + 32GB DDR5 + NVMe，ComfyUI 0.40.0/cu130）；https://m.toutiao.com/article/7684936781942293011（5070 8GB 把 Sage patch 换成 kitchen）
- 日期：均未显示；前者环境为 0.40.0
- 测试条件：掘金——736×416 / 15 s / turbo 8 步 / `--lowvram --fp16-vae --disable-pinned-memory`；头条——8 s 为主，`MiniMaxH3MemoryEfficientSageAttentionPatch` 改为 `ModelAttentionBackend → comfy_kitchen`
- 报告效果：掘金——"8G 显存跑 H3 成立…单条全流程约 35 分钟"，"5 秒一条大约 8~10 分钟"，"cu128 上会静默回退，实测速度差 3 倍"；头条——"8s 最多 3 分钟，15s 测试基本 hung 住了"
- 画质代价：均未量化
- 对 5070 Laptop：**同规格实机存在性证据**（这是 DOSSIER 里"没有任何 5070 Laptop 实测"的最新反例，需在 R11 一并提示）。但对注意力后端**没有对照数字**，头条那条只证明"kitchen 能起来、不崩"，不能证明"更快"
- 等级：**C**（单作者、无对照、无日志）

### L3-11｜Spark-H3：Blackwell 上 kitchen 被列为推荐 dense 后端

- 来源：https://comfyui-wiki.com/en/news/2026-10-11-spark-h3 ｜ 2026-10-11 ｜ 片段读取
- 测试条件：ComfyUI 0.38.x/0.39.x、Python 3.12+、**CUDA 13.0、BF16**、Linux/Windows x86_64；SM120 = RTX 50 系 / RTX PRO 5000·6000 Blackwell；RTX PRO 6000 Blackwell、1344×768、14.4 s
- 报告效果：安装器按 ComfyUI 版本装 comfy-kitchen **0.2.36（0.38.x）/ 0.2.37（0.39.x）**；节点页称 Blackwell 上 `dense_backend=kitchen` "it's marked as the recommended one to try"，但 "Kitchen and Sage dense backends as experimental, not yet through full quality regression testing"；端到端 20 步 1018.5 s → 461.7 s（2.21×）/ 323.3 s（3.15×）
- 画质代价：PSNR/SSIM/LPIPS 表针对**稀疏**方案，非 kitchen；kitchen 本身"no full quality regression"
- 对 5070 Laptop：**"kitchen 在 sm_120 上被推荐"成立**，但① 推荐语境是"稀疏注意力的 dense 预热/回退后端"，不是"H3 端到端提速"；② 2.2×–3.15× **来自稀疏，不能记到 kitchen 头上**；③ 要求 BF16 权重 —— 与 8GB 卡的成本模型冲突
- 等级：**B**（发布页与版本矩阵可核对）；加速归因 **C**

### L3-12｜#15738：静默损坏不止 Sage 一家（只有元数据）

- 来源：https://github.com/Comfy-Org/ComfyUI/issues/15738 ｜ 00akii ｜ 2026-08-19 ｜ Open ｜ **仅标题**
- 测试条件：**未读到**
- 报告效果：标题称 "H3 Ref2VA: safe max frame length drops sharply as reference image/video count increases; corrupted output reports success in the log"
- 画质代价：—
- 对 5070 Laptop：只用于支撑"验证必须逐帧看尾段"这一流程要求，**不引用任何数字**
- 等级：**D**

---

## 三、Windows / sm_120 可用性小结

| 问题 | 结论 | 依据 | 等级 |
|---|---|---|---|
| Windows 有预编译轮子吗 | **有**（`cp312-abi3-win_amd64`，含 cuda 后端） | L3-05 | B |
| 需要自己编译吗 | **不需要**（除非要 cu128 或非主流 Python） | L3-05 | B |
| 需要 triton 吗 | **不强制**：CUDA 后端提供 `int8_attention`；但社区"三件套"把 `triton-windows` 列入，且 triton 路径有 #169 的 65,536 序列上限 | L3-05 / L3-06 / R3-14 | B/C |
| sm_120 在编译目标里吗 | **在**（Windows 默认 `...;80;89;120f`） | L3-05 | B |
| 运行前提 | CUDA Runtime ≥13.0、驱动 r580+、Py ≥3.10、torch ≥2.7.0 | L3-05 | B |
| 版本耦合 | **强耦合**：ComfyUI 精确 pin comfy-kitchen；0.35.0 + k0.2.33 曾黑屏重启 | L3-05 / L3-09 / `R3-07` | C |
| 与 `int8_convrot` 权重共存 | **有已知对齐崩溃报告**（#15529），但 3090 上同权重能跑 → 形状/版本相关 | L3-04 / L3-01 | B/C |

## 四、公开速度/显存对比数据汇总（只列本文件核实过的）

| 平台 | 形状 | SDPA / 原生 | Sage | kitchen | 来源 | 等级 |
|---|---|---|---|---|---|---|
| RTX 3090 24GB（sm_86） | 1344×768×362f，int8_convrot | 420 s/step @230W | 267 s/step @200W（**尾帧坏**） | **188 s/step @230W（无坏帧）** | L3-01 | B |
| RTX 5060 Laptop 8GB（sm_120） | S=8192，56×128，bf16 | 143.1 ms | 29.0 ms | **22.8 ms** | L3-07 | C |
| RTX 5070 12GB（桌面，sm_120） | 1344×768×124f，8 步 | 40.7 s/step attn（342 s 端到端） | 24.8 s/step attn（231 s 端到端，1.48×） | 未测 | R3-15 | D（软文） |
| RTX 5070 Laptop 8GB | 736×416×15 s，8 步 | 未测 | 未测 | 未测（仅"能跑"） | L3-10 | C |

**没有任何一条数据是"RTX 5070 Laptop 8GB 上 SDPA vs Sage vs kitchen 的端到端对照"**——这就是 V1.1 必须靠本机实验补的那一格。

## 五、结论：kitchen 能不能当 V1.1 提速候选

**结论：不进 V1.1 默认项；列为"有条件的一次性实验项"。**

支持（为什么值得一试）：
1. 两个独立平台上 kitchen 都是三者最快（3090：188 s/step；5060 Laptop S=8192：22.8 ms vs SDPA 143.1 ms）。
2. 同形状下 kitchen **没有** Sage 的尾帧噪声（L3-01，该次运行）；且 kitchen 在三者中最快（L3-01、L3-07）。
3. Windows wheel + sm_120 架构都在官方支持矩阵内（L3-05）；Blackwell 上被第三方项目设为推荐 dense 后端（L3-11）。

反对（为什么不能直接开）：
1. **权重格式冲突**：官方模板 `int8_convrot` 有对齐崩溃报告（#15529）；第三方教程转述官方文档称要用 kitchen 就得换 bf16（【C，R3-16】）—— 8GB 卡上 bf16 = 权重体积约翻倍，**搬运成本会吃掉注意力收益**。
2. **长序列 int32 门槛**：#136 / #178 / 第三方量级（15 s≈2.6e9 元素越界）说明 15 s 场景有硬风险，而 5070 Laptop 恰恰只能靠"短一点、少步数"换时间。
3. **版本脆弱**：0.35.0 + k0.2.33 黑屏重启（L3-09）；comfy-kitchen 与 ComfyUI 强耦合（R3-07）。
4. **无 5070 Laptop 数字**：唯一同规格实测（L3-10）没有后端对照，不能当证据。

**对 DOSSIER 的两条连带修正**（已写入 R11）：
- "噪声是 sm_120 专项" → 错，sm_86 也复现；
- "没有任何 5070 Laptop 的公开实测" → 已有（L3-10，736×416/15 s/8 步/约 35 min），但仍**没有后端对照**。

## 六、本机最小实验（3 项）

> 全部在 ComfyUI 层做，不装软件、不改驱动、不下载新权重（除实验 2 明确允许的一次性 bf16 对照，且需 Junlong 批准）。回滚方式统一为"删节点 / 改回下拉 / 重启"。

### E1｜三后端端到端对照（最高优先；直接补上"没有 5070 Laptop 数字"这一格）

- **改动**：在 `UNETLoader` 与采样器之间插入 ComfyUI 内置 `ModelAttentionBackend`，`attention` 分别设 `pytorch attention` / `comfy kitchen attention`（若下拉里没有 kitchen，说明 kernel 未加载，**直接记"不可用"并结束**）。Sage 用 KJNodes 的 patch（`auto`）。
- **固定参数**：官方 `int8_convrot` 权重、736×416、124 帧、Turbo 8 步、同 seed、同 prompt。
- **观测指标**：① 每步秒数（采样器日志时间差 ÷ 步数）；② 峰值 dedicated VRAM（任务管理器）；③ 峰值进程 RAM 与 commit；④ **尾帧质量**：导出最后一帧，算 lag-8 邻域相关（好帧 0.87–0.99，坏帧 0.14–0.28，见 L3-01）；⑤ 控制台是否出现对齐错。
- **停止条件**：
  - kitchen 一开就报 `quant_qk_per_thread_int8 … 4-element alignment` → 记"#15529 在本机复现"，**立即结束 kitchen 分支**；
  - 三种后端步时差 <5% → 判"无提速价值"，不再投入；
  - 出现 illegal memory access / 黑屏 / 重启 → 立即停，锁 ≤124 帧。
- **回滚**：删除 `ModelAttentionBackend` 节点，恢复 `UNETLoader` 直连采样器；重启 ComfyUI（切换后端后显存不自动释放，见 R3-10）。

### E2｜`int8_convrot` vs bf16 在 kitchen 下是否真的不能共存

- **改动**：保持 E1 的 kitchen 设置，只把 `UNETLoader` 换成 bf16 版 H3 权重（**需 Junlong 批准**，且需 ~40GB 磁盘）；先只跑 **1 步** + 解码 1 帧，不跑整片。
- **观测指标**：能否过 step 0；显存峰值；单步耗时 vs int8 版同参数。
- **停止条件**：bf16 加载即 OOM 或单步 >int8 版的 2× → 判"8GB 上不可行"，结束；成功则记录"kitchen 可用但代价 X"。
- **回滚**：把 `UNETLoader` 换回 `int8_convrot`；bf16 文件保留不动（不删用户文件）。

### E3｜长片段的安全边界（验证 int32 溢出与 RowChunk 必要性）

- **改动**：固定 416×736、Turbo 8 步、pytorch attention（先排除注意力后端的干扰），帧数按 **124 → 243 → 362** 递增；每次在 `UNETLoader` 后挂 `MiniMaxChunkFeedForward`（或社区 RowChunk 类节点）与不挂各跑一次。
- **观测指标**：是否出现 `CUDA error: an illegal memory access`；峰值 VRAM；单步耗时；尾帧相关。
- **停止条件**：任一配置出现 illegal memory access → 记下帧数并停，往回退一档；若挂 chunk 后通过但步时 +>30% → 记为"能跑但不能用"。
- **回滚**：删掉 chunk 节点、帧数改回 124、重启。

## 七、链接核实表（本轮）

| # | URL | 读取方式 | 状态 | 等级 |
|---|---|---|---|---|
| 1 | github.com/Comfy-Org/ComfyUI/issues/15263 | **R4 直读（WebFetch）** + API 状态核对 | **已直读全文**：Edit 2026-08-09「0.31.1 上 KJNodes `auto` 可用、1920×1088/15s sage on 54min vs off 94min」、token 阈值表（≤154k 干净 / ≥167k 噪声）、三后端对照；state=**open**，15 评论，created 2026-08-03，updated 2026-08-25。原「片段读取」→ **直读** | **B（直读）** |
| 2 | github.com/Comfy-Org/comfy-kitchen/issues/136 | **R4 直读（WebFetch 页面 + api.github.com REST）** | **已核实 / 仍开着（open）**：created 2026-08-24、closed_at=null、comments=0、无修复版本；建议修复=int64 偏移（仅作者本地验证，未合并）；环境 ck 0.2.31 / ComfyUI 0.32.0 / 3090 24GB / Win11。原「未读到正文」→ **已读全文**；摘录 `sources/R3-136-issue136-int32.md` | **B（直读）** |
| 3 | github.com/Comfy-Org/ComfyUI/issues/15738 | **R4 直读（WebFetch 页面 + API）** | **正文已直读**（原「仅标题」作废）：RTX 4090 24GB / Win11 Pro / 128GB RAM / ComfyUI v0.33.1 / torch 2.13.0+cu130 / `minimax_h3_ref2va_pruned_fp8_scaled` + `minimax_h3_ref2v_turbo_4step` LoRA、**6 步** / 1344×768 / 参考视频 608×352 97 帧；安全帧长随参考数下降（311→294→277）、损坏输出日志仍报成功、码率 24–47 Mbps vs 正常 8–10 Mbps；state=**open**，0 评论，created 2026-08-19 | **C**（直读但单用户 issue、无第三方复现；原 D 升为 C，不写"已修"） |
| 4 | github.com/Comfy-Org/ComfyUI/issues/15529 | **R4 直读（WebFetch 页面 + API）** | **原页已直读**：报错 `quant_qk_per_thread_int8: Q/K base pointers and B/H/N strides must preserve 4-element alignment`（step 0/20 崩，来自 `comfy_kitchen` 的 `sage_sdpa_quantize`）；复现步骤=官方 T2V 模板 + `ModelAttentionBackend=comfy kitchen attention` + `int8_convrot` 双权重；标注 "Debug Logs: fixed"（含义不明确，非关闭）；state=**open**，1 评论，created 2026-08-12，updated 2026-10-17；关联 duplicate #15494。**无明确修复版本**。原「原页未读」→ **直读** | **B（报错原文直读）** / 触发条件与修复状态 **C** |
| 5 | github.com/Comfy-Org/ComfyUI/issues/16150 | 搜索索引正文（未重开） | 片段已读 | C |
| 6 | **https://github.com/Comfy-Org/comfy-kitchen/issues**（官方同源，替换 ghub 镜像） | **R4 直读** | 官方列表标题/日期已直读，编号与镜像一致但部分日期有出入；**#136 不在首屏**（API 另证 open）。镜像 `ghub.com` 本轮未打开、不再作依据 | **B（列表直读）/ 正文未读** |
| 7 | github.com/Comfy-Org/comfy-kitchen/issues/230 | 搜索索引正文（未重开） | 正文片段已读 | C |
| 8 | **https://github.com/Comfy-Org/comfy-kitchen**（官方 README，R4 直读）· https://github.com/Comfy-Org/comfy-kitchen/releases（**直读：「There aren't any releases here」**）· pypi.org / libraries.io / webkkk（R3b 片段，未重开） | **R4 直读 README + Releases 页** | README 直读确认：Win x64 CUDA wheel、`--cuda-archs` Windows 默认含 `120f`、Python≥3.10 / torch≥2.7.0 / CUDA Runtime≥13.0 / 驱动 r580+、`int8_attention` 仅 cuda/triton 两列打勾、NVFP4 SM≥10.0；**Releases 页无任何 release 条目**（版本走 PyPI，故"0.2.37/0.2.38 修复"不能从 GitHub Releases 核实）。原「片段读取」→ **直读（README）** | **B（README 直读）**；Releases 无条目 = 事实 B／**修复版本仍 D** |
| 9 | mail-archive openSUSE changelog（0.2.37）· github.com/Comfy-Org/ComfyUI/pull/17107（0.2.38） | 同上 | 已读 | B |
| 10 | huggingface.co/datasets/FlowForgeLabAi/minimax-h3-8gb-bench | 同上 | README 已读 | C |
| 11 | juejin.cn/post/7697112171168530441 | 同上 | 开头与命令段已读 | C |
| 12 | m.toutiao.com/article/7684936781942293011 | 同上 | 片段已读 | C |
| 13 | smeltcore.com（3090 / 4060Ti 两篇） | 同上 | 片段已读 | C |
| 14 | comfyui-wiki.com/en/news/2026-10-11-spark-h3 · comfy.icu/node/MiniMaxH3SparkAttentionSM120 | 同上 | 已读 | B |
| 15 | huggingface.co/stdstu123/.../commit/5435f13 | 同上 | diff 片段已读 | B（机制） |
| 16 | 163.com/dy/article/L8AT2A240556I5F3.html | 同上 | 装环境段已读 | C |
| 17 | toutiao.com/a7703127136009388544（Veda 软文，5070 12GB） | 同上 | 表格已读 | D |
| 18 | skypray.synology.me/blog/computer/comfyui-minimax-h3-workflow | 同上 | 已读 | C |

> 注：Civitai / Reddit 类页面本轮未被搜索命中，未打开，不作结论。

---

*本文无 A 级结论。所有数字均可在上表 URL 复核；页面正文由搜索引擎代抓，我本人未直连打开——这一限制已在第 0 节写明，请勿把本文当"已逐页直读"的证据。*

---

## 八、修订记录 r1（R4 审核，2026-11-08 JST）

> 方式：**只追加 / 局部改写**，不删除任何原文段落。R3b 时 WebFetch 被禁，全部为搜索片段；R4 本轮 WebFetch 恢复可用，对关键页**逐一直连重读**。凡本表与正文原标注冲突，**以本表为准**。所有摘录存 `./sources/R3-136-*.md`（URL、读取时间 JST、逐字摘录、事实/推测/建议区分）。

### r1.1 改了什么（逐条）

| # | 位置 | 原文（R3b） | 改后（R4 直读） | 依据 |
|---|---|---|---|---|
| 1 | H3 小节标题 + 表 #136 行 | 「#136 本体未读到」「未读到正文」 | **已核实 / 仍开着（state=open）**；`created_at 2026-08-24T20:47:04Z`、`closed_at=null`、`comments=0`、`updated_at=created_at`；标题逐字：`INT8 GEMM kernels index the output in int32: illegal memory access once m*n >= 2**31 (MiniMax H3 at high resolution x length)`；作者 mjungnickel18。**修复状态：未合并、无修复版本**；建议修复=两 Triton kernel 偏移改 `tl.int64`（仅作者本地验证 `m=200000,n=21504` 通过，未测吞吐、未提交）。环境 ck **0.2.31**（0.2.30 同文件）/ ComfyUI 0.32.0 / 3090 24GB / Win11 / torch 2.10.0+cu128 / triton-windows 3.6.0.post26 | `sources/R3-136-issue136-int32.md`（WebFetch issue 页 + `api.github.com/repos/Comfy-Org/comfy-kitchen/issues/136`） |
| 2 | H3「结论」段 | 「修复状态未知…不能写已修复」 | **改写**：机制已由一手 issue 坐实，门槛精确为 `m*n>=2**31`（实测 2.147e9）；实测 1344×768 243f(10s) ok、294f(12s)/362f(15s) 均 `crash after 43 s`，本地 int64 补丁后 410 s / 620 s 跑通；**仍不能写"已修复"**（open、0 评论、无版本），也**不能写"必现"**（作者排除了 dynamic-vram/pinned/0.2.30-31 差异/w4a8/GGUF，且走 `--enable-triton-backend`）。L3-08「release 内核已改 int64 → 可能已修」在 #136 与 Releases 页**均未证实**，维持 D | 同上 + `sources/R3-136-readme-releases-issues.md` |
| 3 | 第七节链接核实表第 2 行 | 「未读到正文（列表首屏无 #136）｜ —」 | 「**已核实 / 仍开着**｜R4 直读页面 + REST API 双证｜**B（直读）**」 | 同上 |
| 4 | L3-06 来源行 | `http://ghub.com/Comfy-Org/comfy-kitchen/issues`（第三方镜像） | 换成 **https://github.com/Comfy-Org/comfy-kitchen/issues**（同源官方，R4 直读）。**镜像 ghub.com 标记为不可靠、不再作为 B/C 依据**；本轮未再打开该镜像。直读确认 #178/#196/#230/#226/#241/#243/#204/#190/#188 编号标题与镜像一致，但部分 opened 日期有出入（#226 09-30、#196 09-24，镜像记 10-01/09-25），以 github 为准；#136 不在首屏但 API 证实 open | `sources/R3-136-readme-releases-issues.md` |
| 5 | L3-01 / L3-02（#15263） | 「片段读取」「B（片段）」 | **直读**；升级 **B（直读）**。逐字复核 Edit 2026-08-09「0.31.1 + KJNodes `auto` 可用，1920×1088/15s sage on 54min vs off 94min」、token 表 ≤154k 干净 / ≥167k 噪声、原「一行修复」已被作者撤回、FP8-PV 根因在正文被标 "Superseded"。API：state=**open**，15 评论，updated 2026-08-25 | `sources/R3-136-issue15263.md` |
| 6 | L3-04（#15529） | 「报错原文经三处转述一致；原页未读」「B/C」 | **原页已直读**：逐字 `RuntimeError: quant_qk_per_thread_int8: Q/K base pointers and B/H/N strides must preserve 4-element alignment`（step 0/20，`comfy_kitchen` 的 `sage_sdpa_quantize`）；复现=官方 T2V 模板 + `ModelAttentionBackend=comfy kitchen attention` + `int8_convrot` 双权重；关联 duplicate #15494；API state=**open**、1 评论、created 2026-08-12、updated 2026-10-17。**无明确修复版本**；"Debug Logs: fixed" 含义不明，不解读为已关。等级 **B（报错原文直读）/ 触发条件与修复状态 C** | `sources/R3-136-issue15529.md` |
| 7 | L3-05（README / 包元数据） | 「片段读取」 | **直读**（github.com/Comfy-Org/comfy-kitchen README）：Win x64 CUDA wheel、Windows 默认 `--cuda-archs` 含 `120f`、Python≥3.10 / torch≥2.7.0 / CUDA Runtime≥13.0 / 驱动 r580+、`int8_attention` 仅 cuda/triton 打勾、NVFP4 SM≥10.0 —— 与 R3b 片段一致。等级 **B（README 直读）** | `sources/R3-136-readme-releases-issues.md` |
| 8 | comfy-kitchen **Releases** | （R3b 只经 openSUSE changelog / PR #17107 二手转述版本节奏） | **直读 https://github.com/Comfy-Org/comfy-kitchen/releases → "There aren't any releases here"**（无任何 release 条目，版本走 PyPI）。因此 0.2.37/0.2.38 是否修了 int32 溢出**不能从 Releases 页核实**；修复版本仍 **D** | 同上 |
| 9 | L3-12（#15738） | 「**仅标题**」「D」「不引用任何数字」 | **正文已直读**（原标注作废）：RTX 4090 24GB / Win11 Pro 26200 / 128GB RAM / Ryzen 9 7950X3D / Python 3.12.11 / torch 2.13.0+cu130 / ComfyUI v0.33.1 / `minimax_h3_ref2va_pruned_fp8_scaled` + `minimax_h3_ref2v_turbo_4step` LoRA **6 步** euler-normal / 1344×768 / 参考视频 608×352 97 帧；安全帧长 311→294→277（参考数 1→2→3），越界=全帧 RGB 噪声，码率 24–47 Mbps vs 正常 8–10 Mbps，**日志仍报成功**；362 帧 `CUDA error: illegal memory access`（重启复现 3 次）。API state=**open**、0 评论、created 2026-08-19。等级 **C**（直读但单用户 issue、无第三方复现；原 D → C，因正文数字现可复核） | `sources/R3-136-issue15738.md` |
| 10 | 第七节表第 1/3/4/6/8 行、§0 方法说明 | 「WebFetch 被禁 / 搜索索引正文 / 镜像」 | 逐行改为**直读 + 具体状态**；§0 追加「R4 复核说明」声明 R3b 限制是历史状态、本轮已直读、冲突以 r1/核实表为准 | 本表 |

### r1.2 未改 / 保留原标注的部分

- L3-03（smeltcore 第三方复盘）、L3-07（FlowForgeLabAi 8GB bench）、L3-08（HF commit）、L3-09（#16150）、L3-10（掘金/头条）、L3-11（Spark-H3）、第七节表第 5/7/9–18 行：**本轮未重开，保留原「片段读取 / 仅标题 / 转述」标注与等级，未编造**。
- `pypi.org` / `libraries.io` / `webkkk` / `mail-archive` / `github.com/Comfy-Org/ComfyUI/pull/17107`：未重开，仍为 R3b 片段。
- **没有任何页面被连续 3 次打不开而记为「无法访问」**；本轮全部目标页均一次直读成功（仅 https://github.com/Comfy-Org/comfy-kitchen/issues 列表不含 #136，属分页而非失效）。
- H1/H2 的判定与第五节「kitchen 不进 V1.1 默认项」结论**未因 r1 改变**：#136 仍 open、#15529 仍 open，反而强化"长序列/权重格式有已知风险、需本机单步验证"的原有结论。

### r1.3 事实 / 推测 / 建议（本轮新增）

- **事实（B，直读）**：#136 state=open、0 评论、无 closed_at、建议修复为 int64 偏移、门槛 `m*n>=2**31`；#15263 state=open 且 0.31.1 报告可用；#15529 state=open 且报错原文如上；#15738 state=open 且正文含完整环境/复现表；comfy-kitchen README 的 Windows/sm_120/依赖矩阵；comfy-kitchen Releases 页无 release 条目。
- **推测（C/D）**：「0.2.37/0.2.38 已修 int32 溢出」——**未证实（D）**；「#15529 的 'fixed' 表示已解决」——**不采信**；把 3090 上的结论外推到 5070 Laptop 8GB——**C**。
- **建议**：V1.1 若走 int8_convrot + `--enable-triton-backend`，15 s / 大分辨率组合需先按 `m*n<2**31` 估安全线或挂 RowChunk 类分块；kitchen attention 仍需先单步验证 4 元素对齐（#15529 未关）；任何"已修复"表述须等 #136/#15529 出现明确合并 commit 后再写。
