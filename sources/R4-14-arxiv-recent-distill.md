# R4-14｜近 3 个月 arXiv：少步蒸馏与特征缓存（4 篇直读）

- URL（本轮实际直连打开，四个）：
  - https://arxiv.org/abs/2609.40037 （Radian）
  - https://arxiv.org/abs/2610.34092 （Leta）
  - https://arxiv.org/abs/2610.02188 （DMAD）
  - https://arxiv.org/abs/2603.01623 （Spectrum 论文，**v2 修订 2026-09-30 落在近 3 个月窗口**）
- 读取时间：2026-11-08 JST
- 读取方式：WebFetch 直连打开（arXiv abs 页）。
- 类别：LOG-004 (5) 近 3 个月 arXiv / 新 release（均有代码或可复核方法）。

## 1. arXiv:2609.40037 — Radian（逐字）

> **Title:** Enhancing Autoregressive Video Generation via Representation Adversarial Distillation
> **Submitted on 30 Sep 2026**；[v1] Wed, 30 Sep 2026 16:09:12 UTC (12,225 KB)
> Authors: Fangyu Lin, Xingtong Ge, Lunjie Zhu, Yi Zhang, Zhening Liu, Tianhang Wang, Mengfei Li, Yumeng Zhang, Guanglu Song, Yu Liu, Jun Zhang
> Few-step autoregressive video generation enables efficient streaming synthesis, but **errors introduced in early temporal blocks are reused as context and can propagate through subsequent rollouts, leading to detail degradation, structural drift, and unstable motion.** Existing distribution matching distillation (DMD) primarily aligns student and teacher distributions in diffusion latent space, but provides no direct supervision over the perceptual quality of decoded videos. We introduce **Radian**, a representation-space adversarial distillation framework that complements on-policy DMD with real-data adversarial supervision in the feature space defined by a frozen visual foundation model (VFM) … These additional components are discarded after training, **leaving the generator architecture and inference-time denoising budget unchanged**.
> Experiments on **Wan2.1-1.3B** cover **four-step chunk-wise, one-step frame-wise, and minute-long autoregressive generation**. Our method achieves a **VBench Total of 0.8444** and a **VideoAlign Total of 0.8033 under four-step generation**, and improves **VBench-Long from 0.7805 to 0.8041** over Rolling Forcing while using fewer denoising steps.

- 对 H2/H1 的意义（推测）：论文明确指出**少步自回归生成的早期误差会沿 rollout 传播 → 细节退化/结构漂移/运动不稳**，这是"低步数放大误差"的机制性佐证（与缓存近似的误差放大同类）。**代码链接未在 abs 页给出 → 代码可得性未核实**。

## 2. arXiv:2610.34092 — Leta（逐字）

> **Title:** Learning the Autoregressive: Direct Score-Difference Estimation with Amortized Future Credit for Video Distillation
> **Submitted on 25 Oct 2026**；[v1] Sun, 25 Oct 2026 06:44:06 UTC (34,048 KB)
> Authors: Zixuan Duan, Yabo Chen, Xunzhi Xiang, Xin Zhang, Haibin Huang, Yin Chen, Chi Zhang, Qi Fan
> Few-step distillation for autoregressive (AR) video generation commonly first initializes an AR student and then refines it with **distribution matching distillation (DMD)** … We introduce **Leta**, a few-step distillation framework that addresses these challenges with **ScoreDelta** and **FutureCredit**. ScoreDelta directly estimates the teacher-student correction from denoising residuals, while FutureCredit learns an auxiliary cross-chunk update signal from vector-Jacobian products (VJPs) of next-chunk teacher residuals …
> In our controlled experiments, **Leta improves over DMD by 0.9 points on VBench and 0.66 on VideoAlign.** This advantage persists at a higher generator update frequency (G:E=1:1), where **Leta improves over DMD by 1.6 on VBench and 0.35 on VideoAlign.**
> Project page: https://scdzx.github.io/Leta/

（HTML 全文片段另见 `https://arxiv.org/html/2610.34092`，经 WebSearch 索引片段读到："On few-step text-to-video distillation with **Wan**, Leta improves over DMD on VBench Overall and VideoAlign visual and motion scores"，并列出 rCM / AnyFlow / CausVid / Self Forcing / DistillAlign / Causal Forcing++ 等同类工作 —— **该段标为片段读取**。）

## 3. arXiv:2610.02188 — DMAD（逐字，**与 MiniMax-H3 直接相关**）

> **Title:** DMAD: Distribution Matching as Adversarial Distillation for Fast Visual Generation
> **Submitted on 1 Oct 2026 (v1), last revised 13 Oct 2026 (this version, v2)**；28 pages, 15 figures
> Authors: Zhengming Yu, Junkun Yuan, Haotian Yang, Gordon Guocheng Qian, Yizhi Wang, Angtian Wang, Yiding Yang, Bo Liu, Xin Li, Wenping Wang, Chongyang Ma
> Distribution Matching Distillation (DMD) trains a few-step student from the difference between separately estimated target and student scores, so it must keep an auxiliary diffusion model fitted to the student's evolving distribution **at extra memory and computation cost**. We introduce **DMAD** … recasts distribution matching as classification and learns the required log-density ratios directly. Two discriminator heads on a shared backbone …
> DMAD reaches a **Fréchet Inception Distance (FID) of 1.04 with one-step generation on ImageNet-64x64**, **14.47 with four-step SDXL on COCO-10K**, and a **VBench total score of 85.15 with four-step Wan2.1-T2V-14B**, the best values among the compared few-step methods and the multi-step teachers. **On MiniMax-H3-33B, our four-step student achieves overall human preference rates of 79.1% over DMD2 and 84.6% over rCM for joint audio-video generation, excluding ties.**
> **Our code, models and demos are available at https://yzmblog.github.io/projects/DMAD**

- 备注：DMAD 是**四步蒸馏 + H3 联合音视频**的近月论文，且明确写了 code/models/demos 可得（B：方法+代码线索）。但**没有 8GB / 显存 / 耗时 / 分辨率 / 帧数数字**，人类偏好率也非速度指标 → 只作"蒸馏路线进展"，不进 8GB 速度栈。
- 二手中文报道（aiboss88 / cloudsai，WebSearch 片段）另称「4 卡 H200 训练更新 233.3s vs DMD2 818.1s（3.51×）」「8GB 消费级显卡可跑」——**均为宣传性转述，标 D，本轮未直读，不采信**。

## 4. arXiv:2603.01623 — Spectrum 原论文（逐字）

> **Title:** Adaptive Spectral Feature Forecasting for Diffusion Sampling Acceleration
> **Submitted on 2 Mar 2026 (v1), last revised 30 Sep 2026 (this version, v2)**；**Comments: CVPR 2026**
> Authors: Jiaqi Han, Juntong Shi, Puheng Li, Haotian Ye, Qiushan Guo, Stefano Ermon
> … recent works resort to the **feature caching and reusing** scheme that skips network evaluations at selected diffusion steps by using cached features in previous steps. However, **their preliminary design solely relies on local approximation, causing errors to grow rapidly with large skips and leading to degraded sample quality at high speedups.** In this work, we propose **spectral diffusion feature forecaster (Spectrum)**, a training-free approach that enables global, long-range feature reuse with tightly controlled error … view the latent features of the denoiser as functions over time and approximate them with **Chebyshev polynomials** … fit the coefficient for each basis via **ridge regression** … **an error bound that does not compound with the step size.**
> Notably, we achieve **up to 4.79× speedup on FLUX.1 and 4.67× speedup on Wan2.1-14B**, while maintaining much higher sample quality compared with the baselines.

- **v2 修订日期 2026-09-30 落在近 3 个月**，故计入 (5)。作者自述「传统缓存依赖局部近似、大跳步误差迅速增长、高倍速下画质下降」是 H2（低步数/大跳步误差放大）的**理论依据**。上游实现 https://github.com/hanjq17/Spectrum（本轮**未打开**，标片段/未读）。
- 注意：comfyui-wiki（R4-07）引用的「14 次网络评估 / 3.5×」是**论文图注示例**（HunyuanVideo/FLUX），与摘要的 4.79× / 4.67× 是不同口径，**不可混写**。

## 事实 / 推测 / 建议

- **事实（B，arXiv abs 直读）**：4 篇的标题、作者、提交/修订日期、摘要数字与代码/项目页线索；其中 3 篇（Radian 09-30、DMAD 10-01/10-13、Leta 10-25）在近 3 个月内，Spectrum v2 修订 09-30 亦在窗口内；DMAD 明确给出 H3-33B 四步蒸馏的人类偏好率。
- **推测（C/D）**：这些是**训练侧方法论文**，不是本机 8GB 推理实测；无一个给出 8GB / 显存 / 耗时数字；代码可得性除 DMAD 项目页外**未逐一克隆核实** → 代码状态标 C。
- **建议（非结论）**：把「DMD/DMD2 → PDD / DMAD / Leta / Radian」记为蒸馏范式的演进线索（供 V2.0 参考），**V1.1 不据此改配置**；Spectrum 论文关于"大跳步误差复合"的结论用于支撑 LOG-004 中"低步数下缓存收益≈0 或为负"的机制解释。

## 等级

**B（arXiv 官方 abs 页直读：方法、日期、数字、代码线索可复核）**；"移到 H3 / 8GB 上的效果" = **C/未核实**；二手中文宣传数字 = **D（未直读、不采信）**。
