# R4-10｜ComfyUI-MiniMaxH3-AdaptiveCache：Turbo × 缓存冲突矩阵（18 组受控实测）

- URL：https://github.com/FFFFFFpy/ComfyUI-MiniMaxH3-AdaptiveCache
- 读取时间：2026-11-08 JST
- 读取方式：WebFetch 直连打开（README 全文）。
- 页面仓库时间（逐字）：`Last commit date` 列显示 Aug 4 / Aug 7 / Aug 4 / **Sep 23, 2026**；commit `fix: keep cache tensors outside Comfy allocation graphs (0.1.3)` 注有 "Aug 4, 2026"；README 版本段写 `当前版本：0.1.1`（两处不一致，照录）。
- 类别：LOG-004 (4) Turbo + cache 正反面；(2) 缓存加速比；(6) 显存；(1) Turbo 步数耗时。

## 1. 机制（逐字）

> 面向 ComfyUI **原生 MiniMax H3** 的独立加速插件。它会在每次采样中重算前部 DiT Block，并在条件允许时复用后部 Block 对目标音视频状态产生的残差。
> 插件不会修改 `comfy/ldm/minimax/model.py` … 它直接使用 H3 现有的 `("double_block", index)` ModelPatcher 扩展点 … 因此，被跳过的尾部 Block 不仅不会执行，也不会被 ComfyUI 的动态权重预取搬入显存。
> 以 50 层 H3、Balanced 预设为例：完整调用执行全部 50 层；执行约第 12 层后截取目标音视频 hidden state；命中后把 residual 加到当前目标音视频状态，**跳过第 13–50 层**。
> 插件只缓存最终输出层会读取的目标音频与目标视频区间。文本、首尾帧条件、参考图片、参考视频与参考音频的尾部 hidden state 不会保存。

预设（逐字）：Safe ~60% 尾部 Block、窗口 15%–85%、连续上限 1；**Balanced ~75%、10%–90%、连续上限 2**；Fast ~84%、8%–92%、上限 3；Sigma Only ~75%、关闭内容保护。

## 2. 缓存位置（逐字，(6) 证据）

> - `gpu`：读取最快，但**较高分辨率下 residual 可能占用数百 MB 显存**。
> - `cpu`：存放在 **pinned host memory**，CUDA 环境下会尽量在前部 Block 计算期间异步搬回 GPU。
> - `auto`：只有 residual 加安全余量能放进当前空闲显存时才选择 GPU，否则选择 CPU。
> 对 **16GB 显卡，建议优先使用 `auto`**。

## 3. RTX 5090 组合实测（逐字，(4) 核心）

> 这组测试比下面的 RTX 4090 初测更完整：同一套工作流连续测试 **18 种组合，每种配置生成 2 次并取平均**。输出为 `1376×768`、124 帧、24 FPS（约 5.17 秒视频）。
> 测试环境：
> - GPU：NVIDIA GeForce **RTX 5090 32GB**；PyTorch `2.12.1+cu130`；ComfyUI `0.30.0`
> - MiniMax H3 DiT：**W4A8 混合量化测试环境**
> - Adaptive Cache：`Balanced`，本轮 residual 落在 GPU
> - EasyCache：`threshold=0.2, start_percent=0.15, end_percent=0.95`

| 配置 | 两次总耗时 | 平均 | 相对 20 步原版 |
|---|---|---|---|
| 原版 20 步 | 303.72 / 290.91 s | **297.32 s** | 1.00× |
| Sol-Attn | 184.73 / 176.93 s | 180.83 s | 1.64× |
| Adaptive Cache | 186.36 / 186.11 s | 186.24 s | 1.60× |
| Sol-Attn + Adaptive Cache | 121.98 / 121.79 s | **121.89 s** | 2.44× |
| EasyCache | 194.54 / 180.58 s | 187.56 s | 1.59× |
| Sol-Attn + EasyCache | 137.64 / 137.44 s | 137.54 s | 2.16× |
| Turbo-4 | 73.21 / 73.17 s | 73.19 s | 4.06× |
| Sol-Attn + Turbo-4 | 48.53 / 48.71 s | 48.62 s | 6.12× |
| Adaptive Cache + Turbo-4 | 51.08 / 51.57 s | 51.33 s | 5.79× |
| **Sol-Attn + Adaptive Cache + Turbo-4** | 39.61 / 39.18 s | **39.40 s** | **7.55×** |
| **EasyCache + Turbo-4** | 73.57 / 73.08 s | **73.33 s** | 4.05× |
| Sol-Attn + EasyCache + Turbo-4 | 48.87 / 48.60 s | 48.74 s | 6.10× |
| Turbo-8 | 131.62 / 131.80 s | 131.71 s | 2.26× |
| Sol-Attn + Turbo-8 | 82.41 / 82.17 s | 82.29 s | 3.61× |
| Adaptive Cache + Turbo-8 | 87.71 / 87.42 s | 87.57 s | 3.40× |
| **Sol-Attn + Adaptive Cache + Turbo-8** | 56.99 / 57.14 s | **57.07 s** | **5.21×** |
| **EasyCache + Turbo-8** | 131.74 / 131.70 s | **131.72 s** | 2.26× |
| Sol-Attn + EasyCache + Turbo-8 | 82.36 / 82.36 s | 82.36 s | 3.61× |

结论（逐字）：

> - **Sol-Attn 与 Adaptive Cache 可以有效叠加。** 20 步原版从 297.32 秒降到 121.89 秒 …
> - **Adaptive Cache 单独也有明确收益。** 20 步从 297.32 秒降到 186.24 秒，约 1.60×；Balanced 典型统计为 `full=10, cache=10, executed=620/1000 blocks, block reduction=38%`。
> - **Turbo-8 与 Sol-Attn + Adaptive Cache 在性能上仍能继续叠加。** 本轮最快的 Turbo-8 组合为 57.07 秒，相对 20 步原版约 5.21×。
> - **Turbo-4 的极限性能更高。** `Sol-Attn + Adaptive Cache + Turbo-4` 达到 39.40 秒，约 7.55×；但 Turbo 相关画质必须单独评估，不能只看耗时。
> - **EasyCache 在正常 20 步下有效，但在本轮 Turbo-4 / Turbo-8 下没有实际跳步。** 日志分别出现 **`skipped 0/4 steps` 与 `skipped 0/8 steps`**，加入 EasyCache 后总耗时也与对应 Turbo 基线几乎一致，因此**不建议为了 Turbo 组合额外挂 EasyCache**。

W4A8 + Turbo 画质说明（逐字）：

> 本轮 Turbo 测试使用 W4A8 混合量化环境。部分 Turbo-4 样本出现明显画质异常，尤其在继续叠加缓存近似时更容易暴露，但**当前数据不足以证明 Adaptive Cache 与 Turbo 本身不兼容**。目前更合理的待验证假设是：Turbo 的少步采样 / LoRA 与 **W4A8 量化近似之间存在兼容性或误差放大问题**；Adaptive Cache 可能进一步放大已有误差，但未必是根因 … 若要评估 Adaptive Cache 与 Turbo 的质量兼容性，应先在**非 W4A8 基线下做同 seed A/B**。
> 另外，Turbo 将采样器本身压缩到很短以后，**端到端瓶颈开始转向固定开销**。以最快的 Turbo 组合为例，sampler 约 23.8 秒，而总工作流约 39.4 秒，说明**约 15～16 秒已经来自 VAE、封装和其他前后处理**。继续只优化 DiT 的边际收益会越来越小。

兼容性（逐字）：

> 其他会替换 MiniMax H3 `double_block` 的**缓存、跳层或 Block Patch 节点仍不建议叠加，相同扩展点只能有一套最终实现**。
> **Turbo LoRA / Turbo sampler 在执行路径上可以与 Adaptive Cache 同时工作**，5090 实测也能继续获得性能叠加；但当前 W4A8 测试中的 Turbo 画质存在未解决变量，**暂不把它列为"已验证无损兼容"**。

RTX 4090 初测（960×640，单次，非受控）另有 10 行表；作者明确写道：

> 它们是实际使用过程中的**单次记录，不是严格控制预热、运行顺序和重复次数的正式 benchmark** … 性能数字应被视为**当前环境的观察值，不应直接外推到其他 GPU、分辨率、帧数、Triton 版本或采样器**。

## 4. 事实 / 推测 / 建议

- **事实（B，仓库 README 直读，含 18 组×2 次取平均的受控表）**：20 步 297.32s 基线；Adaptive Cache 单独 186.24s（1.60×）；与 Sol-Attn 叠加 121.89s（2.44×）；Turbo-8 + Sol-Attn + AdaptiveCache 57.07s（5.21×）、Turbo-4 39.40s（7.55×）；**EasyCache + Turbo-4 = 73.33s（vs Turbo-4 73.19s）、EasyCache + Turbo-8 = 131.72s（vs 131.71s），日志 `skipped 0/4`、`skipped 0/8`**；`cache_device=gpu` 时 residual 可占"数百 MB 显存"；Turbo 到极限后约 15–16s 为 VAE/封装固定开销。
- **推测（C）**：**RTX 5090 32GB + W4A8 + 1376×768**，与 5070 Laptop 8GB 完全不同档，**所有秒数禁止外推到本机**；Turbo+AdaptiveCache 的画质兼容性作者自己写"未验证无损"、需非 W4A8 同 seed A/B；4090 那组单次数据作者明确说不可外推。
- **建议（非结论）**：本机把"Turbo + Block 级残差缓存可叠加"当**方向**（B 机制 + 受控表），但"可叠加"不等于"无损"；EasyCache 类阈值缓存低步数下确认无效（skipped 0）；固定开销占比上升意味着本机继续压 DiT 步数的边际收益会快速衰减。

## 5. 等级

**B（公开仓库 + 18 组受控实测表 + 完整方法/版本/环境，可公开复核；单仓库单作者，无第三方复现故不外推）**；"Turbo + 缓存无损兼容" = **C/证据不足（作者自述未验证）**；对 5070 Laptop 8GB 的秒数 = **C（禁止搬用）**。
