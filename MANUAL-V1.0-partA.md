# MiniMax H3 本地部署技术手册 V1.0（上半部 · 第 0–5 章）

- 配套基线方案：`./PLAN-V1.0.md`（版本 v0.32.0 / DiT int8_convrot / TE nvfp4_awq / video VAE fp16 / audio VAE fp32 / LoRA ema_V4 / 480×832 / 124 帧 / 8 步 / shift 12·6 / SDPA）
- 真机：Win11 家庭版 26200、**Windows PowerShell 5.1**、RTX 5070 Laptop 8GB（Blackwell sm_120，显存 8151 MiB）、驱动 616.92、32GB RAM
- ComfyUI 路径假定 `C:\Users\testuser\WorkBuddy\comfyui\ComfyUI`，版本与 venv 路径**未知**（须按第 1 章只读确认）
- 证据等级约定：**B**=外部可复现（公开源码/官方文档+完整方法+第三方复现）；**C**=研究候选；**D**=未经证实。本手册无任何“本机已验证”结论（真机未跑，等级上限为 B）。
- 语法约定：所有 ```powershell 代码块仅用 Windows PowerShell 5.1 语法；路径均加引号；外部 exe 用 `& "路径"` 调用；不使用 `&&`/`||`/`??`/`?.`/三元 `? :`/`ForEach-Object -Parallel`/`Join-String`/`-AsHashtable`，不使用别名（ls/cat/curl/wget）。

---

## 第 0 章 读前须知

### 0.1 安全红线（违反任意一条即停止并上报 Junlong）

1. **不改驱动**：禁止更新/回滚/安装显卡驱动。驱动版本**只记录**（不设版本门槛）；**任何需要改驱动的情况都停**，把实测版本与现象报给 Junlong 等指示。【依据：任务规格第七节；DOSSIER 第 7 章 R2】
2. **不删文件**：禁止 `Remove-Item` / `del` / `rmdir` / `rd /s`。回滚只用**重命名（Rename-Item）或复制（Copy-Item）**。任何删除（含 `.venv-failed-*`、不完整下载、旧备份）一律需 Junlong 批准。【依据：DOSSIER §7、§8 第 14 项】
3. **不跑来源不明的 exe/脚本**：只用本手册给出的命令与官方 HF / PyTorch 直链。
4. **不关安全软件、不改注册表、不持久改执行策略**：若需运行 `.ps1`，只用 `-Scope Process`（见第 1 章开头）。禁止 `Set-ExecutionPolicy` 改到 `Unrestricted` / `Bypass` 的持久范围。
5. **系统设置改动需批准**：以下两项属于修改系统/驱动级配置，**必须标“⚠ 需 Junlong 本人批准”**，并写清 GUI 操作路径与恢复方法，本手册不自动执行：
   - **NVIDIA Sysmem Fallback**（CUDA 系统内存回退策略）→ 见第 1 章 1.11；
   - **页面文件（pagefile）** 大小 → 见第 1 章 1.9。

### 0.2 预计耗时（均为外部估算，真机待实测，标 D 的部分为外推）

| 阶段 | 预计耗时 | 依据与等级 |
|---|---|---|
| 第 1 章只读预检 | 10–20 分钟（人工操作） | 经验估计【D】 |
| 第 2 章备份 venv | 5–15 分钟（取决于 venv 大小，约 8 GiB） | PLAN §7 venv 备份估算【C】 |
| 第 3 章下载（5 个权重，合计 ≈43.1 GB / 40.1 GiB） | 30 分钟–数小时（取决于网速） | 体积来自 HF API【B】；时长外推【D】 |
| 第 4 章升级（仅在版本不满足时） | 10–30 分钟 | 经验估计【D】 |
| 第 5 章单次生成（480×832 / 124 帧 / 8 步） | 参考 3070 的 345.2 s；5070 Laptop 无公开实测 | Qiita 3070【B，19】；5070 外推【D】 |

> 本手册**不含“执行首个真实生成”**（那是第 6 章及之后的内容，属需 Junlong 批准的操作）。上半部只做到“能启动 ComfyUI、权重就位、预检通过”。

### 0.3 磁盘空间预算

基线 5 个权重（来自 PLAN §2 / `P1-hf-api.md`，HF API 实测字节数）：

| 组件 | GiB(2^30) | GB(10^9) |
|---|---:|---:|
| DiT `minimax_h3_fl2va_pruned_int8_convrot` | 19.53 | 20.97 |
| TE `qwen3vl_32b_minimax_h3_nvfp4_awq` | 14.61 | 15.69 |
| Video VAE `minimax_h3_video_vae_fp16` | 4.85 | 5.21 |
| Audio VAE `minimax_h3_audio_vae_fp32` | 0.563 | 0.605 |
| Turbo LoRA `minimax_h3_turbo_4step_ckpt600_ema_V4` | 0.578 | 0.620 |
| **合计（基线，不含 ref2va）** | **≈40.1 GiB** | **≈43.1 GB** |

附加占用（与 PLAN §7 一致）：
- venv 备份估算 ≈ 8 GiB【C，PLAN §7】；
- 工作空间 / 页面文件增长 / 解压临时占用余量 ≈ 20 GiB【C】。

**下载前最少剩余空间（强制校验）= 40.1 + 8 + 20 ≈ 68 GiB，取整要求 NVMe 剩余 ≥ 70 GiB 才允许开始下载**（含 ref2va 再 +19.53 GiB → 要求 ≥ 90 GiB）。校验命令见第 1 章 1.7。【依据：PLAN §7，B/C】

### 0.4 整体流程图（文字版）

```
[第0章 读前须知：确认安全红线、磁盘≥70GiB]
        │
        ▼
[第1章 只读预检 1.1–1.11]
   ├─ GPU/驱动/nvenc  → 只记录，不改驱动
   ├─ 定位 venv/python.exe
   ├─ ComfyUI 版本   → 须 ≥ v0.32.0（推荐 0.31.1–0.32.x）；0.30.0 不可用
   ├─ torch/CUDA     → 须 cu130（torch.version.cuda == 13.0）
   ├─ 关键包 / 自定义节点 / 磁盘 / 内存 / 页面文件 / 电源 / Sysmem
   └─ 填写“预检记录表”回执
        │
        ▼
[第2章 备份]  Copy-Item 整个 venv 或 pip freeze+Rename；备份 extra_model_paths.yaml 与 workflows
        │
        ▼
[第3章 模型下载]  Junlong 批准后，curl.exe 断点续传 5 个权重 → Get-FileHash 校验 SHA256 与字节数
        │
        ▼
[第4章 ComfyUI 与 torch]
   ├─ 版本已满足 → 跳过
   ├─ 版本不足 → git fetch --tags + checkout v0.32.0 + pip install -r requirements.txt（确认点）
   └─ torch 非 cu130 → 固定版本升级（确认点）
        │
        ▼
[第5章 启动]  五旗启动命令 → 确认日志 VRAM/pytorch/attention/dynamic vram → 等待第6章及之后导入模板与生成
```

---

## 第 1 章 只读预检

> 本章所有命令均为**只读**，不修改任何文件或系统。若需在本机运行 `.ps1` 脚本，用 `-Scope Process` 启动，例如：
> `powershell.exe -Scope Process -File ".\precheck.ps1"`（不持久改动执行策略）。

### 1.1 GPU / 驱动 / 算力

```powershell
# 【只读】
& "$env:SystemRoot\System32\nvidia-smi.exe" --query-gpu=name,driver_version,memory.total,compute_cap --format=csv
```

- **预期输出示例**：
  ```
  name, driver_version, memory.total [MiB], compute_cap
  NVIDIA GeForce RTX 5070 Laptop GPU, 616.92, 8151 MiB, 12.0
  ```
- **判定标准**：
  - `name` 含 “RTX 5070” → 合格；否则记录实际型号。
  - `driver_version`：**只记录，不判定合格与否**。任何需要改驱动的情况（如报错提示驱动不匹配、自检要求升级/回滚）→ **停**，把实测值与现象报 Junlong（本手册不改驱动）。616.92 仅为 PC 回报值，以本次实测为准【D，PC 回报】。
  - `memory.total` ≈ 8151 MiB → 合格（Blackwell sm_120，显存 8GB）。
  - `compute_cap` = 12.0 → 与 sm_120 一致，合格。
  - 找不到 `nvidia-smi` → 停，报告 Junlong（不安装/修复驱动）。

### 1.2 定位 venv 与 python.exe

在 ComfyUI 目录及其上一级查找 `venv` / `.venv` / `python_embeded`，并打印 `sys.executable`。

```powershell
# 【只读】
$ComfyRoot = "C:\Users\testuser\WorkBuddy\comfyui\ComfyUI"
$Candidates = @($ComfyRoot, (Split-Path -Parent $ComfyRoot))
$Found = @()
foreach ($Base in $Candidates) {
    $Dirs = Get-ChildItem -LiteralPath $Base -Force -ErrorAction SilentlyContinue | Where-Object { $_.PSIsContainer -and ($_.Name -like "*venv*" -or $_.Name -eq "python_embeded") }
    foreach ($D in $Dirs) {
        $Py = Join-Path $D.FullName "Scripts\python.exe"
        if (Test-Path -LiteralPath $Py) {
            $Found += $Py
        }
    }
}
$Found
if ($Found.Count -ge 1) {
    & $Found[0] -c "import sys; print(sys.executable)"
}
```

- **预期输出示例**：
  ```
  C:\Users\testuser\WorkBuddy\comfyui\ComfyUI\.venv\Scripts\python.exe
  C:\Users\testuser\WorkBuddy\comfyui\ComfyUI\.venv\Scripts\python.exe
  ```
- **判定标准**：
  - 找到至少一个 `python.exe` 且 `sys.executable` 指向该 venv → 合格；**后续所有命令里的 `<venv>` 一律替换为本步实测路径**（如 `C:\...\ComfyUI\.venv`）。
  - `$ComfyRoot` 不存在 → 停，报告 Junlong（不自行全盘搜索、不移动目录）。
  - 无任何 venv / `sys.executable` 指向系统 Python 或 conda → 停，记录实际路径并报告；**是否新建 venv 需 Junlong 批准**。

### 1.3 ComfyUI 版本

优先读版本文件；`git` 可能未安装，写退路。

```powershell
# 【只读】
$ComfyRoot = "C:\Users\testuser\WorkBuddy\comfyui\ComfyUI"
# 版本文件（PLAN 指定：comfyui_version.py 或 pyproject.toml 的 version）
$VP = Join-Path $ComfyRoot "comfyui_version.py"
$PP = Join-Path $ComfyRoot "pyproject.toml"
if (Test-Path -LiteralPath $VP) { Get-Content -LiteralPath $VP }
if (Test-Path -LiteralPath $PP) { Get-Content -LiteralPath $PP | Select-String -Pattern "version" }
# git 退路（git 可能没装）
if (Get-Command git -ErrorAction SilentlyContinue) {
    git -C $ComfyRoot describe --tags 2>$null
    git -C $ComfyRoot rev-parse --short HEAD 2>$null
    git -C $ComfyRoot status --short 2>$null
} else {
    Write-Host "git 未安装，已改用版本文件读取。"
}
```

- **预期输出示例**（git 可用时）：
  ```
  v0.32.0
  a1b2c3d
  （空，表示工作区干净）
  ```
- **判定标准**（依据 PLAN §2 #1、DOSSIER §3.1、P1-releases）：
  - 版本 **≥ v0.32.0** 且落在 **0.31.1–0.32.x** → 合格，第 4 章可跳过升级。
  - 版本 **= v0.30.0** → **停**（8GB 上 VAE 设备不一致崩，#15251【B，29】），需升级到 v0.32.0（第 4 章，需确认点）。
  - 版本 < v0.30.0 或缺 `comfy\ldm\minimax` → 停，报告 Junlong。
  - 版本 ≥ v0.33.0 → 记录风险（#15663 / #16175 / 0.37.2 aimdo【B，25/27】），**是否降级需 Junlong 批准**，本手册不自动降级。
  - 注意：`v0.31.1` 经实测**无 GitHub release 页**（releases API 404），但 tag 存在；V1.0 固定用 **v0.32.0** 作为推荐 tag（P1-releases §3【B】）。

### 1.4 torch / CUDA 版本

```powershell
# 【只读】
& "<venv>\Scripts\python.exe" -c "import torch; print(torch.__version__); print(torch.version.cuda); print(torch.cuda.is_available()); print(torch.cuda.get_device_capability(0)); print(torch.cuda.get_arch_list())"
```
（将 `<venv>` 替换为 1.2 实测路径）

- **预期输出示例**：
  ```
  2.11.0+cu130
  13.0
  True
  (12, 0)
  ['sm_90', 'sm_100', 'sm_120']
  ```
- **判定标准**（依据 Qiita 3070【B，19】、PLAN §2 #2）：
  - `torch.version.cuda` == **13.0**（cu130）→ 合格。
  - `get_arch_list()` 含 **sm_120** → 合格（Blackwell 快速 kernel 可用）。
  - 若 `torch.version.cuda` **不是 13.0**（如 12.8）→ **停**：cu128 下 int8 路径退回 eager，实测仅 1/3 速度【B，19】；把“升级 torch 到 cu130”列为**需批准操作**（命令见第 4 章 4.3），本手册不执行。
  - `import torch` 失败 / `is_available()` 为 False → 停，报告 Junlong。

### 1.5 关键包版本

```powershell
# 【只读】
& "<venv>\Scripts\python.exe" -m pip list | Select-String -Pattern "comfy-aimdo|comfy-kitchen|sageattention|triton"
```

- **预期输出示例**（部分存在即可，非全部必需）：
  ```
  comfy-aimdo            0.x.x
  comfy-kitchen          0.2.30
  sageattention          2.x
  triton                 3.x
  ```
- **判定标准**：
  - `comfy-aimdo` 存在 → 有利于 DynamicVRAM（#15663 根因与其初始化失败相关【B，25】）。
  - `sageattention` 存在但 V1.0 **默认不用**（SDPA 规避 sm_120 噪声【B，28】）；仅作记录。
  - 以上包**缺失不致命**，仅记录；真正硬门槛是 1.3 的 ComfyUI 版本与 1.4 的 cu130。

### 1.6 已安装自定义节点列表

```powershell
# 【只读】
$ComfyRoot = "C:\Users\testuser\WorkBuddy\comfyui\ComfyUI"
Get-ChildItem -LiteralPath (Join-Path $ComfyRoot "custom_nodes") -Force -ErrorAction SilentlyContinue | Select-Object Name
```

- **判定标准**：V1.0 基线只用 ComfyUI 内置节点（`EmptyMiniMaxH3LatentAV`、`MiniMaxH3ImageToVideo`、`MiniMaxH3SigmaShift`、`LoraLoaderModelOnly` 等），**不需要第三方自定义节点**【B，08/30】。列出仅用于记录与回滚参考。

### 1.7 磁盘剩余

```powershell
# 【只读】
Get-PSDrive -Name C | Select-Object Name, Used, Free
# 备选（更精确，单位字节）
Get-CimInstance -ClassName Win32_LogicalDisk -Filter "DeviceID='C:'" | Select-Object DeviceID, Size, FreeSpace
```

- **判定标准**（依据 PLAN §7）：
  - `Free` ≥ **70 GiB**（约 75.2 GB）→ 允许开始下载（第 3 章）。
  - 含 ref2va 时要求 ≥ **90 GiB**。
  - 不足 → **停，不下载**，清理需 Junlong 批准（本手册不删文件腾空间）。

### 1.8 内存（总量 / 可用 / commit 上限）

```powershell
# 【只读】
Get-CimInstance -ClassName Win32_OperatingSystem | Select-Object TotalVisibleMemorySize, FreePhysicalMemory, TotalVirtualMemorySize, FreeVirtualMemory
Get-Counter -Counter '\Memory\Commit Limit'
```

- **判定标准**（依据 RESEARCH-LOG-002 R2-11【B】、R2-10【C+】）：
  - `TotalVisibleMemorySize` ≈ 32 GB（32768 MB 量级）→ 合格记录。
  - 32GB 在 H3 下偏紧：单任务 worker RAM 峰值实测可达 27.2 GB（16GB 卡外推【B，R2-11】），commit 可冲到 120–145%。
  - 本步仅记录，**不修改**任何内存设置。

### 1.9 页面文件（pagefile）

```powershell
# 【只读】
Get-CimInstance -ClassName Win32_PageFileSetting | Select-Object Name, InitialSize, MaximumSize
Get-CimInstance -ClassName Win32_PageFileUsage | Select-Object Name, CurrentUsage, AllocatedBaseSize
```

- **判定标准**（依据 R2-10【C+】、PLAN §2 #16）：
  - 记录是否“系统管理”（`InitialSize`/`MaximumSize` 为 0 通常表示系统管理）或固定大小。
  - **若页面文件过小（如默认仅 2GB），生成中途可能无报错闪退**【C+，R2-10】；V1.0 建议固定 初始 32768 MB / 最大 65536 MB 置于 NVMe，但**此项属修改系统设置，⚠ 需 Junlong 本人批准**，本手册不执行。恢复方法：同一 GUI 路径改回“系统管理”。

### 1.10 电源模式与 OMEN 性能模式（只记录，不改）

```powershell
# 【只读】
powercfg /getactivescheme
```

- **判定标准**：记录当前电源方案 GUID。OMEN 性能模式需在 **OMEN Gaming Hub /  legion 控制台** 中切换为“性能/狂暴”档；同时**必须用原厂 140W 直插供电**，禁用低功率 USB-C 坞（85W 坞会把功耗墙压到 25W，速度差 2.76×【B，22】）。本步仅记录，不改设置。

### 1.11 NVIDIA Sysmem Fallback 当前值（仅 GUI 查看）

- **本值只能在 NVIDIA 控制面板中查看，无命令行读取方式**。
- **GUI 路径**：右键桌面 → “NVIDIA 控制面板” → “3D 设置” → “管理 3D 设置” → 程序设置（或全局） → 找到 “CUDA - 系统内存回退策略”（英文 “CUDA - System Memory Fallback Policy”）→ 记录当前是“优先无系统内存回退”还是“允许”。
- **重要性**（依据 zenn【B，22】、RESEARCH-LOG-002 R2-13【B】）：VRAM 溢出退避系统内存 + VBS DMA 保护可触发 IOMMU 蓝屏（同规格 8GB 机型 8 次先例）。V1.0 建议设为“**优先无系统内存回退**”，但**⚠ 需 Junlong 本人批准**；恢复方法：同一 GUI 路径改回默认“允许”。本手册不自动修改。

### 1.12 预检记录表（回执，测试员填写后留存）

| 项 | 实测值 | 合格?（Y/N） | 备注 / 去哪章处置 |
|---|---|---|---|
| 1.1 GPU 名 | | | |
| 1.1 驱动版本 | | | 只记录（不改驱动） |
| 1.1 显存总量(MiB) | | | 应≈8151 |
| 1.1 compute_cap | | | 应=12.0 |
| 1.2 venv 路径 | | | 替换后续 `<venv>` |
| 1.2 python.exe | | | |
| 1.3 ComfyUI 版本 | | | <v0.32.0→第4章 |
| 1.4 torch 版本 | | | |
| 1.4 torch.version.cuda | | | ≠13.0→第4章升级 |
| 1.4 含 sm_120? | | | |
| 1.5 comfy-aimdo | | | 记录 |
| 1.5 comfy-kitchen | | | 记录 |
| 1.5 sageattention | | | 记录 |
| 1.5 triton | | | 记录 |
| 1.6 自定义节点数 | | | 记录 |
| 1.7 C 盘剩余(GiB) | | | <70→停 |
| 1.8 物理内存(MB) | | | 应≈32GB |
| 1.8 Commit Limit | | | 记录 |
| 1.9 页面文件 | | | 系统管理/固定 |
| 1.10 电源方案 | | | 记录 |
| 1.11 Sysmem Fallback | | | 记录（GUI） |

---

## 第 2 章 备份

> 原则：**只复制/重命名，不删除**（DOSSIER §7、§8 第 14 项）。所有写操作前必须有“确认点”。

### 2.1 方案 A：整目录复制 venv（推荐，回滚最简单）

```powershell
# 【会修改：复制整个 venv 为备份目录，原 venv 不动；撤销=删除备份目录（⚠ 需 Junlong 批准，本手册不删）】
# 确认点：测试员确认磁盘剩余 ≥ (venv 大小 + 70GiB 下载余量) 后再执行
$ts = Get-Date -Format "yyyyMMdd-HHmm"
$ComfyRoot = "C:\Users\testuser\WorkBuddy\comfyui\ComfyUI"
$Venv = "<venv>"   # 替换为第 1.2 实测路径
$Backup = Join-Path $ComfyRoot ("venv-backup-" + $ts)
Copy-Item -LiteralPath $Venv -Destination $Backup -Recurse
Write-Host ("venv 备份已生成: " + $Backup)
```

- **占用空间**：约 venv 自身大小（含 torch 2.11 cu130 等，PLAN §7 估算 ≈ **8 GiB【C】**），与 venv 同盘。
- **回滚**：失败环境 `Rename-Item $Venv (.venv-failed-<ts>)`，再把备份 `Copy-Item` 回原名；原目录保留不删。

### 2.2 方案 B：pip freeze 快照（省空间）

```powershell
# 【会修改：生成 freeze 文本快照（只新增一个小文本文件，不动 venv）】
# 确认点：测试员确认后再执行
$ComfyRoot = "C:\Users\testuser\WorkBuddy\comfyui\ComfyUI"
$Venv = "<venv>"
# 生成依赖快照（几乎不占空间，<1 MB）
& (Join-Path $Venv "Scripts\python.exe") -m pip freeze | Out-File -LiteralPath (Join-Path $ComfyRoot "freeze-before.txt") -Encoding utf8
```

- **占用空间**：freeze 文本可忽略；本方案本身**不动 venv**。若之后确需重命名旧 venv 并新建，属改动环境的操作，须 Junlong 批准，重命名命令只放在第 11 章回滚块（不在本章给出，避免误执行）。
- **回滚**：用 `freeze-before.txt` 重装依赖，或按第 11.1 把失败 venv 改名回原目录。

### 2.3 备份配置文件与默认工作流（必做，不删原文件）

```powershell
# 【会修改：复制配置文件为 .bak；撤销=删除 .bak（⚠ 需 Junlong 批准）】
# 确认点：测试员确认后再执行
$ComfyRoot = "C:\Users\testuser\WorkBuddy\comfyui\ComfyUI"
$Extra = Join-Path $ComfyRoot "extra_model_paths.yaml"
if (Test-Path -LiteralPath $Extra) {
    Copy-Item -LiteralPath $Extra -Destination ($Extra + ".bak") -Force
}
$Wf = Join-Path $ComfyRoot "user\default\workflows"
if (Test-Path -LiteralPath $Wf) {
    Copy-Item -LiteralPath $Wf -Destination ($Wf + ".bak") -Recurse -Force
}
Write-Host "配置与默认工作流备份完成（.bak）"
```

---

## 第 3 章 模型下载清单

> **⚠ 下载 ~43.1 GB 权重，须 Junlong 明确批准后方可执行。** 所有 SHA256 取自 HF API tree 的 `lfs.oid`（已用 `P1-hf-api.md`，2026-11-04 复核，等级 B），字节数来自同表。目标根目录为 `$ComfyRoot\models`。

### 3.1 基线必下文件（5 个）

| # | 文件名 | HF repo | 完整 URL | 字节数 | GB / GiB | SHA256(HEX) | 目标子目录 |
|---|---|---|---|---:|---:|---|---|
| 1 | `minimax_h3_fl2va_pruned_int8_convrot.safetensors` | Comfy-Org/MiniMax-H3 | `https://huggingface.co/Comfy-Org/MiniMax-H3/resolve/main/diffusion_models/minimax_h3_fl2va_pruned_int8_convrot.safetensors` | 20,970,379,616 | 20.97 / 19.53 | `e889202c41dafb67b10d67b97f0d8541508036a6090af23425a5c2615d03c47a` | `models\diffusion_models` |
| 2 | `qwen3vl_32b_minimax_h3_nvfp4_awq.safetensors` | Comfy-Org/MiniMax-H3 | `https://huggingface.co/Comfy-Org/MiniMax-H3/resolve/main/text_encoders/qwen3vl_32b_minimax_h3_nvfp4_awq.safetensors` | 15,687,142,551 | 15.69 / 14.61 | `35a88d51044231fe332301d7a62aa81e3f2cba62febeb446e2c1e3e0ef76f2c6` | `models\text_encoders` |
| 3 | `minimax_h3_video_vae_fp16.safetensors` | Comfy-Org/MiniMax-H3 | `https://huggingface.co/Comfy-Org/MiniMax-H3/resolve/main/vae/minimax_h3_video_vae_fp16.safetensors` | 5,207,808,496 | 5.21 / 4.85 | `7c1f131492e7eddacaac9069a61b81bdd39de5cc96561e677c5eab1cdce5e522` | `models\vae` |
| 4 | `minimax_h3_audio_vae_fp32.safetensors` | Comfy-Org/MiniMax-H3 | `https://huggingface.co/Comfy-Org/MiniMax-H3/resolve/main/vae/minimax_h3_audio_vae_fp32.safetensors` | 605,254,808 | 0.605 / 0.563 | `8e505d95dd1561d47abd43d4238fd40d9bb1ae9e147ed0a4cba778d76ae4db48` | `models\vae` |
| 5 | `minimax_h3_turbo_4step_ckpt600_ema_V4.safetensors` | Abiray/MiniMax-H3-Turbo-Lora-Pruned-ComfyUI | `https://huggingface.co/Abiray/MiniMax-H3-Turbo-Lora-Pruned-ComfyUI/resolve/main/minimax_h3_turbo_4step_ckpt600_ema_V4.safetensors` | 620,285,656 | 0.620 / 0.578 | `7d68591e1aff9ace24b5138da2af02272cd9de125c5c261f5493ae8aa99c715c` | `models\loras` |

### 3.2 备选按需文件（非基线，见 PLAN §5 F0 / F3）

| # | 文件名 | HF repo | 完整 URL | 字节数 | GB / GiB | SHA256 | 目标子目录 | 用途 |
|---|---|---|---|---:|---:|---|---|---|
| A | `minimax_h3_video_vae_int8_convrot.safetensors` | Comfy-Org/MiniMax-H3 | `https://huggingface.co/Comfy-Org/MiniMax-H3/resolve/main/vae/minimax_h3_video_vae_int8_convrot.safetensors` | 2,811,065,184 | 2.81 / 2.62 | `52a2c8c73583c86e4f41cdcce3a6ad0ea562987bc0bf3d60a0cef5f5c8e60c0e` | `models\vae` | F3：显存吃紧时替换 fp16 VAE【B 来源/C 质量】 |
| B | `minimax_h3_fl2v_turbo_8step_v1.0_comfyui_bf16.safetensors` | Comfy-Org/MiniMax-H3 | `https://huggingface.co/Comfy-Org/MiniMax-H3/resolve/main/loras/minimax_h3_fl2v_turbo_8step_v1.0_comfyui_bf16.safetensors` | 1,956,193,000 | 1.956 / 1.822 | `2339acdf19bfe123f46b971ea35d367a84adb85de43627e1eceafa5a5b2b111e` | `models\loras` | F0：官方 8 步 LoRA 对照（非基线）【B 来源/C 8GB】 |

### 3.3 下载命令（断点续传，Win11 自带 curl.exe）

```powershell
# 【会修改：把权重写入磁盘；确认点：测试员确认 C 盘剩余 ≥70 GiB 且已获 Junlong 批准后再执行】
$ComfyRoot = "C:\Users\testuser\WorkBuddy\comfyui\ComfyUI"
$Models = Join-Path $ComfyRoot "models"
# 5 个基线文件清单：Url / 子目录 / 文件名 / SHA256 / 字节数（逐项取自 3.1 表）
$List = @(
    [PSCustomObject]@{ Url = "https://huggingface.co/Comfy-Org/MiniMax-H3/resolve/main/diffusion_models/minimax_h3_fl2va_pruned_int8_convrot.safetensors"; Sub = "diffusion_models"; Name = "minimax_h3_fl2va_pruned_int8_convrot.safetensors"; Sha = "e889202c41dafb67b10d67b97f0d8541508036a6090af23425a5c2615d03c47a"; Bytes = 20970379616 }
    [PSCustomObject]@{ Url = "https://huggingface.co/Comfy-Org/MiniMax-H3/resolve/main/text_encoders/qwen3vl_32b_minimax_h3_nvfp4_awq.safetensors"; Sub = "text_encoders"; Name = "qwen3vl_32b_minimax_h3_nvfp4_awq.safetensors"; Sha = "35a88d51044231fe332301d7a62aa81e3f2cba62febeb446e2c1e3e0ef76f2c6"; Bytes = 15687142551 }
    [PSCustomObject]@{ Url = "https://huggingface.co/Comfy-Org/MiniMax-H3/resolve/main/vae/minimax_h3_video_vae_fp16.safetensors"; Sub = "vae"; Name = "minimax_h3_video_vae_fp16.safetensors"; Sha = "7c1f131492e7eddacaac9069a61b81bdd39de5cc96561e677c5eab1cdce5e522"; Bytes = 5207808496 }
    [PSCustomObject]@{ Url = "https://huggingface.co/Comfy-Org/MiniMax-H3/resolve/main/vae/minimax_h3_audio_vae_fp32.safetensors"; Sub = "vae"; Name = "minimax_h3_audio_vae_fp32.safetensors"; Sha = "8e505d95dd1561d47abd43d4238fd40d9bb1ae9e147ed0a4cba778d76ae4db48"; Bytes = 605254808 }
    [PSCustomObject]@{ Url = "https://huggingface.co/Abiray/MiniMax-H3-Turbo-Lora-Pruned-ComfyUI/resolve/main/minimax_h3_turbo_4step_ckpt600_ema_V4.safetensors"; Sub = "loras"; Name = "minimax_h3_turbo_4step_ckpt600_ema_V4.safetensors"; Sha = "7d68591e1aff9ace24b5138da2af02272cd9de125c5c261f5493ae8aa99c715c"; Bytes = 620285656 }
)
$Fail = @()
foreach ($F in $List) {
    $Dir = Join-Path $Models $F.Sub
    New-Item -ItemType Directory -Force -Path $Dir | Out-Null
    $Dest = Join-Path $Dir $F.Name
    Write-Host ("开始下载: " + $F.Name)
    # -L 跟随重定向；-C - 断点续传；-o 输出到文件
    curl.exe -L -C - -o $Dest $F.Url
    # 下载后立即逐个比对 SHA256 与字节数
    if (-not (Test-Path -LiteralPath $Dest)) {
        $Fail += ($F.Name + " 未生成文件（下载失败）")
        continue
    }
    $H = (Get-FileHash -Algorithm SHA256 -LiteralPath $Dest).Hash
    $Len = (Get-Item -LiteralPath $Dest).Length
    if ($H -ne $F.Sha) {
        $Fail += ($F.Name + " SHA256 不符：实测 " + $H)
    } elseif ($Len -ne $F.Bytes) {
        $Fail += ($F.Name + " 字节数不符：实测 " + $Len + "，应为 " + $F.Bytes)
    } else {
        Write-Host ("OK: " + $F.Name)
    }
}
if ($Fail.Count -gt 0) {
    Write-Host "FAIL"
    $Fail | ForEach-Object { Write-Host ("  " + $_) }
    throw "基线权重校验失败：停在此处，不进入第 4 章及之后。保留文件并上报 Junlong。"
}
Write-Host "ALL-OK: 5 个基线文件全部通过 SHA256 与字节数校验"
```

- **备选下载方式（venv 内 huggingface_hub）**：若偏好 `hf download`，需 `huggingface_hub` ≥ 0.26（含 `hf` CLI）。示例（须在 venv 内，且同样需 Junlong 批准）：
  ```powershell
  # 【会修改：确认点同上下载】
  & "<venv>\Scripts\python.exe" -m pip install -U "huggingface_hub>=0.26"
  & "<venv>\Scripts\python.exe" -m huggingface_hub hf download Comfy-Org/MiniMax-H3 "diffusion_models/minimax_h3_fl2va_pruned_int8_convrot.safetensors" --local-dir "$Models"
  ```

### 3.4 校验命令（SHA256 与字节数，必须逐文件比对）

```powershell
# 【只读】3.3 的循环已在每个文件下载后自动比对；本段用于事后抽查单个文件
$One = "C:\Users\testuser\WorkBuddy\comfyui\ComfyUI\models\diffusion_models\minimax_h3_fl2va_pruned_int8_convrot.safetensors"
# 1) SHA256 比对（应等于 3.1 / 3.2 表中值）
Get-FileHash -Algorithm SHA256 -LiteralPath $One
# 2) 字节数比对（应等于表中“字节数”）
(Get-Item -LiteralPath $One).Length
# 3) 远程 X-Linked-Size 比对（可选，确认源端大小）
curl.exe -sIL "https://huggingface.co/Comfy-Org/MiniMax-H3/resolve/main/diffusion_models/minimax_h3_fl2va_pruned_int8_convrot.safetensors" | Select-String -Pattern "X-Linked-Size"
```

- **判定标准**：`Get-FileHash` 的哈希值与表内 SHA256 逐字符一致、`.Length` 与字节数一致 → 该文件合格。3.3 的循环只要有一个文件不符就会打印 **FAIL** 并抛出错误**停在此处**（不进入第 4 章）；此时用 `curl.exe -L -C -` 断点续传重下该文件再重跑 3.3，损坏的不完整文件保留，不擅自删除（等 Junlong 指示）。全部 5 个文件输出 **ALL-OK** 才算通过。

---

## 第 4 章 ComfyUI 与 torch

> 分情况决策树。写操作前必须有“确认点”。所有 git / pip 改动属**改动现有环境，需 Junlong 批准**。

### 4.1 决策树

```
[1.3 版本判定]
   ├─ 已 ≥ v0.32.0 且 ∈ 0.31.1–0.32.x  → 跳过本章，直接第 5 章
   ├─ = v0.30.0 或 < v0.32.0           → 走 4.2 升级 ComfyUI 到 v0.32.0（确认点）
   └─ ≥ v0.33.0                        → 记录风险，暂不降级（降级需批准）
[1.4 torch 判定]
   ├─ torch.version.cuda == 13.0       → 跳过 4.3
   └─ ≠ 13.0                           → 走 4.3 升级 torch 到 cu130（确认点）
```

### 4.2 升级 ComfyUI（仅版本不满足时）

```powershell
# 【会修改：git 切换到 v0.32.0，可能改变 ComfyUI 源码与依赖；撤销=git checkout <升级前 commit>】
# 确认点：测试员确认已备份（第2章）且已获 Junlong 批准后再执行
$ComfyRoot = "C:\Users\testuser\WorkBuddy\comfyui\ComfyUI"
# 0) 切标签前先确认工作区干净：有本地改动就停，不做任何 git 写操作
if (-not (Get-Command git -ErrorAction SilentlyContinue)) {
    Write-Host "STOP: 未安装 git，无法执行第 4 章升级，请上报 Junlong。"
    throw "git 不可用，第 4 章已停止。"
}
$Dirty = @(git -C $ComfyRoot status --short)
if ($Dirty.Count -gt 0) {
    Write-Host "STOP: ComfyUI 工作区有本地改动，不执行 checkout。请把下列输出上报 Junlong："
    $Dirty | ForEach-Object { Write-Host ("  " + $_) }
    throw "工作区不干净，第 4 章已停止（未做任何改动）。"
}
# 先记录当前 commit 以便回退
git -C $ComfyRoot rev-parse HEAD | Out-File -LiteralPath (Join-Path $ComfyRoot "comfyui-commit-before.txt") -Encoding utf8
# 取标签并切到 PLAN 指定 tag
git -C $ComfyRoot fetch --tags
git -C $ComfyRoot checkout v0.32.0
# 同步依赖
& "<venv>\Scripts\python.exe" -m pip install -r (Join-Path $ComfyRoot "requirements.txt")
```

- **前置检查（强制）**：checkout 前先跑 `git -C $ComfyRoot status --short`；输出为空才继续，**有本地改动就停**（脚本会打印 STOP 并抛错终止），把改动列表上报 Junlong 由其决定——**不 stash、不丢弃、不覆盖**。
- **依据**：v0.32.0（2026-08-11）含 #15446（VAE 流式）、#15486（H3 内存修复）、#15479（comfy-kitchen attention）、#15477（tiled 音频解码修复），且有 3070 Ti 8GB/cu130/79.74s 实测【B，P1-releases §1、§2】。
- **判定**：`git -C $ComfyRoot describe --tags` 输出 `v0.32.0` → 升级成功。

### 4.3 升级 torch 到 cu130（仅 1.4 判定不满足时）

```powershell
# 【会修改：升级 torch/torchvision/torchaudio；撤销=用 freeze-before.txt 重装回原版本（需 Junlong 批准）】
# 确认点：测试员确认已备份且已获 Junlong 批准后再执行
# 先 freeze 当前版本
& "<venv>\Scripts\python.exe" -m pip freeze | Out-File -LiteralPath "torch-freeze-before.txt" -Encoding utf8
# 固定版本升级（命令固定，勿拉取 latest）
& "<venv>\Scripts\python.exe" -m pip install torch==2.11.0+cu130 torchvision==0.26.0+cu130 torchaudio==2.11.0+cu130 --index-url https://download.pytorch.org/whl/cu130
# 升级后验证
& "<venv>\Scripts\python.exe" -c "import torch; print(torch.__version__); print(torch.version.cuda)"
```

- **依据**：torch 2.11.0+cu130 实测可用（Qiita 3070【B，19】）；Comfy-Org 建议 cu130 以启用 int8_convrot 快速 kernel（PLAN §2 #2【B】）；PyTorch 官方 cu130 wheel 索引 `https://download.pytorch.org/whl/cu130`（本会话已查 PyTorch 安装页确认该索引形态；版本号取自 Qiita 实测【B，19】）。
- **判定**：`torch.version.cuda` 输出 `13.0` → 合格。
- **注意**：`--index-url` 会覆盖默认 PyPI；若 venv 还需其它包，升级后按需补装。

---

## 第 5 章 启动

### 5.1 完整基线启动命令（PLAN §2.1 五旗）

```powershell
# 【会修改：启动 ComfyUI 服务进程；确认点：测试员确认第1章预检与第2章备份均已完成、且权重已就位后再执行】
$ComfyRoot = "C:\Users\testuser\WorkBuddy\comfyui\ComfyUI"
$VenvPy = "<venv>\Scripts\python.exe"   # 替换为第1.2实测路径
Set-Location -LiteralPath $ComfyRoot
& $VenvPy main.py --disable-pinned-memory --reserve-vram 1 --disable-async-offload --cache-none --fast-disk
```

### 5.2 各参数逐条解释（含出处与引入版本）

| 参数 | help 原文（cli_args.py v0.32.0，P1b-cliargs【B】） | 为何 V1.0 用 | 引入版本 | 来源 |
|---|---|---|---|---|
| `--disable-pinned-memory` | `Disable pinned memory use.` | 关掉页锁内存省 RAM（32GB 首要矛盾；Tomiigo 模拟上关后 0 次 swap【C，23】） | v0.31.0 起（P1b-cliargs 三版核对【B】） | R2-14 / docs.comfy.org |
| `--reserve-vram 1` | `Set the amount of vram in GB you want to reserve ...` | 预留 1 GiB 给 OS；#15663 作者三旗之一，shared 溢出是 WDDM 慢路径与蓝屏根因【B，25】 | v0.31.0 起 | #15663（sources/25） |
| `--disable-async-offload` | `Disable async weight offloading.` | #15663 作者用此三旗解决 H3 短片 OOM（根因 comfy-aimdo 未启用 DynamicVRAM）【B，25】 | v0.31.0 起 | #15663（sources/25） |
| `--cache-none` | `Reduced RAM/VRAM usage at the expense of executing every node for each run.` | 省 RAM（R2-11）；注意不省单任务内部峰值 | v0.31.0 起 | R2-14 / docs.comfy.org |
| `--fast-disk` | `Prefer disk-backed dynamic loading and offload over unpinned RAM. Can be faster for users with fast NVME disks.` | 省 RAM 正解（R2-02/R2-03/R2-14）；与 `--disable-pinned-memory` 协同；取代的是 unpinned RAM，非“权重绝不进 RAM” | v0.31.0 起 | R2-14 / docs.comfy.org |

> 五参数自 **v0.31.0** 起均存在于 `comfy/cli_args.py`（已用 v0.31.0/0.31.1/0.32.0 三版 raw 文件核对 help 原文，P1b-cliargs【B】）；“引入版本”不再标 D。`--disable-fast-disk` 仅 master 分支有，v0.32.0 无。真机以 `main.py -h` 实测为准。

### 5.3 启动日志要确认的四类行

启动后观察控制台 / 日志，确认以下四项（真机以实际日志文本为准，下列为常见关键字）：

1. **VRAM 总量**：如 `VRAM available: 8151 MiB` 或 `Total VRAM ...` —— 应 ≈ 8151 MiB。
2. **PyTorch 版本**：如 `PyTorch version: 2.11.0+cu130` —— 应含 `cu130`。
3. **attention 类型**：V1.0 用 SDPA；日志中确认未误启用 SageAttention（SDPA 规避 sm_120 噪声【B，28】）。
4. **DynamicVRAM / aimdo 状态**：如 `DynamicVRAM` 启用、`aimdo` 初始化成功（无 `funchook_prepare ... failed` 报错）。若 aimdo 初始化失败，即 #15663 根因，需走 5.4。

### 5.4 #15663 报错时的替代启动参数

#15663 现象：短片显存被低估 → OOM 且重启仍 OOM（根因 comfy-aimdo 初始化失败致 DynamicVRAM 未启用【B，25】）。本手册 5.1 已包含该 issue 推荐的三旗（`--disable-pinned-memory --reserve-vram 1 --disable-async-offload`）。

若仍出现 #15663 式 OOM，替代做法（微调，仍属需观察的改动）：

```powershell
# 【会修改：提高 reserve-vram 给 OS 更多余量；确认点：测试员确认后再执行】
# 注意：#15663 原文只给出 --reserve-vram 1；下面的 2 是【D，经验值】，非 issue 原始参数
$ComfyRoot = "C:\Users\testuser\WorkBuddy\comfyui\ComfyUI"
$VenvPy = "<venv>\Scripts\python.exe"
Set-Location -LiteralPath $ComfyRoot
& $VenvPy main.py --disable-pinned-memory --reserve-vram 2 --disable-async-offload --cache-none --fast-disk
```

- #15663 原文给出的三个参数是 `--disable-pinned-memory --reserve-vram 1 --disable-async-offload`（【B，25】），**不含 `--reserve-vram 2`**。
- 把 `--reserve-vram 1` 调高为 `--reserve-vram 2`（或更高）属于**微调，无 issue 来源**，标 **【D，经验值】**；若实测更差（生成更早 OOM 或明显变慢）立即退回 5.1 的五旗。
- 仍 OOM → 按 PLAN §7 / §5 上报 Junlong：**不要擅自改系统设置（Sysmem Fallback / pagefile）**，也不要删文件；尝试在 UI 中改任一节点参数或插入无关节点（#15663 作者解法）后重跑。
- 注意：R2-13 实测 `--vram-headroom` 对“权重页被放进共享内存”无效，故不作为首选替代。

---

## 命令自查表（本手册上半部）

| 代码块 | 章节 | 是否只读 | PS5.1 语法已自查 | 备注 |
|---|---|---|---|---|
| 1.1 nvidia-smi | 1.1 | 是（只读） | 是 | 用 `&` 调用 exe，路径引号 |
| 1.2 定位 venv | 1.2 | 是 | 是 | `foreach`/`Where-Object`/`Join-Path`，无别名 |
| 1.3 ComfyUI 版本 | 1.3 | 是 | 是 | `Get-Command` 退路，`2>$null` |
| 1.4 torch/CUDA | 1.4 | 是 | 是 | python `-c` 内联 |
| 1.5 关键包 | 1.5 | 是 | 是 | `pip list | Select-String` |
| 1.6 自定义节点 | 1.6 | 是 | 是 | `Get-ChildItem` |
| 1.7 磁盘剩余 | 1.7 | 是 | 是 | `Get-PSDrive`/`Get-CimInstance` |
| 1.8 内存 | 1.8 | 是 | 是 | `Get-CimInstance`/`Get-Counter` |
| 1.9 页面文件 | 1.9 | 是 | 是 | `Get-CimInstance` 两类 |
| 1.10 电源 | 1.10 | 是 | 是 | `powercfg` |
| 1.11 Sysmem | 1.11 | 是（仅 GUI） | — | 无命令 |
| 2.1 复制 venv | 2.1 | 否（会修改） | 是 | 有“确认点”；`Copy-Item -Recurse` |
| 2.2 freeze | 2.2 | 否（会修改） | 是 | 有“确认点”；仅生成 freeze 文本，无 Rename-Item |
| 2.3 配置备份 | 2.3 | 否（会修改） | 是 | 有“确认点”；`Copy-Item -Recurse` |
| 3.3 下载 | 3.3 | 否（会修改） | 是 | 有“确认点”；`$List` 循环 5 个基线文件，`curl.exe -L -C - -o` |
| 3.3 hf download 备选 | 3.3 | 否（会修改） | 是 | 有“确认点”；venv 内 huggingface_hub |
| 3.4 校验 | 3.4 | 是 | 是 | 3.3 循环内逐个 `Get-FileHash` + 字节数比对，FAIL 即停 |
| 4.2 升级 ComfyUI | 4.2 | 否（会修改） | 是 | 有“确认点”；先 `git status --short`，有改动即停 |
| 4.3 升级 torch | 4.3 | 否（会修改） | 是 | 有“确认点”；固定版本 + `--index-url` |
| 5.1 启动 | 5.1 | 否（会修改） | 是 | 有“确认点”；五旗 |
| 5.4 替代启动 | 5.4 | 否（会修改） | 是 | 有“确认点”；`--reserve-vram 2` 标【D，经验值】 |

- 全部代码块路径加引号；无 `&&`/`||`/`??`/`?.`/三元/`ForEach-Object -Parallel`/`Join-String`/`-AsHashtable`；无 `ls`/`cat`/`curl`(别名)/`wget`（下载用 `curl.exe`，符合 Win11 自带）。
- 所有“会修改”块均带“确认点：测试员确认后再执行”；删除类操作一律不在本手册内。

---

*（MANUAL-V1.0-partA.md 完，含第 0–5 章。第 6 章起为下半部，将由后续任务完成。全文无“本机验证成功”表述；数字均标出来源与等级。）*
