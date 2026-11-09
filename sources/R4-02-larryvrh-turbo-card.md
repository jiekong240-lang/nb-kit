# R4-02｜larryvrh MiniMax-H3 Turbo LoRA 卡（4/6/8 步与画质代价）

- URL：https://huggingface.co/larryvrh/MiniMax-H3-Turbo-Lora/raw/main/README.md
- 读取时间：2026-11-08 JST
- 读取方式：WebFetch 直连打开（AI 摘要超时，返回 7354 字符原始正文全文，仍为直读）。
- 类别：LOG-004 (1) 蒸馏 LoRA / H1 画质代价。

## 逐字摘录

> # MiniMax-H3 Turbo LoRA — few-step audio-video generation
> A LoRA for MiniMax-H3 that renders joint **video + synchronized stereo audio** in as few as **4 sampling steps** instead of the usual ~20 — a ~5× sampling speedup — and keeps getting better as you add steps.

> For **most** work, use **`minimax_h3_turbo_v4_step600_ema.safetensors`**. It's the strongest checkpoint we've released: much better static and small-motion shots, markedly better micro-detail (faces, fingers, fine texture), and the over-sharpening / plastic look of the earlier `v1` (~850) line is **fully resolved**.
> v4 introduced a **static-frame enhancement** … The one trade-off shows up **only at 4 steps with large, fast motion**, where v4 can produce **motion-smear / trailing ghosting** …
> - **Use 6–8 steps.** This **largely removes the smear** and is where v4 looks its best. …
> - For the specific case of **4 steps *and* heavy motion**, the older **`v1` ~850** checkpoint can still be the friendlier pick.
> Still a preview — training continues; the two areas still being improved are **audio** and **behaviour under fast, intense motion**.

> ## Steps and strength — read this
> - **4 steps is the recommended *minimum*; 4–8 is the useful range.** 6–8 steps look noticeably better than 4, so add steps if you can afford them. Past **8 steps** it stops helping and can start to introduce **over-sharp artifacts** — there's no benefit to going higher, so stay in **4–8**.
> - **Keep strength at `1.0`.** … Only reach for the strength dial if a *specific* clip misbehaves — then **blurry ghosting / smear → nudge up** (`~1.05–1.2`), **over-sharp grain → nudge down** (`~0.8–0.95`).
> - Keep the scheduler on `simple`.

权重与 alpha（逐字）：

> All bf16, ~744 MB, applied as a plain low-rank update (`W_eff = W + lora_B @ lora_A`, **alpha = rank, so no extra scaling**). **Prefer the EMA files**; the non-EMA ones are for comparison.
> | `minimax_h3_turbo_v4_step600_ema.safetensors` | **recommended — current best.** …
> | `minimax_h3_turbo_4step_ema_ckpt850.safetensors` | `v1` line (~850) — over-sharpened / plastic in general, but the friendlier pick for **4-step heavy motion** |
> *Naming:* … Older files carry the previous `4step_ckptN` naming, where `4step` referred to the sampler-step count.

ComfyUI 用法（逐字要点）：

> - **Base model**: any MiniMax-H3 base — full (`bf16`, `int8_convrot`) **and the pruned/curve variants** (`pruned_int8`, `pruned_fp8`). The node auto-detects a pruned base and re-injects the time-conditioning at run time, so **one LoRA file covers every base**.
> - **`low_vram`** switch: **off** applies the LoRA at run time (sharpest, recommended); **on** merges it into the weights for the lowest peak VRAM (a bit softer on quantized bases). **Turn it on only if you run out of memory.**
> - insert **MiniMax-H3 Turbo LoRA** between the model loader and the sampler; feed `SamplerCustomAdvanced` from **MiniMax-H3 Turbo Sampler**, and set the scheduler to `simple` at **≥ 4 steps**.

分辨率 / 时长 / 显存（逐字）：

> - **Resolution / duration**: width and height are multiples of 32 (short edge typically 768). Frame count is at 24 fps and snaps to the model's 17·k+5 grid (124 ≈ 5 s). Validated range ~124–362 frames (~5–15 s).
> - **VRAM**: the base model is large (~33 B); an **80 GB GPU is comfortable** at the largest resolutions. The ComfyUI node streams the base and adds the `low_vram` switch, so it runs on much smaller GPUs. In the standalone script, **`--offload-adaln` trades ~13 GB of VRAM for CPU RAM.**
> - **Audio**: 32 kHz stereo … (Audio is one of the two areas still being improved — see the top.)

Standalone 命令示例含 `--width 1344 --height 768 --frames 124 --steps 6`。

## 事实 / 推测 / 建议

- **事实（B，官方模型卡直读）**：4 步为推荐下限、4–8 为有效区间、>8 步无益且过锐；v4 在 4 步 + 大幅快速运动下有 motion-smear / trailing ghosting，6–8 步基本消除；`strength=1.0`、`scheduler=simple`；LoRA ~744MB、`alpha = rank`（**不是 alpha 8**）；`low_vram` 开 = 合并进权重、峰值最低但量化基座上偏软；`--offload-adaln` 用 ~13GB CPU RAM 换 ~13GB VRAM；官方称 80GB GPU 才舒适。
- **推测（C）**：「~5× sampling speedup」作者自己限定为**采样步数**之比（~20→4），不是 wall-clock；8GB 上实际耗时未给。
- **建议（非结论）**：本机 8GB 用 `low_vram=on`（合并权重）作显存兜底，但需在同 seed 下与 off 版比对画质；默认 8 步，4 步只用于快速筛 prompt；音频同步是官方承认的未完善项，需单独验。

## 等级

**B（官方 HF 模型卡直读，参数与画质描述可公开复核）**；其中「~5× 加速」= **C（作者自述为步数比，非端到端实测）**；8GB 外推 = **C**。
