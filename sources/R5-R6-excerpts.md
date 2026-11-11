

---

<!-- file: sources/R5-03-fasth3-trim.md -->

# R5-03｜comfyui-wiki FastH3 Trim

- URL: https://comfyui-wiki.com/zh/news/2026-10-06-fastvideo-fasth3-trim
- 访问日期：2026-11-10 JST
- 发布日期：2026-10-06
- 关键原文段（格式→GPU 与体积）：
  - "fastvideo_fasth3_trim_8step_nvfp4.safetensors 11.9 GB — NVIDIA Blackwell: RTX 50 系列、RTX PRO 6000、DGX Spark"
  - "fastvideo_fasth3_trim_8step_fp8.safetensors 19.7 GB — NVIDIA Ada 及更新架构: RTX 40 系列"
  - "fastvideo_fasth3_trim_8step_int8_convrot.safetensors 18.9 GB — 任意较新的 NVIDIA GPU;与模板默认值一致"
  - "fastvideo_fasth3_trim_8step_bf16.safetensors 37.5 GB — 参考质量"
  - "每个 NVFP4 层都使用在 1,000 条提示词上校准的激活缩放,Trim 共有 294 个已校准层。"
  - "使用 ComfyUI 自带的 FastVideo FastH3: Text to Video 模板(0.36.0 或更高版本)… 8 步、res_multistep、simple,以及 MiniMaxH3SigmaShift 节点在视频上设为 10、音频上设为 3。"
- 用途：L5-03。给出本机最直接可用的 NVFP4 11.9GB（Blackwell 专用）与 int8_convrot 18.9GB（通用）两条路线；二者均 >8GB 需 RAM 卸载。无 8GB 实测秒数。


---

<!-- file: sources/R5-04-civitaiarchives-native.md -->

# R5-04｜civitaiarchives ComfyUI-native 格式对照

- URL: https://civitaiarchives.com/seaart/models/fbc18b5f8dcc729efabaa00d07237c60/versions/6983d621317a819fbdf2a9b20656c4ff
- 访问日期：2026-11-10 JST（镜像归档，未标日期）
- 关键原文段（native/emulated 表与结论）：
  - "50-series (Blackwell) → nvfp4, 12.53 GB. The fastest thing here. NVFP4 needs Blackwell tensor cores. 40-series (Ada) → w4a8, 12.54 GB. 30-series on ComfyUI 0.32+ → w4a8 … Verified on a 3090."
  - 表：5090/0.32 int8_tensorwise/nvfp4/mxfp8/asym_w4a8_int8/float8_e4m3fn 全部 native；3090/0.33.1 int8/asym_w4a8 native，nvfp4/mxfp8/fp8 emulated；3090/0.30 int8 native，w4a8 absent。
  - "A 30-series owner who grabs the 12.5 GB file because it fits will get a worse result than one who takes the 21 GB file that runs native."
  - "Everything here is free … these formats have no GGUF equivalent — and because every one of them is faster than the GGUF ladder."
  - "Measured on two real machines … same scene, same seed, same shape — 544×960, 243 frames, 2 shots, 10 steps"
- 用途：L5-04（C 级，镜像单源）。机制支持 H1（Blackwell nvfp4 native 最快、emulated 更慢）与 H2（原生快于 GGUF）；无 8GB 计时，不可直信为 B。


---

<!-- file: sources/R5-05-nunchaku.md -->

# R5-05｜nunchaku (SVDQuant) NVFP4

- URL: https://github.com/nunchaku-tech/nunchaku （引擎仓库）+ https://github.com/nunchaku-tech/ComfyUI-nunchaku/issues/742 （RTX 5060 8GB 实测）+ arXiv:2411.05007（SVDQuant）
- 访问日期：2026-11-10 JST
- 关键原文段：
  - 引擎 News 2025-02-20："Support NVFP4 precision on NVIDIA RTX 5090! NVFP4 delivers superior image quality compared to INT4, offering ~3× speedup on the RTX 5090 over BF16."
  - 引擎 Performance 段："SVDQuant reduces the 12B FLUX.1 model size by 3.6× and cuts the 16-bit model's memory usage by 3.5×. … INT4 model runs 3.0× faster than the NF4 W4A16 baseline on both desktop and laptop NVIDIA RTX 4090 GPUs. Notably, on the laptop 4090, it achieves a total 10.1× speedup by eliminating CPU offloading. Our NVFP4 model is also 3.1× faster than both BF16 and NF4 on the RTX 5090 GPU."
  - Issue #742（RTX 5060 sm_120 8GB, Win11, CUDA 12.8, PyTorch 2.11 nightly, nunchaku 1.1.0+torch2.11 win_amd64 wheel）："RTX 5060 Laptop (8GB, sm_120) ✅ Fully supported"；"Model Loading: ~1.27 GB VRAM (FP4 quantized)；Inference Time: ~109 seconds (first image on RTX 5060)"；测试为 FLUX 图像（非 H3 视频）。
- 用途：L5-05（B 仓库/arXiv；5060 8GB 图像数字为 C 级旁证）。证明 NVFP4 Windows 预编译轮子可用于 8GB Blackwell（5060 sm_120），但需 PyTorch 2.11 nightly+cu128；H3 视频走 ComfyUI 原生 NVFP4（L5-02/03）。24GB+/16GB 卡秒数禁入 8GB 推荐。


---

<!-- file: sources/R5-06-city96-gguf.md -->

# R5-06｜city96/ComfyUI-GGUF

- URL: https://github.com/city96/ComfyUI-GGUF
- 访问日期：2026-11-10 JST（README 最近 2024-10-23；loader.py 2026-01-12）
- 关键原文段：
  - "GGUF Quantization support for native ComfyUI models … popularized by llama.cpp."
  - "While quantization wasn't feasible for regular UNET models (conv2d), transformer/DiT models such as flux seem less affected by quantization. This allows running it in much lower bits … on low-end GPUs."
  - 安装为自定义节点；支持 flux1-dev/schnell、sd3.5 GGUF；"Initial support for quantizing T5 has also been added"；未列出 MiniMax H3。
  - 仓库 README 无原生 fp4/fp4-tensor-core kernel 描述 → 权重以 GGUF 存、加载时反量化（dequant）到更高精度计算。
- 用途：L5-06（B 机制）。支持 H2：GGUF 反量化后 bf16 计算、无原生低比特 kernel，体积小但速度不如原生 FP8/NVFP4；8GB 上价值在压盘读/内存。H3 GGUF 是否存在未核实（city96 未列）→ D/C。


---

<!-- file: sources/R5-07-sysmem-fallback.md -->

# R5-07｜Sysmem Fallback（官方页失效 + 社区转述）

- 官方 URL（失效）：https://nvidia.custhelp.com/app/answers/detail/a_id/5490 —— 本轮直读返回 "Technical Difficulties / Oracle … Incident Number 0.e9ba1402…"，**无法访问**，故官方原文未核实（D）。
- 社区转述来源：
  - http://runaihome.com/blog/shared-gpu-memory-slow-local-ai-sysmem-fallback-fix-2026
  - https://freakyjolly.com/cuda-out-of-memory-fix
  - https://freakyjolly.com/how-much-vram-for-comfyui-16gb-test/
  - ComfyUI 文档（grokipedia 提及 disabling CUDA system memory fallback via NVIDIA Control Panel）
- 访问日期：2026-11-10 JST
- 关键原文段（社区，均引用 NVIDIA article 5490）：
  - runaihome："The fix: Prefer No Sysmem Fallback … Open NVIDIA Control Panel → Manage 3D Settings → CUDA - Sysmem Fallback Policy. Three options: Driver Default, Prefer Sysmem Fallback, Prefer No Sysmem Fallback."
  - freakyjolly："On Windows, NVIDIA drivers can spill into system RAM … instead of throwing an error. The job keeps running but can become 5-20x slower. … Set it to Prefer No Sysmem Fallback globally or for python.exe."
  - freakyjolly 16GB 实测："1.64 GB made the difference between 16 seconds and 26 minutes."
  - runaihome：python 类工具应指向实际解释器 `ComfyUI\venv\Scripts\python.exe`；策略在 CUDA context 创建时读取，需重启应用。
- 用途：L5-08（D 官方/ C 社区）/ L5-09。支持 H3：8GB 卡溢出风险高，建议对 ComfyUI python.exe 设 Prefer No Sysmem Fallback 并依赖自身卸载；无 H3 专属 A/B 计时（本机实验 2 补）。


---

<!-- file: sources/R5-08-comparedb-tgp.md -->

# R5-08｜comparedb RTX 5070 Laptop TGP 性能陷阱

- URL: https://www.comparedb.com/gpu/rtx-5070-laptop-tgp-specs-performance-brackets-1808313-3
- 访问日期：2026-11-10 JST（未标精确日期）
- 关键原文段：
  - "The RTX 5070 Laptop spans 13 to 23.2 TFLOPS across a 50–125W TGP range."
  - "The boost clock range spans 1410 MHz at the floor to 2520 MHz at the ceiling — a 78% swing. FP32 compute follows that curve: 13 TFLOPS at the low-power configuration, 23.2 TFLOPS at full power."
  - 三档：sub-70W（thin-and-light 税）；80–100W（主流，boost 维持 2000–2200 MHz，"continuous GPU-heavy workloads … Expect thermal throttling after 10–15 minutes"）；115W+（逼近 23.2 TFLOPS）。
  - "OEM TGP Disclosure … Product pages list TGP as a range — '50–125W' — which is compliant and useless."；130W 适配器是 sub-70W 硬上限。
- 用途：L5-10（C）。支持 H4：同「RTX 5070 Laptop」TGP 50–125W，算力差 78%（>30%）；长时负载 10–15 分钟热节流。与 L5-01 官方 50–100W 方向一致，上限更高。


---

<!-- file: sources/R5-09-dazhe-rog14.md -->

# R5-09｜dazhe ROG 幻14 Air RTX 5070 实测

- URL: https://www.dazhe.com/article/36947.html
- 访问日期：2026-11-10 JST（页标 2026 年）
- 关键原文段（恒温 25℃ 连续 30 分钟满载）：
  - "1、在双烤状态下,RTX 5070 独显功耗峰值锁定在 78W,持续 12 分钟后回落至 62W 并保持稳定,较官方标称的 90W TGP 低 31%；表面键盘区 C 面左上角温度达 69.3℃,触发风扇智能降频策略。"
  - "2、单烤 GPU 时功耗可短暂冲至 85W,但维持时间不足 90 秒,随后因热管饱和导致核心频率从 2175MHz 骤降至 1840MHz,降幅达 15.3%；"
  - "3、启用 ROG Armoury Crate 的「狂暴模式」后,GPU Boost Clock 强制锁定 2025MHz,但伴随 CPU Package 功耗被限制在 35W 以内,导致《古墓丽影:暗影》1080P 最高画质帧生成时间增加 17.6ms"
  - 机型：ROG 幻14 Air，整机 1.58kg，标称 RTX 5070（90W TGP）。
- 用途：L5-11（C）。支持 H4「长时降频可观测」：薄机身 5070 持续负载功率/频率显著跌落（−15~31%）。数分钟视频生成后半段明显变慢。


---

<!-- file: sources/R5-10-laptopjudge-tsight.md -->

# R5-10｜laptopjudge TUF A16 / tsight 功耗与持续负载

- URL: https://laptopjudge.com/asus-tuf-gaming-a16-ryzen-9-rtx-value + https://tsight.io/articles/17719159
- 访问日期：2026-11-10 JST
- 关键原文段：
  - laptopjudge（ASUS TUF A16, Ryzen 9 7945HX + RTX 5070, 140W GPU 功率目标）："It can help the RTX 5070 reach its intended performance range, but it does not guarantee 140W forever. In a 30-minute loop … If power falls toward 115W after 15 minutes, that is a sustained cooling limit, not a driver failure." 建议 30 分钟循环 + HWiNFO 记录 power/temp/clock/frame time；nvidia-smi `--query-gpu=name,temperature.gpu,utilization.gpu,power.draw,clocks.gr`。
  - tsight：「理论等效 ≈ 桌面版 RTX 5060 Ti；持续负载等效 ≈ 桌面版 RTX 5060 (或更低)：一旦进入 30 分钟以上的满载渲染…由于 TDP 限制导致的频率跌落(Throttling),其性能将迅速下滑。」「开机前 5 分钟看到的是理论等效,游戏 1 小时后才是真实性能。」
- 用途：L5-12（C）。提供长时生成监测方案（30 分钟循环、HWiNFO/nvidia-smi、平衡电源、垫高后部）；印证 H4「>30% 差异」与「长时降频可观测」。


---

<!-- file: sources/R5-11-pulsequant.md -->

# R5-11｜PulseQuant arXiv:2609.33384

- URL: https://arxiv.org/abs/2609.33384
- 访问日期：2026-11-10 JST
- 提交日期：v1 2026-09-27；v2 2026-09-29（近 3 个月窗口）
- 关键原文段（abstract）：
  - "PulseQuant, a 4-bit post-training quantization method that combines trajectory sensitivity with activation geometry to guide offline calibration."
  - "Evaluations on Wan models, Self Forcing, and MiniMax-H3 demonstrate improvements in key consistency and dense-reference metrics while remaining competitive on other attributes across model scales and generation paradigms."
- 用途：L5-13（C，论文自报）。证明 H3 已被纳入 4-bit 视频 DiT 量化研究；无 Blackwell/8GB 秒数。呼应 L5-02/03 的 NVFP4 实装方向。


---

<!-- file: sources/R5-12-tritile.md -->

# R5-12｜TriTile/Triage arXiv:2610.35706

- URL: https://arxiv.org/abs/2610.35706 （v1 2026-10-26，v2 2026-11-03）
- 访问日期：2026-11-10 JST
- 提交日期：v1 2026-10-26；v2 2026-11-03（近 3 个月窗口）
- 关键原文段（abstract，v2 标题 TriTile）：
  - "a training-free attention method that assigns each tile one of three approximation tiers: skipping it … skimming it in NVFP4, or refining it with NVFP4 residual correction."
  - "A fused kernel executes all three tiers in a single pass on the FP4 tensor cores of RTX Blackwell GPUs."
  - "Over dense FlashAttention, TriTile speeds up generation by 1.38-1.83x on Wan2.2 … 2.28-2.32x on HunyuanVideo … and 1.84x on LTX-2.5 … and in every setting it Pareto-dominates every baseline in speed and PSNR."
- 用途：L5-14（C，论文自报）。显示 NVFP4 不仅在权重、也在注意力层有 Blackwell 原生加速路径（融合 kernel 跑在 FP4 tensor cores）；与 L5-02 互补。无 H3 实测、无 8GB 秒数。


---

<!-- file: sources/R6-01.md -->

# R6-01｜ComfyUI 官方启动参数文档（startup-flags）

- **URL**：https://docs.comfy.org/development/comfyui-server/startup-flags
- **访问日期**：2026-11-11 JST
- **类型**：官方文档（基于 `comfy/cli_args.py`）
- **证据等级**：B

## 原文关键段（逐字摘录）
- VRAM & Memory 段首：「Async weight offloading is on by default on Nvidia and AMD GPUs, and dynamic VRAM is enabled automatically where it is supported. The flags below override those defaults.」
- `--async-offload [NUM_STREAMS]`：default 列「enabled on Nvidia and AMD」，说明「Async weight offloading. Optional stream count (default: 2).」；`--disable-async-offload`：「Disable async weight offloading.」
- `--disable-pinned-memory`：「Disable pinned memory use.」
- `--fast-disk`：「Force disk-backed dynamic loading and offload over unpinned RAM. Useful with fast NVMe. When neither this flag nor --disable-fast-disk is set, disk-backed loading and offload are enabled automatically if every model file is on storage detected as fast.」
- `--reserve-vram GB`：「VRAM in GB to reserve for the OS and other software.」
- `--vram-headroom GB`（default 0）：「Extra VRAM in GB for dynamic VRAM to keep free above the default headroom. ComfyUI tries to keep this much VRAM completely free and unused, even counting VRAM used by other apps.」
- `--lowvram`：「No effect when dynamic VRAM is enabled. Otherwise, runs text encoders on CPU.」
- `--novram`：「Minimal VRAM usage when --lowvram is not enough.」
- `--disable-smart-memory`：「Aggressively offload to RAM instead of keeping models in VRAM.」
- `--enable-triton-backend`：「Enable Triton backend in comfy-kitchen. Disabled by default at launch.」；`--disable-comfy-compiler`：「Disable the Comfy model compiler, including its CUDA graph subfeature.」
- VRAM mode flags（`--gpu-only`、`--highvram`、`--lowvram`、`--novram`、`--cpu`）are mutually exclusive.


---

<!-- file: sources/R6-02.md -->

# R6-02｜triton-windows 官方 README（woct0rdho/triton-windows）

- **URL**：https://github.com/woct0rdho/triton-windows
- **访问日期**：2026-11-11 JST
- **类型**：官方仓库 README
- **证据等级**：B

## 原文关键段（逐字摘录）
- GPU 支持矩阵：「RTX 50xx (Blackwell architecture, sm120) This is officially supported by Triton. It only works with Triton >= 3.3, PyTorch >= 2.7, and CUDA >= 12.8 .」
- What's supported：「triton.jit and torch.compile just work」「All unit tests passed」「It's as fast as on WSL on the same machine」「Windows 10 and 11 are supported」「Nvidia GPU is supported」
- PyTorch↔Triton 映射：2.7→3.3, 2.8→3.4, 2.9→3.5, 2.10→3.6；CUDA 工具链自 3.2 起打包进 wheel。
- 安装示例：「pip install -U "triton-windows<3.7"」；embeded Python：「C:\path\to\python_embeded\python.exe -m pip install -U "triton-windows<3.7"」
- **Windows 已知问题（逐字）**：「Windows file path length limit (260) causes compilation failure — torch.compile may create temp files with very long filenames, causing errors... The solution is to enable Windows' long path support... A reboot is required.」
- vcredist：「vcredist is required... a new enough version, such as 14.42, is required by my Triton wheels.」
- Blackwell fp8/int4 数据类型未列排除项（与 Ampere/Turing 不同），即 sm120 无对应限制。


---

<!-- file: sources/R6-03.md -->

# R6-03｜NVIDIA 官方支持文 5490（System Memory Fallback for Stable Diffusion）

- **URL**：https://nvidia.custhelp.com/app/answers/detail/a_id/5490/~/system-memory-fallback-for-stable-diffusion
- **访问日期**：2026-11-11 JST（R5 轮 2026-11-10 返回 Oracle 故障页，本轮可直读）
- **类型**：NVIDIA 官方支持文
- **发布/更新**：Updated 10/31/2023
- **证据等级**：B
- **用途**：LOG-005 r1 修订 L5-08（升级 B）、L5-09（官方来源更正）

## 原文关键段（逐字摘录）
- 「In driver 536.40, we implemented a new method to allow an application to use shared memory in cases that exhausted the GPU memory. This enabled applications which previously crashed when running out of GPU memory to continue to run, albeit at lower speeds. The switch to use shared memory occurs when running close to maxing out GPU memory to allow for a seamless transition.」
- 「Stable Diffusion happens to require close to 6 GB of GPU memory often. This can cause the above mechanism to be invoked for people on 6 GB GPUs, reducing the application speed.」
- 「In driver 546.01 and above we added a setting to disable the shared memory fallback, which should make performance stable at the risk of a crash if the user uses a setting that requires more GPU memory.」
- 步骤（逐字要点）：Program Settings → 添加 python.exe → 「CUDA - Sysmem Fallback Policy」→ 设「Prefer No Sysmem Fallback」；「Re-launch Stable Diffusion. Note that Stable Diffusion must be restarted following these steps for this setting change to be applied.」


---

<!-- file: sources/R6-04.md -->

# R6-04｜NVIDIA 英文官方 RTX 5070 Laptop 规格对比页

- **URL**：https://www.nvidia.com/en-us/geforce/laptops/compare/
- **访问日期**：2026-11-11 JST
- **类型**：NVIDIA 官方规格页
- **证据等级**：B
- **用途**：LOG-005 r1 修订 L5-01（TGP 冲突核对）

## 原文关键段（逐字摘录，RTX 5070 Laptop 行）
- **GPU Subsystem Power (W)**：「50 - 100 W」（RTX 5070 Laptop）
- Boost Clock (MHz)：1425 - 2347 MHz
- NVIDIA CUDA Cores：4608
- AI TOPS：798
- Standard Memory Configuration：**12 GB GDDR7**（注：官方对比表列 5070 Laptop 为 12GB；本任务目标机是 8GB OEM 变体，官方表未单列 8GB 行）
- Memory Interface Width：128-bit
- CUDA Capability：12.8
- Tensor Cores：Fifth Generation；Architecture：Blackwell
- 脚注仅关于 PCIe(1)/HDMI(2)/DisplayPort(3)，**未找到 Dynamic Boost 额外功率脚注**。

## 冲突记录
- 官方（本页）= 50–100W；第三方 comparedb（LOG-005 L5-10，本轮补日期 Jun 3 2026 / Updated Jul 30 2026）= 50–125W。按规则不自行取舍，记为「冲突未解决」（LOG-005 r1.1）。


---

<!-- file: sources/R6-05.md -->

# R6-05｜freakyjolly · MiniMax H3 16GB OOM 修复（PR #15446 / #15456）

- **URL**：https://freakyjolly.com/minimax-h3-out-of-memory-comfyui-16gb-fix
- **访问日期**：2026-11-11 JST
- **类型**：社区 H3 实测（作者单机）
- **证据等级**：C（16GB 卡，参数名与官方一致；数字禁入 8GB 推荐表）

## 原文关键段
- 「PR #15446 makes H3 decode memory stop scaling with clip length, and PR #15456 frees other models' memory before the tiled-decode retry.」
- 「Before #15456, the 'retrying with tiled VAE decoding' fallback re-ran the same decode with ~14 GB of streamed weights still resident and failed again.」
- 启动参数表（均真实存在于 cli_args.py）：`--reserve-vram 1.0`（Always on Windows with a browser open）、`--cache-none`、`--disable-pinned-memory`、`--disable-smart-memory`、`--disable-async-offload`（More stable, but slower）、`--lowvram`/`--novram`（Older low-VRAM modes）。
- 「System RAM 64 GB recommended. My worker peaked around 27 GB RAM in one job.」
- 「Pagefile matters... My machine at one point sat at 120 to 145% commit, and that is when loads start failing. Set the pagefile to system-managed on a fast NVMe, or give it a large fixed size.」
- 低显存节点：`MiniMaxLowVRAMAttention` + `MiniMaxChunkFeedForward(4096)`。
- 测试配置：RTX 16GB；int8_convrot T2V；Megapixels 0.3；Duration 4–6s（124 帧约 5s）；Steps 6 + turbo LoRA。


---

<!-- file: sources/R6-06.md -->

# R6-06｜instasd · ComfyUI VRAM 卸载指南（pinned memory OOM + --fast-disk + RAM 是瓶颈）

- **URL**：https://www.instasd.com/post/comfyui-vram-offloading-guide
- **访问日期**：2026-11-11 JST
- **类型**：社区多源汇总（Reddit/论坛）
- **证据等级**：C

## 原文关键段（逐字摘录）
- pinned memory 与 OOM：「The source calls cudaHostRegister directly and keeps a running total of how much it has pinned. ... A user with a 5060 Ti and 128 GB of RAM, more memory than almost anyone else reporting, was hitting out-of-memory errors on the default H3 template. The fix was not more capacity anywhere. It was --disable-pinned-memory. In their words: 'It was that stupid pinned memory. I disabled it and the OOM's stopped.' So the current triage order when a card OOMs on a template that smaller cards run fine: try --disable-pinned-memory before you touch anything else.」
- `--fast-disk`：「described as 'Prefer disk-backed dynamic loading and offload over unpinned RAM. Can be faster for users with fast NVME disks.'」层级：VRAM → pinned RAM → system RAM → NVMe。
- 「System RAM is the spec that actually decides ... The 12 GB and 16 GB success stories almost all say 64 GB of RAM in the same breath. The OOM reports cluster at 32 GB.」
- V100 用户测量：5 秒 0.2MP H3 用 28 GB VRAM + 54 GB 系统 RAM。
- 软失败：4070 在 608x352 用 167 秒，864x480 用 11 分钟（4× 慢，因模型 spill）。


---

<!-- file: sources/R6-07.md -->

# R6-07｜Abiray/MiniMax-H3-Pruned-GGUF（HF 真实仓库）

- **URL**：https://huggingface.co/Abiray/MiniMax-H3-Pruned-GGUF （主）+ https://savrn.com/models/minimax-h3-pruned-gguf （镜像含大小/SHA）
- **访问日期**：2026-11-11 JST
- **类型**：Hugging Face 模型卡
- **发布**：First published 2026-08-07（savrn 镜像 revision 76061e7，2026-08-07）
- **证据等级**：B（仓库真实、文件大小可核对）

## 原文关键段（文件大小，逐字/卡页）
- Requirement：「ComfyUI: Version v0.30.0 or higher is required for native MiniMax-H3 architecture support.」「Extension: ComfyUI-GGUF custom node package installed.」「load the model using the UnetLoaderGGUF node.」
- FL2VA（First-and-last-frame）：Q3_K_M 8.9 GB、Q4_K_S 11.6 GB、Q4_K_M 11.6 GB、Q5_K_S 14.1 GB、Q5_K_M 14.1 GB、Q6_K 16.7 GB、Q8_0 21.6 GB。
- Ref2VA（Omni-reference）：Q3_K_M 8.9 GB、Q4_K_M 11.6 GB、Q5_K_M 14.1 GB、Q6_K 16.7 GB、Q8_0 21.6 GB。
- 说明：「The pruned architecture is compressed down to 8.9 GB – 21.6 GB, bringing MiniMax H3 execution directly to consumer hardware.」Q4_K_M 标「Recommended Balance (16 GB GPUs)」、Q3_K_M 标「Low VRAM (~12 GB GPUs)」。


---

<!-- file: sources/R6-08.md -->

# R6-08｜freakyjolly · 最佳 MiniMax H3 GGUF 工作流（社区实测）

- **URL**：https://freakyjolly.com/best-minimax-h3-gguf-workflow-comfyui/
- **访问日期**：2026-11-11 JST
- **类型**：社区 H3 GGUF 工作流（作者单机，16GB 卡）
- **证据等级**：C

## 原文关键段
- 文件：UnetLoaderGGUF 加载 `minimax_h3_fl2va_pruned-Q4_K_M.gguf` **11.4 GB**；CLIPLoaderGGUF(typeminimax) 加载 `qwen3vl-32B-MiniMax-H3-Q2_K.gguf` **8.49 GB**；VAE 用 Comfy-Org/MiniMax-H3 的 fp16/fp32 safetensors（~5.2GB / ~0.6GB）。
- **架构支持坑（逐字）**：「On my install it threw 'Unknown model architecture!' until I added a minimax_h3 entry to the pack's architecture list in loader.py. leejet's model card recommends leejet/ComfyUI-GGUF instead, because the city96 repository appears to no longer be actively maintained. If you're starting fresh, try leejet's fork first.」
- 其它 H3 GGUF 仓库：unsloth/MiniMax-H3-GGUF（pruned FL2VA/Ref2VA 从 Q2_K 6.26GiB 到 Q8_0，含两个 Qwen3-VL 编码器）；Abiray/MiniMax-H3-Pruned-GGUF（16GB 用 Q4_K_M、24GB 用 Q5_K_M）；molbal/MiniMax-H3-GGUF（U16G 混合量化，需自有 ComfyUI-GGUF fork）。
- 收益：切换 GGUF 后单实例 UNet 侧省约 7.4GB 提交 RAM（11.4GB vs 18.8GB int8）、编码器侧省约 6GB。


---

<!-- file: sources/R6-09.md -->

# R6-09｜kijai ComfyUI-WanVideoWrapper Block Swap（deepwiki + comfy.icu）

- **URL**：https://deepwiki.com/kijai/ComfyUI-WanVideoWrapper/14-performance-optimization-guide （基于仓库源码）+ https://comfy.icu/node/WanVideoBlockSwap
- **访问日期**：2026-11-11 JST
- **类型**：DeepWiki（基于 nodes_model_loading.py 等源码）+ 节点说明页
- **证据等级**：B/C（参数/源码可核对偏 B；8GB 推荐档为建议标 C）

## 原文关键段（deepwiki）
- Block Swap 参数：blocks_to_swap（INT 0-48，14B=40 块、1.3B/5B=30、LongCat=48）、vace_blocks_to_swap（0-15）、prefetch_blocks（0-40，1-2 通常抵消速度损失）、use_non_blocking（更快但更占 RAM）、offload_img_emb / offload_txt_emb（省 1-2GB VRAM）、block_swap_debug。
- 按 VRAM 推荐：24GB+（blocks_to_swap 0-10, prefetch 0-1, non_blocking false）；16GB（15-20, 1-2, true）；**12GB（25-30, 1-2, true）；8GB（35-40, 0-1, true）**。
- 对比 Block Swapping vs Model Offloading：「Block Swapping: Faster (selective offload)」「Model Offloading: Slower (more aggressive)」；Block Swap 粒度 per-block，Model Offloading 按参数百分比。
- 计时示例（nodes_sampler 日志）：「Block swap timing for block 15: load=0.023s, compute=0.145s, offload=0.019s」


---

<!-- file: sources/R6-10.md -->

# R6-10｜kentstone84 · PyTorch sm_120 Windows 兼容性核实文档

- **URL**：https://github.com/kentstone84/PyTorch-2.10.0a0/blob/main/docs/REDDIT_RESPONSE.md
- **访问日期**：2026-11-11 JST
- **类型**：社区对「PyTorch cu128 已支持 sm_120」说法的核实文档
- **证据等级**：C（引述用户报告与 GitHub issue）

## 原文关键段（逐字摘录）
- 「PyTorch 2.7.0 Release (April 23, 2025) First stable release with official sm_120 (Blackwell) support. Includes pre-built CUDA 12.8 binaries for Linux.」
- 「Windows support - While CUDA 12.8 binaries exist for Windows in PyTorch 2.7, there are ongoing compatibility issues reported by users」
- 「Performance - PTX JIT compilation fallback still occurs in some scenarios, reducing performance by 20-30%」
- 「Triton support - Custom kernel compilation for sm_120 on Windows has reported issues」
- 用户报告：`RuntimeError: CUDA error: no kernel image is available for execution on the device`、DLL 加载错误、Triton 编译失败；GitHub Issue #159207 持续讨论。


---

<!-- file: sources/R6-11.md -->

# R6-11｜smeltcore · VoxCPM2 on RTX 5070 Ti（同族 Blackwell sm_120 Windows 实战）

- **URL**：https://smeltcore.com/recipes/voxcpm2-on-rtx-5070-ti-30-language-48khz-voice-cloning-in-8-gb-vram
- **访问日期**：2026-11-11 JST
- **类型**：社区 recipe（RTX 5070 Ti 16GB，Blackwell sm_120，~8GB VRAM 目标）
- **证据等级**：C

## 原文关键段
- Blackwell（sm_120）kernel 缺口：「'not compatible with the current PyTorch installation' error, or silent fallbacks/crashes after load, means torch was installed without sm_120 kernels. The fix is to (re)install PyTorch from a CUDA 12.8+ index」；Issue #258 协作者建议升级 CUDA 与 torch；**PR #250 记录 CUDA 13.0 是原生 sm_120 支持最干净的 toolchain**。
- torch.compile / Triton 报错（warm-up）：`torch._dynamo.exc.Unsupported` 或 Triton import 失败，源于版本不匹配；快速修复 = 跳过 torch.compile（`optimize=False`）。
- **Windows 特有问题（逐字要点）**：「on Issue #258 reports that optimize=True (torch.compile + CUDA Graphs) can also hit a thread-local-storage assertion after a few generations when inference runs inside Gradio's background worker thread — optimize=False sidesteps it.」
- PyTorch↔Triton 兼容表：2.7→3.3, 2.8→3.4 等。


---

<!-- file: sources/R6-12.md -->

# R6-12｜arXiv 2609.21849「The Weight Is Over — Interactive Diffusion on Consumer GPUs」

- **URL**：https://arxiv.org/html/2609.21849v1
- **访问日期**：2026-11-11 JST
- **类型**：arXiv（SIGGRAPH Asia 2026 Technical Communications）
- **发布**：v1，2026-09（arXiv ID 2609.21849）；属近 3 月
- **证据等级**：C（自报、无第三方复现、无 H3/8GB 计时）

## 原文关键段（摘要要点）
- 「Central is low-bit quantization (FP8/NVFP4), which yields two separable effects: weight quantization gives a significant reduction in memory footprint, while activation quantization adds a speedup on compute-bound stages.」
- 「For multi-model pipelines whose weights exceed VRAM, a carefully designed GPU-memory offloading scheme is required across text encoders, VAE, and auxiliary conditioning models.」
- 结果：「time-to-first-image-iteration (TTFI) under one second on recent GPUs, and maintained quality of service across a much wider set of hardware.」
- 贡献：embedding translator（小编码器对齐大编码器）、speed/quality/memory 三角 sweep recipe、亚秒 TTFI 交互式编辑器。


---

<!-- file: sources/R6-13.md -->

# R6-13｜arXiv 2611.08023「Precision-Preserving Subtensor Inference (PSI)」

- **URL**：https://arxiv.org/html/2611.08023v1
- **访问日期**：2026-11-11 JST
- **类型**：arXiv（CONSTRAINT 2026 Workshop @ IEEE BigData 2026）
- **发布**：v1，2026-11（arXiv ID 2611.08023）；属近 3 月
- **证据等级**：C（自报、单 GPU 实测、无 H3/8GB 计时）

## 原文关键段（摘要要点）
- 对象：BF16 **LTX-2.5 22B** 联合音视频 transformer；固定 GPU 工作集（fixed-arena）runtime；9 个 matched full-prompt cases on **RTX A6000**；固定 3 GiB available-memory cap。
- 结果（自报）：modified PSI 把 sampled transformer-process **peak memory 从 8,578 MiB 降到 360 MiB**；managed arena 80 MiB；11 次 transformer 调用 mean time 从 **247.11 s 升到 349.00 s**（约 +41%）；固定 3 GiB 上限下完整生成成功（27/27 解码成功）。FFN reuse 降 mean block time 4.0%。
- 方法：reusable weight pages + indexed host state；sub-block weight streaming + 分开管理的 intermediate state。


---

<!-- file: sources/R6-14.md -->

# R6-14｜unsloth/MiniMax-H3-GGUF（HF 真实仓库）

- **URL**：https://huggingface.co/unsloth/MiniMax-H3-GGUF
- **访问日期**：2026-11-11 JST
- **类型**：Hugging Face 模型卡（社区转换，非官方 MiniMax）
- **证据等级**：B（仓库真实）

## 原文关键段
- 提供 GGUF 量化 + 组件文件；含 pruned FL2VA 与 Ref2VA（从 Q2_K 6.26 GiB 到 Q8_0），加两个 Qwen3-VL 编码器（Q2_K_M 12.20 GiB、Q4_K_M 16.97GiB）。
- 与 Abiray 同期（2026-08 前后）；均属社区 H3 GGUF 量化。
- 注：unsloth 卡建议「pair the Q2_K_M encoder with the two smallest denoisers and the Q4_K_M encoder with everything else」。


---

<!-- file: sources/R6-15.md -->

# R6-15｜comfy-aimdo（AI Model Dynamic Offloader）异步权重流式卸载

- **URL**：https://alishangtian.com/article/user-9065771665 （社区博文引述 PyPI 包 comfy-aimdo v0.5.6）
- **访问日期**：2026-11-11 JST
- **类型**：第三方博文（引述 ComfyUI v0.40+ 异步权重流式架构与 PyPI 包）
- **证据等级**：C（未直读源码/基准；机制与官方默认一致）

## 原文关键段
- 「2026 年 ComfyUI 做的一次最重要的架构手术，是把权重搬运从『同步阻塞』改成『异步流水线』，并抽成了独立 PyPI 包 comfy-aimdo（当前版本 0.5.6）。」
- 「它提供 host_buffer 与 vram_buffer 两个缓冲池，配合多 CUDA stream 工作：源码中 NUM_STREAMS 控制流数量，一条流上的 kernel 在消费第 N 层权重时，另一条流已经在把第 N+1 层从主机内存搬入显存、并把冷权重搬回，计算与传输重叠...日志里对应的提示是 Using async weight offloading with N streams。」
- 「README 的说法是 4GB VRAM + 8GB RAM 即可相对快速地跑最大的开源模型且打满 GPU 算力。」（厂商宣称，标 C；8GB RAM 比本机 32GB 更小）
- v0.42.0 提交：「CPU device doesn't support fp8, nvfp4, mxfp8 or int8」——量化格式与搬运路径深度耦合。
