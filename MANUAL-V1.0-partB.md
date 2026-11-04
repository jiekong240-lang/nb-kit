# MiniMax H3 本地部署技术手册 V1.0（下半部 · 第 6–12 章）

- 配套基线方案：`./PLAN-V1.0.md`（版本 v0.32.0 / DiT int8_convrot / TE nvfp4_awq / video VAE fp16 / audio VAE fp32 / LoRA ema_V4 / 480×832 / 124 帧 / 8 步 / shift 12·6 / SDPA）
- 上半部：`./MANUAL-V1.0-partA.md`（第 0–5 章：安全红线、只读预检、备份、下载、升级、启动）
- 真机：Win11 家庭版 26200、**Windows PowerShell 5.1**、RTX 5070 Laptop 8GB（Blackwell sm_120，显存 8151 MiB）、驱动 616.92、32GB RAM
- 证据等级约定同上半部：**B**=外部可复现；**C**=研究候选；**D**=未经证实。本手册无任何“本机已验证”结论（真机未跑，等级上限为 B）。
- 语法约定同上半部：所有 ```powershell 代码块仅用 PS 5.1 语法；路径加引号；外部 exe 用 `& "路径"`；禁用 `&&`/`||`/`??`/`?.`/三元 `? :`/`ForEach-Object -Parallel`/`Join-String`/`-AsHashtable`；禁用别名（ls/cat/curl/wget，下载用 `curl.exe`）。

---

## 第 6 章 工作流：打开模板并改到基线

> 依据：PLAN §3.5（r2）、§2 基线表；模板 JSON 见 `sources/M1b-template-nodes.md`（2026-11-04 实时抓取 `raw.githubusercontent.com/Comfy-Org/workflow_templates/main/templates/video_minimax_h3_t2v.json` 摘录）。接线与参数修改**全部以 PLAN §3.5（r2）为准**。

### 6.1 模板在界面里的位置与名称

1. 启动 ComfyUI（第 5 章 5.1 五旗）后，浏览器打开 ComfyUI 前端（默认 `http://127.0.0.1:8188`）。
2. 顶部菜单 **Templates（模板）** → 分类 **Video（视频）** → 找 **MiniMax H3**（t2v 文本生视频）。
3. 点击载入。v0.32.0 界面绑定的是 **workflow_templates v0.11.39**（PLAN §3.5【B，P1b-template】）。

> ⚠ **版本重要差异（已实测，需测试员注意）**：2026-11-04 实时抓取的 `main` 分支 `video_minimax_h3_t2v.json` 是一个**更新的变体**（含 Lightning LoRA 开关、Video VAE=int8_convrot、BasicScheduler 默认 4 步 + 8 步开关、LoRA 来自 `lightx2v/Minimax-h3-Turbo`）。**该 `main` 版本在 v0.32.0 界面里看不到**（PLAN §3.5【B】）。测试员实际载入的是 v0.11.39，其默认值为：**无 LoRA 节点、Video VAE=fp16、BasicScheduler=simple/20 步、seed 556589502035082、用 `MiniMaxH3ImageToVideo`**。若你的界面版本更高、看到了 Lightning LoRA 开关版本，请按 §3.5 做法 B 处理并上报 Junlong（本手册主路径按 0.11.39 写）。

### 6.2 模板默认值逐项（取自 JSON / PLAN §3.5）

| 项 | 模板默认（v0.11.39，PLAN §3.5【B】） | 备注 / 来源 |
|---|---|---|
| 分辨率选择器 | `ResolutionSelector(0.4MP, 16:9)` 经连线 link 219→width、link 220→height，输出 **864×480** | 依据节点 118 MarkdownNote“Size Settings Reference”说明表：0.4MP + 16:9 → **864×480**。注意 **1344×768 是子图 `MiniMaxH3ImageToVideo` 的 width/height 控件默认值，不是选择器输出**，两者别混（PLAN §3.4） |
| duration | `Float (duration)` 节点默认 **2 秒** | length 由 `ComfyMathExpression` 按 17k+5 网格算出：**2s→56 帧**、5s→124 帧 |
| length（帧数） | 控件里存的旧值 **73**（不参与换算，仅历史残留） | duration=2 时按公式算得 **56 帧**；改 duration=5 才变 124 帧 |
| seed | `RandomNoise` 默认 **556589502035082** | 即 PLAN §2 #14 固定种子，三次重复同值 |
| Lightning LoRA 开关 | 0.11.39 **无此开关、无 LoRA 节点** | main 分支变体才有（Boolean Enable Lightning LoRA，默认 false） |
| BasicScheduler steps | **20** | 模板默认 20 步 |
| sampler / scheduler | **res_multistep** / **simple** | PLAN §2 #12【B】 |
| attention | **SDPA**（默认） | 避 sm_120 Sage 噪声（#15263【B，28】） |
| DiT 加载器文件名 | `minimax_h3_fl2va_pruned_int8_convrot.safetensors` | 目录 `models\diffusion_models` |
| TE 加载器文件名 | `qwen3vl_32b_minimax_h3_nvfp4_awq.safetensors` | 目录 `models\text_encoders` |
| Video VAE 文件名 | **`minimax_h3_video_vae_fp16.safetensors`**（0.11.39 默认 fp16） | 目录 `models\vae`；main 分支变体默认 int8_convrot |
| Audio VAE 文件名 | `minimax_h3_audio_vae_fp32.safetensors` | 目录 `models\vae` |
| 各 loader URL / 字节 / SHA256 | 见上半部第 3 章 3.1 / 3.2 表 | 已含 HF lfs.oid 与字节数【B】 |
| CreateVideo fps | **24** | 节点 130 `widgets_values_named.fps=24`（JSON 实测【B】） |
| SaveVideo 路径 | 默认 `ComfyUI\output`（模板内 SaveVideo 节点，文件名含 prompt/seed 占位） | 真机以 UI 节点显示为准 |

> 上面的 loader 文件名与 `main` 分支 JSON 抓取结果一致（DiT / TE / Audio VAE 三件同名；Video VAE 在 0.11.39 为 fp16、在 main 变体为 int8_convrot）。完整节点摘录见 `sources/M1b-template-nodes.md`。

### 6.3 PLAN 基线要改哪些值（节点 → 参数 → 默认 → 改为）

> 接线依据 PLAN §3.5（r2），节点编号取自 0.11.39 模板：UNETLoader(6)、BasicScheduler(9)、BasicGuider(16)、ResolutionSelector(115)、duration 的 `Float` 节点(111)。

| 节点 | 参数 | 模板默认 | 改为（基线） | 操作 |
|---|---|---|---|---|
| `ResolutionSelector(115)` | width / height 连线 | link 219→width、link 220→height，给 **864×480**（子图 width/height 控件本身默认 1344×768） | **断开 link 219 / 220**，子图输入手填 width=**480**、height=**832** | 断开后手动填（竖版，32 倍数） |
| `Float (duration)`(111) | value | 2 秒 | **5** | 经 `ComfyMathExpression` 得 124 帧（非直接填 length） |
| `BasicScheduler(9)` | steps | 20 | **8** | 改 steps 输入框 |
| `UNETLoader(6)` → `BasicScheduler(9).model` | link 5 | UNETLoader 直连 | 中间插入 `LoraLoaderModelOnly` → `MiniMaxH3SigmaShift` | 见 6.4 |
| `UNETLoader(6)` → `BasicGuider(16).model` | link 193 | UNETLoader 直连 | 同上，SigmaShift 输出同时接两者 | 见 6.4 |
| （新增）`LoraLoaderModelOnly` | model / strength | 无（模板无） | 加载 `minimax_h3_turbo_4step_ckpt600_ema_V4.safetensors`，strength **1.0** | 插在 UNETLoader 之后 |
| （新增）`MiniMaxH3SigmaShift` | shift_video / shift_audio | 无（0.11.39 无此节点） | **shift_video=12.0**、**shift_audio=6.0**（默认 audio=3.0 必须改） | 接在 LoraLoader 之后 |
| `RandomNoise` | seed | 556589502035082 | **保持不变**（即固定种子） | 固定 seed 用于三次重复 |
| attention / sampler / scheduler | — | SDPA / res_multistep / simple | **保持不变** | — |
| Video VAE | 文件名 | fp16 | **保持不变**（fp16） | 可重复证据最强【B，19】 |

### 6.4 接线步骤（严格按 PLAN §3.5 r2）

1. **断开** `ResolutionSelector(115)` 到子图的 width/height 连线（link 219 / 220），在子图 `MiniMaxH3ImageToVideo` 的 width / height 输入上手填 **480 / 832**。
2. 把 `Float (duration)` 节点(111) 的 value 从 **2** 改成 **5**（length 由 `ComfyMathExpression` 自动算成 124 帧）。
3. `BasicScheduler(9).steps` 从 **20** 改成 **8**。
4. **插入 `LoraLoaderModelOnly`**：断开 `UNETLoader(6).MODEL` → `BasicScheduler(9).model`（link 5）和 → `BasicGuider(16).model`（link 193）两条线；把 UNETLoader 的 MODEL 输出接进 `LoraLoaderModelOnly.model`，加载 `minimax_h3_turbo_4step_ckpt600_ema_V4.safetensors`，strength 1.0，取其 MODEL 输出。
5. **插入 `MiniMaxH3SigmaShift`**：接在 `LoraLoaderModelOnly` 的 MODEL 输出之后，设 `shift_video=12.0`、`shift_audio=6.0`。**其 MODEL 输出必须同时接两个地方**：① `BasicScheduler(9).model`（原 link 5 去向）；② `BasicGuider(16).model`（原 link 193 去向）。
   - ⚠ 关键：sigmas 由 `BasicScheduler` 按 model 的 `model_sampling` 算出；**只接 `BasicGuider` 时 shift 不作用到 sigma 调度**（PLAN §3.5 r2-1 实质错误修正）。
6. 确认 `RandomNoise` seed = **556589502035082** 未被动过。

### 6.5 固定 seed 与导出工作流 JSON

- **固定 seed**：在 `RandomNoise`（或 UI 里 seed 输入框）确保值为 `556589502035082`；点击输入框右侧“锁”图标可固定，避免每次提交被随机化。三次重复必须同 seed（PLAN §1）。
- **导出工作流 JSON（随回执交回）**：
  ```
  【只读】在 ComfyUI 界面：菜单 → "Save (API Format)" 或 "Export" → 存为
  "C:\Users\testuser\WorkBuddy\comfyui\ComfyUI\output\workflow-h3-baseline-<run>.json"
  ```
  或用 PowerShell 复制已保存文件（仅复制，不删）：
  ```powershell
  # 【只读】
  $ComfyRoot = "C:\Users\testuser\WorkBuddy\comfyui\ComfyUI"
  Copy-Item -LiteralPath (Join-Path $ComfyRoot "user\default\workflows\MiniMax_H3.json") -Destination "C:\logs\workflow-h3-baseline.json" -Force
  ```
  > 实际文件名以 UI 保存为准；导出后把该 JSON 纳入第 12 章回执的“工作流 JSON 和日志打包清单”。

## 第 7 章 系统设置（⚠ 均需 Junlong 本人批准）

> 以下两项是**修改系统 / 驱动级配置**，本手册**不自动执行**。仅在 Junlong 明确批准后，由测试员手动在图形界面操作，并按“恢复方法”可随时还原。出处见 PLAN §2 #16/#17、RESEARCH-LOG-002 R2-10/R2-13、DOSSIER §9.5。

### 7.1 页面文件（pagefile）

- **为什么要改**：32GB 物理内存 + H3 14.6GB TE 时，Windows 默认页面文件（部分系统仅 ~2GB）会在生成中途**无报错闪退**（R2-10【C+，sources/R2-10】，同款 32GB+Win11 实测）。PLAN §2 #16 建议固定初始 32768 MB / 最大 65536 MB，置于 NVMe。
- **GUI 操作路径（确认点：Junlong 批准后执行）**：
  1. Win + R → 输入 `sysdm.cpl` → 回车 → “高级” 选项卡 → “性能” 区 → “设置” → “高级” 选项卡 → “虚拟内存” 区 → “更改…”。
  2. 取消勾选“自动管理所有驱动器的分页文件大小”。
  3. 选中系统盘（通常是 C:，须是 NVMe 盘）→ 选“自定义大小”→ 初始大小 **32768** MB，最大大小 **65536** MB → “设置” → “确定”。
  4. **需要重启**：改完点确定后会提示重启，必须重启后新页面文件才生效。
- **恢复为“系统管理”的方法**：回到同一窗口 → 勾选“自动管理所有驱动器的分页文件大小” → 确定 → 重启。
- **证据等级**：固定 32–64GB 仅 R2-10 单源【C+】，PLAN 标“依据不足”；方向（页面文件是 32GB 的生死线）成立【B，R2-10/R2-11】。

### 7.2 NVIDIA Sysmem Fallback（CUDA 系统内存回退策略）

- **为什么要改**：VRAM 溢出经 WDDM 退避到系统内存后，每次 GEMM 都经 PCIe 重读权重——实测 >7GB 块带宽从 408GB/s 掉到 12GB/s，步时从秒级变分钟级（R2-13【B，sources/R2-13】）；且 shared 溢出 + VBS DMA 保护可触发 IOMMU 蓝屏（zenn 8 次先例【B，22】）。V1.0 建议设为“**优先无系统内存回退**”。
- **GUI 操作路径（确认点：Junlong 批准后执行）**：
  1. 右键桌面 → “NVIDIA 控制面板” → “3D 设置” → “管理 3D 设置”。
  2. 切到 **“程序设置”** 选项卡（只针对 ComfyUI 的 python.exe，不影响全局）→ “添加” → 浏览到 venv 的 `python.exe`（第 1.2 实测路径，如 `C:\Users\testuser\WorkBuddy\comfyui\ComfyUI\.venv\Scripts\python.exe`）。
  3. 在“指定该程序的设置”列表找 **“CUDA - 系统内存回退策略”**（英文 “CUDA - System Memory Fallback Policy”）→ 改为 **“优先无系统内存回退”**（Prefer No System Memory Fallback）。
  4. 点“应用”。
- **恢复方法**：同一 GUI 路径，把该项改回默认“允许”（Allow）/ 或删掉该程序条目 → 应用。
- **注意**：该值**只能在 NVIDIA 控制面板查看/修改，无命令行读取方式**（上半部 1.11）。如改为“优先无回退”后生成直接 OOM 且无法流式，应恢复默认并上报 Junlong（R2-13 假设，未执行验证【B 方向/C 验证】）。
- **证据等级**：B（zenn 蓝屏先例 + R2-13 机制）/ 关闭后实测效果未验证【C】。

## 第 8 章 监控与日志

> 目的：为第 9 章测试矩阵与第 12 章回执采集四类峰值（dedicated VRAM / shared GPU memory / 进程 RAM / commit）。采集命令**只新增日志文件**（不改系统设置、不改 ComfyUI 配置）。依据：PLAN §1 指标定义、RESEARCH-LOG-002 R2-11/R2-13、上半部 1.1。

**本章第一步：先建日志目录**（8.1–8.3 全部写入 `C:\logs`）：

```powershell
# 【会修改：仅创建目录，已存在则跳过；不覆盖、不删除任何文件】
New-Item -ItemType Directory -Force -Path "C:\logs" | Out-Null
```

### 8.1 GPU 监控（开第二个 PowerShell 窗口）

```powershell
# 【只读】新开一个 PowerShell 窗口后执行；采样 1 次/秒，由 nvidia-smi 自己写文件
& "$env:SystemRoot\System32\nvidia-smi.exe" --query-gpu=timestamp,memory.used,memory.total,utilization.gpu,power.draw,clocks.sm,temperature.gpu --format=csv -l 1 -f "C:\logs\gpu.csv"
```

- **字段有效性核实**：以上 7 个字段均为 `nvidia-smi --help-query-gpu` 的合法字段名（`timestamp`、`memory.used`、`memory.total`、`utilization.gpu`、`power.draw`、`clocks.sm`、`temperature.gpu`）【B，NVIDIA nvidia-smi 文档】。`-l 1` 表示每 1 秒刷新；`--format=csv` 输出带表头，VRAM 列名为 `memory.used [MiB]`。
- **写文件方式**：用 nvidia-smi 自带的 `-f "<路径>"` 参数直接写盘（**不要** `| Out-File`）——`-f` 由程序自己打开/刷新文件，不经过 PowerShell 管道缓冲，Ctrl+C 或崩溃时不会丢掉已刷新内容。
- **首次使用请核对**：跑 10 秒后 `Import-Csv "C:\logs\gpu.csv"` 看行数是否随时间增长（`-l` 循环下应逐次追加）；若发现只有“表头 + 1 行”（说明本驱动版本是覆盖写），改用它自带的 `-l 1` 输出到控制台并用 `Start-Transcript` 落盘（见 8.3 方式 A）。
- **停止**：在采样窗口按 `Ctrl+C`；文件保留已写入内容。**峰值计算见 8.4**。

### 8.2 系统内存 / 页面文件 / 共享显存监控（PowerShell 循环写 CSV）

```powershell
# 【只读】另一个 PowerShell 窗口；无限循环采样，Ctrl+C 停止
# 关键：每次采样后立刻 Export-Csv -Append 落盘，崩溃/蓝屏最多丢最后 1 行（不再攒内存）
$Log = "C:\logs\sys.csv"
$First = $true
while ($true) {
    $Ts = Get-Date -Format "yyyy-MM-dd HH:mm:ss"
    $C = Get-Counter -Counter '\Memory\Committed Bytes','\Memory\Available MBytes','\Paging File(_Total)\% Usage','\GPU Adapter Memory(*)\Shared Usage','\GPU Adapter Memory(*)\Dedicated Usage' -ErrorAction SilentlyContinue

    # _Total 类计数器只取唯一实例值；GPU 类对多实例取最大值（取值法见下方说明）
    $Commit = 0.0
    $Avail  = 0.0
    $Pf     = 0.0
    $SharedMax = 0.0
    $DedMax    = 0.0
    foreach ($P in $C.CounterSamples) {
        $Path = $P.Path.ToLower()
        $Val = [double]$P.CookedValue
        if ($Path -like '*\memory\committed bytes') {
            $Commit = $Val
        } elseif ($Path -like '*\memory\available mbytes') {
            $Avail = $Val
        } elseif ($Path -like '*paging file*% usage') {
            $Pf = $Val
        } elseif ($Path -like '*\gpu adapter memory(*)\shared usage') {
            if ($Val -gt $SharedMax) { $SharedMax = $Val }
        } elseif ($Path -like '*\gpu adapter memory(*)\dedicated usage') {
            if ($Val -gt $DedMax) { $DedMax = $Val }
        }
    }

    # 列名固定为 6 列，后续 8.4 / 第 12 章直接引用，不再“先看表头”
    $Row = [PSCustomObject]@{
        timestamp           = $Ts
        commit_bytes        = [int64]$Commit
        avail_mb            = [int64]$Avail
        pagefile_pct        = [math]::Round($Pf, 2)
        gpu_shared_bytes    = [int64]$SharedMax
        gpu_dedicated_bytes = [int64]$DedMax
    }
    if ($First) {
        $Row | Export-Csv -Path $Log -NoTypeInformation -Encoding utf8
        $First = $false
    } else {
        $Row | Export-Csv -Path $Log -NoTypeInformation -Encoding utf8 -Append
    }
}
```

- **为什么改成逐行落盘**：旧写法把样本攒在 `$Samples` 数组里、循环结束才统一写，一旦进程崩溃或整机蓝屏则**全部丢失**；现在每轮 `Export-Csv -Append`，最多丢最后 1 行。
- **固定列名（6 列）**：`timestamp` / `commit_bytes` / `avail_mb` / `pagefile_pct` / `gpu_shared_bytes` / `gpu_dedicated_bytes`。不再按 `实例名.计数器名` 动态拼接（列名不固定、Excel 里没法直接算）。
- **取值法（写明）**：
  - `_Total` 类（`\Memory\Committed Bytes`、`\Memory\Available MBytes`、`\Paging File(_Total)\% Usage`）→ **只取 `_Total` 实例本身的 `CookedValue`**（这三条本身无多实例）。
  - `\GPU Adapter Memory(*)\*` → 实例按适配器区分，**多实例取最大值**（`max`，不是求和）：本机只有一块 5070，正常只有 1 个有效实例；取 max 可避免系统出现重复/空实例时把显存算成 2 倍。若你的机器确有多块 GPU 且要总量，可改为求和（`$SharedSum += $Val`），但**第 12 章回执必须注明用了哪种取法**。
  - 单位：`commit_bytes` / `gpu_*_bytes` 为**字节**（÷1073741824 得 GiB）；`avail_mb` 为 MB；`pagefile_pct` 为百分数。
- **停止**：Ctrl+C。因每行已落盘，随时中断都不丢历史数据。
- **采样节奏**：`Get-Counter` 自身采集耗时约 1 秒，故循环间隔≈1 秒（未额外加 `Start-Sleep`），与 8.1 的 `-l 1` 大致对齐。
- **自查**：若 `gpu_shared_bytes` / `gpu_dedicated_bytes` 一直为 0，说明计数器集名或实例名不同，先跑下面的自查命令确认。

- **计数器名核实方法（先自查再跑）**：
  ```powershell
  # 【只读】列出所有 GPU 相关计数器集，确认 "GPU Adapter Memory" 存在
  Get-Counter -ListSet "GPU*" | Select-Object CounterSetName
  # 查看该集下具体计数器名（应含 Dedicated Usage / Shared Usage / Total Usage）
  (Get-Counter -ListSet "GPU Adapter Memory").Counter
  ```
  经核实，`\GPU Adapter Memory(*)\Shared Usage` 与 `\GPU Adapter Memory(*)\Dedicated Usage` 是合法路径（`*` 匹配各 GPU 适配器实例），单位**字节**【B，Windows Performance Counter】。`\Memory\Committed Bytes`、`\Memory\Available MBytes`、`\Paging File(_Total)\% Usage` 为标准内存计数器【B】。若 `Get-Counter` 报“计数器不存在”，先跑上面两条自查命令确认集名（部分系统集名为 “GPU Adapter Memory”，实例形如 “lodctr”）。
- **进程 RAM 峰值**：用任务管理器“性能 → 内存”的“已提交”看 commit；单进程 RAM 另开窗口用 `Get-Process -Name python | Select-Object WorkingSet64,PeakWorkingSet64,PrivateMemorySize64` 周期性采样（或用上半部 1.8 的 `Get-Counter '\Process(python)\Working Set - Private'`）。

### 8.3 ComfyUI 控制台日志

```powershell
# 【会修改：启动 ComfyUI 并把控制台输出落盘；确认点：测试员确认第5章五旗参数后再执行】
$ComfyRoot = "C:\Users\testuser\WorkBuddy\comfyui\ComfyUI"
$VenvPy = "<venv>\Scripts\python.exe"
New-Item -ItemType Directory -Force -Path "C:\logs" | Out-Null
Set-Location -LiteralPath $ComfyRoot
# 方式 A：Start-Transcript 记录整个会话（含 stderr）
Start-Transcript -Path "C:\logs\comfyui-transcript.txt" -Append
& $VenvPy main.py --disable-pinned-memory --reserve-vram 1 --disable-async-offload --cache-none --fast-disk 2>&1 | Tee-Object -FilePath "C:\logs\comfyui-console.txt"
Stop-Transcript
```

- **`2>&1` 的红色报错属正常（先看这条）**：PS 5.1 下 `2>&1` 会把 python.exe 写到 stderr 的每一行包成 `NativeCommandError`（`RemoteException`）并以**红色**显示，tqdm 进度条、`Warning`、甚至正常日志都可能触发。**这不代表失败**；判断成败看：① 日志里是否出现 `CUDA out of memory` / `Traceback` / `funchook_prepare ... failed` 等真实错误文本；② 命令结束后的 `$LASTEXITCODE`（0 为正常）。不要因为满屏红字就判定运行失败。
- **PS 5.1 stderr 的其他坑（务必知悉）**：
  1. `2>&1` 在 PowerShell 5.1 中对**外部 exe（如 python.exe）**能把 stderr 合并进成功流，再交给 `Tee-Object`；但对**内部 cmdlet** 的报错（非终止错误写到的错误流）需用 `*|` 之外的重定向，必要时加 `-ErrorVariable` 或 `*>`，但 `*>` 在 5.1 可用。
  2. `Start-Transcript` 能记录会话内所有主机输出（含外部程序 stdout/stderr 在主机上回显的内容），但**不保证逐字节捕获管道内的 `2>&1` 合并流**——两者建议**同时用**（Transcript 兜底 + Tee 拿到纯文本）。
  3. ComfyUI 是**长驻服务**，上面的管道会一直阻塞到进程退出；要在生成完成后手动 `Ctrl+C` 停止，日志才完整落盘。日志中重点搜：VRAM available、PyTorch version、funchook_prepare … failed（#15663 根因）、CUDA out of memory、Device mismatch（#15251）。

### 8.4 跑完后计算峰值（Import-Csv + Measure-Object）

```powershell
# 【只读】生成完成后统计峰值（列名固定，见 8.2 的 6 列）
# ---- GPU（8.1 的 nvidia-smi CSV，列名 "memory.used [MiB]"）----
$Gpu = Import-Csv -Path "C:\logs\gpu.csv"
($Gpu | Measure-Object -Property "memory.used [MiB]" -Maximum).Maximum
($Gpu | Measure-Object -Property "memory.used [MiB]" -Minimum).Minimum
($Gpu | Measure-Object -Property "temperature.gpu" -Maximum).Maximum

# ---- 系统（8.2 的固定 6 列，无需再 Get-Member 猜列名）----
$Sys = Import-Csv -Path "C:\logs\sys.csv"
$Commit = @($Sys | ForEach-Object { [double]$_.commit_bytes })
$Shared = @($Sys | ForEach-Object { [double]$_.gpu_shared_bytes })
$Ded    = @($Sys | ForEach-Object { [double]$_.gpu_dedicated_bytes })
$Pf     = @($Sys | ForEach-Object { [double]$_.pagefile_pct })
Write-Host ("commit 峰值(GiB): "        + [math]::Round((($Commit | Measure-Object -Maximum).Maximum) / 1073741824, 2))
Write-Host ("gpu_shared 峰值(GiB): "    + [math]::Round((($Shared | Measure-Object -Maximum).Maximum) / 1073741824, 2))
Write-Host ("gpu_dedicated 峰值(GiB): " + [math]::Round((($Ded    | Measure-Object -Maximum).Maximum) / 1073741824, 2))
Write-Host ("pagefile 峰值(%): "        + (($Pf | Measure-Object -Maximum).Maximum))
```

- 列名与 8.2 一一对应：`commit_bytes`（commit 峰值，字节）、`gpu_shared_bytes`（shared GPU memory 峰值，字节）、`gpu_dedicated_bytes`（dedicated VRAM 峰值，字节）、`pagefile_pct`（页面文件使用率峰值，%）、`avail_mb`（可用物理内存）。
- shared 峰值判定门槛是 **2 GiB**（= 2147483648 字节），可直接用上面的 `gpu_shared 峰值(GiB)` 与 2 比较。
- 若 `gpu_*` 两列全为 0：先回 8.2 跑计数器自查命令；若确认系统无此计数器，回执中填“缺失”，不编造。
- GPU 采样来自 8.1 的 nvidia-smi（MiB），系统侧来自 8.2（字节），两者口径不同但都填进第 12.2 表（注明单位）。

## 第 9 章 测试矩阵

> 依据：PLAN §1“通过”定义、§2 基线、§5 备选 F0/F1、§7 停止条件。所有配置用第 6 章改好的同一工作流，仅改其中参数。测试员按编号顺序执行。

### 9.1 矩阵表

| 编号 | 配置 | 分辨率 | 帧数 | steps | seed | 重复次数 | 通过条件 | 失败后转到 |
|---|---|---|---|---|---|---|---|---|
| **S0** | 冒烟（最低分辨率、短帧） | **352×608**（宽×高，竖版 9:16，与基线 480×832 方向一致；0.2MP 档） | **56**（duration=2） | 8 | 556589502035082 | 1 | 出片可播放、无纯噪声、无崩 | 查 §10；多半是 shared>2GB 或 OOM → F1 思路或上报 |
| **B1** | 基线（Qiita 全链路） | 480×832 | 124（duration=5） | 8 | 556589502035082 | 1 | 出片、双显存/RAM/commit 齐全 | §10 + §7 |
| **B2** | 基线 第 2 次 | 480×832 | 124 | 8 | 556589502035082 | 1 | 同 B1 | 同上 |
| **B3** | 基线 第 3 次 | 480×832 | 124 | 8 | 556589502035082 | 1 | 同 B1；**B1–B3 全成功=通过** | 同上 |
| **F0** | 官方 8 步 LoRA 对照 | 480×832 | 124 | 8 | 556589502035082 | 1 | 出片可播放（验证官方路径兼容） | 记 C 级，不影响主线 |
| **F1a** | 降分辨率备选 | **416×736** | 124 | 8 | 556589502035082 | 1 | 出片、shared<2GB | 再降 352×608（F1b） |
| **F1b** | 再降分辨率 | **352×608** | 124 | 8 | 556589502035082 | 1 | 出片 | 上报 Junlong |

### 9.2 轮次间是否重启 ComfyUI（按 PLAN 写）

- **S0 与 B1 之间**：建议**重启一次** ComfyUI（清掉冒烟残留，排除 S0 低分辨率对缓存/显存的影响），并按第 12 章填预检/启动记录。
- **B1 → B2 → B3（基线三次）**：**不重启 ComfyUI 进程、不手动清理模型缓存**（PLAN §1“可重复”要求，用于检验内存/显存是否泄漏，对应 R2-11 的“平坦”判定）。若三次中有任意一次失败或任一指标触发 §7 停止条件，则本轮“不通过”。
- **进入 F0 / F1 等备选轮次前**：建议**重启 ComfyUI**，且换 LoRA/分辨率前记录当前状态；备选轮次各自独立，互不继承缓存。
- 任何轮次触发 §7 停止条件（蓝屏 / commit>30GB / shared>2GB / 磁盘<20GB / 需改驱动）→ **立即停止并上报**，不继续后续编号。

### 9.3 三次输出“相同”怎么判定（B1–B3）

- **哈希法（充分非必要）**：对三次 mp4 跑 `Get-FileHash -Algorithm SHA256`：
  ```powershell
  # 【只读】
  Get-FileHash -Algorithm SHA256 -LiteralPath "C:\logs\output-B1.mp4"
  Get-FileHash -Algorithm SHA256 -LiteralPath "C:\logs\output-B2.mp4"
  Get-FileHash -Algorithm SHA256 -LiteralPath "C:\logs\output-B3.mp4"
  ```
  - **哈希不同 ≠ 失败**：视频编码（含 VAE 解码、H.264/H.265 量化、音频编码）在固定 seed 下仍可能因非确定性 kernel、音频 VAE 浮点累加等产生字节级差异；PLAN §1 的“相同”指**视觉/听感一致**，不是逐字节一致。
  - **替代判定（推荐）**：① 文件大小相近（差异 < 5%，排除截断/损坏）；② 人工或脚本逐帧比对——用 `ffmpeg` 抽帧后算帧间 PSNR/SSIM（若本机无 ffmpeg 则人工播放肉眼比对）；③ 三次的**峰值指标（双显存/RAM/commit/耗时）波动在合理范围**（如耗时 ±10%、峰值 ±5%）。以上任一组合满足即判“可重复”，不强制哈希相等。
- 记录：第 12 章回执需填每个输出的“文件名 + SHA256 + 主观质量 1–5 分”。

## 第 10 章 已知错误速查

> 依据：各 issue 原页面（sources/25、28、26、29、27、31、R2-01 及 PLAN §4 风险表）。每条给出“症状关键词 | 原因 | 来源 issue | 处理”。等级均 B（已读原页面）。

| 症状 / 报错原文关键词 | 原因 | 来源 issue | 处理 |
|---|---|---|---|
| VAE 解码 `device mismatch` / `torch.cat` / `addcmul_` 在 CPU↔cuda 之间报错（8GB 卡部分卸载时） | MiniMax H3 Video VAE 部分卸载到 CPU 时张量设备不一致（`comfy/ldm/minimax/vae.py`） | #15251【B，29】 | 升级到含 PR #15224 的 ComfyUI（≥ v0.32.0）；0.30.0 仍不可用，必须 v0.32.0 |
| 视频+音频**纯噪声**、无警告（sm_120） | SageAttention FP8 PV kernel 在 token > ~167k 时出噪点（sm_120） | #15263【B，28】 | V1.0 默认 **SDPA**，不用 Sage；若日后启 Sage，只用 `auto` 模式且 token < 154k（本基线 480×832/124 帧≈14.4k token，远低于阈值，安全区） |
| 短片显存估 514MB、常驻 16GB、`Allocated peak 28.9GB`；**OOM 后每次重跑都同点 OOM**（粘住） | `comfy-aimdo` 初始化失败 → DynamicVRAM 未激活 → VRAM 估算失真 | #15663【B，25】 | 用第 5 章五旗（`--disable-pinned-memory --reserve-vram 1 --disable-async-offload`）；仍 OOM 改任一节点参数或插无关节点；勿擅改系统设置 |
| VAE 解码 `Ran out of memory … retrying with tiled VAE decoding` 后**重试仍 OOM** | `handles_tiling=True` 的 VAE（H3 video VAE）tiled 回退重跑相同 decode，且 ~14GB 流式权重仍常驻、未释放 | #15456【B，26】（修复）/ #15446【B，R2-05】 | 必须升级到含 #15446/#15456 的 ComfyUI（v0.32.0 满足）；锁 124 帧；确保解码前释放 DiT/TE |
| `Model Sparse Attention` 节点开启后 ~50% 处 OOM（`Requested 6.24 GiB`…`Free 2.49 GiB`） | Comfy model compiler 与该节点冲突 | #16175【B，27】 | 启动加 `--disable-comfy-compiler`；V1.0 默认不启用该节点 |
| 音频低音缺失 / 闷 / 振幅减半（用 EasyCache 时） | EasyCache 按视频流选跳步、把缓存残差套到音频（H3 音频不共享调度） | #15326【B，31】 | **V1.0 不使用 EasyCache 节点**（PLAN §6.5 / 风险 R 音频）；遇音频劣化先查是否误加了该节点 |
| 文本编码后进程 RSS 不降、13GB 映射滞留 RAM，卡在 `copy_`/`memcpy_and_sync` | TE mmap 被强引用（LARGEST_CASTED_WEIGHT），“已卸载”仍占 RAM | #17122【B，R2-01】 | 编码后**强制 unload + 清引用**；监控 RSS 采样后是否回落（8.2 的进程 RAM 采样） |
| `CUDA out of memory` | 权重/激活超显存（含 shared 溢出的慢路径） | 通用 | 降分辨率（F1）→ 优先于加 `--vram-headroom`（后者对“权重页进共享内存”无效，R2-13）；shared>2GB 判不通过 |
| 生成中途**窗口无报错消失**（静默崩溃） | 多为 commit/RAM 耗尽 + 页面文件过小 | R2-10/R2-11【B/C+】 | 查 commit 峰值与页面文件（第 7 章）；停止并上报 Junlong |
| **蓝屏 / 整机重启**（如 HYPERVISOR_ERROR 0x20001） | VRAM 溢出回退系统内存 + VBS DMA 保护触发 IOMMU（长片段风险） | zenn【B，22】/R2-13 | **立即停止测试**；收集 `C:\Windows\Minidump` 下 `.dmp`（只复制，不删除）交 Junlong；锁 ≤124 帧 |
| 下载文件哈希/字节数与第 3 章不符 | 下载不完整 / 中断 | 通用 | 用 `curl.exe -L -C - -o` 断点续传重下该文件；损坏的不完整文件**保留不删**，等 Junlong 指示 |

> 来源 issue 原页面：#15251 `github.com/Comfy-Org/ComfyUI/issues/15251`、#15263 `.../issues/15263`、#15446 `.../pull/15446`、#15456 `.../pull/15456`、#16175 `.../issues/16175`、#15326 `.../issues/15326`、#15663 `.../issues/15663`、#17122 `.../issues/17122`（均 2026-11-04 读取，状态：已打开可读）。

## 第 11 章 回滚

> 原则：**只允许重命名（Rename-Item）或复制（Copy-Item），绝不删除**（DOSSIER §7、PLAN §8）。任何删除（含 `.venv-failed-*`、不完整下载、旧备份）一律需 Junlong 批准。依据：PLAN §8、上半部第 2 章。

### 11.1 回滚 venv

```powershell
# 【会修改：把失败 venv 重命名保留，从备份复制回原名；撤销=反向操作】
# 确认点：测试员确认后再执行（属环境改动）
$ComfyRoot = "C:\Users\testuser\WorkBuddy\comfyui\ComfyUI"
$Venv = "<venv>"   # 第1.2实测
$ts = Get-Date -Format "yyyyMMdd-HHmm"
# 1) 失败环境重命名（保留不删）
Rename-Item -LiteralPath $Venv -NewName (".venv-failed-" + $ts)
# 2) 从备份复制回原名
$Backup = Join-Path $ComfyRoot "venv-backup-<备份时ts>"
Copy-Item -LiteralPath $Backup -Destination $Venv -Recurse
Write-Host ("venv 已回滚：失败环境保留为 .venv-failed-" + $ts)
```

### 11.2 回滚 ComfyUI 的 commit

```powershell
# 【会修改：git 切回第 4 章记录的“升级前 commit”；属 git 改动，需 Junlong 批准】
# 确认点：测试员确认后再执行
$ComfyRoot = "C:\Users\testuser\WorkBuddy\comfyui\ComfyUI"
# 先记录当前 commit（如需再次回退）
git -C $ComfyRoot rev-parse HEAD | Out-File -LiteralPath (Join-Path $ComfyRoot "comfyui-commit-after.txt") -Encoding utf8
# 从第 4 章 4.2 写下的 comfyui-commit-before.txt 读出原始 commit
$BeforeFile = Join-Path $ComfyRoot "comfyui-commit-before.txt"
if (-not (Test-Path -LiteralPath $BeforeFile)) {
    Write-Host "STOP: 找不到 comfyui-commit-before.txt（第 4 章未记录），请上报 Junlong。"
    throw "缺少升级前 commit 记录，未执行 checkout。"
}
$c = (Get-Content -LiteralPath $BeforeFile | Select-Object -First 1).Trim()
if ([string]::IsNullOrWhiteSpace($c)) {
    Write-Host "STOP: comfyui-commit-before.txt 内容为空，请上报 Junlong。"
    throw "升级前 commit 为空，未执行 checkout。"
}
Write-Host ("准备切回 commit: " + $c)
git -C $ComfyRoot checkout $c
```

- **为什么要读文件**：`checkout v0.32.0` 只能回到标签，回不到“升级前那个具体 commit”；第 4 章 4.2 已先把原 HEAD 写入 `comfyui-commit-before.txt`，这里读第 1 行作为回滚目标。
- 若第 4 章未升级（4.1 判定已满足、直接跳过），该文件可能不存在——此时无需回滚 commit，跳过本节即可。

### 11.3 回滚页面文件 / Sysmem Fallback

- **页面文件**：按第 7.1“恢复为系统管理”的 GUI 路径勾选“自动管理所有驱动器的分页文件大小” → 确定 → **重启**（恢复不需要 Junlong 额外批准，因为只是还原到默认）。
- **Sysmem Fallback**：按第 7.2 同一 GUI 路径把“CUDA - 系统内存回退策略”改回默认“允许”（或删除该程序条目）→ 应用（还原到默认，不需额外批准）。
- ⚠ 若之前未改过这两项，则无需回滚；只有第 7 章**经 Junlong 批准并实际改过**时才需还原。

### 11.4 模型文件回滚（切换 F1–F5 时）

- 旧权重**保留不删**；切换备选配置时仅新增文件，是否清理旧权重由 Junlong 决定（PLAN §8）。用 `Copy-Item` 备份而非删除。

## 第 12 章 回执模板

> 测试员逐项填写后，连同工作流 JSON、日志打包交回 Junlong。时间一律用**东京时间（JST，UTC+9）**；缺失项填“缺失”，不得编造。

### 12.1 预检记录表（来自上半部 1.12，复填确认）

| 项 | 实测值 | 合格? | 备注 |
|---|---|---|---|
| GPU 名 / 驱动 / 显存(MiB) / compute_cap | | | |
| venv 路径 / python.exe | | | |
| ComfyUI 版本 | | | |
| torch 版本 / torch.version.cuda / 含 sm_120? | | | |
| comfy-aimdo / comfy-kitchen / sageattention / triton | | | |
| C 盘剩余(GiB) | | | |
| 物理内存(MB) / Commit Limit | | | |
| 页面文件（系统管理/固定） | | | |
| 电源方案 / Sysmem Fallback | | | |

### 12.2 每次运行记录表（每行一次，B1/B2/B3 各一行）

| 字段 | 运行 B1 | 运行 B2 | 运行 B3 |
|---|---|---|---|
| 开始时间（JST） | | | |
| 结束时间（JST） | | | |
| 耗时（秒） | | | |
| 显存 dedicated 峰值(MiB) | | | |
| 显存 shared 峰值(MiB/GiB) | | | |
| commit 峰值(GB) | | | |
| 进程 RAM 峰值(GB) | | | |
| 输出文件名 | | | |
| 输出 SHA256 | | | |
| 报错原文（无则填“无”） | | | |
| 截图清单（路径） | | | |
| 主观质量（1–5 分） | | | |
| 工作流 JSON / 日志打包（✓/✗） | | | |

### 12.3 打包清单（随回执交回）

- [ ] 工作流 JSON：`workflow-h3-baseline.json`（第 6.5 导出）
- [ ] GPU 日志：`C:\logs\gpu.csv`（第 8.1）
- [ ] 系统日志：`C:\logs\sys.csv`（第 8.2）
- [ ] ComfyUI 控制台日志：`C:\logs\comfyui-transcript.txt` 与 `comfyui-console.txt`（第 8.3）
- [ ] 三次输出 mp4 及各自 SHA256（第 9.3）
- [ ] 如触发蓝屏：`C:\Windows\Minidump\*.dmp` **复制**件（不删原文件）
- [ ] 本章 12.1 / 12.2 表格填写完整

> 提交前自查：四项峰值（双显存/RAM/commit）是否齐全、是否有任意一次失败、是否有“缺失”未补。任一指标触发 PLAN §7 停止条件即判“不通过”并按 §7 上报。

---

## 命令自查表（本手册下半部）

| 代码块 | 章节 | 是否只读 | PS5.1 语法已自查 | 备注 |
|---|---|---|---|---|
| 6.5 复制工作流 JSON | 6.5 | 是（只读） | 是 | `Copy-Item` 仅复制 |
| 7.1 页面文件 | 7.1 | 否（GUI 手动） | — | ⚠ 需 Junlong 批准；需重启 |
| 7.2 Sysmem Fallback | 7.2 | 否（GUI 手动） | — | ⚠ 需 Junlong 批准 |
| 8 开头 建日志目录 | 8 | 是（仅建目录） | 是 | `New-Item -Force`，不覆盖已有文件 |
| 8.1 nvidia-smi 采样 | 8.1 | 是 | 是 | `&` 调用 exe；自带 `-f` 写文件，无管道 |
| 8.2 系统计数器循环 | 8.2 | 是 | 是 | `while`/`Get-Counter`/`Export-Csv -Append` 逐行落盘；固定 6 列 |
| 8.2 计数器名自查 | 8.2 | 是 | 是 | `Get-Counter -ListSet "GPU*"` |
| 8.3 控制台日志 | 8.3 | 否（会修改：落盘日志） | 是 | 有“确认点”；`2>&1 \| Tee-Object`；Start-Transcript；红字属正常 |
| 8.4 峰值计算 | 8.4 | 是 | 是 | `Import-Csv`/`Measure-Object`；直接用固定列名 |
| 9.3 输出哈希 | 9.3 | 是 | 是 | `Get-FileHash` |
| 11.1 回滚 venv | 11.1 | 否（会修改） | 是 | 有“确认点”；`Rename-Item`/`Copy-Item` |
| 11.2 回滚 commit | 11.2 | 否（会修改） | 是 | 有“确认点”；读 `comfyui-commit-before.txt` 后 checkout |
| 11.3 还原系统设置 | 11.3 | 否（GUI 手动） | — | 还原默认，不需额外批准 |

- 全部代码块路径加引号；无 `&&`/`||`/`??`/`?.`/三元/`ForEach-Object -Parallel`/`Join-String`/`-AsHashtable`；无 `ls`/`cat`/`curl`(别名)/`wget`（外部下载用 `curl.exe`）。
- 所有“会修改”块（8.3、11.1、11.2）均带“确认点：测试员确认后再执行”；删除类操作一律不在本手册内（第 7/11 章系统设置仅还原默认或需 Junlong 批准）。
- nvidia-smi 字段名与 `Get-Counter` 计数器名均有出处（8.1 / 8.2 自查命令 + NVIDIA/Windows 文档【B】）。

---

*（MANUAL-V1.0-partB.md 完，含第 6–12 章。全文无“本机验证成功”表述；数字均标出来源与等级。）*
