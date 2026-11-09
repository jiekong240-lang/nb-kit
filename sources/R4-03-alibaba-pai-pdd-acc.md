# R4-03｜alibaba-pai MiniMax-H3-Acc-LoRAs（PDD 8-step 蒸馏）

- URL：https://huggingface.co/alibaba-pai/MiniMax-H3-Acc-LoRAs/raw/main/README.md
- 读取时间：2026-11-08 JST
- 读取方式：WebFetch 直连打开（返回 7293 字符原始正文全文）。
- 类别：LOG-004 (1) 蒸馏 LoRA；(5) 近月蒸馏方法（PDD）。

## 逐字摘录

> # MiniMax-H3-Acc-LoRAs
> ## Introduction
> We apply **Parallel Decoding Distillation (PDD)** [1] to MiniMax-H3, enabling efficient video generation in only a few inference steps.
> For more details, please refer to our GitHub repo (https://github.com/aigc-apps/VideoX-Fun).

两个权重（逐字）：

> | MiniMax-H3-FL2VA-Acc-8Step.safetensors | MiniMax-H3 (FL2VA) | Official 8 Step Acc LoRA (**`rank=64` and `network_alpha=64` in BF16**) for MiniMax-H3 (FL2VA). |
> | MiniMax-H3-Ref2VA-Acc-8Step.safetensors | MiniMax-H3 (Ref2VA) | Official 8 Step Acc LoRA (**`rank=64` and `network_alpha=64` in BF16**) for MiniMax-H3 (Ref2VA). |

Demo（逐字要点，供 H1 画质对照）：

> ### FL2VA (768p) — 三列对比：`MiniMax-H3-FL2VA`（baseline） / `Minimax-h3-Turbo (fl2v_turbo_4step_v1.1_768p)` / `MiniMax-H3-FL2VA-Acc-8Step`
> ### Ref2VA — 三列对比：baseline / `Minimax-h3-Turbo (ref2v_turbo_4step_v0.1)` / `MiniMax-H3-Ref2VA-Acc-8Step`
> The above test cases are from ModelTC/Minimax-H3-Turbo/tree/main/examples. **Videos are generated with a LoRA weight of 1.0 at both 4 and 8 NFE.**

用法（逐字）：

> Set `model_path` and `pdd_lora_path` … in `predict_t2v.py` / `predict_ref2v.py` … Each example uses `apply_pdd_lora` to load the checkpoint and derive the required number of inference steps from its configuration.
> These scripts use Diffusers' MiniMax-H3 `ModularPipeline` and require **`diffusers >= 0.40.0`**.

方法出处（逐字）：

> [1] Neta Shaul, et al. "Parallel Decoding Distillation for Fast Image and Video Generation.". **arXiv preprint arXiv:2607.26004 (2026)**.

## 事实 / 推测 / 建议

- **事实（B，官方卡直读）**：PDD 蒸馏、两个 8-step Acc LoRA（FL2VA / Ref2VA）、`rank=64`、`network_alpha=64`、BF16、LoRA weight 1.0、在 4 与 8 NFE 均出过 demo、需 `diffusers>=0.40.0`、方法论文 arXiv:2607.26004。**这是本轮唯一明确写出 `network_alpha` 数值的 H3 卡（=64，不是 8）**。
- **推测（C/D）**：本页**未给任何耗时、显存、GPU、分辨率/帧数数字**（demo 只有视频），也**未给画质指标**；「8-step 更快/更好」未量化 → 不写结论。8GB 可行性 = **D（未核实）**。
- **建议（非结论）**：作为 LightX2V / larryvrh 之外的第三条蒸馏路线候选，但需 diffusers 0.40+ 与 ModularPipeline，与本机 ComfyUI 路线不同，V1.1 暂不纳入；若引用 alpha 参数，写 `network_alpha=64`（alibaba-pai），不要写 "alpha 8"。

## 等级

**B（官方 HF 模型卡直读：方法、权重、rank/alpha 可核对）**；效果/耗时/显存 = **D（卡页无数字）**；对 8GB 适用性 = **D**。
