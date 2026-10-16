# 笔记：把《脑部锻炼》Switch 版移植到 iPad 的思路（参考小红书 西门良）

> 个人学习笔记，不属于上游 d2r-mobile 项目文档。写于 2026-10-16，对照的代码是本 fork 的 `main`（v1.0.15，`b94670c`）。
> 文中「已核实」只指在本仓库代码里看到过；博主说的效果都是**博主自述，未核实**；标「猜测」的是推断，动手前要先验证。

## 0. 来源

- 作者：小红书 **西门良 Simon Liang**
- 标题：《我把一个小众游戏移植到iOS了》，发布于 2026-10-16
- 链接：https://www.xiaohongshu.com/explore/6ad21dbf0000000018004094
- 游戏：《大人的 Nintendo Switch 脑部锻炼》（川岛隆太博士监修，Switch 版）
- 作者**没有公开代码或仓库**。下面这些都是他的说法，我们没有核实：
  1. 让 AI agent 参考 @姜海舟 的 d2r-mobile 项目，先在 iOS 模拟器里跑。AI 要把渲染指令（NVN）一条一条对上再翻译，所以花了大半天才出画面。
  2. 一开始帧率很低，优化以后手机上流畅了。
  3. 他的 iPad 是 2018 款 iPad Pro（A12X），不支持 d2r-mobile 用的 Metal 4。AI 找了一套「降级渲染指令」，最后在 iPad 上跑到 60fps。
  4. 手写小游戏（算式填答案）可以用触屏玩（截图 3）。
  5. Joy-Con 红外摄像头的手势玩法（数手指、猜拳）改成用 iPad 前置的 Face ID 深度摄像头（TrueDepth）。截图 4 是低分辨率的假彩色手部深度图，上面盖着「5 = ✋ + ☝」这类题目。
  6. 接上了 iOS 原生的闹钟提醒（截图 5：通知写着「到时间了，可以开始脑部锻炼了。」）。
- 截图里还能看出两点：游戏是**竖着显示**的（Switch 上这个游戏要把主机竖过来玩），顶部有设置和手柄两个图标，看起来是借用了 d2r-mobile 的 App 外壳。

## 1. d2r-mobile 实际怎么工作（已核实，附文件路径）

| 层 | 做法 | 代码位置 |
|---|---|---|
| 改写主程序 | Switch 的 NSO（`main`/`sdk`/`subsdk0`）是 aarch64 代码，离线改写一次：`x18`、`tpidr_el0`/`tpidrro_el0` 改成读写线程槽；`svc #n` 和 `cntpct_el0` 改成跳到「宿主门」。每个游戏构建号有自己的 `CUTS`（逐条比对原字节的定点修改）和 `OBS`（观测门）表 | `dm/tools/patch_nso.py`，说明见 `dm/README.md` §1 |
| 装载 | `loader.c` 扮演 Horizon 的 rtld：映射模块、占线程槽、登记门、跨模块重定位、按函数名接管。iOS 不允许运行时生成可执行代码，所以改写后的映像由 `gen_embed.py` 生成内嵌汇编，作为签名代码编进可执行文件，运行时用 `vm_remap` 按原相对位置拼回去 | `dm/host/loader.c`、`dm/ios/tools/gen_embed.py`、`dm/host/main.c` |
| 系统调用 | `dm_gate` → `gate_dispatch` → `svc_dispatch`，内存相关的在 `hle_svc.c`，线程和同步的在 `hle_thread.c` | `dm/host/entry.S`、`hle.c`、`hle_svc.c`、`hle_thread.c` |
| 系统服务（HLE） | 不模拟 IPC，直接在 SDK 函数名这一层接管：用 `HLE` / `HLE_FORCE` 宏按 C++ 修饰名注册。没实现的导入会打印 `UNIMPL <名字>` 并返回 0 | `dm/host/hle.h`、`hle_*.c` |
| 图形 NVN→Metal | 强制接管 `nvnBootstrapLoader`，NVN 驱动的指令一条都不执行。`hle_nvn.c` 把调用录成命令，`nvn_mtl.m` 提交时回放成 Metal。共 523 个名字，D2R 实际用到的 216 个里有 203 个按语义实现、13 个是空实现，其余 307 个调到会报 `【红】NVN 未实现被调用` | `dm/gfx/nvn/hle_nvn.c`、`nvn_mtl.m`、`names.txt`、`nvn_slots.tsv`、`dm/gfx/README.md` |
| 着色器 | 离线翻译：Maxwell 机器码 → Ryujinx 着色器库（`dmsh_xlate`）→ SPIR-V → spirv-cross（`--msl-version 20300 --msl-argument-buffers`）→ metallib。按控制段 sha256 的前 16 位当 key 查找。`shader_facts.tsv` 补机器码里读不到的信息。D2R 主程序里还内嵌了 4 个计算着色器，是手写的 MSL | `dm/gfx/tools/dmsh_build.py`、`nvn/shader_facts.tsv`、`nvn/nvn_builtin.h` |
| 贴图 | Tegra block-linear 布局反 swizzle。GPU 不支持 BC 时走 bcdec 运行时解码，或者用离线转好的 ASTC 包 | `dm/gfx/tex/`、`docs/LEGACY_DEVICES.md` |
| 输入 | `nn::hid` 的 Npad 状态从前端写入的 `dm_input` 读。触屏走 `GetTouchScreenStates<16>`，坐标系 1280×720，带「短按锁存」（每根手指先报一次 Start，抬起后再报 End）。不支持震动 | `dm/host/hle_hid.c`、`hle_touch.c`、`front.h` |
| 音频 | `nn::audio` AudioOut → CoreAudio（iOS 用 RemoteIO），Opus 解码在 `hle_codec.c` | `dm/host/hle_audio.c`、`hle_codec.c` |
| 存档 | `nn::fs` 的挂载名映射到宿主目录：`rom`→`DM_ROMFS`，存档→`DM_SAVE`，临时→`DM_TEMP`。`MountSaveData`、`Commit` 等都接管了。iCloud 同步在 `cloud_ios.m` | `dm/host/hle_fs.c`、`dm/ios/Sources/cloud_ios.m` |
| 系统消息 | `nn::oe` 的 AppletMessage 队列（焦点变化 15、运行模式变化 30、性能模式变化 31），由前端推进去 | `dm/host/hle_oe.c` |
| 时间 | `nn::time` 直接取系统时钟 | `dm/host/hle_time.c` |

### Metal 4 和「老接口版」（关键发现）

- 默认构建用 Metal 4（`MTL4CommandQueue`、`MTL4ArgumentTable` 等），要求 iOS 26。
- **仓库里已经有非 Metal-4 的后备路径**：编译开关 `DM_NVN_M3=1`。见 `dm/gfx/nvn/nvn_mtl.m` 文件头，所有分叉都在 `#if DM_NVN_M3` 里。这条路用普通的 `MTLCommandQueue` 和各类 encoder，按槽绑定缓冲，参数块引用的资源逐个声明；回放层手里只有 GPU 地址，所以用 `m3_buf()` 按地址区间反查对应的 `MTLBuffer`。
  - iOS 上用 `DM_IOS_M3=1 ./run.sh build` 构建（`dm/ios/run.sh`、`dm/ios/tools/project_m3.py`：部署目标改成 18.0，加 `-DDM_NVN_M3=1`，`-mcpu` 从 apple-a18 改成 apple-a14）。
  - Mac 上用 `DM_MAC_M3=1 dm/tools/mac_app.sh`。说明见 `docs/OLDER_SYSTEMS.md`。
- **但老接口版仍然要求 Metal 3**。`dm/ios/Sources/main.m`（约第 1236 行）启动时会检查 `supportsFamily:MTLGPUFamilyMetal3`，注释写的原因是「参数缓冲里直接写 gpuAddress 的无绑定用法」要 A14/M1 才有。不满足就弹框，不启动游戏。README 的设备表也写明 **A13 及更早不支持**（有玩家在 A12Z 的 iPad Pro 上试过）。
- 推论（猜测）：博主的 A12X 能跑，说明他做的不止是打开 `DM_NVN_M3`，还在 Metal 3 以下又降了一级。A12X 属于 Apple5 家族，没有 Metal 3，argument buffer 只有 Tier 1。可能的做法：不再往参数缓冲里写 `gpuAddress`，改成每次 draw 直接 `setBuffer/setTexture/setSamplerState` 逐槽绑定；着色器也重新翻译成不用 argument buffers 的版本。《脑部锻炼》画面简单、着色器少，这条路比 D2R 好走得多。
- A12X 也不支持 BC 压缩贴图（BC 要 A17 Pro 起才支持，见 `docs/LEGACY_DEVICES.md`）。要么运行时用 bcdec 解码（`nvn_mtl.m` 里有 `k_bcd_msl` 和 `DM_TEST_NO_BC`），要么像 D2R 一样离线转成 ASTC。

## 2. 映射到《脑部锻炼》：分块实施计划

> 总原则：d2r-mobile 的**框架层**（改写器、装载器、门、HLE 宏、NVN 录制/回放、着色器翻译链、iOS 外壳）可以复用。**游戏相关的部分**都要换掉或重做：`CUTS`/`OBS`、`dm_*.c` 里写死的 D2R 地址、`nvn_slots.tsv`、`shader_facts.tsv`、`pso_manifest.tsv`、`nvn_builtin.h`、`hle_*` 里对 D2R 特有行为的假设（比如 `hle_misc.c` 里的 DLC 序号、`hle_time.c` 里的 D2R 地址），还有 `setup.sh`/`hle_fs.c` 里写死的 `3.3.94012` 路径。

### (a) 启动：游戏特有的 NVN 命令和着色器
1. 用自己的主机导出自己的卡带/下载版，拿到 ExeFS（`main`/`sdk`/`subsdk*`）和 RomFS（流程照 `docs/GAME_FILES.md` 和 `dm/tools/extract_game.sh`）。
2. 列出导入表：每个 NSO 导入了哪些 `nn::*` 符号，和现有 `HLE_FORCE` 对比，得到缺口清单。可以参考 `dm/tools/hle_inventory.py`、`xref_sdk.py`，具体用法看脚本头。
3. 关掉所有 D2R 专属钩子：`dm_*.c` 和 `patch_nso.py` 里按构建号登记的 `CUTS`/`OBS`。新游戏的构建号不同，按设计应该不会命中（猜测：需确认匹配不上时是跳过还是报错）。
4. 先在 Mac 命令行宿主上跑（`make -C dm/host` 加上 `dmhost`，带 `DM_WINDOW=1 DM_IMPORTS=1`），盯着日志里的 `UNIMPL` 和 `【红】NVN 未实现被调用`，一个一个补。博主说的「把渲染指令一点点对上」，对应的就是这一步。
5. 着色器：用 `dmsh_build.py` 跑新游戏的着色器包。`shader_facts.tsv` 要为新 key 重新生成，维护方法见 `dm/gfx/tools/dmsh_facts.py` 文件头。主程序里如果也有内嵌的计算着色器，要像 `nvn_builtin.h` 那样单独处理。
6. 方向：游戏是竖屏。iOS 外壳的 `project.yml` 现在只允许两个横屏方向，出图要旋转，触屏坐标也要跟着旋转（见 (d)）。

### (b) 性能
- 先用现成的工具找瓶颈：`DM_SLOWFRAME=1`、`DM_NVN_TSV`、`DM_NVN_PROF=1`、`DM_NVN_PASSGPU`，再用 `DM_PSO_RECORD` 重新录一份 warm-up 清单（`pso_manifest.tsv`）。开关都列在 `dm/gfx/README.md`。
- 常见原因（猜测）：管线在运行时首次编译（换成离线 metallib 加 warm-up）、贴图反复重传（`nvn_dirty.c` 的脏页跟踪）、TBDR 的 load/store 太多（`DM_NVN_TBDR`，以及 `tools/tbdr_*.py`）。
- 用 `DM_FPS` 和 `DM_PRESENT_HZ` 固定帧率。这个游戏画面接近静态，可以考虑 30fps 省电。

### (c) Metal 4 → Metal 3 → A12X（Metal 3 以下）
1. 第一步直接用现成的 `DM_NVN_M3=1`，覆盖 A14 及以上、iOS 18 及以上。
2. 要支持 A12X，需要新开一级后备，下面都是**猜测性设计**：
   - 去掉「参数缓冲里写 `gpuAddress`」的用法，每次 draw 按反射 json（`nvn_refl.m`）逐槽直接绑定 buffer、texture、sampler。M3 分支里的 `m3_buf()` 地址反查可以复用。
   - 着色器翻译去掉 `--msl-argument-buffers`，改成离散绑定，同时检查槽位数量有没有超出 Apple5 的限制。
   - 检查 `MTLResidencySet` 等 iOS 18 新 API 在 A12X 上能不能用；不能用就退回 `useResource`。
   - `main.m` 里的 `MTLGPUFamilyMetal3` 门要改成按后备级别判断。
   - 贴图走 BC→ASTC 离线包，或者用 bcdec 在运行时解码。
   - A12X 的 iPad Pro 内存是 4 GB（1TB 款是 6 GB）。D2R 在 4 GB 机器上会内存不足，但这个游戏资源小，预计问题不大（猜测）。
3. 要做门控的功能：argument buffer 层级、`gpuAddress`、Metal 4 的 barrier/feedback、residency set、BC 贴图，还有 `-mcpu=apple-a14` 要改成 `apple-a12`（宿主汇编要的 LSE 原子指令 A11 起就有，`project_m3.py` 的注释里提到了）。

### (d) 触屏 → 手写
- `hle_touch.c` 已经实现了 `GetTouchScreenStates<16>`，但里面的 1280×720 和「只看第 0 份」都是按 D2R 的读法核实过的。新游戏要重新确认它请求几份状态、读不读直径/角度。
- **游戏竖屏**：Switch 竖过来时，触屏面板仍是物理的 1280×720，游戏自己做旋转。所以前端要把 iPad 的竖屏坐标换算回面板坐标，大致是 `x = (1-ny)·1280, y = nx·720`，具体方向要实测。
- 手写识别的采样率：iPad 触摸是 120Hz 或 60Hz，游戏每帧读一次只拿到最新的点，快速写字时笔画会稀疏。可以考虑在前端插值，或者确认游戏是不是读多份历史状态（猜测）。
- 可以考虑支持 Apple Pencil（UITouch 的 `.pencil` 类型照样当触点上报）。
- 截图 3 里有橡皮擦图标，说明手写和擦除是游戏自己的 UI，宿主只需要把触点送对。

### (e) Joy-Con 红外摄像头 → TrueDepth
- **仓库里现在没有任何 `irsensor` 相关代码**（已搜索），要从零写一个 `hle_irsensor.c`。
- 第一步：看主程序导入了哪些 `nn::irsensor::*` 符号，确定游戏用的是哪种处理器。按 libnx/Ryujinx 公开资料，候选有 ImageTransfer（原始灰度图）、Clustering、Moment、HandAnalysis（手部轮廓）等。**具体是哪种需要核对**，这决定了后面的难度。
  - 如果是 **ImageTransfer**：按 libnx 的公开定义，输出是 8bpp 灰度图，可选 320×240、160×120、80×60、40×30（需核对）。宿主拿到 TrueDepth 的深度图后：裁成 4:3，按距离映射亮度（越近越亮，模拟红外反射），再缩到要求的尺寸，填进 `GetImageTransferProcessorState` 的输出缓冲。
  - 如果是 **HandAnalysis / Clustering**：游戏要的是已经算好的结构化结果（轮廓点、簇等），宿主得自己从深度图算出来。难度高很多，也可以考虑用 Vision 的 `VNDetectHumanHandPoseRequest` 先数手指再合成数据（猜测）。
- iOS 侧（都是公开 API）：`AVCaptureDevice` 的 `.builtInTrueDepthCamera` 加 `AVCaptureDepthDataOutput`，拿到 `AVDepthData`（disparity 或 depth，float16/32）。也可以用 ARKit 的 `ARFaceTrackingConfiguration.capturedDepthData`，但它是为人脸设计的，帧率和取景不一定适合拍手。前者更直接。需要在 Info.plist 里加相机权限 `NSCameraUsageDescription`。
- 截图 4 是蓝底、手部绿到黄的假彩色，像是把深度直接可视化当预览。游戏实际收到的应该是单通道图（猜测）。
- 坐标对齐：Joy-Con 的摄像头是朝外看手的，前置摄像头朝着人。需要做左右镜像，还要根据 iPad 的朝向旋转。

### (f) 音频 / 麦克风
- 音频输出直接复用 `hle_audio.c`。要确认游戏是不是也走 AudioOut，还是用了 FMOD 以外的中间件、`nn::atk` 之类（需核对导入表）。
- Switch 本体没有麦克风。这款游戏预计不用录音（猜测）。如果导入表里有 `audin` 或 `AudioIn`，再接 AVAudioEngine 的输入。

### (g) iOS 原生提醒
- 这是**外壳层的新功能**，不必去钩游戏。思路是在 `ios/Sources` 里加一个设置项，用 `UNUserNotificationCenter` 加 `UNCalendarNotificationTrigger`（`repeats:YES`）安排每天的提醒，文案写成「到时间了，可以开始脑部锻炼了。」。
- 如果想和游戏内的「训练提醒/日历」设置同步，就得找到游戏存这个设置的位置并读出来。是否可行属于猜测，需要逆向。
- 当前代码里没有通知相关实现（已搜索 `UNUserNotification`）。

### (h) 存档
- 复用 `hle_fs.c`：`EnsureSaveData`、`MountSaveData`、`Commit` 已经接管了，存档目录由前端设到 App 沙盒的 `DM_SAVE`。
- 要核对新游戏用的挂载接口是不是同一批（有的游戏用 `MountSaveDataReadOnly`、`MountCacheStorage`、`MountBcatSaveData` 等）。
- 多用户：游戏可能按 `nn::account` 的 Uid 区分档案（家里多人用）。`hle_account.c` 现在只返回固定的本地用户，能不能多档案要看游戏自己存档内部的设计。
- iCloud 同步可以复用 `cloud_ios.m` 的思路。

## 3. 风险和未知
- 博主的帖子没有任何代码，可复现性未知。「A12X 60fps」「深度摄像头完美运行」都是自述。
- d2r-mobile 很多东西是围绕 D2R 的一个固定构建号磨出来的，换游戏后补洞的工作量取决于新游戏用到的 NVN 和 SDK 范围。307 个未实现的 NVN 函数里，新游戏可能会用到一些。
- `hle_nvn.c` 的已知限制（`dm/gfx/README.md`「Known limitations」）：pass 中途 clear、quad 索引图元、8 位索引等还没实现。新游戏碰上了就得补。
- irsensor 用的是哪种处理器，直接决定 (e) 的工作量。
- Metal 3 以下的后备在上游没有，等于自己维护一个分叉。
- 用免费 Apple ID 签名的 App 每 7 天要重签（README 里有写），不适合直接交给长辈日常用。

## 4. 法律和边界
- **只用自己主机导出、合法拥有的那一份游戏**。游戏程序、RomFS、`prod.keys`、着色器包，以及从它们生成的任何资源，**一律不提交 git、不上传、不分发**。这和 d2r-mobile 的原则一致（见 `docs/GAME_FILES.md`、`docs/DERIVED.md`）。
- 公开仓库里最多放：自己写的宿主代码、接口信息（地址、函数表等，参照 `docs/DERIVED.md` 的尺度），以及这类思路笔记。
- 在 Nintendo 的作品上做衍生移植，法律风险比 D2R 更高（Nintendo 维权更积极）。只做自用研究，不对外发包。
