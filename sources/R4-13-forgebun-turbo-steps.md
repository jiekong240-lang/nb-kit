# R4-13｜forgebun / Stable Yogi：Turbo 20/8/4 步 warm 耗时（明确为 24GB+ 卡）

- URL：https://forgebun.com/go/civit_h3_turbo_ref
- 读取时间：2026-11-08 JST
- 读取方式：WebFetch 直连打开。
- 页面标注（逐字）：`Stable Yogi · August 25, 2026 · 5 min read` — `MiniMax H3 twice as fast, and keeping the same person across shots`
- 类别：LOG-004 (1) Turbo 步数-耗时；(4) 画质观察。**注意：非 8GB，禁止冒充 8GB。**

## 逐字摘录

> ## Turbo: 8 steps instead of 20
> A "step" is one pass the model makes at the picture. Twenty of them is the normal setting. The turbo file teaches the model to get there in eight.
> **What it actually saved here. 960 × 544, 124 frames (five seconds) on one card, with the model already loaded:**
> | Steps | Time |
> | 20, no turbo file | **113 s** |
> | **8, turbo file** | **53 s** |
> | 4, turbo file | **34 s** |
> About **twice as fast at eight steps, three times at four**.

> **The first number was measured warm.** The very first run of any session also pays for loading 19.5 GB off your disk, which can be minutes on its own and has nothing to do with steps. Compare a cold run against a warm one and you can "prove" almost anything. **These are all warm.** To test your own card, ignore the first run and compare warm runs.
> **We looked at the frames.** A faster setting that softens faces or kills motion is a worse setting. **Nothing smeared, nothing melted, and faces held at both 8 and 4.**
> **Use 8 as your normal setting.** Drop to 4 while you are hunting for a prompt that works and want answers quickly, then go back to 8 for the one you keep. **A 4-step clip can differ from the final 8-step shot in small details such as camera position.**

Ref2V 补充表（逐字）：

> 960 × 544, 73 frames, warm: 20, no turbo file **69 s** / 8, reference turbo **31 s** / 4, reference turbo **21 s**. The likeness held at four steps as well as at twenty. … the reference turbo file is still at v0.1.

文件与要求（逐字）：

> The turbo file is a LoRA … it goes in `ComfyUI/models/loras`, and the file is **`minimax_h3_fl2v_turbo_8step_v1.0_comfyui_bf16.safetensors`** (1.9 GB) … change the step count from 20 to 8.
> You also need **ComfyUI 0.30 or newer** and **a card with 24 GB or more**.

FAQ 逐字复述：`At 960 x 544 and 124 frames, warm: 113 seconds at 20 steps, 53 seconds at 8 steps with the turbo file and 34 seconds at 4 steps.`

## 事实 / 推测 / 建议

- **事实（B，页面直读）**：960×544 / 124 帧 / warm：20 步 113s → 8 步 53s（2.13×）→ 4 步 34s（3.32×）；Ref2V 960×544/73 帧 20/8/4 = 69/31/21s；作者称 8 与 4 步「faces held、无 smear/melt」、4 步与 8 步在机位等小细节有差异；LoRA 文件名与 1.9GB 体积；**页面明确要求 24GB+ 显卡**。
- **推测（C/D）**：**这组数字出自 ≥24GB 卡，绝对不是 8GB 实测**；本轮按其原文标注，**不把 113/53/34s 当本机数字，也不据此给 8GB 下结论**（遵守 R4 §8「禁止用 24GB+ 卡数字冒充 8GB」）。画质为作者目视、无指标；"faces held" 与 larryvrh 卡「4 步大幅运动有拖影」、JUNI「Turbo 剥掉运动平滑度」并不完全一致，属题材/检查方法差异。
- **建议（非结论）**：只借用其**方法论**（warm-to-warm 对照、忽略首跑加载、8 步为常规 / 4 步用于筛 prompt）与"步数比 ≠ 端到端比"的定性形状；本机必须自己测 4/6/8 步。

## 等级

**B（页面直读，条件/数字/显卡要求可核对）**；但**对 5070 Laptop 8GB 的适用性 = 不适用（24GB+ 卡）**，仅作方法参考。不进入 V1.1 速度栈表的 8GB 依据栏。
