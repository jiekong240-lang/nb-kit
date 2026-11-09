# R4-01｜LightX2V / ModelTC MiniMax-H3 Turbo 规格 + H4「alpha 8」核实

- URL（本轮实际打开，三个）：
  - https://github.com/ModelTC/Minimax-H3-Turbo
  - https://comfyui-wiki.com/en/news/2026-08-11-minimax-h3-turbo-lightx2v-v1
  - https://huggingface.co/lightx2v/Minimax-h3-Turbo/raw/main/README.md （HF 卡 raw）
  - https://huggingface.co/lightx2v/Minimax-h3-Turbo （HF 页面本体）
- 读取时间：2026-11-08 JST
- 读取方式：WebFetch 直连打开（HF 页面/raw 两次均因 AI 摘要超时返回原始正文，仍为直读）。
- 类别：LOG-004 (1) 蒸馏 LoRA；H4 假设检验。

## 1. ModelTC/Minimax-H3-Turbo GitHub 规格表（逐字）

仓库标题：`Distill Minimax-H3 into 4 steps`；文件时间：`README.md` / `LICENSE Initial commit Aug 7, 2026`，`DIFFUSERS_SETUP_AND_INFERENCE.md feat: ref2va … Aug 13, 2026`。

> ## 1. Model specs
> | Model | Tasks | Training resolution | Training shifts (video / audio) | Distillation steps (NFE) | Recommended inference steps (NFE) |
> | **FL2VA Turbo 4-step v0.1** Diffusers · ComfyUI | FL2VA / T2VA | 544p mixed aspect ratio | 12 / 3 | 4 | 4 |
> | **FL2VA Turbo 8-step v1.0** | FL2VA / T2VA | 544p mixed aspect ratio | 12 / 3 | 8 | 8 / 4 |
> | **FL2VA Turbo 4-step v1.0 768p** | FL2VA / T2VA | 768p 1344×768 | 6 / 3 | 4 | 4 |
> | **FL2VA Turbo 8-step v1.0 768p** | FL2VA / T2VA | 768p 1344×768 | 6 / 3 | 8 | 8 |
> | **Ref2VA Turbo 4-step v0.1** | Ref2VA | 544p mixed aspect ratio | 12 / 3 | 4 | 4 |

> ### Note on shift
> For `NFE = N`, define the N transformer evaluation points on the unshifted grid as `q_i = (N - i) / N` … For example, with `NFE = 4`, `video shift = 12`, and `audio shift = 3`, the shared grid is `q = [1, 0.75, 0.5, 0.25]`, giving video sigma `[1, 0.9730, 0.9231, 0.8000] -> 0` and audio sigma `[1, 0.9000, 0.7500, 0.5000] -> 0`; each list therefore uses exactly four NFEs.

> ### 2. Online Studio and API
> The Studio currently uses the FL2VA Turbo 8-step v1.0 768p LoRA.
> - **Current model:** FL2VA Turbo 8-step v1.0 768p
> - **Recommended settings:** 8 NFE, video shift 6, audio shift 3

（另：三种参考图缩放策略 `match` / `max` / `diffusers`，蒸馏训练用 `match`；`--reference-resize-mode` 默认 `match`。）

## 2. comfyui-wiki 2026-08-11（逐字）

> Last updated: August 11, 2026 ComfyUI Wiki news
> On August 11, the **Lightx2v** team and **ModelTC** released **v1.0** of their **MiniMax H3 Turbo LoRA**, following the early v0.1 preview from August 7.

> **8-step v1.0** (`minimax_h3_fl2v_turbo_8step_v1.0_bf16.safetensors`): distilled at 8 NFE with 12/3 video/audio shifts, and usable at 8 or 4 inference steps.
> **4-step v1.0 768p** (`minimax_h3_fl2v_turbo_4step_v1.0_768p_bf16.safetensors`): a 4-step distillation trained at 768p (1344x768) with 6/3 shifts …
> Each checkpoint has a ComfyUI-format twin (`..._comfyui_bf16.safetensors`) hosted in the same repository, so no separate conversion is needed.
> For the 8-step checkpoint, run 8 sampling steps (4 also works); for the 768p variant, run 4 steps at 768p resolution.

## 3. HF 卡页（H4 核实，逐字）

`https://huggingface.co/lightx2v/Minimax-h3-Turbo/raw/main/README.md` 全文（仅有的技术内容）：

> ---
> library_name: minimax-h3
> tags: [video-generation, audio-video-generation, distillation, lora, lightx2v]
> ---
> # MiniMax-H3 Turbo
> > 🌐 **[Create online with LightX2V Studio](https://x2v.light-ai.top/)** — generate videos with audio from text, images, or reference materials. No local installation or GPU setup required.
> For local inference and reproducibility, see the [MiniMax-H3 Turbo GitHub repository](https://github.com/ModelTC/Minimax-H3-Turbo), [LightX2V MiniMax-H3 examples](https://github.com/ModelTC/LightX2V/blob/main/scripts/minimax_h3/README.md), and [model specifications](https://github.com/ModelTC/Minimax-H3-Turbo#model-specs).
> ## Online App: LightX2V Studio … ## Online API: LightX2V Studio …

页面本体另可见：`Downloads last month 1,793,272`、`Community 60`、`Spaces using … 100`、tag `distillation`。

**H4 判定（逐字结论）**：卡页（含 raw README）**没有任何 `alpha` / `alpha 8` / `network_alpha` 字样，也没有步数、shift、分辨率、帧数、耗时、显存的技术参数表**；它已变成**在线 Studio 引流页 + 三个外链**。`SOURCES.md` 第 34 行所记「lightx2v/Minimax-h3-Turbo（4 步，alpha 8）」在本轮打开的卡页中**无法核实** → **保持 D，不得升格**。技术参数只存在于 **GitHub 仓库**（本文件第 1 节）与**其它 HF 卡**（larryvrh：`alpha = rank`；alibaba-pai：`rank=64 and network_alpha=64`），**均不是 "alpha 8"**。

## 4. 事实 / 推测 / 建议

- **事实（B，直读）**：5 个 Turbo 检查点的 NFE / shift / 训练分辨率 / 推荐推理步数；v1.0 发布 2026-08-11、v0.1 预览 08-07；8-step 检查点可用 8 或 4 步；HF 卡已无技术参数、无 "alpha 8"。
- **推测（C）**：「4/8 步在 5070 Laptop 8GB 上的耗时与画质」——本页无 8GB 数字，**未核实**；"alpha 8" 的出处（SOURCES.md 二手笔记）**未核实 → D**。
- **建议（非结论）**：本机先用 8-step 检查点、8 NFE、shift 6/3（768p 版）作基线；4 步仅作 prompt 筛选；任何"alpha 8"参数组合在找到一手卡页前不写入配置。

## 5. 等级

规格表 / 发布记录 = **B（官方仓库 + 发布页直读）**；HF 卡「4 步 / alpha 8」= **D（本轮仍无法核实，保持 D）**；对 8GB 的耗时/画质外推 = **C**。
