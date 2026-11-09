# R4-06｜EasyCache 论文（arXiv 2507.02860）+ 官方仓库实测数字

- URL（本轮实际打开，两个）：
  - https://arxiv.org/abs/2507.02860
  - https://github.com/H-EmbodVis/EasyCache
- 读取时间：2026-11-08 JST
- 读取方式：WebFetch 直连打开。
- 类别：LOG-004 (3) EasyCache 视频 DiT 数字；(5) 缓存方法；(4) 叠加。

## 1. arXiv abs（逐字）

> **Title:** Less is Enough: Training-Free Video Diffusion Acceleration via Runtime-Adaptive Caching
> **Submitted on 3 Jul 2025**；Authors: Xin Zhou, Dingkang Liang, Kaijin Chen, Tianrui Feng, Xiwu Chen, Hongkai Lin, Yikang Ding, Feiyang Tan, Hengshuang Zhao, Xiang Bai
> We conduct comprehensive studies on various large-scale video generation models, including **OpenSora, Wan2.1, and HunyuanVideo**. Our method achieves leading acceleration performance, **reducing inference time by up to 2.1-3.3×** compared to the original baselines while maintaining high visual fidelity with a significant **up to 36% PSNR improvement** compared to the previous SOTA method.
> The code is available at https://github.com/H-EmbodVis/EasyCache. Project page: https://h-embodvis.github.io/EasyCache/

（注：**2025-07-03，不属于"近 3 个月"**；近 3 个月的 arXiv 另见 R4-14。）

## 2. 官方仓库 README 实测表（逐字）

> **Tested on Vbench prompts with NVIDIA A800.**

HunyuanVideo：

| Method | Latency (s)↓ | Speedup ↑ | PSNR ↑ | SSIM ↑ | LPIPS ↓ |
|---|---|---|---|---|---|
| HunyuanVideo (Baseline) | 1124.30 | 1.00x | - | - | - |
| PAB | 958.23 | 1.17x | 18.58 | 0.7023 | 0.3827 |
| TeaCache | 674.04 | 1.67x | 23.85 | 0.8185 | 0.1730 |
| SVG | 802.70 | 1.40x | 26.57 | 0.8596 | 0.1368 |
| **EasyCache (Ours)** | **507.97** | **2.21x** | **32.66** | **0.9313** | **0.0533** |

Wan2.1-1.3B：

| Method | Latency (s)↓ | Speedup ↑ | PSNR ↑ | SSIM ↑ | LPIPS ↓ |
|---|---|---|---|---|---|
| Wan2.1 (Baseline) | 175.35 | 1.00x | - | - | - |
| PAB | 102.03 | 1.72x | 18.84 | 0.6484 | 0.3010 |
| TeaCache | 87.77 | 2.00x | 22.57 | 0.8057 | 0.1277 |
| **EasyCache (Ours)** | **69.11** | **2.54x** | **25.24** | **0.8337** | **0.0952** |

与 SVG 叠加（逐字，HunyuanVideo 720p，**NVIDIA H20**）：

> | Method | Latency (s)↓ | Speedup ↑ | PSNR (dB) ↑ |
> | Baseline | 6594s | 1.00x | - |
> | SVG | 3474s | 1.90x | 27.56 |
> | SVG (w/ TeaCache) | 2071s | 3.18x | 22.65 |
> | **SVG (w/ Ours)** | **1981s** | **3.33x** | **27.26** |

> **EasyCache is orthogonal to other acceleration techniques**, such as the efficient attention mechanism SVG, and can be combined with them for even greater performance gains.

机制 / News（逐字要点）：

> EasyCache monitors the relative transformation rate and detects local stability via a dynamic, runtime-adaptive thresholding scheme … **Unlike prior methods, which rely on costly offline statistics and fixed heuristics (e.g., polynomial fitting), EasyCache makes fully online decisions** based on lightweight first-order dynamics.
> [2025/08/01] EasyCache for Wan2.2 I2V … [2025/07/30] Wan2.2 TI2V-5B … [2025/07/05] Wan2.1 T2V … [2025/07/02] HunyuanVideo … [2025/07/04] Release the paper.
> - `ComfyUI-WanVideoWrapper` (kijai) supports EasyCache4Wan2.1/EasyCache4Wan2.2.
> - `ComfyUI` (comfyanonymous) supports EasyCache for several model includes Kontext, Qwen Image Edit, Wan2.2 14B.

**To Do 列表中无 MiniMax H3**。Demo 标注 `Ours (2.28x)` / `TeaCache (1.68x)` / `PAB (1.19x)`（HunyuanVideo）；`Ours (2.63x)` / `TeaCache (1.46x)` / `PAB (2.10x)`（Wan2.1-14B）。

## 3. 事实 / 推测 / 建议

- **事实（B，论文 + 官方仓库直读）**：EasyCache 是 runtime-adaptive、无需离线标定的 training-free 缓存；A800 上 HunyuanVideo 1124.30→507.97s（2.21×，PSNR 32.66）、Wan2.1-1.3B 175.35→69.11s（2.54×）；H20 上与 SVG 叠加 6594→1981s（3.33×，PSNR 27.26）；作者称与其它加速"正交"。**注意：这里的"叠加"是与稀疏注意力 SVG 叠加，不是与少步蒸馏 Turbo LoRA 叠加**。
- **推测（C）**：把 A800/H20 的 2.2–3.3× 搬到 8GB + Turbo 4/8 步 —— **不成立**；论文/仓库**无 H3、无 8GB、无 peak VRAM/RAM 数字**；"正交"未被 Turbo 场景验证（反证见 R4-09 / R4-10 / R4-11）。
- **建议（非结论）**：把 EasyCache 当"高步数原生基线"的杠杆；Turbo 低步数下按 (4) 类证据关闭；若要在 H3 上用，需先确认 timestep 变换率稳定性。

## 4. 等级

**B（arXiv 论文 + 官方仓库 benchmark 直读，含完整方法与作者数字）**；与 Turbo 叠加 = **不成立/证据不足（见 R4-10）**；对 8GB = **C**。
