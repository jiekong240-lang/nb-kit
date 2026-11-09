# R4-09｜comfy.icu WanVideo EasyCache 节点页（Turbo + cache 冲突，反面）

- URL：https://comfy.icu/node/WanVideoEasyCache
- 读取时间：2026-11-08 JST
- 读取方式：WebFetch 直连打开。
- 页面标注（逐字）：`By kijai · Created 2 years ago · Updated 6 months ago · 6,811`
- 类别：LOG-004 (4) Turbo + cache 冲突（反面）；(2) 缓存开关条件；(6) VRAM/RAM。

## 逐字摘录

标题（逐字）：

> # WanVideo EasyCache
> **Skip redundant Wan steps — but not with speed LoRAs**

节点默认参数（逐字）：

> `cache_args` ◄`easycache_thresh` **0.015**► ◄`start_step` **10**► ◄`end_step` **-1**► ◄`cache_device` **offload_device**►

> EasyCache is a speedup that works by being lazy in a smart way: across a diffusion run, consecutive steps often produce nearly identical output, so instead of computing every step in full, it reuses ("caches") a previous result when the change is small enough. It's **the same family as TeaCache and MagCache — a step-skipping cache** — sourced from the H-EmbodVis/EasyCache project and wired into WanVideoWrapper as a native node.
> Whether it's worth it is genuinely mixed — one community answer to "do you use EasyCache/MagCache in Wan 2.2?" was blunt that **"they didn't give me much speedup."**

参数语义（逐字）：

> - **`easycache_thresh`** (default 0.015) - the whole ballgame. Higher = more steps get skipped = faster, but more quality loss (softer motion, mushier detail).
> - **`start_step`** (default 10) - **don't start caching until this step. The early steps set up composition and motion**, so it's deliberately conservative about touching them.
> - **`end_step`** (default -1) - where to stop caching; `-1` means run to the end.
> - **`cache_device`** (`offload_device` / `main_device`) - **where the cached tensors live. Default `offload_device` keeps them off your GPU (saves VRAM); switch to `main_device` only if you've got VRAM headroom and want to shave the transfer overhead.**

Inputs 表（逐字）：`easycache_thresh FLOAT 0.015 0–1`；`start_step INT 10 0–9999`；`end_step INT -1 -1–9999`；`cache_device COMBO offload_device`。

**冲突段（逐字，(4) 类核心反面证据）**：

> ## Common issues & troubleshooting
> **Don't stack it on a speed LoRA. This is the big one. Caches and step-distillation accelerators (LightX2V, CausVid) don't play together — the community guidance is explicitly that you shouldn't use these caches with accelerators, and speed LoRAs already cut you to a handful of steps, so there's almost nothing left for a cache to skip. Pick one acceleration path. Caching earns its keep on full-quality, many-step runs, which is exactly where speed LoRAs aren't in the graph.**
> **Barely faster.** As above, some users see little speedup. It depends heavily on step count, resolution, and threshold.
> **Face consistency wobbles.** Caching skips real computation, and the known tradeoff with TeaCache-style caching is that faces drift a little; disabling the cache improves face consistency at the cost of speed.

## 事实 / 推测 / 建议

- **事实（B，节点目录页直读）**：EasyCache 节点默认 `easycache_thresh=0.015`、`start_step=10`、`end_step=-1`、`cache_device=offload_device`；`start_step=10` **意味着 4/6/8 步的 Turbo 采样根本不会进入缓存窗口**（机制上解释了为什么 Turbo+EasyCache 无效）；维护方明确「不要叠在 speed LoRA 上，缓存与 LightX2V/CausVid 类步数蒸馏不兼容，二选一」；缓存张量默认放 `offload_device`（省 VRAM），改 `main_device` 才占显存（(6) 证据）；另有「收益微弱 / 人脸一致性漂移」的社区口径。
- **推测（C）**：该页是**节点目录/文档页**，非受控 benchmark，无 8GB 实测耗时数字；"不兼容"是社区指导而非量化实验。
- **建议（非结论）**：本机若用 4/8 步 Turbo，默认关闭 EasyCache/TeaCache/MagCache；若退回 20+ 步原生基线再单独测缓存，并把 `cache_device` 留在 `offload_device`。

## 等级

**B（节点文档直读：默认参数与「不要叠加」指导可公开复核）**；「不兼容」的量化验证 = **C（社区指导）**，另有 R4-10 / R4-11 的一手实测佐证。
