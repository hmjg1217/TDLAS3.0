# TDLAS 测氢系统 V3.2.2 — 开发与维护技术指南

> 文档基线：2026-11-12，Asia/Shanghai。适用目录：`D:\V3.1`。本文面向维护人员与编码代理；产品介绍和操作说明见同目录 `README.md`。
> 本文根据当前独立源码、构建脚本和交付验证记录编写。本文的编写不表示重新执行了编译、CTest 或真实硬件验收。

## 1. 工作区与版本事实

本工作区是交付目录，不能直接当作 CMake 源码根目录。

| 位置（相对 D:\V3.1） | 定位 |
| --- | --- |
| `源码包/tdlas-hydrogen-ui-3.2.2-source/` | 当前唯一正式源码根目录，下文简称 SOURCE |
| `发布包/tdlas-hydrogen-ui-3.2.2-win64/` | 当前正式便携运行目录，下文简称 PACKAGE |
| `源码包/tdlas-hydrogen-ui-3.2.2-source.zip` | 独立源码交付压缩包 |
| `发布包/tdlas-hydrogen-ui-3.2.2-win64.zip` | Windows x64 便携交付压缩包 |
| 两类目录中的 `SHA256SUMS.txt` | 对应 ZIP 的交付校验文件；包内另有运行文件校验清单 |
| `待删除旧文件/` | 旧版本、历史工作区、备份及临时产物；不作为开发入口 |

- CMake 工程：`tdlas_hydrogen_ui_suite`，`VERSION 3.2.2`，C++17。
- 唯一正式应用：`apps/integrated` → CMake target `tdlas_hydrogen_ui` → `tdlas_hydrogen_ui.exe`。
- `startup`、`admin`、`client` 的独立可执行目标用于 UI 预览和截图。它们的窗口源码、QSS 和资源同时被正式应用复用，不能因其名称带 Template 就删除。
- 当前公共数据合同版本是 **1.7**，定义在 `core/contracts/MeasurementTypes.h`。应用版本、合同版本、参数集 ID、算法版本、频率标定版本分别管理，禁止相互替换。
- 正式 UI 参数集为 `tdlas-v1.6-ui`，算法版本为 `tdlas-v1.8-20-cycle-average-cpp`。名称包含历史版本号不等于交付版本错误。
- 不把历史 `N.UI`、`camera-work`、`dist` 或 `build-codex-ui` 绝对路径当作当前入口，不使用旧工作区覆盖正式源码。
- 阅读 SOURCE 下现有 `AGENTS.md`；发生历史说明冲突时，核对当前代码、最新明确决定及验证证据，不根据旧阶段记录撤销已实现功能。

## 2. 开始任务前阅读什么

路径均相对于 SOURCE。

| 文件 | 用途与注意事项 |
| --- | --- |
| `CMakeLists.txt`、`apps/integrated/CMakeLists.txt` | 真实版本、依赖、源码组合、目标和测试注册 |
| `docs/PROJECT_STATUS.md` | 开发轨迹与最新状态；包含被后续记录覆盖的历史行为 |
| `docs/PROJECT_LAYOUT.md` | 当前交付目录布局 |
| `docs/Software_Architecture.md` | 分层和职责边界 |
| `docs/Interface_Definition.md` | 接口设计；与头文件交叉核对版本 |
| `docs/Module_Development_Plan.md` | 模块与阶段计划，不等于当前完成情况 |
| `docs/OPEN_QUESTIONS.md` | 未决问题及验收输入，部分临时策略已演进，不能整表视为实时实现清单 |
| `docs/RELEASE_NOTES_V3.2.2.md` | 本版 PTZ 修正 |
| `docs/DATA_SAVE_WORKFLOW.md` | 保存流程概述；目录名与实际写入行为以控制器和存储代码核对 |
| `docs/validation/DELIVERY_VALIDATION.md` | 本次交付的已有构建与便携验证证据 |
| `docs/validation/CAMERA_CONTROLS_PREVIOUS_REPORT.md`、`docs/CAMERA_MANUAL_REVIEW.md` | 已有相机验证及人工复验边界 |
| `docs/ui/`、`apps/*/docs/ui/`、`apps/*/docs/ui-specs/` | UI 规格、设计令牌与参考清单 |
| `docs/reference/` | 原始算法、技术参数、MATLAB 参考及硬件资料 |

判断顺序：用户当前明确要求 → 当前实现及可复现结果 → 最新已批准的决定/验证记录 → 历史计划和参考资料。发现不一致应说明证据，不能为“统一文档”而修改算法或设备参数。没有依据的公式、单位、设备能力和验收结论保持待确认。

### 2.1 外部参考资料来源与现有副本

2026-11-12 补充核查了 `C:\Users\91463\Documents\ChatGPT\N.UI`。该目录顶层的以下 **6 份参考文件**，均已收录在 SOURCE 的 `docs/reference/`，逐文件 SHA-256 对比完全相同，包括相机 PDF。无需再复制一套，也不应把外部目录设为编译、运行或资料查阅的必要依赖。

| 已随源码保存的资料（相对于 SOURCE） | 应参考的内容 | 使用边界 |
| --- | --- | --- |
| `docs/reference/TDLAS_模块化开发与增量集成规范_V1.0.md` | 单一主工程、统一接口、七类功能职责、模块测试和逐步集成规则 | 属于开发方法与历史路线图；候选接口、目录和阶段编号不覆盖当前实现 |
| `docs/reference/技术参数.txt` | 电压/吸光度单位、NI 接入背景、AO 峰值及软件边界、标定缺口 | 包含旧工程的存储、录像、权限与设备记录，不作为 V3.2.2 功能清单 |
| `docs/reference/算法模块说明.md` | 旧 v1.6 算法来源、处理步骤、Voigt 参数及参考配置解释 | `TDLAS::Algorithm`、`tdlas::TDLASProcessor`、旧头文件路径和安装方式不是本工程公开 API |
| `docs/reference/测氢算法.txt` | MATLAB 原始处理流程、多项式、周期/光谱平均、光强估计与拟合参考 | 固定路径、索引、窗口和保存片段仅供追溯；不是现场配置或浓度标定证明 |
| `docs/reference/vvoigtshape.m` | Voigt 四项有理近似公式、系数和参数顺序 | 不包含完整优化器、设备配置或浓度反演；验证公式与验证最终算法结果分开处理 |
| `docs/reference/索尼IMX415摄像头模组.pdf` | 相机模组硬件背景，设备接入前按需查阅原文 | 模组规格不等于当前 USB 桥接/驱动的可用能力，不据此承诺实际分辨率、帧率或 PTZ |

外部目录还包含 `TDLAS-Hydrogen-UI-Suite/` 和 `.git`；其 CMake 工程版本为 **3.0.0**。它是历史工程与版本线索，不是当前 V3.2.2 源码入口，不得整目录覆盖 SOURCE，也不得把其中的 Git 状态当作当前交付源码的状态。

后续若外部参考文件发生变化，先按文件比较差异，记录来源和适用版本，再判断是否更新 `docs/reference/` 及本文；不要自动同步整个历史工程。

### 2.2 历史资料与 V3.2.2 的差异处理

| 旧资料中的描述 | 当前维护时采用的事实 |
| --- | --- |
| 照片 PNG/JPG/BMP、录像 MP4/H.264、目标 20 fps | 当前媒体输出是 PNG 与逐帧 JPEG 的 MJPEG；不能将历史编码和帧率承诺带入本版 |
| 没有应用内账号或角色权限 | 当前有管理员/客户端路由、应用授权和审计边界；生产账号系统仍未接入 |
| 只保存时间/峰值/光强 CSV，没有完整波形或元数据 | 当前同时有完整实验归档、附加 TXT/CSV 和摄像索引，具体路径见第 10 节 |
| 算法不推断周期，输入必须预先按相位对齐 | 当前正式 UI 算法跨块识别锯齿边界，组装连续 20 个完整周期后平均拟合 |
| `Sample=100000`、`single_fre=200`、`num1=500` | 这组变量用于历史参考窗口划分；`Sample` 在该文本中是点数，不能直接作为现场采样率或输出频率配置 |
| `Fs=1/Sample` 和 `Fc=Fs/20` | 用于追溯原文的归一化滤波参数；不能把这里的变量名当成真实 DAQ 采样时钟 |
| 旧用户名路径、MATLAB addpath 与固定 G/E 盘保存路径 | 不属于本版路径配置，不复制进运行配置、源码接口或操作说明 |
| 曾识别 NI PCIe-6363、列出 IMX415 模组资料 | 仅为设备背景，实际设备、通道、格式与范围由目标机枚举和现场确认 |

`vvoigtshape.m` 的历史中文注释可能因编码方式显示异常；需要阅读时先识别原编码，不为注释显示问题批量转码或改动公式。原始资料可保留历史描述，当前结论应在版本化说明中解释，避免改写原文后失去来源。

### 2.3 将模块化规范落到当前工程

原规范中的七类职责对应当前 Parameter、Acquisition、Algorithm、Display/Presentation、Camera、Storage、System。Display/Presentation 主要由展示模型、应用层映射和各窗口/ChartWidget 承担，不要求为其再建一个空库或独立程序。

新增模块或较大功能前，在任务说明或相应设计文档中写清：职责、输入、输出、对外接口、依赖、线程、错误处理、测试方法、修改范围和验收标准。完成一个可验证的小闭环即接入正式 integrated 应用并回归，避免模块全部完成后才首次整合。

原规范的“等待用户验收，不自动进入下一阶段”用于约束未授权的里程碑扩展：完成用户已授权任务中的实现与验证，再报告验收结果；用户已授权的必要步骤无需反复请求确认。真实设备操作仍受当前授权范围约束。历史阶段模板不要求重新从架构准备或模拟骨架阶段开始。

## 3. 技术栈和构建依赖

- 平台：当前交付与验证为 Windows 10/11 x64。
- 语言与构建：C++17、CMake 3.21+、Visual Studio 2022/MSVC v143 x64、Windows SDK。
- 已验证的交付构建组合：Qt 6.8.3 `msvc2022_64`、CMake 3.30.5、MSVC 2022 x64 Release。
- Qt 模块：Core、Gui、Widgets、Multimedia；CMake 启用 AUTOMOC 和 AUTORCC。
- Windows 相机控制链接 `strmiids`、`ole32`、`oleaut32`。NI 适配器运行时加载 `nicaiu.dll`，不以硬编码驱动路径替代发现逻辑。
- 构建与打包脚本使用 PowerShell，开发建议 PowerShell 7。运行发布包不要求安装 Qt SDK、编译器、CMake、Python 或 PowerShell 7。
- 包内运行库不替代系统驱动安装。NI-DAQmx、摄像头/UVC 或厂商驱动按目标机器配置。

## 4. 源码导航与修改落点

| 路径 | 职责 / 主要入口 |
| --- | --- |
| `apps/integrated/src/main.cpp` | QApplication、便携环境初始化、版本与命令行、控制器入口 |
| `apps/integrated/src/PortableEnvironment.h` | 读取 `portable-config/runtime.ini`，将相对路径解析到 EXE 目录 |
| `apps/integrated/src/IntegratedApplicationController.*` | 服务装配、登录角色路由、权限门禁、UI 意图绑定、参数持久化、附加导出 |
| `apps/integrated/src/TemporaryCredentialAuthenticator.*` | 临时开发凭据路由，非生产账号系统 |
| `apps/startup/` | 登录窗口、样式和资源 |
| `apps/admin/` | 管理员主页面、调试、峰值数据、摄像与图表 |
| `apps/client/` | 客户端主页面、参数、摄像与图表 |
| `core/contracts/` | RunId、ExperimentId、参数快照、原始块、结果、有效性、错误和展示模型 |
| `core/parameter/` | 参数合法性校验、提交、快照与 SHA-256 |
| `core/acquisition/` | 采集接口、设备发现、mock、混合路由、NI 驱动及缓冲工具 |
| `core/algorithm/` | 流式周期识别、平均、滤波、基线、吸光度、Voigt 与参考算法 |
| `core/application/` | 测量状态机、应用协调、算法 worker、存储 worker、峰值构建及归档应用门禁 |
| `core/storage/` | 文件 repository、配置、归档检查/回放/导出/显式恢复、峰值 CSV 导出 |
| `core/camera/` | 相机接口、Qt Multimedia/mock、Windows PTZ、相机 worker、媒体与预览状态 |
| `core/system/` | SessionContext、授权策略、审计接口及文件审计 |
| `shared/resources/` | 共享侧栏图标与 qrc |
| `tests/`、`apps/integrated/tests/` | 核心/集成/性能/便携/相机测试及人工复验工具 |
| `tests/golden/tdlas-algorithm-v3.1-p0.json` | 固定算法回归金标，不随实现变化自动重写 |
| `tools/` | 工具链发现、截图、源码打包、发布校验等脚本 |

主要链接关系：`tdlas_measurement_core` 依赖 Qt Core；`tdlas_camera_core` 依赖测量核心与 Qt Core/Gui/Multimedia；`tdlas_integrated_core` 组合控制器和三类窗口；正式 EXE 组合核心与资源。

## 5. 分层与线程约束

1. 窗口收集 typed intent 并展示 display model。采集调度、权限判断、实验关联和算法决策放在应用服务或控制器，不能靠隐藏按钮替代授权。
2. 算法不得依赖 Widgets、NI 驱动或摄像头。设备特定逻辑通过 `IAcquisitionDevice`、`INidaqmxDriver`、`ICameraDevice` 等端口隔离。
3. 当前算法由独立 `AlgorithmWorker` 顺序处理，完整实验存储由独立 `StorageWorker` 顺序处理，相机设备/编码/媒体 I/O 由独立相机 worker 处理。
4. 核对真实对象装配和线程归属，不因存在 `ThreadedAcquisitionDevice` 文件就断言正式路径已使用该包装器。NI 调用阻塞性、停止时限和目标机吞吐仍需要专项确认。
5. 跨线程载荷保留运行 ID、序号、时间戳、有效性与错误；不得传递生命周期不明确的裸帧指针或 UI 控件。
6. 运行切换、停止、故障和析构必须清除待发任务并拒绝旧运行迟到结果。正常停止应排空已接收的处理/存储工作，随后完成实验清单。
7. UI 刷新默认 33 ms，原始波形可以先于完整处理结果展示；刷新节流不能改变算法窗口或覆盖完整存储数据。
8. 新增持续 I/O 不进入 UI 线程。当前控制器中的附加 TXT/流式 CSV 仍同步写入并 flush，这是现有实现的限制，不能描述为全部异步或已证明慢盘下不卡顿。

## 6. 测量流程、合同与状态

标准流程：登录生成 session → 参数校验/提交 → 得到不可变 snapshot → 开始测量建立 run/experiment → 采集 RawDataBlock → 算法 worker → ProcessingResult → 同源 PeakRecord/展示模型 → 存储 worker → 停止排空与归档结束。

- `MeasurementState`：`Idle → Preparing → Running → Stopping → Idle`；故障进入 `Faulted`，使用显式 `resetFault()` 清理后回到 Idle。不要直接绕过状态机修改展示状态。
- `RunId` 标识一次测量运行；`ExperimentId` 标识实验，格式由 `createExperimentId()` 生成；`Sequence` 在运行内递增。运行内去重和跨运行去重必须区分。
- `RawDataBlock` 按 channel series 保存，每通道 `samplesV`；保留采样率、每通道点数、UTC/单调时间、overrun 与 droppedBefore。
- `ParameterSnapshot` 保留 ID、revision、创建 UTC、哈希、AI/AO 配置和算法参数集；不在已启动运行中偷偷替换快照。
- `ProcessingResult` 保留完整平均周期、裁剪时轴、频率轴、基线、电压/拟合电压、吸光度/拟合吸光度、Voigt 参数、质量和来源。
- `DataValidity` 有 `Valid`、`Invalid`、`Incomplete`、`Stale`、`Uncalibrated`；数值为 0 不等于有效，空字段不等于 0。
- 电压用 V，时间轴用 s，采样率/输出频率用 Hz，相对频率用 cm⁻¹，吸光度无量纲，光强展示为百分比。
- 浓度 `concentration` 保持空、单位空、未标定语义。不得把峰值、吸光度或光强直接命名为 ppm/% 氢浓度。
- 错误用 `ModuleError` 传播，保持稳定 code、category、severity、用户信息、诊断、时间和停止/重试标志；不得吞错后伪造正常结果。

## 7. 算法维护规则

### 7.1 正式 UI 行为

- 依据真实锯齿边界跨采集块识别连续 **20 个完整周期**，先按相位对齐平均，再拟合并同步发布。不能恢复成逐周期发布，也不能把一个 read block 当成一个周期。
- 周期长度变化按相位线性插值；显示时间轴保留平均周期时长。run、snapshot、采样率变化、序号不连续、overrun 或丢块时重置未完成窗口。
- 主页面、调试页、FV 和峰值必须消费同一组结果。原始实时采集块与处理组的平均电压具有不同更新节奏，应保留其来源差异。
- `cycleVoltageV` 保存完整平均周期，`timeSeconds` 对应裁剪处理区间；`displayVoltageOffsetV` 用于与原始 AI 电压叠加，不能把显示偏移当成设备电压修正。

### 7.2 参考配置与数值边界

- `tdlas-v1.6-reference` 默认 500 点/周期、10 周期/光谱、20 光谱，即 100000 点参考窗口，与正式 UI 的 20 周期组窗分别回归。
- 现有处理包含四阶 Butterworth 前后向低通、最高点对齐、4.2 V 内部偏置、30%～90% 截取、首 35%/末 40% 区间四阶多项式基线、`ln(baseline/voltage)`、固定相对频率映射及 Voigt 拟合。
- 固定多项式、`0.8761/30` 比例和频率标定版本来自当前参考配置，不是对任意硬件/光路的通用标定。
- Voigt 使用 `vvoigtshape.m` 对应的四项有理近似及有界搜索；当前 C++ 优化器不能直接宣称与 MATLAB `lsqcurvefit` 等价。
- 更改滤波、截取、对齐、偏置、拟合初值/边界、频率系数或光强公式，必须说明数值影响，更新相应算法版本/文档，并使用固定数据与金标验证。
- 禁止通过修改金标、反转符号、倒置坐标、夹紧异常结果等手段掩盖算法偏差。新增/替换金标需要明确参考来源与容差。

## 8. 采集参数与 NI-DAQmx

- `ParameterDraft` → `ParameterService::validate/commit` → `ParameterSnapshot` 是配置入口；控制器负责 INI 恢复与保存。
- 必填设备和至少一个不重复的非空 AI 通道；采样率为正有限数，块点数为正，输入量程上下限有序。
- AO 启用时必须有通道，频率为正，峰值幅值非负且有限，偏移有限；输出采样率至少为频率的 10 倍；`offset ± amplitudePeak` 必须位于 `[-10 V,+10 V]` 软件边界。
- AO 幅值使用峰值，不得默认为峰峰值。软件校验通过不代表现场设备量程、接线或安全值已批准。
- `HybridAcquisitionDevice` 按策略和明确 device ID 路由；`auto` 可发现真实设备并优先选择真实 NI，也允许显式模拟选择；不把“auto”解释为真机故障时静默制造模拟结果。
- NI 适配器动态加载驱动、创建/配置 AI 与可选 AO，读取实际采样率并产生电压块；配置失败、超时、停止、断开、故障及析构路径必须停止并清除任务。
- 未获任务范围内明确实机授权时使用 mock/注入驱动。现场型号、terminal configuration、AI/AO 同步触发、周期相位、拔出行为、停止时限和 AO 安全输出仍不能仅凭测试替身证明。

## 9. 相机与 V3.2.2 的关键约束

- 默认 `system` 使用 Qt Multimedia 枚举系统/UVC 输入；未指定设备 ID 时，由操作员明确选相机后连接。mock 仅用于显式模拟。
- 左右/上下对应设备 Pan/Tilt，放大/缩小对应设备 Zoom；Windows 适配器按设备实例匹配并执行写入与回读验证。
- 使用驱动实际范围和步长。控制不可用、到达边界或请求失败应反馈并禁用相应操作；不能静默回退软件平移/缩放。
- Zoom 显示的是驱动原始值，不能擅自换算成倍数，更不能据此宣称光学变焦或机械云台。
- 仅顺时针 90° 为软件预览旋转；四次回到原方向。原始 PNG/MJPEG 不叠加该预览旋转。
- 复位恢复驱动 PTZ 默认值并清除旋转，不是恢复启动时取景，不重置曝光或自动对焦。
- 设备控制在相机 worker；长按限频并合并最新意图，切换会话、停止和故障清除待发控制，避免松手后长时间执行旧请求。
- 设备 PTZ 改变源视频，因此照片/录像自然反映设备取景。预览缩小副本不能替代原始媒体帧。
- 拍照/录像要求 Running 实验和合法实验 ID。图片为 PNG，录像是逐帧 JPEG 的 `.mjpeg` 流，不是 MP4/H.264。
- 正常停止提交媒体；断线/写失败取消未完成临时录像并明确报错；关闭时停录像、停预览、断设备并等待 worker。
- V3.2.2 已有 `4K ZOOM CAMERA` 两角色按钮写入/回读记录及用户“取景位移、缩放和旋转正常”反馈。该证据不覆盖所有摄像头、长期录像或全部硬件性能。

## 10. 存储路径与实际实现差异

### 10.1 完整实验归档

`FileStorageRepository` 在控制器构造时获取存储配置，通常使用 PACKAGE 下 `data/`，按 UTC 保存：

```text
<archive-root>/yyyy/MM/dd/<experiment-id>/
  manifest.json
  parameters/parameter-r<revision>-<hash>.json
  raw/raw-<sequence>.csv
  results/result-<sequence>.json
  logs/events.jsonl
```

参数、原始块和结果关联 run/experiment；数据文件采用 `QSaveFile` 原子提交，日志为追加 JSONL。拒绝覆盖已存在实验或重复序号。正常停止排空后结束 manifest；写入失败显式进入错误路径。

系统审计在启动归档根的 `system-logs/` 下按月写入，不具备生产防篡改保证。

### 10.2 侧栏“数据保存”是另一条路径

当前 `chooseDataRoot` 创建以下实际名称：

```text
<operator-selected-root>/
  CSV数据保存/峰值数据-<local-timestamp>.csv
  算法处理数据/
  拍照摄像/                 # PNG、MJPEG、index.jsonl
  TXT/算法处理数据.txt
```

- 选目录后，附加 TXT 追加 `[时间戳,峰值吸光度,光强百分比,氢气浓度]`；没有标定时浓度字段为空。
- 管理员峰值页“导出 CSV”当前发出 toggle intent：第一次开启后续结果连续记录，第二次关闭。完整 PeakRecord CSV 导出函数另有实现，不能与四列流式 CSV 混为同一格式。
- `算法处理数据/` 当前创建目录，不代表完整结果 JSON 已写入这里。
- 选目录会更新相机存储根和控制器配置成员，**未重新配置已构造的 FileStorageRepository**；不能声称完整实验归档和系统审计随这个按钮迁移。
- 目录选择仅当前进程生效；参数 INI 的跨启动恢复与保存目录选择不是同一功能。
- 流式 CSV 以 sequence 判断增量，跨 run 重置、目录切换时已打开文件归属和慢盘同步 I/O 是后续维护重点，尚无充分依据宣称多次运行无遗漏。
- `docs/DATA_SAVE_WORKFLOW.md` 中的 `CSV` 简称和旧状态文档的 `<experiment>/media/` 不应覆盖当前代码的实际名称与媒体路径。

### 10.3 归档服务边界

`ExperimentArchiveService` 与应用门禁提供清单检查、原始回放、结果 CSV 导出、显式单目录恢复。恢复将遗留 running 实验按已有文件标记 aborted，不是继续采集。当前没有完整 UI 数据管理页、自动全目录崩溃扫描、跨进程锁、数据库或远程存储。新接入 UI 时使用专用 worker 并执行权限检查。

## 11. 配置与启动接口

便携模式以 EXE 同目录的 `portable-config/runtime.ini` 为标记，相对路径基于 EXE 所在目录解析，而非启动终端的工作目录。非空显式环境变量优先于该文件。

| 环境变量 | 作用 |
| --- | --- |
| `TDLAS_STORAGE_ROOT` | 完整实验归档根目录覆盖 |
| `TDLAS_STORAGE_CONFIG` | `storage.json` 配置文件路径覆盖 |
| `TDLAS_PARAMETER_CONFIG` | 参数 INI 路径覆盖 |
| `TDLAS_ACQUISITION_BACKEND` | `auto` / `mock` / `nidaqmx` |
| `TDLAS_CAMERA_BACKEND` | `system` / `mock` |
| `TDLAS_CAMERA_DEVICE_ID` | 明确指定初始系统相机；不设置时由用户选择 |

- `storage.json` schema 为 1.0；参数 INI `[measurement]` 的 `schemaVersion=1`。不承诺跨版本自动迁移。
- 源码构建输出若没有便携标记，配置/数据采用 `QStandardPaths` 默认路径；不能假设 build EXE 总往其旁边写数据。
- 原生 EXE 支持 `--help`、`--version`、`--capture <path>`。截图参数只保存登录页后退出。
- `--mock` 是 `启动便携版.cmd` 的参数，不是原生 EXE 的参数；该 CMD 不透传任意 EXE 参数。
- 源码中附带的启动脚本面向部署目录；先构建/部署，不能直接在 SOURCE 双击脚本期待存在正式 EXE。

## 12. 身份、权限与审计

- 临时管理员 `00/00`、客户端 `01/01` 定义在 `TemporaryCredentialAuthenticator`；只用于开发/演示路由。
- session 必须有合法 ID、principal、角色、签发/到期时间及 authority；过期或缺失会话拒绝操作。
- 管理员获当前所有受保护操作；客户端允许查看/开始/停止/复位、修改参数、保存实验、使用相机；客户端不允许 ConfigureStorage、ViewAuditLog、ManageAuthorization。
- 权限可用不等于窗口已提供对应页面。例如底层有审计查看操作名，不表示已实现完整审计管理界面。
- 保存目录选择当前受 SaveExperiment 门禁；归档恢复受 ConfigureStorage 门禁。新增操作须同步策略、调用点、审计与测试。
- 不将硬编码凭据、本地 session 和普通 JSONL 说成生产认证、安全账号库或防篡改系统。

## 13. 标准构建与运行命令

以下命令从 PowerShell 执行，先进入真实 SOURCE；Qt 路径按安装位置调整。

```powershell
Set-Location -LiteralPath 'D:\V3.1\源码包\tdlas-hydrogen-ui-3.2.2-source'

# 全量 Release 构建并执行全部已注册测试
.\build.ps1 -QtRoot 'C:\Qt\6.8.3\msvc2022_64' -RunTests

# 只编译正式应用（不等于构建全套测试）
.\build.ps1 -QtRoot 'C:\Qt\6.8.3\msvc2022_64' -Target integrated

# 对已有完整构建重新运行测试
& 'C:\Qt\Tools\CMake_64\bin\ctest.exe' --test-dir .\build -C Release --output-on-failure

# UI 预览截图；默认使用 build，输出到 artifacts/captures
.\tools\capture-all.ps1 -QtRoot 'C:\Qt\6.8.3\msvc2022_64'
```

`build.ps1` 通过 `tools/resolve-environment.ps1` 查找工具链，使用 VS 2022 x64 generator；支持 Configuration、BuildDirectory、Target、RunTests。`-RunTests` 必须配合默认 `-Target all`，不可与 `-Target integrated` 混用。默认正式产物是 `build/bin/Release/tdlas_hydrogen_ui.exe`。

新电脑使用新 build 目录，不复制原电脑的 CMake cache；不要为了路径问题批量升级 Qt 或更换 generator。构建成功不代表 build 目录已完成便携部署。

## 14. 测试与验收策略

当前默认全量 CMake 配置注册 **19 项 CTest**。不是每个 tests 下的 EXE 都注册为 CTest；人工硬件复验和存储吞吐工具单独使用。

| 关注点 | CTest 名称 |
| --- | --- |
| 核心、参数/状态与数据基础设施 | `tdlas_measurement_core`、`tdlas_data_infrastructure` |
| 周期、算法回归、金标 | `tdlas_single_cycle`、`tdlas_algorithm_regression`、`tdlas_algorithm_golden` |
| 峰值提取/导出 | `tdlas_peak_records` |
| 显示与性能基线 | `tdlas_display_performance`、`tdlas_performance_baseline` |
| NI 注入驱动 | `tdlas_nidaqmx_acquisition` |
| 相机核心与 PTZ | `tdlas_camera_core`、`tdlas_camera_hardware_controls` |
| 归档与稳定性 | `tdlas_experiment_archive`、`tdlas_system_stability` |
| 便携环境 | `tdlas_portable_environment` |
| 身份与登录路由 | `tdlas_authentication_routes`、`tdlas_login_window_routes` |
| 界面集成 | `tdlas_camera_view_interactions`、`tdlas_peak_page_interactions`、`tdlas_measurement_ui_flow` |

执行原则：

1. 根据修改范围先运行相关测试；共享合同、生命周期、算法或正式装配变化完成全套回归。
2. GUI CTest 的 offscreen 插件路径已在 CMake 设置；通过 offscreen 不等于 Windows qwindows 视觉验收。
3. UI 修改比较登录/管理员/客户端相关状态，记录客户区、DPI/DPR、Qt、字体、主题与测试数据；环境不一致只判断结构回归。
4. 算法修改比较完整数组、坐标、元数据和质量语义；性能优化同时守住数值金标。
5. 设备修改覆盖失败中途清理、超时、断线、重复启停、旧 run 迟到结果、驱动范围和默认值恢复。注入驱动通过不等于实机通过。
6. 保存修改覆盖权限、不可写/慢盘、重复序号、多次运行、目录切换、停止排空、空浓度和媒体实验关联。
7. 纯文档变更核对路径、命令、版本及事实即可，不为文档编写重复实现的自动测试。

已有交付记录：独立路径干净 Release 构建成功、19/19 CTest 通过（21.04 秒），部署依赖和中文/空格路径受限 PATH 启动验证通过。这些是 `DELIVERY_VALIDATION.md` 的历史结果；报告本次工作时必须另写自己实际执行的检查。

## 15. 部署、打包与交付

默认交付不包含三个预览程序。优先输出到新的目录供审查，避免覆盖正在使用的正式运行目录。

```powershell
Set-Location -LiteralPath 'D:\V3.1\源码包\tdlas-hydrogen-ui-3.2.2-source'

# 使用新目录生成便携包与 ZIP
.\deploy.ps1 -QtRoot 'C:\Qt\6.8.3\msvc2022_64' `
    -BuildDirectory build -PackageDirectory 'D:\V3.1\交付候选'

# 校验已有正式包，不启动采集
.\tools\verify-package.ps1 -PackagePath 'D:\V3.1\发布包\tdlas-hydrogen-ui-3.2.2-win64'

# 源码打包默认输出到 SOURCE 的上一级；需要覆盖既有 ZIP 前先核对输出路径
.\tools\package-source.ps1 -OutputPath 'D:\V3.1\交付候选\tdlas-hydrogen-ui-3.2.2-source.zip'
```

- `deploy.ps1` 处理隔离 staging、Qt/MSVC/多媒体运行库、Windows 平台启动冒烟、manifest、哈希和 ZIP。日常验证不要无依据加 `-SkipSmokeTest`。
- 保持 CMake、部署 ReleaseVersion、资源和说明版本一致；当前校验脚本也固定检查 3.2.2，未来升级须同步。
- 复制完整便携目录，不能只交 EXE。源码 ZIP 保留 apps/core/shared/tests/docs/tools、资源、脚本和金标。
- 不把实验数据、机器专属构建缓存、凭据、临时 QA 输出或旧版本打进源码包/发布包。
- 根目录新增本文和 README 不会自动更新现有源码 ZIP、发布 ZIP 或其中的文档；需要单独明确执行重新打包。
- 哈希清单证明交付文件完整性，不等于代码签名、安装器认证或另一台物理机器已验收。

## 16. 修改纪律与完成报告

- 修改之前确认目录、版本和用户已有改动；不创建嵌套仓库，不假设当前源码包自带 Git 元数据。
- 保留批准的布局、导航、QSS、资源和图表语义。没有对应需求时不重建主窗口、不批量改名、不开展无关重构或依赖升级。
- 只在用户任务范围内修改正式源码。不要直接手改发布 EXE、生成的 MOC/RCC、build cache 或清单来伪造一致性。
- 公共字段/语义变化同步头文件、序列化、读取兼容、显示映射、测试和接口文档；修改算法参数同时说明版本与适用范围。
- 不因目录名为“待删除旧文件”而自动删除。清理、覆盖现有交付、真实设备操作和发布行为按当前用户授权范围处理。
- 新增文档使用 UTF-8，命令注明工作目录；保留中文/空格路径兼容。产品说明避免承诺未实现的界面按钮或后台能力。
- 完成报告说明：修改文件、实际验证及结果、必要截图/日志、未执行项目、未关闭事项。分别报告软件回归、数值参考、真实设备、便携部署和目标机长时验收。

## 17. 尚待关闭的主要事项

1. 浓度反演、单位、量程、误差、标准样气、标定模型及批准输入/输出。
2. 同一批准数据集上的 MATLAB/C++ 对比与容差；固定频率映射和偏置对目标光路的验证。
3. NI 实机接线、terminal configuration、采样时钟/触发、AI/AO 相位与安全输出、设备异常及吞吐。
4. 生产身份来源、会话撤销、角色策略批准与审计保留/防篡改。
5. 目标磁盘持续吞吐、队列上限/背压、磁盘不足、跨进程写入及小时/日级稳定性。
6. 目标摄像头长时录像、格式/帧率/分辨率、热插拔、图像质量、播放器与目标机资源占用。
7. 目标电脑部署、杀毒/权限、字体/DPI、签名与升级回滚要求。
8. 本文第 10 节记录的保存路径、流式记录和同步 I/O 限制。

新增证据应更新相应状态和未决事项，不能把 mock、短时软件 soak 或单一相机反馈扩大为整机最终验收。
