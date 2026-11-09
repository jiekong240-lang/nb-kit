# R4-11｜note JUNI：8GB 上 Turbo + EasyCache A/B 实测（-0.7%，无收益）

- URL：https://note.com/j_alpha_lab/n/n2661aa9a31cd?hl=en
- 读取时间：2026-11-08 JST
- 读取方式：**部分直读**（WebFetch 返回前 15000/38641 字符，含标题、日期、环境、实测值表；**EasyCache A/B 段落在截断之后，取自 WebSearch 返回的该页英文正文片段，标为「片段读取」**）。
- 页面标注（逐字）：`Running MiniMax H3 on an 8GB GPU (All Measured Values)｜JUNI` — `Aug 11, 2026 21:21`
- 类别：LOG-004 (4) Turbo + cache 冲突（反面，8GB 一手）；(1) Turbo 步数/耗时。

## 1. 直读部分（逐字）

环境（逐字）：

> - **GPU** / **RTX 3060 Ti 8GB** (Ampere / SM86)
> - **Driver** / 591.86 ｜ **System RAM** / **64GB** ｜ **ComfyUI** / **0.31.0** (portable) ｜ Python 3.13.14 ｜ PyTorch **2.13.0+cu130** ｜ comfy-kitchen 0.2.28 ｜ triton-windows 3.7.1.post27

> ### 🔴 What matters more than VRAM is system RAM … **the actual constraint is system RAM, not VRAM**. I have **64GB** installed. And the launch flag **`--disable-pinned-memory`** is what makes this work … **I have seen reports that it runs on 32GB, but it seems to hit a wall with either resolution or duration. If you have an 8GB card, I think you should check your RAM before your VRAM.**

模型合计 39.55GB（DiT pruned int8_convrot 19.53 + TE nvfp4_awq 14.61 + video VAE 4.85 + audio VAE 0.56）；启动 flags `--enable-triton-backend --disable-pinned-memory --preview-method none --port 8188`。

VAE 流式解码（PR #15446，2026-08-09 合并）：362 帧 / 15.08s / **7,677 MiB**，8192 上限余 515 MiB；但 243→362 帧耗时 **605.7s → 2113.2s（3.5×）**。

实测值（逐字，832×480 / 24fps / 带音频 / i2v / 同图同 prompt 同 seed）：

> ### Comparison at 10 seconds (243 frames)
> - **turbo (lightx2v)** / steps **8** / sampler **er_sde** / time **605.7 seconds** (10 minutes 06 seconds)
> - **turbo (Tutu)** / steps 8 / sampler euler / time 725.8 seconds
> - **raw** / steps 20 / res_multistep / 1491.8 seconds
> - **raw** / steps 25 / res_multistep / 1677.0 seconds
> - **turbo (lightx2v) · 15 seconds** / steps 8 / er_sde / **2113.2 seconds** (35 minutes 14 seconds)
> Measured turbo 8 steps twice, resulting in **605.7 seconds and 610.7 seconds**. The variance was 0.8%.
> ### Comparison at 5 seconds (124 frames)
> - **raw** / steps 25 / cold 625.8 / warm **595.7 seconds**
> - **turbo (lightx2v)** / steps **6** / cold 310.5 / warm **185.5 seconds**

> **📌 Increasing the number of steps does not increase the time much** — Going from 20 steps to 25 steps, despite the step count being 1.25x, the time was **1.12x**. I think it's because **the fixed cost of loading and unloading the model with 8GB is high**.

Turbo 画质（逐字）：

> **Turbo LoRA "always cuts something"** … QrusherZA Rough / lightx2v (Kijai conversion) **awkward** (walking scene) / Tutu AudioVideo 20→8 **Degraded** … **Step distillation strips away not only lines and sound but also the smoothness of movement.** … it is probably best to avoid it for shots where characters are walking or running. **However, depending on the subject, turbo was better** — 「Two girls standing under a starry sky」 两个 turbo 版本反被判定画质更好。

> ⚠️ All numbers are **as of August 11, 2026**.
> *All measurements were conducted on the same machine with the same input image between 2026-08-09 and 10. … only warm-to-warm comparisons were used. Any figures affected by queueing during parallel execution were discarded.*

## 2. EasyCache A/B 段落（**片段读取**，来自 WebSearch 抓取的该页英文正文，逐字）

> **EasyCache did not have any effect when used in combination with turbo.** I hadn't measured the "shortening rate when adding EasyCache," so I did that. **The conditions were completely identical (832×480 / 243 frames / 8 steps / er_sde / turbo lightx2v), with only the presence or absence of EasyCache as the variable.**
> **Without EasyCache / Time 740.8 seconds**
> **With EasyCache / Time 735.9 seconds**
> **The difference is -0.7%. It's within the margin of error.** I also compared the images by lining up frames from the same time, but I couldn't tell them apart. The result is that there is neither gain nor loss. **The reason is likely simple: there are only 8 steps. Since EasyCache is a mechanism that "skips calculations similar to the previous step," there are simply no opportunities to skip anything. I suspect it never once hit the default threshold (reuse_threshold 0.2).** It might be effective when running at 20–25 steps without using turbo. However, since the reason for using turbo in the first place is speed, I doubt there is any point in a configuration that "removes turbo and adds EasyCache." **If you are running with turbo, the conclusion for today is that it makes no difference whether you use EasyCache or not.**

## 3. 事实 / 推测 / 建议

- **事实（C，单作者 8GB 实机）**：RTX 3060 Ti 8GB / 64GB RAM / Win11 / ComfyUI 0.31.0 / cu130；832×480/243 帧/8 步/er_sde/turbo lightx2v **605.7s**（两次 605.7 / 610.7，方差 0.8%）；5s/124 帧/6 步 warm **185.5s**；20→25 步只 1.12×（8GB 固定开销高）；**EasyCache 开/关 740.8 vs 735.9s（-0.7%，在误差内）**，作者归因「只有 8 步、没有可跳的步、没触发 reuse_threshold 0.2」。
- **推测（C/D）**：作者自称"likely / I suspect"，阈值未触发是**推测**；64GB RAM 比本机 32GB 宽裕，**605.7s 等耗时不可直接搬到 5070 Laptop 8GB/32GB**；Turbo 画质"按题材可好可坏"是主观目视，无指标。
- **建议（非结论）**：本机 V1.1 若用 8 步 Turbo，默认关闭 EasyCache；把"20 步→8 步"的省时与"缓存"分开评估；记录 warm-to-warm、固定 seed 的对照方法可直接复用。

## 4. 等级

**C**（第一手 8GB 实机数字、方法清楚，但**单作者单机、无第三方复现**；按 LOG-001 修订记录 r-grade 的同一标准，R1-27 亦已由 B 降 C）。EasyCache A/B 段落额外标 **片段读取（非直读）**，不冒充直读。
