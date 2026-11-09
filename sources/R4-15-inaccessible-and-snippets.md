# R4-15｜无法访问正文 / 仅搜索片段 / 只看了标题与摘要的页面（不作数字依据）

> 本文件登记本轮「打开失败」或「只拿到片段/标题」的页面，遵守 CODEBUDDY.md「只写真实打开过的内容、打不开写无法访问、不编造」。**这些页面不进入 LOG-004 的编号条目，也不作任何结论依据。**
> 读取时间：2026-11-08 JST（WebSearch 索引显示检索日 2026-11-07）。

## 1. 直连打开但只拿到标题（视为「正文无法访问」）

| URL | 直读结果 | 处置 |
|---|---|---|
| https://modelscope.cn/learn/435669 《8GB 显存运行 MiniMax-H3：量化、动态卸载与 Spectrum 加速的真实边界》 | WebFetch 返回**仅文章标题**，正文、日期、v0.2.5 配置块、「Spectrum 的内存账本」节均未返回 | **正文无法访问**。WebSearch 片段曾显示该文含 `blend_weight=0.50 / degree=1 / warmup_steps=1 / max_history=8 / history_storage=system_ram` 及「Spectrum 会保留隐藏特征历史」的内存段，但**本轮未逐字核实，不引用其中任何数字**；相关参数已由 R4-08（minimax3.org + 上游 README）直读确认 |

## 2. 只看了 WebSearch 标题/摘要片段、未直连打开（不作结论）

| URL | 片段里可见的宣称（仅登记，不采信） | 处置 |
|---|---|---|
| https://www.virse.ai/zh/blog/minimax-h3-review | 「一项 RTX 4060 Ti 8GB 案例使用 EasyCache 后，从约 20 分钟降至 12 分钟」；另有 3050 6GB / 3060 Laptop 6GB / 3070 8GB / 4090 Laptop / 5090 的耗时表 | **未直读**。这是与 R4-09/R4-10/R4-11「Turbo+cache 无收益」方向相反的**正面 8GB 报告**，但为聚合评测站、无方法/无原始出处，**标 D，仅登记为待核实的反例线索**，不写入结论、不进速度栈 |
| https://arxiv.org/html/2609.38853 《Visualizing Distribution Coverage in Generative Diffusion Models》 | 片段称 DMD2 / Self-Forcing DMD 在 best-of-k 下出现多样性收缩与 mode collapse，而 consistency 蒸馏（Causal-CD）保持较好；VBench-20、n=64、k∈[1,64] | **未直读 abs 页**，仅 HTML 索引片段。作为「DMD 类 vs consistency 类」的线索登记，**不作数字依据** |
| https://hellogen.ai/zh-cn/blog/minimax-h3-local-deployment | 显存档位表（8GB 640×360–832×480 / 3070 9–10 分钟；12GB 20 步 <9 分钟；16GB 配 Turbo 可上 768p）、「Turbo LoRA 是最大单项收益」 | **未直读**，聚合站，数字无原始出处 → **D** |
| http://www.lpqz.cn/news/151751 《Turbo 加速实战…21 步砍到 4 步》 | 4 卡 NPU 768P 端到端表：FL2VA Turbo 8 步 BF16 212.33s / INT8 113.51s；Ref2VA Turbo 8 步 pruned INT8 454.77s；Ref2VA 21 步 944.96s / 50 步 2263.03s | **未直读**；且明确为 **4×NPU 多卡、非 8GB**，按 R4 §8 **禁止冒充 8GB**，仅登记 |
| https://minimax3.com/blog/minimax-h3-turbo-steps 《4 vs 6 vs 8 Steps》 | 「两位作者都推荐 6–8 步」「~5× 是采样步数比，不是 wall clock」「4 步肢体/运动先坏」 | **未直读**（仅 WebSearch 片段）。其观点与 R4-02（larryvrh 卡直读）一致，但**不重复引用、不用其数字** |
| https://blog.mushroom.cv/blog/minimax-h3-video-generation-local-acceleration-engineering-guide | 「8 步 vs 20 步差异轻微，4 步可能运动模糊和细节缺失」 | **未直读**，聚合工程文 → **D** |
| https://www.aiboss88.com/news/project-dmad ・ https://cloudsai.cn/ai-tutorials/7054.html | DMAD 宣传数字（4 卡 H200 233.3s vs DMD2 818.1s；「8GB 消费级显卡即可」） | **未直读**，中文宣传站 → **D**，不采信（DMAD 一手数据见 R4-14 的 arXiv abs） |
| https://comfy.icu/node/WanVideoEasyCache 之外的 `deepwiki.com/chengzeyi/Comfy-WaveSpeed/2-first-block-cache-(fbcache)` | FBCache 推荐阈值表：flux-dev fp8 28 步 0.12、ltx-video-2b 30 步 0.1、hunyuan_video_720 **20 步 0.1**、sd3.5 30 步 0.12 | **未直读**（WebSearch 片段）。机制与 R4-04 一致；「推荐阈值总是绑在 20–30 步」这一点与 LOG-004 (2) 结论方向一致，但**仅作线索，不作依据** |
| https://arganzheng.life/timestep-redundancy-caching-and-step-skipping.html | TeaCache / FBCache / DBCache 公式；「diffusers FirstBlockCacheConfig 默认阈值 0.05（保守），社区常用 0.08–0.12（1.6–2×）」；「FBCache 每步至少算一个 block（1/57≈2%）」 | **未直读**，个人技术博客 → **C 线索**；不写进条目 |
| https://hysenlabs.com/projects/xmarre-comfyui-spectrum-minimax-h3 | 「v0.2.26/0.2.27/0.2.28 均在 2026 年 9 月」；`steps` 是 outer sigma 区间，SEEDS-2/3 与 PECE 会把 10 步变成 19/28 次模型调用 | **未直读**；同一事实已由 R4-08 上游 README 直读确认，不重复 |
| https://blog.gitcode.com/ad2ecec9b48fbe3fe8866803f0b4781c.html ・ https://blog.gitcode.com/198f4fe1d777a7031142408f0dc656eb.html | Spectrum A/F 调度、8 个 Turbo 权重与 flow_shift 速查表 | **未直读**；8 个权重/参数已由 R4-01 官方仓库直读覆盖一部分，但 v1.1/v1.2/v1.3 等更新版本**未核实**，不引用 |
| https://octohz.com/p/1795 ・ https://cloudthrill.ca/?p=8039/ ・ https://builderai.tools/tool/teacache ・ https://deepwiki.com/ali-vilab/TeaCache | TeaCache 二手加速比（4.41× Open-Sora-Plan / 2.1× HunyuanVideo / 2.25× FLUX / 1.3–2.7×）、「缓存 residual 而非 full state，内存开销小」 | **未直读**，均为第三方转述 → **D/C**；一手数字见 R4-05 / R4-06 |
| https://blog.csdn.net/gitblog_00650/article/details/155084767 ・ http://www.hqwc.cn/a/1513961.html ・ https://closerai.douyoubuy.cn/?p=422238 ・ https://blog.gitcode.com/034cecf8cbea8019c8eec26c3cbd19f3.html | Turbo 4–8 步参数建议、`--offload-adaln` 省 ~13GB、`cache_device='cpu'` 显存紧张建议、TeaCache `clear_all()` / `cache_report()` | **未直读**，聚合/转载 → **D**；`--offload-adaln ~13GB` 已由 R4-02 官方卡直读确认 |

## 3. 连续打开情况

- 本轮**没有任何 URL 连续 3 次打不开**；WebFetch 对 github.com / arxiv.org / huggingface.co / comfyui-wiki.com / comfy.icu / note.com / minimax3.org / forgebun.com 均一次成功。
- HF 页面与部分长页出现「AI 摘要超时 → 返回原始正文前 N 字符」，属**部分直读**（已在 R4-01/02/03/08/11 逐文件标注），不记为失效。
- Reddit r/StableDiffusion / r/comfyui、知乎、B 站专栏、Qiita/Zenn 新文章：**本轮 WebSearch 未命中可打开的 8GB + 少步/缓存开关 + 耗时**的新页面，如实写「未找到」，不用旧条目凑数（LOG-001/002/003 已覆盖的基线按编号引用，不抄数字）。

## 事实 / 推测 / 建议

- **事实**：上表所列页面的「直读结果」与「可见片段」为本轮真实所见。
- **推测**：所有片段里的数字都可能是不完整转述，**一律不作依据**。
- **建议**：下一轮若要补 8GB 正面缓存反例，优先直读 virse.ai 引用的原始出处（若存在），以及 modelscope 435669 的正文；在此之前两者均标 D。
