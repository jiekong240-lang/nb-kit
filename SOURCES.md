# FrontierLab-H3 原始资料笔记（研究指挥 BOT 于 2026-11-03 夜侦察所得，2026-11-04 01:09 JST 整理）

> 用途：给 WorkBuddy 编写 DOSSIER.md / RESEARCH-LOG 的输入。这是**二手笔记**，WorkBuddy 必须自己重新打开每个链接核实，不得直接照抄数字。
> 证据等级（规格第五节）：A=本机已复现（目前无）；B=外部可复现（公开源码+完整方法+第三方复现记录）；C=研究候选（有真实代码/论文，无直接复现）；D=未经证实（宣称/无方法的榜单/推测）。
> 目标硬件：Junlong 的 HP OMEN 16 笔记本，RTX 5070 Laptop 8GB（Blackwell，sm_120，驱动 616.92，8151 MiB），i9-14900HX，32GB RAM，Win11 家庭版 26200。
> 已知 PC 现状（来自 WB-REPLY-20261103.md §G1）：ComfyUI 为源码+venv 安装，位于 `C:\Users\testuser\WorkBuddy\comfyui\ComfyUI`，版本未知；顶层有 temp-gguf、weights 等目录。

## 1. 官方发布
- 官方公告（2026-08-03 开源）：https://www.minimax.io/news/minimax-h3-open-source  【B】
- HF 模型卡：https://huggingface.co/MiniMaxAI/MiniMax-H3 （ModelScope：MiniMax/MiniMax-H3）【B】
  - H3-Omni-Transformer：33B 稠密单流 Transformer，约 13B 在 AdaLN 分支，推理可预计算/裁剪（pruned ≈20B）
  - 文本编码器：Qwen3-VL-32B 全量权重，取第 50 层 hidden states
  - VisualVAE f16t4d24；AudioVAE 32kHz 立体声 40Hz latent
  - 输出：短边 768（默认）、24fps、4–15 s、32 kHz 立体声；2K 需 H3-Regenerate-2K（未开源，仅 API）；H3-Context-IR 未开源（仅 API）
  - 两个检查点：FL2VA（t2va/fl2va）、Ref2VA（参考生成）；CFG 蒸馏，BF16
  - 首发仅 full attention，稀疏注意力“未来发布”
- GitHub：https://github.com/MiniMax-AI/MiniMax-H3/ 【B】
- 许可证：https://huggingface.co/MiniMaxAI/MiniMax-H3/raw/main/LICENSE （MiniMax H3 Community License）【B】
  - Excluded Territories：欧盟、英国、韩国、美国（日本、中国不在排除列表）
  - 年营收 > 2000 万美元的商用需书面授权
- Comfy 官方教程：https://docs.comfy.org/tutorials/video/minimax/minimax-h3 【B】—— 写明本地生成内容商用需通过 Comfy 购买 MiniMax 商用许可
- MiniMax 自托管文档：https://platform.minimax.io/docs/guides/local-deploy-h3 （SGLang 参考配置 8×B200）【B】

## 2. 权重变体
- Comfy-Org/MiniMax-H3（ComfyUI 重新打包）：https://huggingface.co/Comfy-Org/MiniMax-H3 ；文件清单见 https://comfyui-wiki.com/en/news/2026-08-03-minimax-h3-open-weights-comfyui 【B】
  - DiT：fl2va_bf16 61.7GB / fl2va_int8_convrot 31.7GB / fl2va_pruned_int8_convrot 19.5GB（及 ref2va 对应）
  - TE：qwen3vl_32b bf16 48.0GB / int8_convrot 25.3GB / nvfp4_awq 14.6GB
  - VAE：video_vae_fp16 4.9GB；audio_vae_fp32 0.6GB
- unsloth GGUF：https://huggingface.co/unsloth/MiniMax-H3-GGUF 【C】pruned Q2_K 6.26GiB … Q8_0 19.97GiB；TE GGUF Q2_K_M 12.20GiB / Q4_K_M 16.97GiB；示例用 stable-diffusion.cpp `sd-cli --mode vid_gen ... --cfg-scale 1.0 --backend te=cpu --diffusion-fa`
- joeygambino GGUF：https://huggingface.co/joeygambino/MiniMax-H3-GGUF 【C】需要 ComfyUI-GGUF + 架构补丁（节点包 https://github.com/jlucasmcrell/comfyui-h3-multishot ）；city96 原版 ComfyUI-GGUF 不认 H3（据 genvidkit，D 级来源，需核实）
- DiffSynth NF4：https://huggingface.co/DiffSynth-Studio/MiniMax-H3-NF4 【C】bitsandbytes NF4，磁盘卸载，宣称“最低 8GB”；pruned DiT NF4 约 9.8GB；VRAM 管理文档 https://github.com/modelscope/DiffSynth-Studio/blob/main/docs/en/Pipeline_Usage/VRAM_management.md
  - Colab T4 实测仅在 288×160 / 22 帧 / 6 步完成：https://zenn.dev/nakanishitf/articles/bec3dc44dd1b65?locale=en 【C】
- Kijai：https://huggingface.co/Kijai/MiniMax-H3_comfy （pruned + Turbo LoRA）【C】
- Turbo LoRA：Abiray https://huggingface.co/Abiray/MiniMax-H3-Turbo-Lora-Pruned-ComfyUI （推荐 8–12 步、video shift 12 / audio shift 6、res_multistep）；larryvrh/MiniMax-H3-Turbo-Lora（8 步）；lightx2v/Minimax-h3-Turbo（4 步，alpha 8）【B（8GB 上有第三方实测）】
- W4A8：realrebelai/Rebels_w4a8s（仅 Ref2VA，非 pruned，22.8GiB）——来源为下方 Qiita 3070 文章【C】

## 3. 运行时
- ComfyUI 原生支持：v0.30.0 起（PR #15224，2026-08-03），模板 video_minimax_h3_t2v / i2v / r2v，无需自定义节点【B】
- SGLang cookbook：https://docs.sglang.io/cookbook/diffusion/MiniMax/MiniMax-H3 【C（对本机）】
  - 消费级最小配方 Recipe A：12GB VRAM + 32GB host，`--performance-mode memory --layerwise-offload-components dit,text_encoder,vae --layerwise-resident-layers video_vae=36`，864×480/124 帧/20 NFE 16.8–18.7 s/it（实验室 2TB 主机模拟，真实 32GB 主机会更慢；要求 NVMe）
  - 没有 8GB 配方；SGLang 仅 Linux → Windows 需 WSL2
  - 4090 24GB 配方：convrot_int8 + layerwise offload，GPU 峰值约 18GB；Spectrum 跳步 3.53×

## 4. 8GB 社区实测（Windows，除注明外均为 ComfyUI）
- RTX 3070 8GB / 48GB RAM（2026-08-11）：https://qiita.com/Kiyoaki_SUgiyama/items/bff51bdf389a732dc87c 【B】
  - ComfyUI 0.31.0，torch 2.11.0+cu130（初测 cu128）；DiT minimax_h3_fl2va_pruned_int8_convrot；TE qwen3vl_32b_minimax_h3_nvfp4_awq；video_vae_fp16；Turbo LoRA minimax_h3_turbo_4step_ckpt600_ema_V4
  - 12 steps、sigma shift video 12/audio 6、res_multistep/simple、SDPA
  - 124 帧（cu128）：352×608 8 步 213.4s / 6,682MiB；480×832 8 步 345.2s / 7,322MiB；544×800 I2V 8 步 392.0s；544×960 558.0s；608×896 I2V OOM
  - cu130 后 int8 路径 1.56×（R2V 313.4→200.9s）；宽高必须 32 的倍数；SageAttention 致纯噪声（ComfyUI #15263）；ckpt850_V1 LoRA 出环状伪影
- RTX 3070 Ti 8GB / 16GB RAM（2026-08-13）：https://qiita.com/makuragadekasugiru810114514/items/471ecc2c041e431359c6 【B】
  - ComfyUI 0.32.0，torch 2.9.1+cu130，comfy-kitchen 0.2.30；W4A8 mixed DiT+TE、int8 video VAE、Turbo LoRA
  - 864×480/124 帧：4 步 warm 79.74s（cold 89.56s）vs 20 步 271.84s；峰值 6.47–6.98GiB；1344×768 4 步 255.35s；DynamicVRAM 2-stream 最快
- RTX 4060 Laptop 8GB / 32GB RAM（2026-08-17）：https://note.com/rono_ai/n/n982a96de3812?hl=en 【B】
  - pruned int8 + nvfp4 TE；0.3MP（约 416×736，9:16）I2V；5s 8:53，换 comfy kitchen attention 后 6:30；10s 可行；Infinite workflow 拼接约 18s：https://civitai.com/models/2852849/h3-infinite-video-continuation-and-automatic-stitching-with-fl2va-model
- 8GB 笔记本 15 秒（zenn）：https://zenn.dev/cosara/articles/vram8gb-minimax-h3-15s 【B/C】
  - 800×448/15.083s 成功（908.1s）；832×448 及以上 OOM；8 次蓝屏原因=NVIDIA“CUDA - 系统内存回退策略”，设为“优先无系统内存回退”后 0 次；换坞/供电 655.8s→237.2s
- Tomiigo 8GB 指南（5070 Ti 模拟 8,188MiB 上限，非实体 8GB 卡）：https://github.com/Tomiigo/minimax-h3-16gb 【C】`python main.py --disable-pinned-memory --reserve-vram 1.5`，峰值 7,130–7,669MiB
- 其它待核实：note JUNI https://note.com/j_alpha_lab/n/n2661aa9a31cd?hl=en （搜索摘要：10s 8 步 10:06，15s 35:14，未打开）【D 直至打开】；ezlaunch https://github.com/yume-arasaki/ezlaunch-minimax-h3 （未打开）【D】

## 5. 已知问题与修复（GitHub）
- ComfyUI #15663 短片 VRAM 估计偏低 OOM；根因 comfy-aimdo 初始化失败导致 DynamicVRAM 未启用；临时 flags `--disable-pinned-memory --reserve-vram 1 --disable-async-offload`：https://github.com/Comfy-Org/ComfyUI/issues/15663 【B】
- ComfyUI PR #15456 handles_tiling VAE 解码 OOM 重试前释放其它模型：https://github.com/Comfy-Org/ComfyUI/pull/15456 【B】
- ComfyUI #16175 Model Sparse Attention 节点在 H3 OOM（已标修复）：https://github.com/Comfy-Org/ComfyUI/issues/16175 【B】
- ComfyUI #15263 SageAttention 输出纯噪声（来源：Qiita 3070 文章转述，链接需核实）【C】

## 6. 第三方“榜单”站（无方法，D 级，只可作线索）
- https://whichh3.com/en/workflows/low-vram-gguf
- https://www.genvidkit.com/blog/minimax-h3-gguf
- https://openclawdc.com/blog/can-i-run-minimax-h3-locally/
- https://localrigcheck.com/minimax-h3/ ；https://minimaxh3.cc/guides/hardware-requirements ；https://vidirect.org/models/hailuo-3-h3

## 7. 替代模型（不是 MiniMax H3，只能作兜底，禁止冒充 H3 成功）
- Wan2.1 T2V-1.3B：README 写 8.19GB VRAM https://github.com/Wan-Video/Wan2.1 【B】
- LTX-Video：README 写 RTX 4060 8GB 上 720×480×121 一分钟内 https://github.com/Lightricks/LTX-Video 【C】
- Wan2.2 TI2V-5B：官方 24GB+offload https://github.com/Wan-Video/Wan2.2 ；Qiita 3070 作者在 8GB 上运行【C】

## 8. 侦察者的推断（未验证，C/D）
- 5070 Laptop 为 Blackwell，NVFP4 应有硬件加速（SGLang 文档写 NVFP4 需 CC≥10.0；sm_120=12.0）【C】
- 驱动 616.92 应支持 cu130（Qiita 作者称 r580+ 即可）【C】
- 最小 FL2VA 下载约 43GB；32GB RAM 下需 NVMe + 足够 pagefile（5070 Ti 16GB 有“32GB RAM 加载卡死”传闻，来源 D）【D】
