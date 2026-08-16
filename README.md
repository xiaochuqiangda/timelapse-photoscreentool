# 延时摄影筛图工具（LightningPicker）

- 库里已有的压缩包为轻量包，不包含所需的Python库，需要提前安装环境
- 懒人一键包请前往https://pan.baidu.com/s/1_pP3rcylSSfg7ZJy-yYiTg?pwd=se28 提取码: se28 自行下载

基于 **Tauri + Python 子进程** 的闪电照片自动筛选应用。从延时摄影连拍中自动挑出拍到闪电的帧。

- **前端**：文件管理面板、参数滑块、区域选择画布（矩形/多边形）、任务状态、实时日志、首次启动环境向导
- **Rust 后端**：扫描文件夹（含 EXIF 拍摄时间）、常驻 Python 子进程管理（崩溃自动重启）、JSON IPC、进度事件推送
- **Python 后端**：stdin/stdout JSON 服务入口，复用现有差分检测算法（rawpy 解码 NEF 等 RAW），支持连续帧差分 / 单帧阈值两种模式、归一化选区过滤、命中文件复制与标记图

---

## 目录结构

```
autophoto/
├── src/                        # 前端（React + TypeScript + Vite）
│   ├── App.tsx                 # 主界面编排
│   ├── api.ts                  # Tauri 命令封装 + 事件订阅
│   ├── components/             # EnvWizard / FilePanel / PreviewCanvas / ParamPanel / StatusBar / LogPanel
│   └── types.ts                # 共享类型 + 内置预设
├── src-tauri/                  # Rust 后端（Tauri v2）
│   ├── src/
│   │   ├── lib.rs              # 应用装配、退出清理
│   │   ├── commands.rs         # Tauri 命令（扫描/解码/检测/预设/日志…）
│   │   ├── python_worker.rs    # Python 子进程：请求-响应路由 + 事件转发 + 自动重启
│   │   └── environment.rs      # 环境检测 / 引导下载 / python 目录定位
│   ├── .cargo/config.toml      # crates.io 镜像（rsproxy，可删）
│   └── tauri.conf.json
├── python/                     # Python 后端
│   ├── service.py              # JSON 服务入口（常驻，stdin/stdout）
│   ├── lightning_detector.py   # 检测算法库（由现有脚本重构）+ CLI
│   ├── check_env.py            # 环境检查（Rust 探针）
│   └── requirements.txt
├── scripts/gen_icons.ps1       # 图标生成（PNG + ICO）
└── 框架.txt / 一个简单的闪电检测脚本，不完善.txt   # 原始设计文档
```

---

## 快速开始

### 方式一：双击启动（推荐）
直接双击项目根目录的 **`启动应用.bat`**：
- 已编译过 → 立即启动应用（无控制台窗口）
- 首次运行 → 自动完成 `npm install` + release 编译（约 5-15 分钟），完成后自动启动

需要开发模式（Vite 热更新 + 调试编译）时双击 **`启动应用-开发模式.bat`**。

### 方式二：命令行
#### 环境要求
- [Node.js](https://nodejs.org/) 18+、[Rust](https://rustup.rs/) 1.77+
- Python 3.10+（Windows 也可用 `winget install Python.Python.3.12`）
- Windows 需 WebView2（Win10/11 自带）；macOS 需 Xcode CLT

#### 1. 安装依赖
```bash
npm install            # 前端依赖（已配置 npmmirror 镜像，可删 .npmrc 换官方源）
pip install -r python/requirements.txt   # Python 依赖
```
> Python 依赖也可在应用内「环境设置 → 一键安装依赖」完成。

#### 2. 开发运行
```bash
npm run tauri dev
```
首次启动会弹出**环境准备向导**：检测 `python / python3 / py` 与 numpy、cv2、rawpy、Pillow；
缺失时提供 打开下载页 / 一键装依赖 / 手动指定解释器 三种途径。

#### 3. 打包
```bash
npm run tauri build     # 产出 NSIS/MSI 安装包（Windows）
```

---

## 使用流程

0. **切换筛图功能**：顶栏左上角点击当前功能名（如「⚡ 闪电检测」）弹出下拉列表，
   选择功能切换右侧控制界面（列表只含已实现的功能：闪电检测 / 流星检测）
1. **扫描**：左侧选择输入文件夹 →「扫描文件夹」（支持 RAW/JPG/PNG/TIFF…，按拍摄时间排序）
2. **预览**：点击缩略图查看大图；可拖动绘制**矩形/多边形选区**（坐标归一化，多张图通用），
   勾选「仅检测框选区域」后只检测选区
3. **调参**：右侧灵敏度/阈值等滑块（各功能参数独立保存），可保存/应用预设（预设按功能分组）
4. **测试单张**：对当前图片立即跑一次检测，画布上绿色框为命中区域
5. **开始检测**：批量处理，底部进度条 + 实时日志；命中帧复制到输出文件夹
   （默认 `输入目录\Lightning_Export`（闪电）/ `输入目录\Meteor_Export`（流星），
   文件名 `lightning_001_score0.917_原名.ext` / `meteor_001_...`），
   可选同时保存叠加检测框的标记图（`marked/` 子目录）
6. **视频检测**（可选）：左侧「📹 打开视频」选择 mp4/mov/avi/mkv 等，
   点「开始检测」自动**流式拆帧**（不落盘全部帧）并按当前筛图功能逐帧检测，
   只把命中帧输出为图片（文件名带原始帧号，如 `lightning_001_f000123.jpg`），
   便于回溯到视频中的位置；「抽帧间隔」可设每 N 帧取 1 帧（帧数过多自动加大间隔）
   - 帧间差分类检测对视频天然适配：闪电/流星都可用；流星滑动窗口模式在视频中
     以"最近 N 帧中值"作底图（流式窗口）
   - 检测过程中内存恒定（只保留前一帧 / 最近窗口）；输出目录默认在视频旁 `视频_导出/`
   - **格式兼容**：基于 OpenCV FFmpeg 解码，实测支持 mp4/mov/avi/mkv/webm/wmv/ts 中的
     H.264、H.265/HEVC、MPEG-4、VP8/VP9、AV1、MJPG 等编码；打不开时依次回退
     Windows 系统解码器（Media Foundation）与 imageio-ffmpeg（自带完整 FFmpeg），
     仍失败会给出明确提示；检测开始时日志栏会打印视频信息（分辨率/帧率/编码/解码后端）
   - **进度与任务锁**：视频拆帧进度在底部状态栏实时显示（抽帧间隔为 1 时逐帧刷新）；
     任务进行中「开始检测/测试单张/扫描文件夹/更换视频」等按钮全部锁定，防止任务冲突，
     完成后自动解锁并汇总命中帧
   - **流星视频底部伪影过滤**：视频底部边缘常见编码条带/黑边伪影（画面内容不变的
     紧贴底边区域被误检），视频模式自动排除「紧贴画面边缘的轮廓」并忽略底部约 1%
     的条带区域，不影响画面内真实流星（批量图片检测不受影响）

### 流星检测（☄️，算法参考文件夹内《流星检测算法一/二》）
- **帧间差分**：相邻帧相减，用长宽比区分"流星/飞机细长亮线"与"星点移动产生的近圆形斑点"，
  综合得分 = 面积×0.3 + 长宽比×0.35 + 亮度变化×0.2 + 集中度×0.15（内存极低）
- **滑动窗口**：窗口内帧取中值造底图（自动适配星点漂移），当前帧减底图找增亮的细长轨迹；
  边界帧自动回退为帧间差分；窗口越大越抗漂移、越慢
- 输出前缀默认 `meteor`；其余行为（标记图/并行/取消/选区）与闪电检测一致

### 飞机过滤（✈️，参考《飞机轨迹过滤算法》，流星检测专属，可折叠参数区自由开关）
- **原理**：逐帧检测所有细长亮线（流星+飞机+卫星），跨帧按「角度 + 位置」关联成轨迹；
  轨迹出现帧数 ≤ 最大连续帧数（默认 2）才保留——流星只出现 1-2 帧，
  飞机/卫星连续多帧同方向移动 → 整条轨迹丢弃
- 参数：最大连续帧数 / 角度容差 / 位置容差（随线长自适应）/ 允许断帧（应对飞机灯闪烁漏检）
- 对参考算法的优化：匹配时取「角度+距离」组合代价最小的最佳匹配（而非首个匹配）；
  位置容差随线长自适应（长曝光轨迹帧间位移更大）；支持滑动窗口模式的闪烁飞机过滤
- **仅批量/视频检测生效**（单帧测试无法判断连续性）；视频模式下命中帧缓冲到流结束统一判定再落盘

### 多功能切换架构（扩展新筛图功能）
注册表位于 `src/types.ts` 的 `FILTER_FUNCTIONS`，新增一个筛图功能（实现后）只需三步：
1. `FilterFunctionId` 追加新 id
2. `FILTER_FUNCTIONS` 追加元数据（`icon` / `name` / `desc`）
3. `src/App.tsx` 右栏渲染处挂载对应的控制面板组件

顶栏下拉会自动列出注册表中的全部功能。

---

## 架构说明

### 通信协议（Rust ↔ Python）
Python 以常驻进程运行，Rust 通过 `std::process` 管道与其交换**逐行 JSON**：

```
请求:  {"id": 1, "cmd": "batch_detect", "task_id": "t1", "files": [...], "params": {...}, ...}
响应:  {"id": 1, "ok": true, "data": {...}}
事件:  {"type": "progress", "task_id": "t1", "current": 3, "total": 100, "hits": 1, "file": "..."}
       {"type": "log", "level": "info", "message": "..."}
```

- Rust 侧 `PyWorker` 按 `id` 路由响应到对应请求（支持并发在途请求），无 `id` 的行作为事件推送给前端
- 批量检测在 Python 子线程执行，主循环继续读 stdin（可随时接收 `cancel`）；`quit` 时等待在途任务收尾
- 子进程崩溃自动重启；stderr 落日志文件（`%APPDATA%/LightningPicker/python_stderr.log`）

### 多核并行（各筛图功能独立开关）
- 各功能参数面板均有「多核并行加速（批量检测）」开关（默认开启），属于该功能自身执行选项
- 开启后按帧拆分成**多进程**（`ProcessPoolExecutor`，worker 数 = min(CPU 核数, 4)）并行分析，
  绕开 GIL；文件输出在父进程**按文件顺序**统一写盘，命中编号/文件名与串行模式完全一致
- 并行/串行结果一致性由 `python/_selftest3.py`（闪电）、`python/_selftest4_meteor.py`（流星）验证
- 取消任务 = 停止提交 + 取消排队中的帧（在途帧自然完成）；文件数 < 4 时自动回退串行
  （进程池启动开销对小批量无收益）
- 单帧测试（test_single）始终单线程，不受该开关影响

### 检测算法（`python/lightning_detector.py` / `python/meteor_detector.py` / `python/video_detect.py`）
闪电（`lightning_detector.py`）保留原脚本的**帧间差分 + 颜色特征 + 空间集中度**打分：
- `diff` 模式：相邻帧亮度突变像素统计 → 蓝白特征 → 集中度 → 加权得分
- `single` 模式：单帧高亮阈值 → 形态学去噪 → 连通域过滤 → 同上打分
- 二者都支持归一化选区掩码（rect/polygon）与命中框输出

流星（`meteor_detector.py`）参考《流星检测算法一/二》：
- `diff` 模式：相邻帧绝对差分 → 细长轮廓（长宽比过滤）→ 面积/长宽比/亮度/集中度加权得分
- `window` 模式：滑动窗口中值底图差分（只保留增亮部分），边界帧回退差分
- 服务层 `detect` / `batch_detect` 指令带 `function` 字段（lightning | meteor）分发到对应检测器；
  `video_detect` 指令流式读取视频并复用两者的"内存帧"检测核心（analyze_rgb / analyze_frames），
  只把命中帧落盘；解码多后端回退（cv2 FFmpeg → Media Foundation → imageio-ffmpeg），
  流星视频模式开启贴边轮廓/底部条带伪影过滤（drop_border + bottom_band），
  `video_detect` 与 `batch_detect` 一样为异步指令（立即回包会导致前端误判任务结束）
  （`python/_selftest5_video.py` 验证两种功能 + 抽帧间隔 + 取消 + 服务协议 + 底部条带伪影过滤）
- 流星检测含飞机过滤：`filter_aircraft`（跨帧轨迹关联）+ `apply_aircraft_filter` / `rebuild_from_kept`
  （批量两阶段：先收集候选线 → 关联过滤 → 再写盘；并行路径在父进程执行过滤，与串行一致；
  `python/_selftest6_aircraft.py` 验证 diff/window/视频三场景 + 并行一致性）

### 环境层
- 启动时后台探测 Python 与依赖（`python -c` 探针，12s 超时）
- 向导提供：自动装依赖（`pip install -r requirements.txt`，输出实时回显）、
  引导下载（Python 官方页 / python-build-standalone 嵌入式运行时）、手动指定解释器路径
- 解释器路径与预设持久化在 `%APPDATA%/LightningPicker/`（config.json / presets.json）

---

## 自定义图标

图标文件位于 **`src-tauri/icons/`**（`tauri.conf.json` → `bundle.icon` 引用）：
- `icon.ico` — Windows 窗口标题栏 + exe 文件图标（编译时嵌入）
- `32x32.png` / `128x128.png` / `128x128@2x.png` — 通用尺寸图标

两种改法：
1. **替换文件**：用图片工具生成同名文件直接覆盖（`icon.ico` 建议含 256x256 尺寸）
2. **改代码重生成**：编辑 `scripts/gen_icons.ps1` 中的 `New-BoltBitmap`
   （背景渐变颜色、闪电多边形坐标 `$pts` 等），运行
   `powershell -ExecutionPolicy Bypass -File scripts\gen_icons.ps1` 重新生成全部图标

### 重新编译（改完图标后）
```bash
npm run tauri build -- --no-bundle     # 项目根目录执行（推荐，标准方式）
```
只改了图标、想省去前端构建时，也可在 `src-tauri` 目录执行：
```bash
cargo build --release --features custom-protocol
```
> ⚠️ 千万不要用不带 `--features custom-protocol` 的 `cargo build --release`：
> 那样构建出的是**开发模式** exe，启动时会去连 `localhost:1420`（vite 开发服务器），
> 没有 vite 就会报「localhost 拒绝连接」。`--features custom-protocol` 才会把前端资源
> 内嵌进 exe 成为独立版。两条命令请固定用一条（混用会触发一次较慢的全量重编）。

> 图标变化会自动触发重编译（`build.rs` 已用图标内容哈希强制重链接），
> 不需要手动清理。exe 图标是编译期嵌入的，改完必须重新编译才能看到效果。

## 常见问题

| 问题 | 处理 |
| --- | --- |
| 找不到 Python | 环境向导 → 打开下载页安装，或「手动指定 Python 路径」 |
| rawpy 安装失败 | 需与 Python 版本匹配的预编译 wheel，可升级 Python 或降级至 3.11/3.12 |
| 杀毒软件误报 | Python 子进程调用为正常行为，请将应用加入白名单 |
| 检测偏松/偏紧 | 调低/调高「灵敏度」，或降低「最小突变像素数」/「得分阈值」 |
| 操作时弹出命令行窗口 | 已默认隐藏 Python 子进程控制台（CREATE_NO_WINDOW）；若再出现请确认使用最新编译的 exe |

## 已知限制
- HEIF/AVIF 需要额外安装 `pillow-heif`，当前未内置
- 嵌入式 Python 自动下载未实现，采用「引导下载 + 离线包手动导入」方案（见环境向导）
- 系统托盘、系统通知为框架预留项，尚未实现
