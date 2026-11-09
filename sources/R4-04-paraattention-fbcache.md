# R4-04｜ParaAttention First Block Cache（FBCache）机制与视频 DiT 加速比

- URL（本轮实际打开，两个）：
  - https://github.com/chengzeyi/ParaAttention
  - https://github.com/chengzeyi/ParaAttention/blob/main/doc/fastest_hunyuan_video.md
- 读取时间：2026-11-08 JST
- 读取方式：WebFetch 直连打开。
- 类别：LOG-004 (2) FirstBlockCache 开关条件与加速比；(6) 显存副作用。

## 1. README（逐字）

> ### First Block Cache (Our Dynamic Caching)
> Inspired by TeaCache and other denoising caching algorithms, we introduce **First Block Cache (FBCache)** to use the residual output of the first transformer block as the cache indicator. If the difference between the current and the previous residual output of the first transformer block is small enough, we can reuse the previous final residual output and skip the computation of all the following transformer blocks. This can significantly reduce the computation cost of the model, **achieving a speedup of up to 2x while maintaining high accuracy**.

FLUX.1-dev 单卡 NVIDIA L20 表（逐字，Wall Time (s)）：

> | Optimizations | Original | FBCache rdt=0.06 | FBCache rdt=0.08 | FBCache rdt=0.10 | FBCache rdt=0.12 |
> | Wall Time (s) | 26.36 | 21.83 | 17.01 | 16.00 | 13.78 |

（26.36 → 13.78 = **1.91×**，rdt 越大跳步越多。）

> Adjust the `residual_diff_threshold` to balance the speedup and the accuracy. **Higher residual_diff_threshold will lead to more cache hits and higher speedup, but might also lead to a higher accuracy drop.**

> **Officially Supported Models**（单 GPU FBCache）：FLUX / HunyuanVideo / Mochi / CogVideoX（`first_block_cache_examples/run_*.py`）。

> **NOTE**: To run `HunyuanVideo`, you need to install `diffusers` from its latest master branch. **It is suggested to run `HunyuanVideo` with GPUs with at least 48GB memory, or you might experience OOM errors, and the performance might be worse due to frequent memory re-allocation.**

> **NOTE**: The speedup of iterations per second is generally higher than the speedup of wall time, because the wall time includes the overhead of calling the text encoder and vae decoder.

（另：Context Parallelism 性能表为 A100-SXM4-80GB / mochi-1-preview 多卡，**不含 FBCache**，不用于本条结论。）

## 2. HunyuanVideo 专文（逐字，FBCache 在视频 DiT 上的实测）

> To apply the first block cache on HunyuanVideo, we can call `apply_cache_on_pipe` with **`residual_diff_threshold=0.06`, which is the default value for HunyuanVideo.**
> When the difference is small enough, we can reuse the residual difference of previous inference steps, meaning that we in fact **skip this denoising step** … we can achieve an **up to 2x speedup on HunyuanVideo inference** with very good quality.

Baseline（逐字）：

> `residual_diff_threshold=0.0` to `apply_cache_on_pipe` **to disable the cache mechanism** now … **This is our baseline. On one single NVIDIA L20 GPU, we can generate 129 frames with 720p resolution in 30 inference steps in 3675.71 seconds.**

FBCache 开启（逐字）：

> Now, on one single NVIDIA L20 GPU, we can generate **129 frames with 720p resolution in 30 inference steps in 2271.06 seconds. This is a 1.62x speedup compared to the baseline.**

显存 / OOM（逐字）：

> most people will experience OOM … The size of `attn_mask` is proportional to the square of the input sequence length … In ParaAttention, we patch the original implementation in `diffusers` to cut the text conditions before calling `scaled_dot_product_attention`. We implement this in our `apply_cache_on_pipe` …
> If you still experience OOM errors, you can try calling `pipe.enable_model_cpu_offload` or `pipe.enable_sequential_cpu_offload` after calling `apply_cache_on_pipe`.
> The NVIDIA L20 GPU only has **48GB memory** and could face OOM errors after compiling the model … because the HunyuanVideo has very large activation tensors …
> `pipe.vae.enable_tiling()`

> Due to the fact that large video generation models usually have performance bottleneck on the attention computation rather than the fully connected layers, we don't observe a significant speedup with quantization and compilation.

## 3. 事实 / 推测 / 建议

- **事实（B，官方仓库直读）**：FBCache 用第一 transformer block 残差差作缓存指示、`residual_diff_threshold` 控速质；FLUX 单 L20 26.36→13.78s（1.91×）；HunyuanVideo 720p/129 帧/**30 步**、L20 48GB，baseline 3675.71s → FBCache 2271.06s（**1.62×**），默认 rdt=0.06；`rdt=0.0` 即关闭缓存；作者自述 "up to 2x"；≥48GB 才建议跑 HunyuanVideo，且需切 text conditions 避免 OOM；作者明确"wall time 的加速低于迭代加速（含 TE/VAE 开销）"。
- **推测（C）**：把 L20 48GB / 30 步的数字搬到 5070 Laptop 8GB / 4–8 步 Turbo —— **不成立**。关键机制含义：**FBCache 的收益来自"多次 ≥20–30 步的迭代里跳步"，步数越少可跳的步越少**。
- **建议（非结论）**：本机若退回 20 步原生基线，可试 FBCache（rdt 0.06 起步）；若用 4/8 步 Turbo，按 (4) 类证据应关闭。

## 4. 等级

**B（官方仓库 + 官方专文直读，含完整方法/代码/作者实测数字，可公开复核）**；对 8GB 外推 = **C**。
