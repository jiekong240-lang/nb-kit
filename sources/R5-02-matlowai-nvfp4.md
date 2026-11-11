# R5-02｜MATLOWAI/minimax-h3-nvfp4 HF 卡

- URL: https://huggingface.co/MATLOWAI/minimax-h3-nvfp4
- 访问日期：2026-11-10 JST
- 发布/量化日期：卡页写 "post-training weight quantisation … performed 2026-08-19"
- 关键原文段：
  - "An NVFP4 quantisation of the MiniMax H3 pruned FL2VA diffusion model for ComfyUI. Every one of the 200 block linears (qkv, out, fc1, fc2 in all 50 blocks) is stored as fp4 e2m1 with fp8 e4m3 block scales of 16 and one fp32 per-tensor scale; embeds, token refiner, norms and heads stay bf16."
  - "On Blackwell (sm120) the GEMMs run natively on the fp4 tensor cores."
  - "Measured against the W4A8 file on the same seed and graph: 0.84x wall at 29k tokens, 0.90x at 45k, on an RTX PRO 6000. Same file size as W4A8 (12.5 GB, both 4.54 bits per parameter), so this is a speed play, not a memory play."
  - "Any other GPU: no. ComfyUI falls back to dequantised matmuls, which is slower than int8_convrot. Use Comfy-Org's int8_convrot file instead."
  - "The lightx2v turbo LoRA stacks cleanly … measured 12-step turbo, 210 s vs 239 s on the W4A8 file (0.88x)"
  - "Median weight error is 9.4 percent relative rms vs bf16 (the W4A8 file carries 7.3, int8_convrot 1.0)."
- 用途：L5-02。所有速度数字在 RTX PRO 6000（96GB，非 8GB）→ 禁入 8GB 推荐；但「sm120 原生 fp4 TC」「画质代价 9.4% > int8 1.0%」「文件 12.5GB 需卸载」对 8GB 适用。
