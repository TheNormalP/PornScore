# PornScore v2 — 本地媒体"色情质量"评分器（自包含 Windows 桌面版）

对**本地**图片/视频打分（0–100），分数越高代表越"色情"（越能唤起性欲的视觉刺激），
用于从媒体库中**精选**高刺激度内容。**完全离线运行，不上传任何文件，不要求安装任何东西**：
解压后双击 `pornscore_gui.bat`（GUI）或 `pornscore.bat`（CLI）即可。

```
 pornscore.bat --check                       rem 新机器先跑环境体检
 pornscore_gui.bat                           rem 启动图形界面（可视化选目录 + 实时等级列表）
 pornscore.bat D:\videos --top 20
 pornscore.bat D:\media --model inception --copy-top 10
 pornscore.bat D:\pics --exts jpg,png,mp4 --min-score 60 --copy-top 20
```

> ⚠️ **安全与法律声明**
> 本工具仅用于整理**你自有的、全部由成年人出演的合法媒体**。
> 机器学习分类器**无法**验证画面中人物的年龄或同意状态，也不会替你审核内容合法性，
> 请自行确保媒体库中不含任何违法内容。本工具不对任何违法用途负责。

**v2 与 v1 的关系**：评分逻辑（权重/五档等级/视频时间聚合/报告/GUI）与 v1 完全一致；
唯一变化是推理层——v1 的 Node.js + nsfwjs(WASM) 换成了**内置 ONNX 模型 + ONNX Runtime**，
因此**不再需要系统安装 Node.js**：整个 `PornScore/` 目录就是一个自包含程序。
模型权重与 v1 同源（原版随包的 nsfwjs 内置权重解包重建），
并经与原版推理逐图交叉验证（100+ 图，逐类概率 |Δ|≤0.02，见 `DECISIONS.md`/`SIZE_AUDIT.md`）。

---

## 一、"色情质量"如何量化（调研结论）

"色情质量"（pornographic quality）在学术上通常指**唤起度（sexual arousal）的视觉可预测性**。
综合公开研究，没有单一的"质量分"模型，业界与学界的做法是：

### 1. 单帧层面：NSFW 多分类模型（业界基线）
- **GantMan/nsfw_model**（[github.com/GantMan/nsfw_model](https://github.com/gantman/nsfw_model)）：
  60+GB 数据训练的 5 分类 InceptionV3，精度 93%，类别为
  `Drawing / Hentai / Neutral / Porn / Sexy`。它正是 [nsfwjs](https://github.com/infinitered/nsfwjs)
  背后的模型，也是 Yahoo/LAION 等 NSFW 过滤系统的血统来源。
- 本工具直接使用与该模型同源的内置权重（**离线、内置 ONNX**），
  `Porn` 权重最高、`Hentai` 次之（同属显性内容）、`Sexy` 再次、`Drawing` 仅微量计权。

### 2. 视频层面（重点）：关键帧聚合 + 时间维度
学术界对色情视频的经典做法（[LSPD 数据集论文](https://www.researchgate.net/publication/358734386_LSPD_A_Large-Scale_Pornographic_Dataset_for_Detection_and_Classification)、
NPDI-800/2k 数据集）：
1. **按 1fps 提取关键帧**（本工具默认上限 96 帧、约 2fps 采样）；
2. 逐帧做显性分类；
3. **显性帧数超过阈值 → 视频判为色情**；LSPD 还发现"≥3 个**连续**关键帧显性"
   是最稳健的判据（误报率最低）。

而唤起度研究（[AGAIN 数据集](https://arxiv.org/abs/2104.02643)，IEEE TAC 2022；
VEATIC、AVDOS 等情感视频库）表明：**人的唤起是随时间连续变化的信号**，
高质量内容的特征是"高唤起、持续、有峰值"。

综合两条线，本工具对视频按**每帧色情分的时间序列**聚合：

| 分量 | 定义 | 权重 | 对应研究依据 |
|---|---|---|---|
| 强度 intensity | 帧分的 p90（避免单帧离群值） | 0.40 | 唤起峰值（AGAIN/VEATIC 的 peak arousal） |
| 覆盖度 coverage | 帧分 ≥ 阈值的帧占比（默认阈值 40） | 0.30 | 显性帧占比（NPDI/LSPD 关键帧判据） |
| 持续段 sustained | 最长连续"高分段"占全片比例 | 0.20 | LSPD "连续关键帧"稳健性判据 |
| 特写 closeup | 人脸占据画面较大比例的帧占比（Haar 检测） | 0.10 | 面部特写与唤起正相关的启发式先验 |

再乘**制作质量系数**：
- 时长 <3s ×0.75；<10s ×0.90（过短片缺乏"铺垫-峰值"结构）；
- 高度 <240p ×0.90；<360p ×0.95（低清降低视觉吸引力）。

### 3. 图片层面
直接按 5 类概率加权和：`100 × (1.0·P + 0.85·H + 0.60·S + 0.15·D)`，权重可用 `--weights` 调整。

### 4. 已知局限（务必知晓）
- NSFW 分类器存在**偏见**（[FAccT'24 审计](https://facctconference.org/static/papers24/facct24-78.pdf)：
  对女性/特定肤色的误判率更高），分数只是启发式排序，不是绝对真理；
- 模型只"看懂"画面，不懂**情节、剪辑节奏、声音**，对剧情向内容会低估；
- 分数 ≠ 质量 ≠ 合法性。它只是一个**粗排索引**。

---

## 二、安装（= 解压）

**无需安装任何运行时**。`runtime/`（自包含 Python 3.13 + OpenCV/numpy/ONNX Runtime）
与 `models/`（两个 ONNX 分类模型 + Haar 人脸级联）均已内置，任意目录解压即用：

- 不联网、不下载模型、不写注册表、不装服务；
- 支持 Windows 11 x64（Windows 10 1809+ x64 亦可）；
- 中文/非 ASCII 路径已实测兼容（OpenCV 的已知坑已全部绕开，见下文）。

### 环境体检（推荐新机器第一步）

```bat
pornscore.bat --check
```

检查项：自包含 Python 版本、OpenCV/numpy、onnxruntime、两个 ONNX 模型（各跑一次
空张量推理）、推理模块自检（内置测试图 → 5 类概率）、Haar 级联、PYTHONPATH 污染；
每项失败都会给出具体的修复提示，全部通过才可运行。

### 重建 runtime（仅当 `runtime/` 丢失/损坏时，需联网）

```bat
powershell -ExecutionPolicy Bypass -File build\build.ps1
```

`build.ps1` 下载官方 Python 3.13.2 完整版 → 组装 embed 布局 → 按 `requirements.txt`
锁定版本安装依赖（约 200MB 下载，仅构建期联网）。

### ⚠️ Windows 常见坑：PYTHONPATH 被第三方工具污染

SVP/mpv 等播放器常把含 `python3xx.dll` 的目录写进系统 `PYTHONPATH`，与当前解释器
冲突，导入任何 C 扩展都会崩溃（报 `Module use of python3xx.dll conflicts with
this version of Python`）。**请一律用 `pornscore.bat` 启动**——它会先清空 PYTHONPATH
并把控制台切到 UTF-8（chcp 65001），任何 Win10/11 机器都可直接运行；
`--check` 也会探测此类污染并告警。

### ⚠️ Windows 非 ASCII 路径

OpenCV 4.x 的 C++ 文件 I/O（`imread`/`imwrite`/CascadeClassifier）在中文路径下不可靠
（实测 `imwrite` 甚至会损坏文件名）。本工具已全部绕开：图片用 `np.fromfile+imdecode`，
人脸级联先复制到 ASCII 临时目录再加载；视频读取（VideoCapture）经测试支持中文路径。

---

## 三、使用

```bat
rem 环境体检（新机器/迁移后建议先跑）
pornscore.bat --check

rem 基本：扫描目录，打印 Top20 + 生成报告
pornscore.bat D:\videos

rem 高精度模型（InceptionV3，约 6 倍慢，更准）
pornscore.bat D:\videos --model inception

rem 只挑高分并复制到"精选"文件夹
pornscore.bat D:\media --min-score 60 --copy-top 50 --copy-to D:\精选

rem 指定扩展名、输出目录、JSON
pornscore.bat D:\pics --exts jpg,png,webp,mp4,mkv --out D:\report --json D:\report\result.json

rem 调权重（让你偏好"sexy"风而非纯 porn）
pornscore.bat D:\videos --weights porn=0.8,sexy=0.9
```

### GUI（图形界面，推荐）

```bat
pornscore_gui.bat
```

- **左侧**：选择输入（文件夹/文件可多选混合）、报告输出目录、可选的精选输出目录
  （Top N + 复制最低分）、模型/抽帧数/覆盖阈值/四类权重；
- **右侧**：评分过程中实时刷新的结果列表 —— 等级（彩色行）、分数、类型、文件名、
  详情（时长/分辨率/帧数/强度/覆盖/持续/特写），双击行直接打开文件；
- 完成后自动在报告目录生成 CSV/HTML 报告，并按需把 Top N 复制到精选目录
  （"无法判断"的文件不参与精选复制）；
- **高分屏支持**：自动启用 Windows 高 DPI 感知，4K/2K 高缩放屏上窗口与字体清晰不糊；
- **悬浮提示**：左侧每个选项（输入/目录/Top N/最低分/模型/帧数/阈值/权重/按钮）
  鼠标移入即显示详细说明；
- **结果列表排序**：点击列标题（等级/分数/类型/文件名/修改日期）多级排序——
  第 1 次=正序（最优在前：分数高→低、等级夯→拉完了、日期新→旧、文本 A→Z），
  第 2 次=倒序，第 3 次=移除该条件；可叠加多列，先点的列为主排序、后点为次级排序
  （标题上的 ↑↓①② 标注当前条件与优先级）；全部移除后恢复默认顺序（按文件名，
  不分文件类型）。
- **选中态**：点击空白处（列表外任意位置，或列表内行下方/边缘空区）取消选中行，
  恢复行的等级颜色。
- **事后重新输出精选**：评分完成后，随时可修改 Top N / 复制最低分 / 精选输出目录，
  点"📁 按当前参数输出精选"按当前参数重新复制（无需重新评分）。
- **精选参数空值容错**：Top N / 复制最低分 留空时，点任意空白处或点"开始评分/输出精选"
  会自动填入默认 0；Top N = 0 表示不输出精选。

**支持的格式**：图片 `jpg jpeg jpe jfif png webp bmp gif tiff`；视频
`mp4 mkv avi webm mov flv m4v ts mts m2ts wmv mpg mpeg 3gp`。
`.jfif`/`.jpe` 等异常扩展名按 JPEG 内容识别（下载图片常见 `.jfif`）；
个别编码不支持的视频会记为"处理失败"并给出原因，不影响其他文件。

**五档等级（非均匀阈值）+ 无法判断：**

| 等级 | 分数区间 | 含义 |
|---|---|---|
| 夯 | ≥ 88 | 天花板、神作、强得离谱 |
| 顶级 | 72 – 88 | 很强，非常优秀 |
| 人上人 | 55 – 72 | 超出预期，中上水平 |
| NPC | 35 – 55 | 中规中矩，路人感 |
| 拉完了 | < 35 | 拉胯、差劲、让人失望 |
| 无法判断 | 不按分数，按信号触发 | 需人工观阅：模型最大类概率 < 40%，或视频有效帧 < 4/时长未知，或分数落在 35–55 模糊带且覆盖度 5%–40%（边界信号） |

> 阈值依据（启发式）：单张纯 Porn 帧（P≈0.9+）才逼近 90+；视频经时间聚合后真正
> 显性的好片一般落在 70–90；30 分以下基本是 SFW/中性。定义见 `app/scoring.py`。

### 输出
- 控制台：Top-N 排行（含等级列）
- `<out>/pornscore_report.csv`：全量明细（UTF-8 BOM，Excel 可直接打开）
- `<out>/pornscore_report.html`：**可视化报告**（浏览器打开，图片内嵌缩略图，
  视频带峰值帧海报 + 可播放预览）
- `--copy-top N`：把 Top N 文件复制到 `--copy-to` 目录（默认 `<out>/selected`）

### 主要参数
| 参数 | 默认 | 说明 |
|---|---|---|
| `--model` | mobilenet | `mobilenet`(快) / `inception`(准、约 6 倍慢) |
| `--max-frames` | 96 | 每视频最多采样帧数 |
| `--coverage-thresh` | 40 | 视频覆盖度/持续段的帧分阈值 |
| `--weights` | 见上文 | 类别权重，如 `porn=1.0,sexy=0.6` |
| `--min-score` | 0 | 复制筛选的最低分 |
| `--exclude` | 内置 | 追加排除 glob（目录名或文件路径模式） |
| `--images-only` / `--videos-only` | | 只处理某一类 |

### 性能参考（典型桌面 CPU，MobileNet + ONNX Runtime CPU）
- 图片：约 0.02–0.06 s/张
- 视频：约 2–10 s/部（96 帧），Inception 模型 ×6

### 自测（测试套件零额外依赖）

测试代码/素材内置于 `pornscore_test/`（`testtool/` 测试代码 + `_testdata/` 冒烟样本）。
素材均为程序生成的中性图像/视频（无人物内容），stdlib unittest 实现：

```bat
rem 单元/集成/端到端
pornscore_test\testtool\run_tests.bat
rem 只跑纯函数单测（<1s）
pornscore_test\testtool\run_tests.bat test_pure
rem GUI 冒烟（用内置 runtime 的 python 运行）
runtime\python.exe pornscore_test\gui_smoke.py
runtime\python.exe pornscore_test\sort_smoke.py
runtime\python.exe pornscore_test\export_smoke.py
```

覆盖：评分函数/权重解析/排除规则/视频时间序列聚合、ONNX 推理模块等价契约
（加载/尺寸/灰图数值合理/批处理顺序/坏帧不崩）、完整 CLI 流程（中文路径扫描、
坏文件降级、CSV/HTML/JSON 报告、`--copy-top`/`--min-score`、`--check`）。

---

## 四、文件结构

```
PornScore/
├─ pornscore.py        # CLI 入口（薄封装）；同时保留旧模块 API 供测试使用
├─ app.py              # GUI 入口（tkinter，无第三方 GUI 依赖）
├─ app/                # 核心包（分层设计，新功能加在这里）
│  ├─ scoring.py       #   分数→五档等级（非均匀阈值）/ 权重解析 / 视频聚合纯函数
│  ├─ infer.py         #   ONNX Runtime 推理客户端（uint8 输入，归一化在图内）
│  ├─ engine.py        #   扫描 / 抽帧 / 人脸检测 / 回调式批处理评分（可停止）/ 精选复制
│  ├─ report.py        #   CSV / HTML（内嵌缩略图+视频海报）/ JSON 报告
│  ├─ cli.py           #   命令行接口（v1 全部参数保留）
│  ├─ doctor.py        #   环境体检（--check）/ 运行时预检
│  └─ ui.py            #   GUI（左参数右实时列表，事件队列线程模型）
├─ models/
│  ├─ mobilenet_v2.onnx                  # 224×224，5 分类（uint8 输入，图内 /255）
│  ├─ inception_v3.onnx                  # 299×299，5 分类（同上）
│  └─ haarcascade_frontalface_default.xml   # 人脸级联（内置，绕开 OpenCV 中文路径问题）
├─ runtime/            # 自包含 Python 3.13（解释器+标准库+tcl/tk+opencv/numpy/onnxruntime，解压即用）
├─ pornscore.bat       # CLI 启动器（清空 PYTHONPATH + UTF-8 控制台 + tcl/tk 指向）
├─ pornscore_gui.bat   # GUI 启动器（同上）
├─ requirements.txt    # Python 依赖锁定清单（重建 runtime 时用）
├─ build/              # 构建工具（开发用：build.ps1 重建 runtime / convert_models.py 重转模型）
├─ pornscore_test/     # 回归测试套件 + GUI 冒烟 + 中性测试素材
└─ LICENSE             # ISC
```

---

## 五、工作原理速览

1. Python 递归扫描媒体文件（自动排除系统/回收站目录、报告目录）。
2. ONNX Runtime（CPU）加载内置 5 分类模型（MobileNetV2 224 / InceptionV3 299）。
3. 图片：解码 → 224×224 原始 RGB 字节 → 图内 Cast+Div(255) 归一化 → 推理 → 加权得分。
4. 视频：OpenCV 均匀抽帧（≤96 帧）→ 每帧 Haar 人脸特写检测 + 推理
   → 时间聚合（强度/覆盖/持续/特写）× 质量系数 → 得分；峰值帧存为海报。
5. 输出 CSV/HTML/JSON，可选复制 Top-N。

## 六、许可

[MIT](LICENSE)。
使用即代表你已知晓并同意文首的**安全与法律声明**。
