# MiniMax H3 本地部署技术手册 · 总入口（V1.0）

- **版本**：V1.0（2026-11-04），与 `./PLAN-V1.0.md` **r2** 配套
- **适用机器**：Win11 家庭版 26200 / **Windows PowerShell 5.1** / RTX 5070 Laptop 8GB（Blackwell sm_120，8151 MiB）/ 驱动 616.92 / 32GB RAM
- **基线配置**：ComfyUI v0.32.0 · DiT int8_convrot · TE nvfp4_awq · Video VAE fp16 · Audio VAE fp32 · Turbo LoRA ema_V4 · 480×832 · 124 帧 · 8 步 · shift 12/6 · SDPA
- **证据等级**：B=外部可复现 · C=研究候选 · D=未经证实。真机未跑，全文无“本机已验证”结论，等级上限 B。

## 阅读顺序

1. `./MANUAL-V1.0-partA.md` —— **第 0–5 章**（读前须知、只读预检、备份、模型下载、升级、启动）
2. `./MANUAL-V1.0-partB.md` —— **第 6–12 章**（工作流改造、系统设置、监控日志、测试矩阵、错误速查、回滚、回执）

> 先读完 partA 的第 0 章再动手；partA 与 partB 都需在开跑前通读，命令按章顺序执行。

## 总目录

| 章 | 标题 | 文件 |
|---|---|---|
| 0 | 读前须知（安全红线、耗时、磁盘预算、流程图） | partA |
| 1 | 只读预检（GPU/驱动、venv、版本、torch、包、节点、磁盘、内存、页面文件、电源、Sysmem） | partA |
| 2 | 备份（venv 复制、freeze 快照、配置与工作流） | partA |
| 3 | 模型下载清单（5 个基线文件 + 校验） | partA |
| 4 | ComfyUI 与 torch（版本决策树、升级） | partA |
| 5 | 启动（五旗命令、参数解释、日志确认、#15663 替代参数） | partA |
| 6 | 工作流：打开模板并改到基线 | partB |
| 7 | 系统设置（页面文件、Sysmem Fallback，⚠ 需批准） | partB |
| 8 | 监控与日志（GPU / 系统 / 控制台 / 峰值统计） | partB |
| 9 | 测试矩阵（S0、B1–B3、F0、F1a/F1b） | partB |
| 10 | 已知错误速查 | partB |
| 11 | 回滚（venv、commit、系统设置、模型） | partB |
| 12 | 回执模板 | partB |

## 安全红线（摘要，详见 partA 第 0.1 节）

1. **不改驱动**：驱动版本只记录；任何需要改驱动的情况都**停**并上报。
2. **不删文件**：禁止 `Remove-Item`/`del`/`rmdir`；回滚只用 `Rename-Item` 或 `Copy-Item`。
3. **不跑来源不明的 exe/脚本**：只用手册内命令与官方 HF/PyTorch 直链。
4. **不关安全软件、不改注册表、不持久改执行策略**（`.ps1` 仅 `-Scope Process`）。
5. **系统设置改动需 Junlong 本人批准**：页面文件、NVIDIA Sysmem Fallback 只给 GUI 路径，手册不自动执行。

## 修订记录

**r1（2026-11-04，审核返修）**

| 编号 | 位置 | 修订内容 |
|---|---|---|
| A1 | partA 1.1 / 0.1 / 流程图 / 1.12 | 删除无来源的“驱动 <580 即停”，改为“只记录；任何需要改驱动的情况都停” |
| A2 | partA 3.3 | 改为 `$List` 数组循环 5 个基线文件，逐个 `Get-FileHash` 比对，不符打印 FAIL 并停止 |
| A3 | partA 4.2 | checkout 前先 `git status --short`，有本地改动即停、上报，不执行 checkout |
| A4 | partA 5.4 | 注明 #15663 原文为 `--reserve-vram 1`；`2` 标【D，经验值】 |
| A5 | partA 2.2 | 删除注释里对现役 venv 的 `Rename-Item` 示例 |
| B1 | partB 6.2 / 6.3 | 修正分辨率：选择器 0.4MP/16:9 → 864×480；1344×768 是子图控件默认；length 73 为旧值，duration=2 → 56 帧 |
| B2 | partB 8 开头 / 8.1 | 第 8 章开头即建 `C:\logs`；nvidia-smi 改用自带 `-f` 写文件，不走管道 |
| B3 | partB 8.2 / 8.4 | 改为逐次 `Export-Csv -Append` 落盘，固定 6 列，GPU 多实例取最大值，无限循环 Ctrl+C；8.4 同步列名 |
| B4 | partB 8.3 | 补充说明：PS5.1 下 `2>&1` 把 python stderr 包成红色 NativeCommandError 属正常现象 |
| B5 | partB 11.2 | 改为从 `comfyui-commit-before.txt` 读出 commit 再 checkout |
| B6 | partB 9.1 | S0 明确写成宽×高 352×608（竖版，与基线方向一致） |

- 未完成项：无。
