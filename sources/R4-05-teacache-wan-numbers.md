# R4-05｜TeaCache 官方仓库 + Wan2.1 实测数字（CVPR 2025）

- URL（本轮实际打开，两个）：
  - https://github.com/ali-vilab/TeaCache
  - https://github.com/ali-vilab/TeaCache/blob/main/TeaCache4Wan2.1/README.md
- 读取时间：2026-11-08 JST
- 读取方式：WebFetch 直连打开。
- 类别：LOG-004 (3) TeaCache 在视频 DiT 的公开数字；(6) 显存；H3 支持性。

## 1. 主仓库 README（逐字要点）

> # [CVPR 2025 Highlight] Timestep Embedding Tells: It's Time to Cache for Video Diffusion Model
> We introduce Timestep Embedding Aware Cache (TeaCache), a training-free caching approach that estimates and leverages the fluctuating differences among model outputs across timesteps, thereby accelerating the inference. TeaCache works well for Video Diffusion Models, Image Diffusion models and Audio Diffusion Models.

News 日期（逐字，均为 **2025 年**，说明这不是近 3 个月的新工作）：

> [2025/06/08] Update coefficients of Lumina-Image-2.0 … [2025/04/05] Recommended as a **highlight** in CVPR 2025, top 16.8% … [2025/03/13] Optimized TeaCache for Wan2.1 … [2025/03/05] Support Wan2.1 for both T2V and I2V … [2025/02/27] Accepted in CVPR 2025 … [2024/12/30] Support Mochi and LTX-Video … [2024/12/24] Support HunyuanVideo … [2024/11/28] Release the paper … arXiv:2411.19108

**Supported Models 逐字列表（Text to Video）**：`TeaCache4Wan2.1`、`TeaCache4Cosmos`、`TeaCache4CogVideoX1.5`、`TeaCache4LTX-Video`、`TeaCache4Mochi`、`TeaCache4HunyuanVideo`、`TeaCache4CogVideoX`、`TeaCache4Open-Sora`、`TeaCache4Open-Sora-Plan`、`TeaCache4Latte`；I2V 含 Wan2.1 / Cosmos / CogVideoX1.5 / ConsisID；另有 FLUX / Lumina / HiDream（图）、TangoFlux（音频）。

> **MiniMax H3 不在支持列表中**（本轮直读确认）。

社区接入（逐字）：`ComfyUI-TeaCache` (welltop-cn)、`ComfyUI-WanVideoWrapper` (kijai) supports TeaCache4Wan2.1、`ComfyUI-HunyuanVideoWrapper` (kijai)、`Teacache-xDiT`、SD.Next、DiffSynth Studio、FastVideo、FramePack、EasyAnimate。

> If the custom model is based on or has similar model structure to the models we've supported, you can try to directly transfer TeaCache to the custom model … Otherwise, you can refer to these successful attempts, e.g., [1](issues/20), [2](issues/18).

（机制：`rel_l1_thresh` 控速质、按模型拟合多项式重标定系数；每模型独立子目录与脚本，非 drop-in 库。）

## 2. TeaCache4Wan2.1 README 实测表（逐字）

> TeaCache can speedup Wan2.1 **2x** without much visual quality degradation, in a training-free manner.

> ## 📈 Inference Latency Comparisons on a **Single A800**

| 配置 | 原生 | TeaCache (thresh) | TeaCache (thresh) | TeaCache (thresh) |
|---|---|---|---|---|
| Wan2.1 t2v **1.3B** | ~175 s | ~117 s (0.05) | ~110 s (0.07) | ~88 s (0.08) |
| Wan2.1 t2v **14B** | ~55 min | ~38 min (0.14) | ~30 min (0.15) | ~27 min (0.2) |
| Wan2.1 **i2v 480P** | ~735 s | ~464 s (0.13) | ~372 s (0.19) | ~300 s (0.26) |
| Wan2.1 **i2v 720P** | ~29 min | ~17 min (0.18) | ~15 min (0.2) | ~12 min (0.3) |

`use_ret_steps`（Retention Steps）表（逐字，节选）：

| use_ref_steps | 1.3B | Slow | Fast |
|---|---|---|---|
| False | ~97 s (0.00) | ~64 s (0.05) | ~49 s (0.08) |
| True | ~97 s (0.00) | ~61 s (0.05) | ~41 s (0.10) |

命令示例（逐字，含 `--offload_model True --t5_cpu`）：

> `python teacache_generate.py --task t2v-1.3B --size 832*480 … --base_seed 42 --offload_model True --t5_cpu --teacache_thresh 0.08`
> `… --task t2v-14B --size 1280*720 … --teacache_thresh 0.2`
> `… --task i2v-14B --size 1280*720 … --frame_num 61 --teacache_thresh 0.3`

> Using Retention Steps will result in faster generation speed and better generation quality (except for t2v-1.3B).

## 3. 事实 / 推测 / 建议

- **事实（B，官方仓库直读）**：TeaCache 是 CVPR2025 training-free 缓存，按模型拟合多项式系数、`rel_l1_thresh` 控速质；Wan2.1 在**单 A800** 上 1.3B 175→88s（~1.99×）、14B 55→27min（~2.04×）、i2v 480P 735→300s（2.45×）、i2v 720P 29→12min（2.42×）；**H3 不在支持模型列表**；社区由 kijai 的 WanVideoWrapper 等接入。
- **推测（C）**：把 A800（>40GB）/ 832×480–1280×720 / 常规多步采样搬到 5070 Laptop 8GB + Turbo 4/8 步 —— **不成立**；本页**没有 8GB、没有少步蒸馏叠加、没有 peak VRAM 数字**。
- **建议（非结论）**：H3 要用 TeaCache 类机制需自行移植并按 H3 的 timestep 波动重新标定系数（仓库明确说"结构相似才可直接搬系数"）；低步数下按 (4) 类证据应关闭。

## 4. 等级

**B（官方仓库 + 官方 benchmark 表直读，含完整方法与作者实测数字）**；对 8GB / H3 / 少步叠加 = **C / 未核实**。
