# MiniMax H3 本地部署技术档案

**目标机器**：HP OMEN 16，RTX 5070 Laptop 8GB（Blackwell，sm_120，8151 MiB），i9-14900HX，32GB RAM，Win11 家庭版 26200
**驱动记录值**：616.92（来自 PC 回报，本任务未复核，【D】）
**编写日期**：2026-11-04（T3 合并版；由 `DOSSIER-part1.md` 第 1–5 章与 `DOSSIER-part2.md` 第 6–9 章合并，并按审核意见 R1–R10 修正）
**证据等级**：A=本机已复现（本任务不可能出现 A）；B=外部可复现（公开源码/官方文档 + 完整方法 + 第三方复现记录）；C=研究候选（有真实代码或论文，无直接复现）；D=未经证实
**引用约定**：每条事实后的【】为等级，括号内为 `./sources/` 留底编号；链接总表见第 11 章

---

## 目录

- 执行摘要
- 第 1 章 模型与许可证
- 第 2 章 权重变体总表
- 第 3 章 运行时
- 第 4 章 8GB 社区实测汇总
- 第 5 章 已知坑与修复
- 第 6 章 推荐 V1.0 基线配置
- 第 7 章 Windows 执行单草稿（草稿·未执行·需 Junlong 批准）
- 第 8 章 V1.1–V2.0 候选技术
- 第 9 章 风险与待决事项
- 第 10 章 审核修正记录（R1–R10）
- 第 11 章 合并后总链接核实表（去重）
- 附录 A 证据强度自评
- 附录 B 未完成 / 未核实项

---

## 执行摘要

1. **本机尚无 A 级验证，全文所有结论均为 B/C/D 级**；第 7 章执行单是草稿，未在任何机器上执行。
2. H3 是三段式系统，只有中间段 H3-Base（FL2VA / Ref2VA）开源；H3-Context-IR 与 H3-Regenerate-2K 仅 API【B，01/02】。
3. 许可证排除 EU / UK / KR / US；中国、日本不在排除列表，可直接使用与商用；年营收 >2000 万美元需书面授权；商用 UI 须标 “MiniMax H3”【B，04】。
4. ComfyUI 官方教程另称“本地输出商用需经 Comfy 购买许可”，与许可证正文口径不一致，合规按更严格一方处理【B，04/05】。
5. 480p/5 秒在 8GB 卡上是 B 级事实：5 张不同 8GB 卡、5 个作者、7 组配置，耗时 80 秒–9 分钟【B，19/20/21/22/24】。
6. **没有任何 5070 Laptop（sm_120、8GB）的公开实测**；本档案对它的所有性能预期都是 C/D 级外推【C/D】。
7. 成功案例的共同前提是三件套：pruned int8 / W4A8 量化 + Turbo LoRA 压到 4–8 步 + torch cu130；缺一件即退化到 OOM 或数十分钟【B，19/20】。
8. SageAttention 的噪声是 **sm_120 上 >约 167k token** 的 FP8 PV kernel 问题，154k 以下干净；0.31.1 起报告可用，但只能设 `auto`——**这条直接适用于 5070 Laptop**【B，28】。
9. ComfyUI 0.30.0 在 8GB 上不可用（VAE 部分卸载设备不一致，#15251）；推荐区间 **0.31.1–0.32.x**，等级 C（由 issue 时间线反推，无官方兼容矩阵）【B/C，29/28/25/27】。
10. 8GB 上最危险的坑不是 OOM，而是 Windows + VBS 下 CUDA 系统内存回退触发 IOMMU 蓝屏（8 次先例，8151 MiB 机型与目标机完全同规格）【B，22】。
11. 最低磁盘组合约 39.55–43 GB（FL2VA + NVFP4 TE + 双 VAE + LoRA）；加 Ref2VA 再 +19.5 GB【B，24/19】。
12. 内存压力不可忽视：3070 Ti（16GB RAM）cold 进程 RAM 已达 **9.74 GiB**，且 TE 为 32B 级；32GB 是否够用属外推【B（实测）/ C（外推），20】。
13. 第 7 章已移除“强删 .venv”“更新驱动”两类动作，改为重命名备份 + 只读检查 + 报告 Junlong【按 R2/R3/R4 修正，D（流程）】。
14. 本档案不含任何密钥/令牌，不指导修改系统配置；所有需改动系统的动作单列为“需 Junlong 批准”【D（流程）】。

> 全文无“本机验证成功”一类表述；凡未打开或无法核实的条目，均标“未核实/无法访问”，不做推算填补。

---

## 第 1 章 模型与许可证

### 1.1 系统构成

H3 不是单一模型，而是三段式系统【B，01/02】：

| 模块 | 作用 | 是否开源 |
|---|---|---|
| H3-Context-IR | 解析自由形式多模态指令，序列化成 Context IR | **否**，仅 API |
| H3-Base | 由 IR 生成 768p 视频 + 立体声 | **是**（FL2VA、Ref2VA 两个 checkpoint） |
| H3-Regenerate-2K | 用原上下文把 768p 重生成到 2K | **否**，仅 API |

官方原文：“Because H3-Context-IR relies on a multi-stage workflow and multiple hosted models and services, it is not included in this open-source release.”【B，01】“Due to the complexity of the system, this module is not yet open-sourced.”【B，01】

推论（重要）：本地部署**永远得不到**官方产品的完整链路。本地只有 768p 短边、4–15 秒、无 2K、无 IR 提示词重写。任何“本地等于官网效果”的说法都是错的【B，01/06】。

### 1.2 架构与参数

- **H3-Omni-Transformer**：33B 稠密、单流（single-stream）Transformer；“approximately 13B parameters residing in AdaLN-related branches”【B，01】。这 13B 在推理时可预计算/缓存，“do not need to be loaded”【B，01】，因此 Comfy-Org 的 “pruned” 变体把参数量降到约 20B【B，12】。
- **打包序列**：`[text | cond/ref blocks | audio | video]` 一条扁平序列，视频与音频联合去噪【B，30】。
- **位置编码**：三维 MM-RoPE，作用在 (t, h, w)【B，01】。
- **注意力**：首发**只有 full attention**；“The sparse-attention implementation is not included in the initial open-source release and will be published separately in a future update.”【B，01/02】
- **视觉 VAE**：f16t4d24（空间 16×、时间 4×、24 latent 通道），latent 再按 1×2×2 patch 化，等效空间下采样 32×，时间仍 4×【B，01/02】。解码器是 ViT3D，内部做空间 tiling + 时间分块【B，30】。
- **音频 VAE**：左右声道共用编解码器、各自独立处理；32 kHz → 40 Hz latent token；DAC 系编码器 + BigVGAN 解码器，每 latent 帧 800 样本【B，01/30】。
- **文本编码器**：Qwen3-VL-32B 全量权重，取**第 50 层** hidden states，非 chat template，带 `<Picture i>/<Video k>/<Audio j>` 标签与 2 fps 时间戳视频块【B，01/30】。许可页脚注：Qwen3-VL-32B 本身是 Apache 2.0【B，04】。

### 1.3 输出规格

| 项 | 规格 | 来源 |
|---|---|---|
| 时长 | 4–15 秒 | 【B，01/02】 |
| 画幅 | 21:9、16:9、4:3、1:1、3:4、9:16 等 | 【B，01】 |
| 分辨率 | 短边默认 768 px；2K 需 H3-Regenerate-2K | 【B，01/02】 |
| 帧率 | 24 FPS | 【B，01/02】 |
| 音频 | 32 kHz 立体声，与视频同一遍生成 | 【B，01/02】 |
| 对白语言 | 11 种：阿、中、英、法、德、意、日、韩、葡、俄、西 | 【B，01】 |
| 帧数网格 | 17k+5（5, 22, … 124, … 209, … 362） | 【B，22/35】 |

### 1.4 未开源部分（汇总）

1. H3-Context-IR（仅 API）【B，01】；
2. H3-Regenerate-2K（仅 API）【B，01】；
3. 稀疏注意力实现（承诺未来发布）【B，01/02】；
4. 官方托管侧的安全护栏与提示词增强（模型卡称不影响许可证义务）【B，01】。

### 1.5 许可证：地域与商用

许可证名：**MiniMax H3 Community License Agreement**，协议日期 2026-08-02，许可方 Nanonoble Pte. Ltd.【B，04】

关键条款（逐字）：
- 排除地域：“‘Excluded Territories’ means the European Union, the United Kingdom, the Republic of Korea and the United States of America.”【B，04】
- 适用地域：“‘Applicable Territory’ means worldwide, excluding the Excluded Territories.”【B，04】
- 商用门槛：“…if your commercial products and services generate more than 20 million US dollars (or equivalent in other currencies) in yearly revenue.” 超过需书面授权（api@minimax.io）【B，04】
- 商用 UI 必须显著展示 “MiniMax H3”【B，04】
- 输出归属：“MiniMax claims no rights over the Outputs you generate.”【B，04】
- 禁止用 H3 或其输出改进其它 AI 模型【B，04】
- 禁止在适用地域外使用/复制/修改/分发/展示【B，04】
- 分发需附协议副本、标注修改、附 NOTICE 文本【B，04】
- 管辖：香港特别行政区法律与法院【B，04】
- Exhibit A 禁止军事用途、绕过护栏、未披露的机器生成内容、高风险自动化决策等【B，04】

**对中国用户的含义**：中国不在排除列表，属于 Applicable Territory，可直接依本协议使用与商用；≤2000 万美元年营收无需额外授权，但商用界面须标 “MiniMax H3”；争议受香港法院管辖；军事用途被明令禁止【B，04】。

**对日本用户的含义**：同上，日本不在排除列表，可直接使用；日文是官方稳定支持的 11 种对白语言之一【B，01/04】。

**必须点出的矛盾（不替用户下结论）**：ComfyUI 官方教程写 “Commercial use of locally generated outputs requires a MiniMax commercial license, available through Comfy, the only official reseller.”【B，05】而许可证正文只对“年营收 >2000 万美元”要求书面授权，并未要求所有商用都向 Comfy 购买【B，04】。两者口径不一致；合规上应按更严格的一方（Comfy 渠道）处理，或直接向 MiniMax 书面确认。这是本档案里唯一一处“两个官方来源互相冲突”的地方【B，04/05】。

---

## 第 2 章 权重变体总表

### 2.1 官方与 Comfy-Org（safetensors）

大小均来自 ComfyUI Wiki 原文表【B，08】；文件名为 Comfy-Org 仓库实际清单【B，07】。Comfy-Org 页面本身**不给大小**。

| 仓库 | 文件名 | 大小 | 精度 | 适配 GPU | 加载方式 |
|---|---|---|---|---|---|
| Comfy-Org/MiniMax-H3 | `minimax_h3_fl2va_bf16.safetensors` | 61.7 GB | BF16 | 任意，需大显存/强卸载 | ComfyUI `Load Diffusion Model` |
| 同上 | `minimax_h3_fl2va_int8_convrot.safetensors` | 31.7 GB | INT8 ConvRot | sm_86+，**cu130 才有快速 kernel** | 同上 |
| 同上 | `minimax_h3_fl2va_pruned_int8_convrot.safetensors` | 19.5 GB | INT8 + 剪枝 | 同上；8GB 社区首选 | 同上 |
| 同上 | `minimax_h3_fl2va_pruned_bf16.safetensors` | 原文未给 | BF16 + 剪枝 | — | 同上 |
| 同上 | `minimax_h3_fl2va_pruned_w6a8.safetensors` | 原文未给 | W6A8 | — | 同上 |
| 同上 | `minimax_h3_fl2va_pruned_fp8_scaled.safetensors` | 原文未给 | FP8 scaled | 不能跑 int8 时的后备 | 同上 |
| 同上 | `minimax_h3_ref2va_*`（同 6 种） | 19.5 GB（pruned int8）等 | 同上 | 同上 | 同上 |
| 同上 | `qwen3vl_32b_minimax_h3_bf16.safetensors` | 48.0 GB | BF16 | — | `Load CLIP`（H3 专用） |
| 同上 | `qwen3vl_32b_minimax_h3_int8_convrot.safetensors` | 25.3 GB | INT8 | 需 cu130 | 同上 |
| 同上 | `qwen3vl_32b_minimax_h3_nvfp4_awq.safetensors` | 14.6 GB | NVFP4 AWQ | **不要求 Blackwell**（原文） | 同上 |
| 同上 | `minimax_h3_video_vae_fp16.safetensors` | 4.9 GB | FP16 | 任意 | `Load VAE` |
| 同上 | `minimax_h3_video_vae_int8_convrot.safetensors` | 原文未给 | INT8 | 需 cu130 | 同上 |
| 同上 | `minimax_h3_audio_vae_fp32.safetensors` | 0.6 GB | FP32 | 任意 | `Load VAE`（音频口） |
| MiniMaxAI/MiniMax-H3 | `FL2VA/*`、`Ref2VA/*`（diffusers 目录） | 页面未给 | BF16 / FP32 混合 | 多卡或强卸载 | `ModularPipeline.from_pretrained`【B，03】 |

官方说明：pruned INT8 比标准 INT8 小约 40%，靠预计算 adaLN 曲线表；NVFP4 AWQ 文本编码器任何 GPU 都能跑【B，08】。Comfy-Org 明确建议：“For diffusion models prefer `int8_convrot` if you are able to use pytorch with cu130.”“`fp8_scaled` should only be used if you cannot use `int8_convrot`.”【B，07】

### 2.2 GGUF

| 仓库 | 代表文件 | 大小 | 精度 | 适配 | 加载 |
|---|---|---|---|---|---|
| unsloth/MiniMax-H3-GGUF | `minimax_h3_fl2va_pruned-Q2_K.gguf` | 6.26 GiB | Q2_K | 极限低显存，质量存疑 | sd.cpp 或 ComfyUI-GGUF（需补丁） |
| 同上 | `…-UD-Q2_K_XL.gguf` | 7.51 GiB | 动态 Q2 | sd.cpp 示例默认 | sd.cpp |
| 同上 | `…-Q3_K.gguf` / `UD-Q3_K_XL` | 8.16 / 8.90 GiB | Q3 | 8–12GB | 同上 |
| 同上 | `…-Q4_K.gguf` | 10.64 GiB | Q4 | 12GB+ | 同上 |
| 同上 | `…-Q8_0.gguf` | 19.97 GiB | Q8 | 24GB | 同上 |
| 同上 | `qwen3vl_32b_minimax_h3-Q2_K_M.gguf` | 12.20 GiB | Q2_K_M | 配最小 DiT | `--llm` |
| 同上 | `qwen3vl_32b_minimax_h3-Q4_K_M.gguf` | 16.97 GiB | Q4_K_M | 其余组合 | `--llm` |
| joeygambino/MiniMax-H3-GGUF | fl2va Q4_0 / Q5_1 | 19.9 / 25.9 GB | Q4_0 / Q5_1 | 16GB / 24–32GB | ComfyUI-GGUF + 架构补丁 |
| joeygambino（curve 变体） | pruned Q8_0 | 21.5 GB | Q8_0 | 24GB | 需 ComfyUI 0.30.0+【B，10】 |

GGUF 的关键限制：
- joeygambino 的文件**没有 K-quant**，原因是 “H3's hidden width is 2688, and K-quants require weight rows divisible by 256 (2688 % 256 = 128)”【B，10】。
- 原版 city96/ComfyUI-GGUF（最后更新 2026-01）不支持 `minimax_h3` 架构，会抛 “Unexpected architecture type in GGUF file: 'minimax_h3'”【B，40/10】。需要 ComfyUI-H3-Multishot v1.5.2+ 在运行时把架构名注入，或用其 `apply_gguf_arch_patch.py`【B，11】。

### 2.3 4-bit 与 W4A8

| 仓库 | 文件 | 大小 | 精度 | 适配 | 加载 |
|---|---|---|---|---|---|
| DiffSynth-Studio/MiniMax-H3-NF4 | `minimax-h3-fl2va-pruned-nf4.safetensors` | 约 9.8 GB | NF4（bitsandbytes） | 8GB（配磁盘卸载） | DiffSynth `MiniMaxH3Pipeline` |
| 同上 | `minimax-h3-text-encoder-nf4.safetensors` | 页面未给 | NF4 | — | 同上 |
| AX1Y2JP/MiniMax-H3-W4A8-ConvRot | `minimax_h3_fl2va_pruned_w4a8_mixed.safetensors`（据 Qiita 用法） | 页面未给 | W4A8 | 8GB，需最新 ComfyUI + comfy-kitchen | ComfyUI |
| realrebelai/Rebels_w4a8s | `MiniMax-H3-REF2VA-w4a8.safetensors` | 22.8 GiB（Qiita 实测） | W4A8 | 仅 Ref2VA、**非剪枝** | ComfyUI |

NF4 的 8GB 是“磁盘卸载下的理论下限”，模型卡原文：“The actual GPU memory usage depends on available GPU memory, and the model can run with as little as 8GB of GPU memory.”【B，12】实际 T4（14.56 GiB）上只跑通 288×160/22 帧/6 步【B，14】。

### 2.4 8GB 实际最小组合

社区跑通 5 秒 480p 的最小磁盘组合（JUNI 实测合计 39.55GB）【B，24】：

`fl2va_pruned_int8_convrot` 19.53 GB + `qwen3vl_32b_nvfp4_awq` 14.61 GB + video VAE 4.85 GB + audio VAE 0.56 GB。

若还要 Ref2VA，再 +19.5GB（Qiita 3070 实测总磁盘约 63GB）【B，19】。

---

## 第 3 章 运行时

| 运行时 | H3 支持状态 | Windows 原生 | 8GB 适配 | 证据 |
|---|---|---|---|---|
| **ComfyUI** | 原生，v0.30.0 起（PR #15224，2026-08-03），4 个节点 + 5 个本地模板 | **是**（社区全部 8GB 实测都在 Windows） | 是，社区主路径 | 【B，06/08/30】 |
| **SGLang** | v0.5.17+，Preview；官方 cookbook | **否/未声明**；依赖 NCCL、Docker、Linux | **无 8GB 配方**；最小可用是 4090 24GB（峰值 ~18GB） | 【B，06/18】 |
| **DiffSynth-Studio** | 2026-08-03 起支持 FL2VA/Ref2VA、NF4、低显存 | **未声明**（pip 安装，未见 Windows 说明） | 宣称 8GB（磁盘卸载） | 【B，37/12/13】 |
| **stable-diffusion.cpp** | 2026-08-04 Day-1；`--mode vid_gen` | **是**（明确列 Windows，示例即 `.exe` 路径） | 可用 GGUF Q2/Q3 + `--backend te=cpu` | 【B，34/35/09】 |
| **diffusers** | 官方 `ModularPipeline`，官方 checkpoint | **是**（PyTorch 通用），但无 H3 专用 Windows 说明 | 未发现 8GB 实测；有二手站称 8-bit 文件需 “over 80GB of RAM”（D 级转述） | 【B，03】/【D，38.3】 |
| **vLLM-Omni** | 社区 recipe，vLLM 0.26.0 | 未声明 | 不适合：双卡卸载要求 ≥200 GiB 系统内存 | 【B，06】 |

### 3.1 ComfyUI 版本门槛（这是最容易踩的坑）

| 版本 | 状态 | 等级 |
|---|---|---|
| **0.30.0** | 首个原生支持版本（PR #15224，2026-08-03）【B，06/08】；但**对 8GB 不可用**：VAE 部分卸载触发设备不一致崩溃（#15251），zenn 实测 “ComfyUI v0.30.0 では動きません”【B，22/29】 | B |
| **0.30.1 及更早** | 修复推进期；sm_120 上 SageAttention 在 **167k token 以上**出噪声（154k 以下干净）【B，28】 | B |
| **0.31.1** | SageAttention 噪声在该版本上被报告为“可用”（1920×1088、15s、54 min vs 94 min，KJNodes 节点设 `auto`）【B，28】 | B |
| **0.32.0** | 3070 Ti 8GB 的 79.74 s 实测即在此版本上（cu130）【B，20】 | B |
| **0.33** | `--disable-dynamic-vram` 下的短片段显存估计问题（#15663）【B，25】 | B |
| **0.34 / 0.35** | Model Sparse Attention + 新 compiler 的 OOM（#16175），0.35 仍有复发报告【B，27】 | B |
| **0.37.2** | 有用户报告升级后出现 aimdo 初始化失败型 OOM【B，25】 | B |

**推荐版本区间（本档案唯一结论）：0.31.1–0.32.x，等级【C】。**

理由（逐条，均为 B 级事实 + C 级推断）：
1. 0.30.0 在 8GB 上不可用（#15251，B），必须跳过；
2. 0.31.1 是 issue #15263 作者亲测“可用”的最低版本（B）；
3. 0.32.0 有 3070 Ti 8GB / cu130 / 79.74 s 的完整实测（B）；
4. 0.33 起出现 #15663 的短片显存低估问题，0.34–0.35 有 #16175 的 sparse attention OOM，0.37.2 有 aimdo 初始化失败报告（均为 B）——这些都不解决 8GB 的核心问题，只增加变量；
5. 因此“0.31.1–0.32.x”是由 issue 时间线反推的**推断**，不是官方兼容矩阵，也不是任何一份第三方对 5070 Laptop 的实测结论 → C。

反对“直接用最新稳定版”的理由：最新版把 4 个开放性故障（#15663 / #16175 / 0.37.2 aimdo）一次性引入，而 8GB 用户没有余量做故障归因【C，25/27】。

节点清单（原生，无需第三方）【B，08/30】：
`EmptyMiniMaxH3LatentAV`、`MiniMaxH3ImageToVideo`、`MiniMaxH3ReferenceToVideo`、`MiniMaxH3SigmaShift`；VAE 用 `VAELoader`，文本编码器用 `CLIPLoader`（type 选 minimax_h3）。

### 3.2 SGLang 为什么不是 8GB 的答案

- 官方 Quickstart 是 **8×B200**，并明说 “The 8 × B200 topology below is a reference configuration, not a minimum-hardware claim.”【B，06】
- Cookbook 消费级最低配方是 **1×4090 24GB**，峰值仍约 18GB【B，18】。
- 网传 “12GB 配方” 的真相：是在 **2TB RAM 的机器**上把 allocator 硬限到 12 GiB、并把 host 预算设成 32GB 的**模拟**；实测 16.8–18.7 s/iter，匿名内存 24.5 GiB【C，38.5】。localrigcheck 同样标注 “lab run / memory capped”【C，38.4】。
- 2×5090 32GB 配方需要 “a 377 GiB host”【B，18】。

结论：SGLang 在 32GB RAM 的笔记本上不具备可行性；Windows 还需 WSL2，而 WSL2 的显存管理与 pinned memory 行为与原生不同，无任何 8GB 实测【C】。

### 3.3 stable-diffusion.cpp 的定位

唯一“原生 Windows + 纯 C/C++ + 无 Python”的路径，且明确列 Windows 平台【B，34】。H3 用法要求四个组件分开传，且有**三个不可省略的 flag**：“`--mode vid_gen` … `--cfg-scale 1.0` … `--backend te=cpu`”【B，09】。它的优势是 TE 可以完全放 CPU（省 12–15GB 显存），劣势是缺少 ComfyUI 生态的 Turbo LoRA/注意力补丁；目前**没有 8GB 的 sd.cpp H3 实测**（未核实）【D】。

---

## 第 4 章 8GB 社区实测汇总

筛选标准：显存 ≤8GB（或明确限制在 8GB 上限）、写明分辨率与耗时、有可追溯原文。Tomiigo 的 8,188 MiB 是**在 16GB 卡上模拟**，单列在表后，不混入。原文确实没给的单元格一律写“未写”，不做推算填补。

| # | 来源 | 日期 | GPU | RAM | OS | ComfyUI / torch | 模型文件 | 分辨率 | 帧数 | 步数 | LoRA | attention | 耗时 | 显存峰值 | 等级 |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| 1 | Qiita 3070【19】 | 2026-08-11 | RTX 3070 8GB | 48GB | Win11 原生 | 0.31.0 / 2.11.0+cu128（§7 另测 cu130） | `fl2va_pruned_int8_convrot` + `qwen3vl_32b_nvfp4_awq` + video VAE fp16 | 480×832 | 124 | 8 | `turbo_4step_ckpt600_ema_V4` | SDPA | 345.2 s | 7,322 MiB | B |
| 2 | Qiita 3070【19】 | 2026-08-11 | RTX 3070 8GB | 48GB | Win11 原生 | 0.31.0 / 2.11.0+cu128 | 同上 | 352×608 | 124 | 8 | 同上 | SDPA | 213.4 s | 6,682 MiB | B |
| 3 | Qiita 3070 Ti【20】 | 2026-08-13 | RTX 3070 Ti 8GB | 16GB | Windows | 0.32.0 / 2.9.1+cu130 | `fl2va_pruned_w4a8_mixed` + TE w4a8_mixed + video VAE int8 | 864×480 | 124 | 4 | `fl2v_turbo_8step_v1.0` | SageAttention | 79.74 s（cold 89.56 s） | 6.47 GiB | B |
| 4 | note 4060【21】 | 2026-08-17 | RTX 4060 Laptop 8GB | 32GB | Windows | 未写 / 未写 | `fl2va_pruned_int8_convrot` + `nvfp4_awq` + video VAE fp16 | 416×736 | 未写（5 s） | 未写 | 无 | “comfy kitchen attention” | 6 分 30 秒（换前 8 分 53 秒） | 未写 | B |
| 5 | note JUNI【24】 | 2026-08-11 | RTX 3060 Ti 8GB | 64GB | 未写 | 0.31.0 / 2.13.0+cu130 | `fl2va_pruned_int8_convrot` + `nvfp4_awq` | 832×480 | 124 | 6 | Turbo（5 s 档未标明具体 LoRA；10 s 档用 lightx2v） | 未写 | 185.5 s（warm）/ 310.5 s（cold） | 6,304 MiB | B |
| 6 | Zenn cosara【22】 | 2026-08-15 | RTX PRO 2000 Blackwell Laptop 8,151 MiB | 95.5GB | Win11 26100 | commit 2a68ce33 / 2.13.0+cu130 | NVFP4 DiT 11.66GB + 其余未写 | 800×448 | 362 | 未写 | 未写 | 未写 | 908.1 s | 未写 | B |
| 7 | Zenn cosara【22】 | 2026-08-15 | 同上 | 95.5GB | Win11 26100 | 同上 | 同上 | 864×480 | 124 | 20 | 未写 | 未写 | 未写（仅称稳定） | 未写 | B |

补充说明（不占表行）：

- **Tomiigo（模拟上限，等级 C，不混入上表）**：RTX 5070 Ti 16GB，用 sidecar 把可用显存压到 8,188 MiB，RAM `mem=32GB`，Linux；四变体峰值 7,130–7,669 MiB，165–190 s，磁盘读 270 GiB；原文反复强调 “No physical 8 GB card was used”“Timings here will not transfer to a real 8 GB card”【C，23】。**不能当作 8GB 实测引用**（本条按 R8 统一降为 C）。
- **失败边界（同一作者）**：3070 上 608×896 与 640×928 的 I2V 均 OOM；544×960 可跑但效率骤降（2.05×/步）【B，19】。cosara 上 832×448 起 “VRAM grow failed”，832×480/864×480 OOM【B，22】。
- **15 秒**：cosara 在 8,151 MiB 上做出 800×448/15.083 s（908.1 s）【B，22】；JUNI 在 8,192 MiB 上做出 362 帧（2113.2 s），但依赖 PR #15446 的流式 VAE 解码【B，24】。可见“5 秒 480p”与“15 秒 480p”是两个难度档【B】。

### 4.1 从表里能读出的三条硬结论

1. **480p/5s 在 8GB 上是 B 级事实**，不是传闻：5 张不同的 8GB 卡（3070、3070 Ti、3060 Ti、4060 Laptop、RTX PRO 2000 Laptop）、5 个独立作者、7 组配置，耗时 80 s–9 min【B，19/20/21/22/24】。
2. **决定性变量不是显存，而是三件事**：① 是否用 pruned int8/W4A8（19.5GB vs 61.7GB）；② 是否用 Turbo LoRA 把步数压到 4–8；③ 是否 cu130（int8 快速 kernel 开启，1.56×）【B，19/20】。
3. **16GB RAM 也能跑 864×480/4 步**（3070 Ti，79.74 s）【B，20】；由此推断 32GB RAM 的目标机在内存侧不构成硬瓶颈——但 cold 进程 RAM 已达 9.74 GiB 且 TE 为 32B 级，仍属紧张【C，20】。真正的瓶颈是 8GB 显存下的模型流式加载带宽与 VAE 解码峰值【B，19/22】。

### 4.2 对 5070 Laptop 的外推（明确标注为推断）

- 5070 Laptop 是 Blackwell（sm_120）；SGLang 文档写 NVFP4 执行需 “compute capability 10.0+”【B，18】。把 sm_120 视作 CC 12.0 属常识映射，本文按【C】处理。但 Comfy-Org 说其 `nvfp4_awq` **文本编码器**并不需要 Blackwell【B，07】，所以“NVFP4 加速”对 TE 意义有限，真正受益的是 DiT 的 NVFP4/INT8 kernel【C】。
- 目标机的显存带宽/功耗墙没有公开实测。可比对象是 cosara 的 RTX PRO 2000 Blackwell Laptop（同样 8,151 MiB）：仅把 85W 坞换成 140W 原装电源就带来 2.76× 差异【B，22】。**5070 Laptop 上没有任何同类数据，属未核实**【D】。

---

## 第 5 章 已知坑与修复

按“症状 → 根因 → 修复/规避 → 适用版本”排列。每条都给 issue/PR 链接。

### 5.1 8GB 必踩：VAE 部分卸载设备不一致

- 链接：https://github.com/Comfy-Org/ComfyUI/issues/15251
- 症状：VAE 解码时 `RuntimeError: Expected all tensors to be on the same device … cuda:0 and cpu`，两处：`ViT3DDecoder.forward` 的 `register_tokens`、`TransformerBlock.forward` 的 `scale1/scale2`【B，29】。
- 根因：8GB 卡触发 VAE 部分 CPU 卸载，注册缓冲/标量未跟随主张量迁移【B，29】。
- 修复：官方后续版本修复；用户侧可把 `register_tokens`、`scale1/scale2` 显式 `.to(device=h.device, dtype=h.dtype)`【B，29】。**规避：不要用 0.30.0**。
- 适用：0.30.0；8GB 及以下。等级 B【29/22】。

### 5.2 SageAttention 出纯噪声（视频+音频）——sm_120 专项

- 链接：https://github.com/Comfy-Org/ComfyUI/issues/15263
- **issue 标题（逐字）**：“MiniMax H3: SageAttention FP8 PV kernels produce noise above ~160k tokens on sm_120”【B，28】。
- 症状：整段视频与音频同时是纯噪声，无报错、无警告；55 分钟采样白跑【B，28】。
- 根因（按原作者 2026-08-09 更新）：**sm_120 上 SageAttention 的 FP8 PV kernel 在约 154k token 以下干净、167k token 起出噪声**；作者给出的阈值表：142k / 151k / 154k clean，167k / 175k / 186k noise【B，28】。音频 VAE 不经过 `optimized_attention`，故故障在 DiT【B，28】。
- **作者更新（逐字）**：“**This works now on ComfyUI 0.31.1.** 1920x1088, 15 s, sage on via the KJNodes node set to `auto`, 54 min against 94 with sage off.”【B，28】
- **模式限制**：作者 08-08 给出的 `sageattn_qk_int8_pv_fp16_cuda` workaround 已被撤回——“doesn't work on sm_120 … `auto` is the only mode that runs on that arch”；在 sm_120 上首次调用该 `pv_fp16` 路径即失败/中止（abort），只有 `auto` 能跑【B，28】。
- 升级建议（逐字）：“If you're on 0.30.1 or older and getting noise on sm_120, update first.”【B，28】
- 修复/规避：升级到 ≥0.31.1；KJNodes 的 sage 节点必须设 `auto`。**480×832 / 124 帧按“宽/32 × 高/32 × 帧/4”估算约 12k token，远低于 167k 阈值，属安全区**【C（阈值换算；该公式用 issue 表中 1920×1088/362 帧≈186k 反算一致），28】。
- 适用：0.30.1 及更早（sm_120）；0.31.1+ 已报告可用但仍受 token 阈值约束【B，28】。
- **对 5070 Laptop 的直接含义**：目标机就是 sm_120、8GB，本条不是“传闻”，是**同一架构上的已知缺陷与已知修复**；V1.0 仍默认 SDPA，Sage 只在确认安全分辨率下作为提速项【B/C，28】。
- **3070（Ampere）的噪声报告是另一现象**：Qiita 3070 作者记录 “SageAttention → 视频与音频全噪声（引 ComfyUI #15263），用 SDPA”【B，19】。Ampere 不在 #15263 的 sm_120 结论覆盖范围内，该报告可能是另一 bug 或旧版本，**未核实，单独列出，不与 sm_120 结论混用**【D（归因）】。

### 5.3 短片段显存估计偏低 → OOM 且“粘住”

- 链接：https://github.com/Comfy-Org/ComfyUI/issues/15663
- 症状：2 秒片段估计仅 514 MB，保留 16GB 权重常驻，峰值 28,928 MiB（卡只有 24,564 MiB）；一旦 OOM，同一图重启也必 OOM，改一个参数即恢复【B，25】。
- 根因：`comfy-aimdo` 初始化失败（`funchook_prepare(cuMemAlloc_v2) failed: 8`）导致 DynamicVRAM 未启用，估计失真【B，25】。
- 修复：`LD_PRELOAD` 预加载 `aimdo.so`（Linux 自定义 glibc 场景）；通用规避 `--disable-pinned-memory --reserve-vram 1 --disable-async-offload`【B，25】。
- 适用：0.33.0 / 0.37.2。等级 B【25】。

### 5.4 Model Sparse Attention 节点 OOM

- 链接：https://github.com/Comfy-Org/ComfyUI/issues/16175
- 症状：启用核心 `Model Sparse Attention` 后约 50% 进度 OOM；关闭即正常【B，27】。
- 根因：与 Comfy model compiler 的交互（`Comfy model compiler graph breaks: 10`），不是稀疏注意力数学本身【B，27】。
- 修复：`--disable-comfy-compiler` 可规避；官方称已修，但 v0.35.0 仍有复发报告【B，27】。
- 适用：0.34–0.35。等级 B【27】。

### 5.5 EasyCache 劣化音频（未修复）

- 链接：https://github.com/Comfy-Org/ComfyUI/issues/15326
- 症状：视频 SSIM 仍 0.95–0.96，但音频低音缺失、响度减半（RMS 0.503–0.971）、频谱质心上移【B，31】。
- 根因：EasyCache 用视频流决定跳步，却把缓存残差同样施加到音频；H3 音频有自己的 shift 调度，残差缩放系数从 4.00 变到 0.26【B，31】。
- 修复：无。降阈值无效（0.1 仍 mel L1 0.283）【B，31】。**规避：8GB 上不要用 EasyCache**；Tomiigo 的图用了它并注明“音频是否受损未判定”【C，23】。
- 适用：全部 H3 版本。等级 B【31】。

### 5.6 VAE 解码 OOM 重试无效

- 链接：https://github.com/Comfy-Org/ComfyUI/pull/15456（修 #15453）
- 症状：长片段解码 OOM，日志提示 “retrying with tiled VAE decoding”，但重试同样失败【B，26】。
- 根因：H3 video VAE 的 `handles_tiling=True`，`decode_tiled()` 直接调 `decode()`；重试前不释放其它模型，约 14GB 流式权重仍常驻，而解码只需约 1.2GB【B，26】。
- 修复：重试前 `free_memory(..., keep_loaded=[VAE patcher])`。页面显示仍是 “wants to merge”，**是否已合并未核实**【B，26 + D（合并状态）】。
- 适用：长片段 + 动态显存。等级 B【26】。

### 5.7 宽高不是 32 的倍数 → reshape 报错

- 症状：`RuntimeError: shape '[1, 24, 1, 1, 29, 2, 20, 2]' is invalid for input of size 56640`（640×944）【B，19】。
- 根因：latent 先 1/16，再切 2×2 patch，故 `height//16` 必须为偶数 → 高宽须为 32 的倍数。节点 schema 声明 `step=32`，但 API 不校验【B，19】。
- 修复：脚本侧校验；1280×720 也是同样原因失败（720÷32=22.5）【B，24】。
- 来源：Qiita 3070【B，19】、JUNI【B，24】。等级 B。

### 5.8 长片段导致整机重启（Windows + VBS + CUDA 系统内存回退）

- 症状：362 帧生成 5–15 分钟后蓝屏/重启；`HYPERVISOR_ERROR (0x20001)`，Arg1=0x28（IOMMU 内部错误）【B，22】。
- 根因：驱动把溢出显存静默回退到系统内存，回退数据经 PCIe DMA，而 VBS 的 DMA 保护要求所有 DMA 过 IOMMU，长片段回退量大到触发 IOMMU 内部错误【B，22】。
- 修复：NVIDIA 控制面板 → “CUDA - 系统内存回退策略” → “优先无系统内存回退”。崩溃 8 次 → 0 次；代价是 864×480 可通过长度从 209 帧降到 124 帧【B，22】。
- 适用：Win11 + VBS/Hyper-V + 8GB。等级 B（单机、8 次复现）【22】。
- 这是本档案里对 5070 Laptop 最有价值的一条：目标机 8151 MiB 与作者 8151 MiB **完全一致**【B，22】。

### 5.9 其它

| 症状 | 根因 | 修复 | 来源 | 等级 |
|---|---|---|---|---|
| int8 只跑出 1/3 速度 | cu128 下 `comfy_kitchen` CUDA backend 被禁用，回落到 eager | 换同版本 +cu130 的 torch/torchvision/torchaudio；日志确认 `disabled: False` | 【19】 | B |
| W4A8 比 int8 慢 3.2× | 同上，cu128 无融合 int8 GEMM；cu130 后反超 | 升级 cu130 | 【19】 | B |
| 15 秒 8GB OOM | VAE 解码峰值随帧数线性涨 | PR #15446 流式 VAE 解码（增长速率变 1/2.5） | 【24】 | B |
| I2V 不响应运镜 | first_frame 约束构图 | 改用 R2V | 【19】 | B |
| GGUF 无法加载 | ComfyUI-GGUF 架构白名单无 `minimax_h3` | ComfyUI-H3-Multishot ≥1.5.2 或 apply 脚本 | 【10/11】 | B |
| GGUF 文本编码器加载失败 | 视觉塔是 `-mmproj` sidecar，stock 只对 qwen2vl 合并 | 用 `H3 Clip Loader (Any)` | 【11】 | B |
| Turbo LoRA 出同心圆伪影 | 特定 ckpt（ckpt850_V1） | 换 ckpt600_ema_V4 | 【19】 | B |
| Turbo LoRA 音频变噪声 | 4 步 joyfox LoRA 与音频调度不匹配 | 换 lightx2v 8 步或降 LoRA 强度 | 【24/16】 | B |
| 功耗被限到 25W，速度差 2.76× | 85W USB-C 坞供电不足 | 原装 140W 直插；`nvidia-smi` 看 `SW Power Cap` | 【22】 | B |

---

## 第 6 章 推荐 V1.0 基线配置

### 6.1 技术假设（基线定义）

以 **ComfyUI 0.31.1–0.32.x + pruned int8 DiT + NVFP4 TE + fp16 video VAE + Turbo LoRA + SDPA** 作为 V1.0 基线，在 RTX 5070 Laptop 8GB + 32GB RAM 上可重复生成 5 秒（124 帧）视频。该假设基于多份 8GB 社区实测（RTX 3070/3070 Ti/3060 Ti/4060 Laptop/RTX PRO 2000 Blackwell Laptop），但**未在本机验证**【C】。

关键推理链（C 级，待本机验证）：

- NVFP4 的**计算加速**需 Blackwell 一代硬件。**Tomiigo 的指南（经 part2 转述，本任务留底未逐字命中该句）称**低于 Blackwell 时 `supports_nvfp4_compute` 返回 false、权重在运行时展开、无加速；而该指南本身是在 **RTX 5070 Ti 16GB 上用 sidecar 模拟 8,188 MiB 上限**测的，原文自述 “No physical 8 GB card was used”“Timings here will not transfer to a real 8 GB card”【C，23】。本条按 R8 统一标 **C**，不能当作 8GB 实卡结论。
- 5070 Laptop 是 Blackwell（sm_120，按常识映射为 CC 12.0）；SGLang cookbook 记 NVFP4 执行需 “compute capability 10.0+”【B，18】，故 **DiT 的 NVFP4 计算在本机应受支持**属推断【C】。但 Comfy-Org 明确说其 `nvfp4_awq` **文本编码器**在任何 GPU 都能跑、不要求 Blackwell【B，07】，所以 NVFP4 对 TE 主要是体积优势，加速收益应在 DiT【C】。无任何第三方 5070 Laptop 数据【D】。
- int8_convrot DiT 在 Ampere 上有 INT8 张量核加速【B，19】；Blackwell 同样支持 INT8，预期不弱于 3070【C】。

### 6.2 组件参数表（每项含来源与等级）

| # | 组件 | 推荐值（V1.0） | 依据来源 | 等级 |
|---|------|----------------|----------|------|
| 1 | ComfyUI 版本 | **0.31.1–0.32.x**（统一区间）；**0.30.0 在 8GB 上不可用**（#15251）；不建议无脑用最新稳定版（0.33 #15663 / 0.34–0.35 #16175 / 0.37.2 aimdo） | 3.1 节；#15251【B，29】；#15263 作者称 0.31.1 可用【B，28】；0.32.0 有 3070 Ti 实测【B，20】 | **C**（区间推断） |
| 2 | PyTorch / CUDA | torch 2.11.0~2.13.0 **+cu130**（Comfy-Org 明确建议 cu130 以启用 int8_convrot） | Qiita 3070（2.11.0+cu130）【B，19】；zenn（2.13.0+cu130）【B，22】；comfyui-wiki【B，08】 | B |
| 3 | DiT（FL2VA） | `minimax_h3_fl2va_pruned_int8_convrot.safetensors`（19.5 GB，官方列表；Qiita 实测约 21 GB） | Comfy-Org/MiniMax-H3【B，wiki 列 19.5，08】；Qiita 3070【B，测 ~21，19】 | B |
| 4 | 文本编码器 TE | `qwen3vl_32b_minimax_h3_nvfp4_awq.safetensors`（14.6 GB，官方；Qiita 实测约 15.7 GB） | Comfy-Org【B，08】；Qiita 3070【B，19】；Tomiigo（Blackwell NVFP4 加速，模拟环境）【C，23】 | B（尺寸）/ C（加速） |
| 5 | Video VAE | `minimax_h3_video_vae_fp16.safetensors`（4.9 GB） | Comfy-Org【B，08】；Qiita 3070【B，测 5.2，19】 | B |
| 6 | Audio VAE | `minimax_h3_audio_vae_fp32.safetensors`（0.6 GB） | Comfy-Org【B，08】 | B |
| 7 | Turbo LoRA | Abiray `MiniMax-H3-Turbo-Lora-Pruned-ComfyUI` → `minimax_h3_turbo_4step_ckpt600_ema_V4.safetensors`（约 0.6 GB） | Abiray 卡【B，16】；Qiita 3070（ema_V4 无伪影，ckpt850_V1 有环状伪影）【B，19】 | B |
| 8 | 分辨率 | 必须是 **32 的倍数**（见 6.4） | Qiita 3070【B，19】；zenn（800×448 等）【B，22】 | B |
| 9 | 帧数 | 124 帧 = 5.17 秒 @24fps | Qiita 3070【B，19】；zenn（17k+5 阶梯）【B，22】 | B |
| 10 | 步数 | 8–12（推荐 12；8 步肢体易融、16 步无改善） | Qiita 3070【B，19】；Abiray（8–12）【B，16】 | B |
| 11 | sigma shift | video **12** / audio **6**（不守此值音频会坏/噪） | Abiray README【B，16】；Qiita 3070【B，19】 | B |
| 12 | sampler / scheduler | `res_multistep` + `simple` | note 4060【B，21】；zenn【B，22】；Abiray 仅列 `res_multistep` 未点名 scheduler【B，16】 | B（sampler）/ C（scheduler 名） |
| 13 | attention | **SDPA**（V1.0 默认，最稳） | Qiita 3070【B，19】；#15263：sm_120 的 Sage 噪声已于 0.31.1 报告可用，但仅 `auto` 模式、且 >167k token 仍会出噪声【B，28】 | B |
| 14 | 启动参数 | `--disable-pinned-memory --reserve-vram 1 --disable-async-offload`（OOM 规避，见 6.5） | #15663 作者用此三旗解决 OOM【B，25】；Tomiigo（`--disable-pinned-memory --reserve-vram 1.5`，模拟环境）【C，23】 | B（来源）/ C（迁移） |
| 15 | NVIDIA 控制面板 | “3D 设置 → CUDA - 系统内存回退策略 → **优先无系统内存回退**”；并接原厂 140W 供电（避免 USB-C 低功率坞） | zenn（此设置消除 8 次蓝屏；供电 25W→60W 提速 2.76×）【B，22】 | B |

**关于第 15 项的性质说明**：它属于**修改系统/驱动级设置**，与“规格第七节禁止改驱动/系统配置”的约束直接相关——本档案只记录该做法及其证据，不把它列为自动执行项；是否采用须 Junlong 单独批准【D（流程）】。

**最小下载体积（仅 FL2VA，不含 ref2va）：** 19.5 + 14.6 + 4.9 + 0.6 + 0.6(LoRA) ≈ **40.2 GB**（官方列表）；Qiita 3070 记为“约 43 GB（FL2VA 最小构成）”【B，19】，差异来自 safetensors 实际略大于列表值。加 ref2va 再 +19.5 GB。

### 6.3 两套参数：首次冒烟测试（保守）vs 基线目标

**A. 首次冒烟测试（保守，先求“能跑完”）**

| 项 | 值 | 依据 |
|----|----|------|
| 分辨率 | **352×608**（214K 像素，9:16） | Qiita 3070：352×608 / 8 步 / 213.4s / 6,682 MiB【B，19】 |
| 帧数 | 124（5.17s） | 同上【B，19】 |
| 步数 | **8**（先验证通路，再升 12） | Qiita 3070【B，19】 |
| shift | video 12 / audio 6 | Abiray【B，16】 |
| sampler/sched | res_multistep / simple | note 4060、zenn【B，21/22】 |
| attention | SDPA | Qiita 3070【B，19】 |
| 预期 | 3070 上 213s / 6.7GB；5070 Laptop 未知（同代 8GB 量级，待实测） | 【C/D】 |

**B. 基线目标（V1.0 正式）**

| 项 | 值 | 依据 |
|----|----|------|
| 分辨率 | **480×832**（399K 像素，9:16）— 8GB int8 DiT 的 T2V 实用上限 | Qiita 3070：480×832 / 8 步 / 345.2s / 7,322 MiB（最高稳过 T2V）【B，19】 |
| 帧数 | 124（5.17s） | Qiita 3070【B，19】 |
| 步数 | **12**（质量最优，16 无改善） | Qiita 3070【B，19】 |
| shift | video 12 / audio 6 | Abiray【B，16】 |
| sampler/sched | res_multistep / simple | note 4060、zenn【B，21/22】 |
| attention | SDPA | Qiita 3070【B，19】 |
| 预期 VRAM 峰值 | 约 7.3 GB（3070）；5070 Laptop 待实测 | Qiita 3070【B，19】/【C（外推）】 |

> I2V 上限低于 T2V：Qiita 3070 实测 I2V 实用上限约 544×800（435K），更高即 OOM【B，19】。若做 I2V，冒烟测试分辨率应降到 416×736（0.3MP，note 4060 实测稳过）【B，21】。

### 6.4 分辨率必须是 32 的倍数的依据

Qiita 3070 原文明确：宽高必须 32 的倍数（8 章）【B，19】。所有社区成功分辨率（352×608、480×832、416×736、800×448、864×480、1344×768）均满足【B，19/20/21/22】。zenn 指出 VRAM 峰值由分辨率而非时长决定，因此调分辨率比调时长更能控 OOM【B，22】。

### 6.5 启动参数 / 控制面板设置依据

- `--disable-pinned-memory --reserve-vram 1 --disable-async-offload`：#15663 中作者在 RTX 4090/3090 上靠这三旗解决 H3 短片 OOM（根因 `comfy-aimdo` 初始化失败导致 DynamicVRAM 未启用）【B，25】。Tomiigo 在 8GB 模拟上也用 `--disable-pinned-memory --reserve-vram 1.5`（默认会页锁高达 90% RAM，关掉后 0 次 swap）【C，23】。
- “CUDA - 系统内存回退策略 → 优先无系统内存回退”：zenn 在 RTX PRO 2000 Blackwell Laptop 8GB 上因 VRAM 溢出退避到系统内存触发 IOMMU 蓝屏 8 次，改此设置后 0 次；代价是“VRAM 超支时作业以 CUDA OOM 停止而非整机崩溃”，且可过长度从 209 帧降到 124 帧【B，22】。
- 供电：zenn 同一生成 655.8s→237.2s（2.76×）仅因换 140W 原厂供电（USB-C 坞把功耗墙压到 25W，SW Power Cap 激活）【B，22】。OMEN 16 应使用原厂大功率适配器，勿用低功率 USB-C 坞【B/C，22】。

---

## 第 7 章 Windows 执行单草稿（草稿·未执行·需 Junlong 批准）

> **本执行单为草稿，未在任何机器上执行。所有命令、路径、耗时均为外部资料推断或占位，未经本机验证（无 A 级证据）。执行前须经 Junlong 批准。本单全部步骤为【D】级流程草稿，不含技术结论。**
>
> **本单遵守以下约束（按审核意见 R2–R4 修正后）**：
> 1. **不更新、不修改显卡驱动**；只记录实际版本，低于 580 即停止并报告；
> 2. **不删除任何用户文件**（含 `.venv`）；回滚只用“重命名 + 解压备份”；
> 3. **不修改系统配置**（NVIDIA 控制面板、pagefile、电源计划均只列为“需单独批准”，本单不执行）；
> 4. 所有检查步骤默认**只读**；写操作单独标注“需 Junlong 批准”；
> 5. **本单不包含任何凭据**（无口令、无访问令牌），也不访问需登录的私有仓库。
>
> 路径前提为假设：`C:\Users\testuser\WorkBuddy\comfyui\ComfyUI`，venv 在 `.venv`。PC 回报只说“源码 + venv”，**`.venv` 是假设路径，必须用步骤 2 实测确认**。

### 步骤 1 · 记录环境（只读，不改驱动）

```powershell
# PowerShell 5.1；普通用户即可，无需管理员
$ComfyRootGuess = "C:\Users\testuser\WorkBuddy\comfyui\ComfyUI"   # 假设值，步骤 2 校验
& "$env:SystemRoot\System32\nvidia-smi.exe" --query-gpu=name,driver_version,memory.total --format=csv
```

- **预期结果**：输出 GPU 名（含 “RTX 5070”）、驱动版本、显存总量（约 8151 MiB）。PC 回报的记录值是 **616.92**，但以本次实测为准。
- **失败停在哪**：
  - 找不到 `nvidia-smi` → 停，报告 Junlong（本单不安装/修复驱动）。
  - **驱动版本 < 580 → 停，把实测版本号报告 Junlong，等指示**。
  - **本单不执行任何驱动安装、更新、回滚操作**（规格第七节禁止改驱动/系统配置）。

### 步骤 2 · 确认 ComfyUI 根目录与 venv 实际路径（只读）

```powershell
$ComfyRoot = $ComfyRootGuess
Test-Path $ComfyRoot
Get-ChildItem -Force $ComfyRoot | Where-Object { $_.Name -like "*venv*" -or $_.Name -eq "main.py" } |
    Select-Object Name, Mode, LastWriteTime
```

```powershell
# 以实际存在的解释器为准；先试 .venv，若不是再按上一步列出的目录名替换
& "$ComfyRoot\.venv\Scripts\python.exe" -c "import sys; print(sys.prefix); print(sys.executable)"
```

- **预期结果**：`Test-Path` 为 True；列表里能看到 `main.py` 与某个 `*venv*` 目录；`sys.prefix` 指向 `$ComfyRoot\.venv`（或实测到的实际目录）。
- **失败停在哪**：
  - `$ComfyRoot` 不存在 → 停，报告 Junlong（不自行搜索全盘、不移动目录）。
  - 没有 venv 目录，或 `sys.prefix` 指向系统 Python / conda → 停，记录实际路径并报告；**是否新建 venv 需 Junlong 批准**，本单不创建。
  - **后续所有命令中的 `.venv` 一律替换为步骤 2 实测到的路径**。

### 步骤 3 · 确认 torch / CUDA 版本（只读；判断是否 cu130）

```powershell
& "<步骤2实测的 python.exe>" -c "import torch;print(torch.__version__, torch.version.cuda, torch.cuda.get_device_name(0))"
& "<步骤2实测的 python.exe>" -m pip show torch | Select-String -Pattern "Version|Location"
```

- **预期结果**：`torch.__version__` 为 2.11–2.13 量级，`torch.version.cuda` 为 **13.0**，`torch.cuda.get_device_name(0)` 含 “RTX 5070”。
- **失败停在哪**：
  - `import torch` 失败 / 无 CUDA → 停，报告 Junlong。
  - `torch.version.cuda` **不是 13.0**（例如 12.8）→ **停**。把实际版本写入记录，并把“升级 torch 到 cu130”列为**需批准操作**；命令如下，**本单不执行**：

```powershell
# 【需 Junlong 批准后才可执行；本单不执行】
# 先完成步骤 4 的 venv 备份，再考虑执行：
# & "<venv>\Scripts\python.exe" -m pip install --index-url https://download.pytorch.org/whl/cu130 torch==2.11.0+cu130 torchvision==0.26.0+cu130 torchaudio==2.11.0+cu130
```

- **说明**：cu128 下 int8 路径会退回 eager，实测只有 1/3 速度（【B，19】），所以 cu130 不是“可选项”而是基线前提；但它属于改动现有环境，必须批准。

### 步骤 4 · 备份 venv（只复制，不删除）

```powershell
$ts = Get-Date -Format "yyyyMMdd-HHmm"
Compress-Archive -Path "$ComfyRoot\.venv" -DestinationPath "$ComfyRoot\.venv-backup-$ts.zip"
Write-Host "venv 备份: $ComfyRoot\.venv-backup-$ts.zip"
```

- **预期结果**：生成 `.venv-backup-<时间戳>.zip`，大小与 `.venv` 同量级。
- **失败停在哪**：磁盘不足 → 停（见第 9 章磁盘空间），**不要删除任何现有文件腾空间**；确需清理须 Junlong 批准。
- 说明：若步骤 2 实测 venv 路径不同，这里同步替换。

### 步骤 5 · 确认 ComfyUI 版本（只读）

```powershell
git -C $ComfyRoot describe --tags 2>$null
git -C $ComfyRoot rev-parse --short HEAD
Test-Path "$ComfyRoot\comfy\ldm\minimax"
```

- **预期结果**：版本 ≥0.31.1，且首选落在 **0.31.1–0.32.x**；`comfy\ldm\minimax` 存在。
- **失败停在哪**：
  - 版本为 0.30.0 → 停（#15251：8GB 上 VAE 设备不一致会崩）【B，29/22】。
  - 版本 <0.30.0 或缺 `comfy\ldm\minimax` → 停，报告 Junlong；**`git pull` / 同步 requirements 属改动现有环境，需单独批准，本单不执行**。
  - 版本 ≥0.33 → 记录风险（#15663 / #16175 / 0.37.2 aimdo），**是否降级需 Junlong 批准**，本单不做。

### 步骤 6 · 下载清单（仅列 URL、目标路径、校验；**需 Junlong 批准**）

> 以下为 HF `resolve/main` 直链（Comfy-Org/MiniMax-H3）。**下载 ~40GB，需 Junlong 明确批准后方可执行。** 校验方法：用 `curl -sIL <url>` 取 `X-Linked-Size` 头，与本地 `Get-Item <dest> | Select Length` 比对字节数（Qiita 3070 强烈建议）【B，19】。可用 `curl -C -`（断点续传）+ `.part` 方式防中断。

| 组件 | URL（前缀 `https://huggingface.co/Comfy-Org/MiniMax-H3/resolve/main/`） | 目标路径（相对 `$ComfyRoot\models`） | 约大小 |
|------|------|------|------|
| DiT FL2VA | `diffusion_models/minimax_h3_fl2va_pruned_int8_convrot.safetensors` | `diffusion_models/` | 19.5 GB |
| TE | `text_encoders/qwen3vl_32b_minimax_h3_nvfp4_awq.safetensors` | `text_encoders/` | 14.6 GB |
| Video VAE | `vae/minimax_h3_video_vae_fp16.safetensors` | `vae/` | 4.9 GB |
| Audio VAE | `vae/minimax_h3_audio_vae_fp32.safetensors` | `vae/` | 0.6 GB |
| Turbo LoRA | `https://huggingface.co/Abiray/MiniMax-H3-Turbo-Lora-Pruned-ComfyUI/resolve/main/minimax_h3_turbo_4step_ckpt600_ema_V4.safetensors` | `loras/` | ~0.6 GB |

- **预期结果**：5 个文件字节数与 `X-Linked-Size` 一致，置于正确子目录。
- **失败停在哪**：任一文件大小不符/下载中断 → 停，重下该文件（断点续传），**不得用损坏文件继续**；**下载失败产生的不完整文件保留，不擅自删除**。

### 步骤 7 · 启动参数（控制面板设置单列，需批准）

```powershell
# 【前置】"CUDA - 系统内存回退策略 → 优先无系统内存回退"需在 NVIDIA 控制面板手动设置。
#         该项属修改系统/驱动级设置：本单不执行，需 Junlong 单独批准（见 6.2 第 15 项说明）。
# 启动 ComfyUI（先不带系统级改动）
Push-Location $ComfyRoot
& "<venv>\Scripts\python.exe" main.py --disable-pinned-memory --reserve-vram 1 --disable-async-offload
Pop-Location
```

- **预期结果**：Web UI 在 `http://127.0.0.1:8188` 启动，日志无 `aimdo` / OOM 报错。
- **失败停在哪**：启动即 OOM/崩溃 → 停，记录日志，检查 `--reserve-vram` 值；按步骤 11 回滚，不擅自改系统设置。

### 步骤 8 · 导入模板

- 在 UI：Templates → Video → **MiniMax H3 Image to Video**（覆盖 T2V 与 first/last-frame I2V）。无需自定义节点【B，08】。
- **预期结果**：模板加载，节点含 `MiniMaxH3ImageToVideo`、`EmptyMiniMaxH3LatentAV`、`MiniMaxH3SigmaShift`、`SamplerCustomAdvanced` + `BasicGuider`（CFG-less 设计）【B，19/30】。
- **失败停在哪**：模板缺失 → 停，确认版本落在 0.31.1–0.32.x 且含 `comfy/ldm/minimax/`；不自行改源码。

### 步骤 9 · 冒烟测试（用 6.3-A 保守参数）

- 在模板设：width 352、height 608、length 124、steps 8、shift_video 12、shift_audio 6、sampler res_multistep、scheduler simple、LoRA = `minimax_h3_turbo_4step_ckpt600_ema_V4.safetensors`、attention = SDPA。
- **预期结果**：3070 参考为约 213 s / 6.7 GB（【B，19】）；**5070 Laptop 的耗时未知，属未核实**【D】。
- **失败停在哪**：OOM → 停，降分辨率或检查步骤 7 参数；音频噪声/环状伪影 → 检查 shift 值与 LoRA 文件是否为 ema_V4（非 ckpt850_V1）【B，19】。

### 步骤 10 · 记录（每轮必记）

```powershell
& "$env:SystemRoot\System32\nvidia-smi.exe" --query-gpu=memory.used,memory.total --format=csv
# 任务管理器内存：手动读取“性能→内存”并提交值
# 耗时：从 UI 提交到产出；输出文件：SaveVideo 的 filename_prefix 路径
```

- 记录项：nvidia-smi 显存峰值、任务管理器内存峰值、耗时、输出文件路径与大小、实际驱动版本、torch/CUDA 版本。
- **失败停在哪**：无法取得任一指标 → 仍记录“缺失”，不得编造。

### 步骤 11 · 回滚（重命名，不删除）

```powershell
# 若步骤 7 改过启动参数导致环境异常：
$ts = Get-Date -Format "yyyyMMdd-HHmm"
Rename-Item -LiteralPath "$ComfyRoot\.venv" -NewName ".venv-failed-$ts"
Expand-Archive -Path "$ComfyRoot\.venv-backup-<备份时间戳>.zip" -DestinationPath "$ComfyRoot\.venv"

# 若步骤 5 做过 git 变更：按备份前 commit 还原（需 Junlong 批准）
# git -C $ComfyRoot checkout <备份前 commit>
```

- **预期结果**：原 `.venv` 被改名为 `.venv-failed-<时间戳>`（**保留，不删除**），备份解压为新的 `.venv`。
- **失败停在哪**：备份缺失 → 停，报告 Junlong；**任何删除（含 `.venv-failed-*`）都必须先获 Junlong 批准，本单不执行删除**。

---

## 第 8 章 V1.1–V2.0 候选技术

> 每项：来源链接、原文测试条件、报告收益、等级、在 5070 Laptop 上的不确定点、最小验证实验、回滚方法。

### 8.1 V1.1 提速

**8.1.1 Turbo 4/8 步 LoRA**

- 来源：3070 Ti 作者（`minimax_h3_fl2v_turbo_8step_v1.0`）、larryvrh（8 步）、Abiray 基线 8–12 步【B，20/16】；lightx2v 的 HF 页面**只有引流链接，无步数、无 alpha、无文件列表**【B，17】。
- 原文条件：3070 Ti 用 4 步 preview 79.74 s（warm）/ 8 步 138.24 s / 864×480 / 124 帧【B，20】。
- 报告收益：4 步相对 20 步 baseline 3.41×；Abiray 称“可进一步降步数但需平衡 LoRA 强度”【B，16/20】。
- **关于“lightx2v 4 步 / alpha 8”**：该参数组合在 HF 模型页 `lightx2v/Minimax-h3-Turbo` **无法核实**（页面无技术参数），part1 附录 C 已列为未核实项。**本档案按【D，未核实】处理**，不作为基线依据【D，17】。可核实的只有：JUNI 文中出现 lightx2v turbo **8 步**配置（10 s / 605.7 s）与 6 步配置（5 s / cold 310.5 s、warm 185.5 s）【B，24】。
- 等级：**B**（4/8 步 LoRA 在 8GB 有第三方实测）；lightx2v 4 步 alpha 8 单独为 **D**。
- 5070 不确定点：4 步质量退化（肢体/指）在 Blackwell 上是否更严重未测【C】。
- 最小验证：基线 12 步跑通后，同分辨率同 seed 改 8 步、再 4 步，目测质量与音频【C 实验】。
- 回滚：LoRA 强度/步数改回 12 步 ema_V4【B，19】。

**8.1.2 cu130 + comfy-kitchen（attention / W4A8 后端）**

- 来源：3070 作者 cu130 后 int8 路径 1.56×（R2V 313→201 s）【B，19】；3070 Ti 用 comfy-kitchen 0.2.30 + W4A8【B，20】；note 4060 用 “comfy kitchen attention” 提速 27%（8m53s→6m30s）【B，21】。
- 原文条件：3070 cu130；4060 0.3MP/5s/Kitchen Attention【B，19/21】。
- 报告收益：cu130 ~1.5×；Kitchen Attention ~27%；W4A8 4 步 79.74 s【B，19/20/21】。
- 等级：B。
- 5070 不确定点：comfy-kitchen 在 Win11 + Blackwell 的兼容性、W4A8 DiT 是否已合入 ComfyUI 0.31–0.32（SOURCES 记 w4a8 已合）【C】。
- 最小验证：确认 torch cu130 已装；装 comfy-kitchen，单 clip 用 Kitchen Attention 对比 SDPA 耗时【C】。
- 回滚：卸载 comfy-kitchen / 改回 SDPA（无持久改动）【B】。

**8.1.3 NVFP4 / W4A8 量化变体**

- 来源：zenn 用 NVFP4 DiT 11.66GB 在 RTX PRO 2000 Blackwell Laptop 8GB 跑通 800×448/15s【B，22】；3070 Ti W4A8 mixed DiT+TE 4 步 79.74s【B，20】；Tomiigo（NVFP4 计算需 Blackwell，低于则运行时展开无加速；**模拟 8GB 上限**）【C，23】。
- 原文条件：Blackwell Laptop 8GB + NVFP4 DiT；3070 Ti W4A8【B，22/20】。
- 报告收益：NVFP4 DiT 更小（11.66GB vs int8 19.5GB），Blackwell 有硬件加速；W4A8 4 步极快【B，22/20】。
- 等级：B（Blackwell 8GB 实卡 NVFP4）/ B（3070 Ti W4A8）/ C（Tomiigo 的 Blackwell 加速陈述）。
- 5070 不确定点：**V1.0 基线用 int8 DiT；切换 NVFP4 DiT 是否为同一 Comfy-Org 仓库文件、质量差异未在本机比过**【C】。
- 最小验证：下载 NVFP4 DiT（如可用），同参数对比 int8 DiT 的 VRAM/耗时/质量【C】。
- 回滚：Loader 指回 int8；已下载的 NVFP4 文件**保留不删**，是否删除由 Junlong 决定【B】。

**8.1.4 attention 后端（SDPA / SageAttention / comfy kitchen）**

- 来源：SDPA 为 3070 稳定默认【B，19】；3070 Ti 用 SageAttention 端到端比 SDPA 短 14%、microbench 2.43×【B，20】；note 4060 用 Kitchen Attention +27%【B，21】；Tomiigo 用 KJNodes SageAttention patch（实验性，需源码编 sageattention 2，模拟环境）【C，23】。
- **SageAttention 在 sm_120 的正确表述（按 R1 修正）**：issue #15263 标题为 “SageAttention FP8 PV kernels produce noise above ~160k tokens on sm_120”；约 154k token 以下干净、167k 起噪声；作者 2026-08-09 更新称 **ComfyUI 0.31.1 上已可用**（KJNodes 的 sage 节点设 `auto`），**`sageattn_qk_int8_pv_fp16_cuda` 在 sm_120 首次调用即失败/中止，`auto` 是该架构唯一能跑的模式**【B，28】。
- 这与 5070 Laptop **同架构（sm_120）**，因此不是“是否仍有 bug 未证实”，而是“**有明确阈值 + 明确修复版本 + 明确唯一可用模式**”【B，28】。480×832/124 帧约 12k token（按 宽/32 × 高/32 × 帧/4 估算），远低于 167k，落在安全区【C（阈值换算）】。
- 报告收益：SageAttention 最快但风险随 token 数上升；Kitchen Attention 居中较稳【B，20/21】。
- 等级：B（Kitchen/Sage 实测）；B（sm_120 阈值与修复，issue 原文）；C（5070 Laptop 上实测收益）。
- 5070 不确定点：>167k token 的长片段/高分辨率仍会出噪声；本机 8GB 通常跑不到该 token 量，但 15 s + 高分辨率组合需单独验证【C】。
- 最小验证：先 SDPA 基线；再单 clip 试 Kitchen Attention；Sage 仅在 480×832/124 帧下试，且必须检查音频【C】。
- 回滚：attention 节点改回 SDPA / 移除 patch【B】。
- **3070（Ampere）的噪声报告单列**：Qiita 3070 称 SageAttention → 视频与音频全噪声，并引用 #15263，改用 SDPA【B，19】。Ampere 不属 #15263 的 sm_120 结论范围，**归因为“另一现象/旧版本”，未核实**【D（归因）】。

**8.1.5 FirstBlockCache / Spectrum**

- 来源：3070 Ti FirstBlockCache Safe 在 20 步省 271.84→211.52s（5/20 步复用）；但 Turbo 4/6/8 步时 cache 关（步少近似误差大）【B，20】；zenn 用 ComfyUI-Spectrum-MiniMax-H3 v0.2.3 节点【B，22】。
- 报告收益：非 Turbo 多步时约 22% 提速；Spectrum 跳步（SGLang 4090 配方 3.53×）【B，20/18】。
- 等级：B。
- 5070 不确定点：与 Turbo LoRA 叠加是否冲突（3070 Ti 明确 Turbo 步数下关 cache）【C】。
- 最小验证：基线 12 步 + FirstBlockCache 对比无 cache 耗时/质量【C】。
- 回滚：关 cache 节点【B】。

**8.1.6 DynamicVRAM 双流（async offload 2-stream）**

- 来源：3070 Ti：1-stream 96.85s / 2-stream 89.56s（最快，block 传输与计算重叠）【B，20】；Tomiigo 用 DynamicVRAM + async offload【C，23】。
- 报告收益：2-stream 比 1-stream 快 ~8%【B，20】。
- 等级：B。
- 5070 不确定点：与 `--disable-async-offload`（OOM 规避旗）互斥——OOM 规避与双流加速是 trade-off【C】。
- 最小验证：在已确认不 OOM 的分辨率下，去掉 `--disable-async-offload` 试 2-stream【C】。
- 回滚：加回 `--disable-async-offload`【B】。

### 8.2 V1.2 稳定性

**8.2.1 连续 N 次运行 + 内存/显存泄漏监测**

- 来源：zenn 在 124 帧下“15 本以上全完走”无掉；但 209/362 帧长片触退避崩溃【B，22】；issue #15663 短片估计偏低 OOM 且失败后“同图重复 OOM，重启无效”，插入任意无关节点或改任一参数即恢复【B，25】。
- 报告收益/风险：长片与特定图易复发 OOM；疑似预算决策按“图结构+参数”缓存且跨进程存活【B，25】。
- 等级：B。
- 5070 不确定点：5070 Laptop 8GB + 32GB RAM 下长片（>124 帧）是否触同 bug 未测【C】。
- 最小验证：同参数连跑 5 次 124 帧，记录显存/内存峰值是否单调上升（泄漏）【C】。
- 回滚：遇顽固 OOM，改任一节点参数或插入 ShowAny 节点（issue #15663 解法）【B】。

**8.2.2 sysmem fallback 策略（蓝屏根因）**

- 来源：zenn 8 次整机蓝屏（HYPERVISOR_ERROR / IOMMU）根因是 VRAM 溢出退避系统内存 + VBS DMA 保护；改“优先无系统内存回退”后 0 次，但可过长度降至 124 帧【B，22】。
- 等级：B（现象与效果）/ D（本机适用性）。
- 5070 不确定点：OMEN 16 是否启用 VBS/DMA 保护未知；若未启用可能不受影响【C/D】。
- 最小验证：先跑 124 帧确认不崩；再试 209 帧看是否崩【C】。
- 回滚：控制面板改回默认（允许回退）【B】。
- **约束**：此项属修改系统设置，须 Junlong 单独批准；第 7 章执行单不包含该操作【D（流程）】。

**8.2.3 pagefile / NVMe**

- 来源：3070 Ti 冷启动 pagefile +2GiB；warm 降至 0.02–0.15GiB；监控 pagefile 增量>0.25GiB 告警【B，20】。32GB RAM + NVMe 为硬前提（SOURCES §8；5070 Ti 有“32GB RAM 加载卡死”传闻 D 级）【D】。
- 等级：B（pagefile 实测）/ D（卡死传闻）。
- 5070 不确定点：32GB RAM 是否够 TE 32B 常驻+offload，未在本机验证【C】。
- 最小验证：跑首生成时观察任务管理器“已提交”是否逼近 32GB【C】。
- 回滚：pagefile 改回系统管理【B】（改 pagefile 需批准）。

### 8.3 V2.0 720p

**8.3.1 1344×768 / 1280×720 在 8GB 的已知数据**

- 来源：3070 Ti 1344×768 / 4 步 255.35s，峰值 6.47–6.98GiB（W4A8）【B，20】；zenn 在 Blackwell Laptop 8GB 上 800×448 为 15s 上限（832×448 即失败）【B，22】。
- 报告收益/风险：720p 直出在 W4A8+4 步下可行但慢；int8 DiT + 高分辨率在 8GB 极易 OOM【B，20/22】。
- 等级：B。
- 5070 不确定点：V1.0 用 int8 DiT（非 W4A8），720p 直出在 8GB 大概率 OOM；需换 W4A8/NVFP4 或降帧【C】。
- 最小验证：V1.1 切换 W4A8/NVFP4 后，单 clip 试 1280×720 / 4 步，记录 OOM【C】。
- 回滚：分辨率降回 480×832【B】。

**8.3.2 低分辨率生成 + 放大（upscale）**

- 来源（直接证据）：3070 Ti：864×480 direct + NVENC/Lanczos 768p 90.09s vs 1344×768 direct 255.35s，低清+放大快 2.83×（但非超分模型，未生成细节不复原）【B，20】。
- 来源（分段拼接，**按 R5 修正归属**）：**note rono_ai（RTX 4060 Laptop 8GB / 32GB RAM）** 用 Infinite workflow 生成两段 **10 秒 / 416×736**，去重叠后拼成 **约 18 秒**；作者原文：“there was a behavior where the cat's movement paused once around the 10-second mark”，即**边界附近出现一次动作停顿**；该文未给拼接耗时、未给显存峰值【B，21 + T3-note-4060-recheck】。
- **zenn cosara 的“5 秒×3 拼接”续报**：原文相关链接区确实存在续报 `https://zenn.dev/cosara/articles/minimax-h3-concat-15s`（标题意为“把 3 段 5 秒接起来再升分辨率”）。**该续报本任务未打开**，其内容、数字、结论一律不引用【D，未核实；链接见 sources/T3-zenn-cosara-recheck.md】。
- 报告收益：分段 + 放大在 8GB 上是绕开长片/高分辨率 OOM 的现实路径，3070 Ti 的“低清+放大”为 2.83× 提速【B，20】；拼接的代价是边界运动可能停顿【B，21】。
- 等级：B（低清+放大提速、18 s 拼接与停顿）/ D（zenn 续报内容）。
- 5070 不确定点：外部超分模型在 8GB 的显存余量；拼接边界停顿在 5070 Laptop 上的表现未测【C】。
- 最小验证：480×832 生成后用 Lanczos/超分节点升到 1280×720，目测细节损失与边界连贯性【C】。
- 回滚：去掉放大/拼接节点，直出低清【B】。

---

## 第 9 章 风险与待决事项

### 9.1 许可证

- **地域排除**：MiniMax H3 Community License 排除欧盟、英国、韩国、美国；**日本、中国不在排除列表**（Junlong 合规）【B，04】。
- **商用门槛**：年营收 > 2000 万美元需 MiniMax 书面授权（api@minimax.io）【B，04】。本地生成内容商用另需经 Comfy 购买 MiniMax 商用许可（Comfy 自称唯一官方转售）【B，05】——两者口径冲突，按更严格一方处理【B，04/05】。
- **其他限制**：禁止用 H3 输出改进竞品模型、禁止军事用途、公开机器生成内容需明确披露等【B，04】。
- 等级：B。

### 9.2 磁盘空间

- 最小 FL2VA 下载约 40.2–43 GB（官方列表 19.5+14.6+4.9+0.6 + LoRA）【B，08/19】；加 ref2va +19.5 GB；ComfyUI 本体 + venv + 缓存另计。需 NVMe 余量 >60 GB 为宜【C】。
- 等级：B（体积）/ C（余量建议）。

### 9.3 32GB 内存压力

- TE 为 Qwen3-VL-32B，即便 NVFP4(14.6GB)/int8(25.3GB) 仍需大量 RAM offload【B，08】。zenn 测试机有 95.5GB RAM【B，22】。
- **3070 Ti（16GB RAM）的进程 RAM 实测（按 R6 修正）**：**cold 9.74 GiB，warm 3.65 GiB**；同文表格中 **9.17 GiB 是 20 步 baseline 一行的 Comfy RAM**，不是 cold 峰值——两者不可混用【B，20】。
- 5070 Laptop 仅 32GB，TE 常驻 + offload + 系统 + 桌面共享 8GB 显存，余量紧张【C】；SOURCES §8 记 5070 Ti 16GB 有“32GB RAM 加载卡死”传闻（未证实）【D】。
- 缓解：pagefile 扩容（NVMe，需批准）、`--disable-pinned-memory` 防页锁 90% RAM【B，25/23】。
- 等级：B（机制与实测）/ C（32GB 是否够）/ D（卡死传闻）。

### 9.4 笔记本供电 / 散热

- 供电：zenn 同生成 655.8s→237.2s（2.76×）仅因换 140W 原厂供电；USB-C 低功率坞触发 SW Power Cap（25W）严重降频【B，22】。OMEN 16 须用原厂大功率适配器【B/C，22（迁移推断）】。
- 散热：笔记本持续高负载易降频；建议外接供电、垫高散热、监控 `nvidia-smi -q -d PERFORMANCE` 的 Power Cap【B/C，22】。
- 等级：B（外部实测）/ C（迁移到 OMEN 16）。

### 9.5 需要 Junlong 批准的操作

> 以下 9 项均为流程项，等级【D】（本机未执行、未验证），不是技术结论。

1. 下载 ~40GB 模型权重（第 7 章步骤 6）【需批准】。
2. 升级 torch 到 cu130（步骤 3 给出的命令，**本单不执行**）【需批准】。
3. 改动 ComfyUI（`git pull` / 同步 requirements / 装 comfy-kitchen）（步骤 5、8.1.2）【需批准】。
4. 修改 NVIDIA 控制面板“系统内存回退策略”（6.5、8.2.2）——**属修改系统设置，规格第七节明确禁止擅改**【需批准】。
5. 修改 pagefile / 电源计划（8.2.3、9.4）【需批准】。
6. 执行首个真实生成（可能触发整机蓝屏重启，zenn 8 次先例）【B，22】【需批准】。
7. 切换量化变体（NVFP4/W4A8 DiT）与 attention 后端（SageAttention 有 token 阈值噪声风险）【B/C，28】【需批准】。
8. **删除任何文件**（含 `.venv-failed-*`、下载失败的不完整文件、旧备份）【需批准】。
9. **更新/回滚显卡驱动**——不在本档案范围，规格第七节禁止；如需变更由 Junlong 自行决定【禁止执行】。

---

## 第 10 章 审核修正记录（R1–R10）

> 每条给出：编号、原位置与原文（摘要/逐字）、改后内容、依据（sources 文件或 URL）。凡“原文”为摘要者标注“摘要”。

### R1 · #15263 的表述与适用范围

| 项 | 内容 |
|---|---|
| 位置 | 第 5 章 5.2；第 6 章 6.2 第 13 行（attention）；第 8 章 8.1.4 |
| 原文（摘要） | 5.2 标题“SageAttention 出纯噪声（视频+音频）”；6.2 第 13 行依据写“SageAttention 在 3070 出纯噪声（#15263 转述）【C】”；8.1.4 写“SageAttention 在 sm_120 是否仍有噪声 bug 未证实”，并称 3070 噪声报告与 #15263“不完全一致” |
| 问题 | 把一条 **sm_120 专属、有明确 token 阈值和明确修复版本**的 issue，写成了“3070 上的现象 + sm_120 未证实”；对 5070 Laptop（sm_120）恰好把最相关的结论降级成了不确定 |
| 改后 | 5.2 改为“sm_120 专项”：标题逐字引用 “SageAttention FP8 PV kernels produce noise above ~160k tokens on sm_120”；写明 154k 以下 clean / 167k 起 noise、作者 2026-08-09 更新“**This works now on ComfyUI 0.31.1**”（KJNodes 节点设 `auto`，54 min vs 94 min）、`sageattn_qk_int8_pv_fp16_cuda` 在 sm_120 首次调用即失败/中止、`auto` 是该架构唯一可运行模式；6.2 第 13 行依据同步改写；8.1.4 删除“是否仍有 bug 未证实”，改为“有明确阈值 + 修复版本 + 唯一模式”，并注明 5070 Laptop 为同架构。**3070（Ampere）的噪声报告单列为“另一现象，未核实”** |
| 依据 | `sources/28-comfyui-issue-15263.md`（issue #15263 原文与标题）；`sources/19-qiita-3070-8gb.md`（3070 作者“SageAttention → 全噪声，用 SDPA”） |
| 等级变化 | 原【C】→ 改后【B】（issue 原文 + 作者复测）；3070 归因保持【D】 |

### R2 · 第 7 章步骤 1 的驱动处理

| 项 | 内容 |
|---|---|
| 位置 | 第 7 章 步骤 1（原） |
| 原文（逐字） | “**失败停在哪**：若找不到 nvidia-smi 或驱动非 616.92 → 停，先装/更新驱动，**不要继续**。” |
| 问题 | 违反规格第七节“禁止改驱动/系统配置”，且把 616.92 当作硬门槛（该值是 PC 回报，未经复核） |
| 改后 | 步骤 1 改为**只读记录**：`nvidia-smi --query-gpu=name,driver_version,memory.total`；预期只要求“能看到 GPU 名与驱动版本”；失败条件改为“找不到 nvidia-smi → 停并报告”“**驱动 < 580 → 停，把实测版本报告 Junlong，等指示**”；并明写“本单不执行任何驱动安装、更新、回滚操作” |
| 依据 | 任务规格第七节（禁止改驱动/系统配置）；`sources/T2-*` 无驱动硬门槛来源，616.92 仅为 PC 回报值【D】 |
| 等级 | 流程项【D】，不含技术断言 |

### R3 · 第 7 章步骤 9 的删除操作

| 项 | 内容 |
|---|---|
| 位置 | 第 7 章 步骤 9（原，现为步骤 11） |
| 原文（逐字） | `Remove-Item "$ComfyRoot\.venv" -Recurse -Force` 然后 `Expand-Archive ...` |
| 问题 | 直接递归强删用户环境目录，违反“不得删除用户文件”；且 `-Force` 会连带删除回滚后仍可能需要的内容 |
| 改后 | 步骤 11 改为：`Rename-Item -LiteralPath "$ComfyRoot\.venv" -NewName ".venv-failed-$ts"` → `Expand-Archive` 备份到 `.venv`；明写“原目录**保留，不删除**”“**任何删除（含 `.venv-failed-*`）都必须先获 Junlong 批准，本单不执行删除**”；步骤 4 的 `Compress-Archive -Force` 也去掉了 `-Force` |
| 依据 | 任务规格（禁止删除用户文件）；R3 审核意见 |
| 等级 | 流程项【D】 |

### R4 · 第 7 章缺少“venv 实际路径”与“torch/CUDA 版本”检查

| 项 | 内容 |
|---|---|
| 位置 | 第 7 章（原步骤 1–9） |
| 原文（摘要） | 直接把 `.venv` 当既定事实；全章没有任何 torch/CUDA 版本检查步骤 |
| 问题 | PC 回报只说“源码 + venv”，路径与 cu130 均为假设；cu128 下 int8 只有 1/3 速度，若不自检会把环境问题误判为模型问题 |
| 改后 | 新增**步骤 2**（只读）：`Test-Path $ComfyRoot`、列出 `*venv*`/`main.py`、`python -c "import sys;print(sys.prefix);print(sys.executable)"`；约定“后续命令中的 `.venv` 一律替换为实测路径”，路径不符/无 venv → 停并报告（新建 venv 需批准）。新增**步骤 3**（只读）：`python -c "import torch;print(torch.__version__, torch.version.cuda, torch.cuda.get_device_name(0))"`；**若 `torch.version.cuda` ≠ 13.0 → 停**，把“升级 torch cu130”列为需批准操作，并写出命令（`pip install --index-url https://download.pytorch.org/whl/cu130 torch torchvision torchaudio`）但**明确不执行**。原步骤 3–9 顺延为 5–11，步骤编号在文内已更新 |
| 依据 | 任务 R4 指定的检查命令；cu128 性能损失见 `sources/19-qiita-3070-8gb.md`（313.4s → 200.9s，1.56×）；Comfy-Org 建议 cu130 见 `sources/07-hf-comfy-org.md` |
| 等级 | 命令与判据来自上游文档【B】；“本机是否 cu130”为【D，未核实】 |

### R5 · 第 8 章 8.3.2 的拼接来源错配

| 项 | 内容 |
|---|---|
| 位置 | 第 8 章 8.3.2 |
| 原文（摘要） | “zenn 续报‘5秒×3 拼接升分辨率’【B】”；“拼接边界运动停顿（zenn 18s 拼接有 1 次停顿）【C】” |
| 问题 | “约 18 秒拼接”和“10 秒附近动作停顿”实际出自 **note rono_ai（RTX 4060 Laptop）**，不是 zenn；“5 秒×3”又是 zenn 的另一篇续报，被混成一条 |
| 改后 | 拆成两条来源：① **note rono_ai**【B，21】：两段 10 s / 416×736，去重叠后约 18 s，原文 “the cat's movement paused once around the 10-second mark”；② **zenn cosara 续报**：给出链接 `https://zenn.dev/cosara/articles/minimax-h3-concat-15s`，明确写“**本任务未打开**，内容/数字不引用”【D】。原“zenn 18s 拼接有 1 次停顿”整句删除 |
| 依据 | `sources/T3-note-4060-recheck.md`（原文逐字）；`sources/T3-zenn-cosara-recheck.md`（续报链接存在于原文“関連リンク”区，续报本身未打开）；`sources/21-note-4060-laptop.md` |
| 等级 | 18 s / 停顿【B】；zenn 续报内容【D，未核实】 |

### R6 · 第 9 章 9.3 的 RAM 数字错位

| 项 | 内容 |
|---|---|
| 位置 | 第 9 章 9.3 |
| 原文（逐字） | “3070 Ti 16GB RAM 下 Comfy 进程峰值 9.17GB（cold）【B】” |
| 问题 | 9.17 GiB 是 3070 Ti 基准表中 **20 步 baseline 行**的 “Comfy RAM”，不是 cold 峰值；cold 进程 RAM 是 **9.74 GiB** |
| 改后 | 改为“**cold 9.74 GiB，warm 3.65 GiB**；同文表格中 **9.17 GiB 是 20 步 baseline 一行的 Comfy RAM**，不是 cold 峰值——两者不可混用” |
| 依据 | `sources/20-qiita-3070ti-8gb.md`：基准表 `baseline / 20 / 271.84s / ... / 9.17GiB`；正文“进程 RAM：warm 3.65GiB，cold 9.74GiB” |
| 等级 | B |

### R7 · 第 8 章 8.1.1 把 lightx2v 参数标为 B

| 项 | 内容 |
|---|---|
| 位置 | 第 8 章 8.1.1 |
| 原文（逐字） | “larryvrh（8 步）、lightx2v（4 步 alpha 8）【B，3070 Ti 文末链接】”；“等级：B（8GB 有第三方实测）” |
| 问题 | `lightx2v/Minimax-h3-Turbo` 的 HF 页面只有引流链接，无步数/alpha/文件列表；part1 附录 C 已把“4 步 / alpha 8”列为无法核实，part2 却按 B 引用 |
| 改后 | 明写“**该参数组合在 HF 模型页无法核实，按【D，未核实】处理**，不作为基线依据”；仅保留可核实项：JUNI 文中出现 lightx2v turbo **8 步**（10 s / 605.7 s）与 6 步（5 s / cold 310.5 s、warm 185.5 s）【B，24】；8.1.1 的总体等级仍为 B（4/8 步 LoRA 有第三方实测），lightx2v 4 步 alpha 8 单列 D |
| 依据 | `sources/17-hf-lightx2v-turbo.md`（“页面无步数、无 alpha…不能从本页核实”）；`sources/24-note-juni-3060ti.md`（8 步/6 步实测）；part1 附录 C 第 5 条 |
| 等级变化 | 原【B】→ 改后【D，未核实】 |

### R8 · 第 6 章 6.1 把 Tomiigo 的陈述标为 B

| 项 | 内容 |
|---|---|
| 位置 | 第 6 章 6.1（技术假设的推理链） |
| 原文（逐字） | “Tomiigo 指南明确指出：低于 Blackwell 时 `supports_nvfp4_compute` 返回 false…【B，Tomiigo 源码级陈述】” |
| 问题 | Tomiigo 的数据来自 **RTX 5070 Ti 16GB 上用 sidecar 把可用显存压到 8,188 MiB 的模拟**，原文自述 “No physical 8 GB card was used”“Timings here will not transfer to a real 8 GB card”；按本档案的等级定义，模拟环境不构成 B（外部可复现的 8GB 结果） |
| 改后 | 统一改为 **【C，23】**，并在句中直接引出上述两句原文自述；第 4 章表后补充说明也由“B/C”改为“**C，不混入上表**”；6.2 第 4、14 行的 Tomiigo 依据同步标 C |
| 依据 | `sources/23-github-tomiigo-sim8gb.md`（逐字原文）；第 4 章筛选标准（“Tomiigo 的 8,188 MiB 是在 16GB 卡上模拟，单列在表后”） |
| 等级变化 | 原【B】→ 改后【C】 |

### R9 · ComfyUI 版本建议不统一

| 项 | 内容 |
|---|---|
| 位置 | 第 3 章 3.1；第 6 章 6.1、6.2 第 1 行 |
| 原文（摘要） | part1：“给 5070 Laptop 的建议版本区间：0.31.1–0.32.x…这是推断【C】”；part2 6.2：“≥0.30.0…**建议最新稳定版**（实测 0.31/0.32/0.33/0.37 均有）”【B】 |
| 问题 | 两个区间互相矛盾；part2 的“最新稳定版”没有说明理由，也回避了 0.30.0 在 8GB 不可用、以及 0.33+ 的四个已知故障 |
| 改后 | 全文统一为“**推荐 0.31.1–0.32.x，等级 C**”，并在 3.1 给出 5 条理由（0.30.0 不可用 #15251；0.31.1 修 sm_120 Sage 噪声；0.32.0 有 3070 Ti 8GB 实测；0.33 #15663、0.34–0.35 #16175、0.37.2 aimdo 均为新变量）；6.2 第 1 行改为同一区间并把等级从 B 降为 **C**；另加“反对直接用最新稳定版”的说明；**“0.30.0 在 8GB 上不可用（#15251）”在三处（3.1、6.1/6.2、5.1）均保留** |
| 依据 | `sources/29-comfyui-issue-15251.md`（0.30.0 不可用）；`sources/28-comfyui-issue-15263.md`（0.31.1 可用）；`sources/20-qiita-3070ti-8gb.md`（0.32.0 实测）；`sources/25-comfyui-issue-15663.md`（0.33/0.37.2）；`sources/27-comfyui-issue-16175.md`（0.34/0.35） |
| 等级变化 | 原【B】（“最新稳定版”）→ 改后【C】（区间推断），各版本故障事实仍为【B】 |

### R10 · 链接核实表自相矛盾（note JUNI）

| 项 | 内容 |
|---|---|
| 位置 | part2 第 10 章表注；合并后第 11 章 |
| 原文（逐字） | “未打开的 SOURCES 条目（**note JUNI**、ezlaunch、第三方榜单站）保持 D 级未核实，未写入结论。” |
| 问题 | note JUNI 在 part1 已打开（`sources/24`），且第 2.4、4 章已引用其 39.55GB 与 185.5 s 数据——同一份档案里“已引用其数字”与“未打开”并存 |
| 改后 | 合并为**一张总表（第 11 章）**，note JUNI 状态记为“**已打开可读（sources/24）**”，等级 B；删除“note JUNI 未打开”的表述；ezlaunch 与榜单站按实际打开状态单列（均为已打开，等级 B/C 视语气而定）；Civitai 保留“无法访问” |
| 依据 | `sources/24-note-juni-3060ti.md`（已打开，含环境/时间/峰值/依赖 PR #15446）；part1 附录 A 第 25 行 |
| 等级 | B |

### F2 · 第 7 章步骤 3 的 torch 升级命令固定版本

| 项 | 内容 |
|---|---|
| 位置 | 第 7 章 步骤 3（被注释掉的升级命令，本单不执行） |
| 原文（逐字） | `# & "<venv>\Scripts\python.exe" -m pip install --index-url https://download.pytorch.org/whl/cu130 torch torchvision torchaudio` |
| 问题 | 未固定版本，执行时会拉取 cu130 通道的**最新** torch/torchvision/torchaudio；若该通道未来发布不兼容版本，可能破坏现有 venv，且与 Comfy-Org 实测所用的 2.11.0/2.13.0 区间不符 |
| 改后 | 改为固定版本：`# & "<venv>\Scripts\python.exe" -m pip install --index-url https://download.pytorch.org/whl/cu130 torch==2.11.0+cu130 torchvision==0.26.0+cu130 torchaudio==2.11.0+cu130`。出处：Qiita 3070 文（sources/19）；该文实测 torch 2.11.0+cu130 可用 |
| 依据 | `sources/19-qiita-3070-8gb.md`（2.11.0+cu130 实测）；Comfy-Org 建议 cu130 见 `sources/07-hf-comfy-org.md` |
| 等级 | 流程项【D】（本单不执行）；版本号来源为 B 级 |

> 说明：本修改为**纯注释文本**修正，不改变第 7 章“本单不执行 torch 升级”的约束；是否执行仍属需 Junlong 批准的操作（见第 9 章 9.5 第 2 项）。

---

## 第 11 章 合并后总链接核实表（去重）

> 读取时间统一为 2026-11-04（本机无 `date` 权限，未能记录精确时分；以会话日期为准）。本表合并 part1 附录 A（40 条 + 7 条补充）与 part2 §10（12 条，全部与前者重复），并按 R10 以“实际打开状态”为准。共 **48 条**：**46 条已打开可读、1 条无法访问（Civitai）、1 条已知链接但未打开（zenn 续报）**。

| # | URL | 状态 | 留底 |
|---|---|---|---|
| 1 | https://www.minimax.io/news/minimax-h3-open-source | 已打开可读 | sources/01 |
| 2 | https://huggingface.co/MiniMaxAI/MiniMax-H3 | 已打开可读 | sources/02 |
| 3 | https://github.com/MiniMax-AI/MiniMax-H3/ | 已打开可读 | sources/03 |
| 4 | https://huggingface.co/MiniMaxAI/MiniMax-H3/raw/main/LICENSE | 已打开可读 | sources/04 |
| 5 | https://docs.comfy.org/tutorials/video/minimax/minimax-h3 | 已打开可读 | sources/05 |
| 6 | https://platform.minimax.io/docs/guides/local-deploy-h3 | 已打开可读 | sources/06 |
| 7 | https://huggingface.co/Comfy-Org/MiniMax-H3 | 已打开可读 | sources/07 |
| 8 | https://comfyui-wiki.com/en/news/2026-08-03-minimax-h3-open-weights-comfyui | 已打开可读 | sources/08 |
| 9 | https://huggingface.co/unsloth/MiniMax-H3-GGUF | 已打开可读 | sources/09 |
| 10 | https://huggingface.co/joeygambino/MiniMax-H3-GGUF | 已打开可读 | sources/10 |
| 11 | https://github.com/jlucasmcrell/comfyui-h3-multishot | 已打开可读 | sources/11 |
| 12 | https://huggingface.co/DiffSynth-Studio/MiniMax-H3-NF4 | 已打开可读 | sources/12 |
| 13 | https://github.com/modelscope/DiffSynth-Studio/blob/main/docs/en/Pipeline_Usage/VRAM_management.md | 已打开可读 | sources/13 |
| 14 | https://zenn.dev/nakanishitf/articles/bec3dc44dd1b65?locale=en | 已打开可读 | sources/14 |
| 15 | https://huggingface.co/Kijai/MiniMax-H3_comfy | 已打开可读（文件页未展开） | sources/15 |
| 16 | https://huggingface.co/Abiray/MiniMax-H3-Turbo-Lora-Pruned-ComfyUI | 已打开可读 | sources/16 |
| 17 | https://huggingface.co/lightx2v/Minimax-h3-Turbo | 已打开可读（**无技术参数**，4 步/alpha 8 未核实） | sources/17 |
| 18 | https://docs.sglang.io/cookbook/diffusion/MiniMax/MiniMax-H3 | 已打开可读（末尾截断） | sources/18 |
| 19 | https://qiita.com/Kiyoaki_SUgiyama/items/bff51bdf389a732dc87c | 已打开可读 | sources/19 |
| 20 | https://qiita.com/makuragadekasugiru810114514/items/471ecc2c041e431359c6 | 已打开可读 | sources/20 |
| 21 | https://note.com/rono_ai/n/n982a96de3812?hl=en | 已打开可读（T3 复核过） | sources/21、T3-note-4060-recheck |
| 22 | https://zenn.dev/cosara/articles/vram8gb-minimax-h3-15s | 已打开可读（T3 复核过） | sources/22、T3-zenn-cosara-recheck |
| 23 | https://civitai.com/models/2852849/…（无限续写工作流） | **无法访问**（Cloudflare 拦截，3 次） | sources/33 |
| 24 | https://github.com/Tomiigo/minimax-h3-16gb | 已打开可读（**模拟 8GB，非实卡**） | sources/23 |
| 25 | https://note.com/j_alpha_lab/n/n2661aa9a31cd?hl=en（note JUNI） | **已打开可读**（R10 修正） | sources/24 |
| 26 | https://github.com/yume-arasaki/ezlaunch-minimax-h3 | 已打开可读 | sources/32 |
| 27 | https://github.com/Comfy-Org/ComfyUI/issues/15663 | 已打开可读 | sources/25 |
| 28 | https://github.com/Comfy-Org/ComfyUI/pull/15456 | 已打开可读（**是否合并未核实**） | sources/26 |
| 29 | https://github.com/Comfy-Org/ComfyUI/issues/16175 | 已打开可读 | sources/27 |
| 30 | https://github.com/Comfy-Org/ComfyUI/issues/15263 | 已打开可读 | sources/28 |
| 31 | https://github.com/Comfy-Org/ComfyUI/issues/15251 | 已打开可读 | sources/29 |
| 32 | https://github.com/Comfy-Org/ComfyUI/pull/15224 | 已打开可读 | sources/30 |
| 33 | https://github.com/Comfy-Org/ComfyUI/issues/15326 | 已打开可读 | sources/31 |
| 34 | https://github.com/leejet/stable-diffusion.cpp | 已打开可读 | sources/34 |
| 35 | https://github.com/leejet/stable-diffusion.cpp/blob/master/docs/minimax_h3.md | 已打开可读 | sources/35 |
| 36 | https://huggingface.co/AX1Y2JP/MiniMax-H3-W4A8-ConvRot | 已打开可读（模型卡为空壳） | sources/36 |
| 37 | https://github.com/modelscope/DiffSynth-Studio | 已打开可读 | sources/37 |
| 38 | https://whichh3.com/en/workflows/low-vram-gguf | 已打开可读（二手榜单，C） | sources/38（38.1） |
| 39 | https://www.genvidkit.com/blog/minimax-h3-gguf | 已打开可读（二手榜单，C） | sources/38（38.2） |
| 40 | https://openclawdc.com/blog/can-i-run-minimax-h3-locally/ | 已打开可读（二手站，“80GB RAM”为 D 级转述） | sources/38（38.3） |
| 41 | https://localrigcheck.com/minimax-h3/ | 已打开可读（“lab run / memory capped”，C） | sources/38（38.4） |
| 42 | https://minimaxh3.cc/guides/hardware-requirements | 已打开可读（“12GB 配方”实为 2TB RAM 模拟，C） | sources/38（38.5） |
| 43 | https://vidirect.org/models/hailuo-3-h3 | 已打开可读（二手站） | sources/38（38.6） |
| 44 | https://github.com/Wan-Video/Wan2.1 | 已打开可读（替代模型，非 H3） | sources/39（39.1） |
| 45 | https://github.com/Lightricks/LTX-Video | 已打开可读（替代模型，非 H3） | sources/39（39.2） |
| 46 | https://github.com/Wan-Video/Wan2.2 | 已打开可读（替代模型，非 H3） | sources/39（39.3） |
| 47 | https://github.com/city96/ComfyUI-GGUF | 已打开可读 | sources/40 |
| 48 | https://zenn.dev/cosara/articles/minimax-h3-concat-15s | **已知链接、未打开**（“5 秒×3 拼接”续报；内容不引用） | T3-zenn-cosara-recheck（仅记录链接与标题） |

**统计**：46 条已打开可读，1 条无法访问（Civitai，第 23 行），1 条已知但未打开（zenn 续报，第 48 行）。
**去重说明**：part2 §10 的 12 条全部与 part1 附录 A / 补充表重复，已并入本表；重复项不另计。part1 附录 A 称“40 个 URL”（39 可读 + Civitai 1 条拦截）加 7 条补充 = 46 条可读，本表按实际 URL 重排为 46 已打开 + 1 无法访问 + 1 未打开 = 48。

**未打开/无法访问的条目对结论的影响**：
1. Civitai 无限续写工作流：其“18 秒拼接”只在 note rono_ai 文中出现【B，21】，本表不把它算作 Civitai 已核实内容【D】。
2. zenn 续报：只记录链接与标题，其“3×5 秒”、分辨率提升幅度、耗时等一概不引用【D】。

---

## 附录 A 证据强度自评

| 结论 | 等级 | 理由 |
|---|---|---|
| H3 是 33B 单流 DiT + 13B AdaLN + Qwen3-VL-32B 第 50 层 | B | 官方公告与模型卡逐字一致 |
| 排除地域为 EU/UK/KR/US，中国/日本可用 | B | 许可证原文 |
| 8GB 可生成 480p/5s | B | 5 张卡、5 个作者、7 组配置；但无 5070 Laptop |
| Turbo LoRA 4–8 步可用 | B | 多篇实测，但有质量退化与音频伪影 |
| NF4 的 “8GB” 是卸载下限 | B | 模型卡原文 + T4 只跑通 288×160 |
| SageAttention 在 sm_120 上 >167k token 出噪声，0.31.1 起可用 | B | issue 原文 + 作者复测（1920×1088/15s，54 min） |
| 0.30.0 在 8GB 上不可用 | B | #15251 + zenn 实测 |
| 0.31.1–0.32.x 是 8GB 最佳版本区间 | **C** | 由多条 issue 时间线反推，无官方矩阵 |
| 12GB SGLang 配方适用于笔记本 | **D** | 是 2TB 主机上的模拟，且无 Windows |
| 5070 Laptop 能达到 3070 Ti 的 80 s/5s | **D** | 无任何公开实测，纯外推 |
| NVFP4 在 sm_120 上有实际加速 | C | 门槛满足（CC≥10.0），但无 5070 Laptop 对照 |
| Tomiigo 的 8GB 结论可直接迁移到实卡 | **D** | 模拟环境，原文自述 “Timings here will not transfer to a real 8 GB card” |
| 两段 10s→约 18s 拼接、边界停顿一次 | B | note rono_ai 原文逐字 |
| zenn “5 秒×3 拼接”续报的内容 | **D** | 仅有链接，未打开 |

## 附录 B 未完成 / 未核实项

1. **无 5070 Laptop 实测**：全网检索中未发现该机型的 H3 数据；第 4 章全部是 3070/3070 Ti/3060 Ti/4060 Laptop/RTX PRO 2000 Laptop【D】。
2. **Civitai 无限续写工作流**无法访问；“拼接 18 秒”仅在 note 4060 文中出现【B，21】，未在该页面核实【D】。
3. **ComfyUI PR #15456 是否合并**未确认（页面显示 “wants to merge”）【D】。
4. **SGLang cookbook 后半部分**（VDN-H3、5090 单卡、§10 ComfyUI）因抓取截断未读到【D】。
5. **lightx2v “4 步 / alpha 8”** 在 HF 模型页无法核实（页面只有引流链接）→ 全文按【D】处理。
6. **AX1Y2JP W4A8 的文件名与大小**未能从模型页取得（模型卡为空壳）；3070 Ti 文中的文件名只能视为该作者的用法记录【D】。
7. 第 4 章第 4、6、7 行的部分单元格原文确实未给（步数/显存/帧数），已标“未写”，未做任何填补【B（原文缺失本身）】。
8. **zenn cosara 的“5 秒×3 拼接”续报**（https://zenn.dev/cosara/articles/minimax-h3-concat-15s）只有链接，未打开；其内容、数字、结论均未引用【D】。
9. **本机 venv 实际路径、torch/CUDA 版本、ComfyUI 版本、实际驱动版本**全部未核实，均需按第 7 章步骤 1–3 只读确认【D】。
10. **3070（Ampere）的 SageAttention 噪声**与 #15263 的 sm_120 结论是否同一根因，未核实【D】。

---

*（本档案完。第 7 章为草稿、未执行；全文无 A 级证据，无“本机验证成功”表述。）*
