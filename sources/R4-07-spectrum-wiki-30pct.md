# R4-07｜ComfyUI Wiki：Spectrum Apply MiniMax H3 ~30% 加速与实测

- URL：https://comfyui-wiki.com/en/news/2026-08-03-comfyui-spectrum-minimax-h3
- 读取时间：2026-11-08 JST
- 读取方式：WebFetch 直连打开。
- 类别：LOG-004 (3) Spectrum 在 H3 的公开数字；(1)/(4) 画质代价。

## 逐字摘录

> **Last updated: August 3, 2026** ComfyUI Wiki news
> `xmarre`'s ComfyUI-Spectrum-MiniMax-H3 … it skips H3 transformer evaluations for **~30% faster** MiniMax H3 video, placed after H3 Sigma Shift.
> By forecasting post-transformer hidden features and skipping selected transformer evaluations, it delivers roughly **30% faster generation without loss of quality in community tests**, and has already racked up over 200 GitHub stars within days of release.
> The node is an independent port of the Spectrum acceleration method ([Adaptive Spectral Feature Forecasting for Diffusion Sampling Acceleration](https://arxiv.org/abs/2603.01623)) for the H3 omni-modal model …
> During sampling, the node fits a **Chebyshev ridge regression** model to the actual post-transformer hidden features of H3, then uses it to forecast those features on selected future solver steps. On forecast steps, the expensive transformer blocks, RoPE construction, conditioning projections, and reference embedding are all skipped — only the native output heads, video/audio reconstruction, and sigma mapping still execute.
> *Spectrum paper samples (top: text-to-video on HunyuanVideo, bottom: text-to-image on FLUX.1), generated with only **14 network evaluations for a 3.5× speedup**.*

> ## Speedups
> On a **20-step run with the conservative preset**, the sampler-aware scheduler skips 6-7 of 20 H3 transformer evaluations:
> | Sampler | Actual steps | Forecasted steps | Transformer-step reduction |
> | Euler | 13 | 7 | 35% |
> | RES multistep / CFG++ | 14 | 6 | 30% |
> Community reports match those numbers: BecauseReasons shared the node with a "~30% faster without quality loss, unlike cache" verdict, and nghtdrp confirmed similar results. **An AMD ROCm user reported a 20-step RES multistep run (864×480, 107 frames) dropping from 212.7s to 161.0s — about 1.32× throughput on a Radeon AI PRO R9700 with ComfyUI 0.30.0.**

> ## Quality Caveats
> Spectrum is an **approximate accelerator**, not a lossless path. The README is explicit that forecasted steps change the denoising trajectory:
> - During fast or brief actions, **motion, pose, timing, or gaze can diverge** from a fully native run
> - Rapidly moving or briefly visible details (**eyes, fingers, fingernails**) can occasionally become malformed or unstable
> - Outputs can differ even with identical prompt, seed, model, and workflow
> For quality-critical work, compare the same prompt and seed with the node enabled and disabled.

> **Requirements:** ComfyUI with native MiniMax H3 support (**0.30.0 or later**, native H3 merged August 3). The node accepts and returns `MODEL`; recommended placement is **after the H3 Sigma Shift and before the guider and sampler**. **Euler, RES multistep, and RES multistep CFG++ are supported; ancestral samplers and multi-GPU parallel sampling automatically fall back to the native H3 path.**
> Links: GitHub xmarre/ComfyUI-Spectrum-MiniMax-H3；Spectrum paper arXiv 2603.01623。

## 事实 / 推测 / 建议

- **事实（B，发布页直读）**：Spectrum H3 节点的机制（Chebyshev 岭回归预测 hidden feature、跳过整段 transformer block）；20 步保守预设下 Euler 13 实际 / 7 预测（35%）、RES 14/6（30%）；AMD ROCm 用户 20 步 RES multistep、864×480、107 帧、212.7→161.0s（1.32×）、ComfyUI 0.30.0；画质 caveat（运动/姿态/眼睛/手指/指甲、同 seed 也可能不同）；支持采样器与自动回退条件。
- **推测（C）**：「~30% 更快 / 无损」是**社区口径**（"community tests"），非受控基准；**Radeon AI PRO R9700 不是 8GB 卡**（该卡为 32GB 级），其 1.32× **不可外推到 5070 Laptop 8GB**；Wiki 所引「14 次网络评估 / 3.5×」出自 Spectrum **论文图注**（HunyuanVideo/FLUX 示例，非 H3）。
- **建议（非结论）**：本机若试 Spectrum，必须同 seed 开/关各跑一次逐帧比对；20 步以上才可能有效，低步数 Turbo 按 (4) 证据关闭；「30%」只当 transformer-step 削减比例，不当端到端耗时保证。

## 等级

**B（发布页直读：机制、跳步表、AMD 用户数字、版本要求可核对）**；但「30% 端到端更快」= **C（社区口径）**；AMD 1.32× = **C（第一手但单用户、非 8GB、无第三方复现）**；8GB 外推 = **C/未核实**。
