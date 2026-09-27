# mperf 核心功能与实现原理

> 面向想读懂这个项目、或准备在其上做二次开发的人。
> 命令行参数速查请看 [`docs/cli.md`](../cli.md)，日常操作请看 [使用方法](usage.md)。

## 1. 它解决什么问题

Android 和 iOS 的性能采集工具各自为政：Perfetto、Simpleperf、ART Method Trace、Instruments，
各有各的启动方式、产物格式、查看器。mperf 把它们收进一个 CLI，统一成
**「选目标 → 采集 → 转格式 → 打开查看器」** 一条流水线。

在这个基础上，它额外做了一件其他工具没做的事：**可信的冷启动缺页（page fault）归因分析**——
精确到每一次缺页发生在哪个文件的哪个偏移、属于哪个 DEX/编译方法、当时的调用栈是什么。

所以整个项目实际上是两条相对独立的管线：

| 管线 | 入口命令 | 代码位置 | 定位 |
|---|---|---|---|
| A. 常规 profiling | `android start` / `android collect` / `ios start` / `ios convert` | `profilers/`、`gecko/` | 薄封装，统一多个平台 profiler |
| B. 启动缺页分析 | `faults android` / `faults ios` | `faults/`、`resources/faults-engine/` | 自研，代码量占大头 |

## 2. 整体架构

```
                        Main.kt  (Clikt 命令树装配)
                              │
        ┌─────────────────────┴──────────────────────┐
        │                                            │
   管线 A: profilers/                         管线 B: faults/
        │                                            │
  ┌─────┴─────┬──────────┬──────────┐      ┌─────────┴─────────┐
Perfetto  Simpleperf  Method   Instruments  AndroidFault*    IosFault*
  │           │          │          │        Collector        Collector
  │           │          │          │        Processor        Processor
  │           │          │          │        Report           Report
  └─────┬─────┴──────────┴────┬─────┘             │
        │                     │                   │
   utils/Adb (adb 传输)   utils/XcodeUtils    resources/faults-engine/
        │                  (xctrace)          (native C + 离线 viewer)
        │                     │                   │
   tools/ (SHA-256 钉死的外部二进制供应)            │
        │                     │                   │
        └──────── utils/ProfileOpener ────────┘    └──> 自包含 report.html
             (Perfetto UI / Firefox Profiler / Instruments)
```

分层职责（对应 `src/main/kotlin/com/bromano/mobile/perf/`）：

- `commands/` — Clikt 命令定义、参数解析、交互式设备选择、默认值兜底
- `profilers/` — `Profiler` 接口的四个实现（见 §3）
- `faults/` — 缺页采集编排、预处理、归因、报告生成（见 §4）
- `gecko/` — Instruments `.trace` → Gecko（Firefox Profiler）格式转换器，自研 XML 解析
- `tools/` — 外部二进制供应：tracebox、simpleperf prebuilt、Perfetto trace_processor
- `utils/` — adb/shell 传输、配置、下载校验、浏览器打开、重试
- `resources/faults-engine/` — 随 JAR 打包的 native C 采集器源码 + 离线报告前端资产

## 3. 管线 A：常规 profiling

### 3.1 统一抽象

所有采集器实现同一个接口（`profilers/Profiler.kt`）：

```kotlin
interface Profiler {
    val targetProcessId: Long?                                   // 平台能给出稳定 PID 时提供
    fun execute(packageName, output)                             // ad-hoc 会话
    fun executeTest(packageName, runner, testCase, output)       // 单次 Macrobenchmark 迭代
}
```

`ProfilerExecutorImpl` 只做三件事：按 `--format` 选工厂 → 调 `execute`/`executeTest` → 交给
`ProfileOpener` 打开。新增一种 profiler 只需实现接口并在 `Main.kt` 的工厂表里注册一行。

### 3.2 Perfetto（Android 默认）

`profilers/perfetto/PerfettoProfiler.kt`

1. **版本适配**：Android 10（API 29）及以上直接用设备自带的 `perfetto` 服务；更老的设备
   旁加载钉死版本的 `tracebox`（v58.2）到 `/data/local/tmp/`。API 28+ 还要 `setprop persist.traced.enable 1`。
2. **配置下发**：用户给 `--configPb` 就用用户的，否则由 `DefaultPerfettoConfig.kt` 按包名生成，
   push 到 `/data/local/tmp/perfetto_config.pb`，以 `cat config | perfetto -c - -o <trace>` 启动。
3. **生命周期**：轮询 `pidof` 确认进程起来（10s 超时），等用户按键，然后 `kill -TERM`；
   失败回退 `killall`，再失败就限时等待。这是为了让 perfetto 有机会正常 flush 而不是被硬杀。
4. **回传**：用 `cat <file> > <output>` 而不是 `adb pull`（注释说明是为了绕开某些老系统的权限限制）。

### 3.3 Simpleperf

`profilers/simpleperf/SimpleperfProfiler.kt`

- **root 优化**：设备可 root 时旁加载上游钉死的 simpleperf prebuilt，并加上 `--user-buffer-size 1G`。
  这个参数只有 root 能改，而默认缓冲区会导致调用栈损坏（代码里附了 AOSP commit 链接）。
- **停止方式**：`kill -2`（SIGINT）而不是 SIGTERM，让 simpleperf 自己收尾；之后轮询等它真正退出
  （大会话 / 大缓冲区 finalize 很慢，默认给 60s）。
- **格式转换在 host 侧**：pull 回 `perf.data`，调用上游 `gecko_profile_generator.py`（Python 3）
  转成 Gecko JSON，再 gzip。`--symfs` / `--mapping`（R8）/ `--remove-method` / `--show-art-frames`
  都是直接透传给这个脚本的。

### 3.4 ART Method Trace

`profilers/method/MethodProfiler.kt`

分两种情况：app 已在跑就 `am profile start --streaming`；没跑就 `am start --start-profiler --streaming`
冷启动。API 35+ 额外加 `--clock-type wall`。停止后要轮询等 trace flush 完再 pull。

### 3.5 Instruments（iOS）

`profilers/instruments/InstrumentsProfiler.kt` + `utils/XcodeUtils.kt`，底层是 `xcrun xctrace`。
产出 `.trace` 包，可以直接用 Instruments 打开，也可以走 `gecko/InstrumentsConverter.kt` 转成
Gecko/Perfetto 格式。

> **Simulator 的已知限制**：较新的 Xcode 不再可靠地接受 simulator 进程作为 `xctrace` target。
> mperf 的处理是退化为「录制 host 进程」并明确告警，转换输出按目标 PID 过滤——但原始 `.trace`
> 仍然含有其他 host 进程，体积大得多，应当视为 host 级诊断数据。物理设备采集不受影响。

### 3.6 Macrobenchmark 模式（`android collect`）

`profilers/BenchmarkInstrumentation.kt` 负责拼 `am instrument` 命令，关键参数：

- `androidx.benchmark.dryRunMode.enable=true` — 让 Macrobenchmark 只跑一次测量迭代
- `androidx.benchmark.suppressErrors=EMULATOR` — 允许在模拟器上跑（验证采集链路用，性能数字不可比）
- Simpleperf 模式额外加 `androidx.benchmark.profiling.mode=StackSampling`

产物定位用的是「跑之前列目录 / 跑之后列目录 / 取差集」的办法（`findNewBenchmarkOutput`），
而不是猜文件名。注意 Simpleperf 的 `collect` 走的是 AndroidX 自己的栈采样，产出的是
**Perfetto trace**，所以默认在 Perfetto UI 里打开。

### 3.7 查看器

`utils/ProfileOpener.kt` + `ProfileViewer.kt`，三选一：`PERFETTO` / `FIREFOX` / `INSTRUMENTS`。

- 配了 `traceHostUrl` 就先 multipart POST 上传，拿回 `{"id": ...}`，用公网 URL 拼查看器链接
- 没配就在本机起一个临时 loopback HTTP 服务，只暴露那一个文件，浏览器打开失败会清理掉。
  用官方 Perfetto UI 时固定监听 9001（官方 UI 只放行这个端口）；Firefox Profiler 和自建
  `perfettoUrl` 用随机端口
- **trace 字节从不上传到第三方**：Perfetto 走 `postMessage` 交接，Firefox Profiler 走 `from-url` 拉本地端口

## 4. 管线 B：启动缺页分析（项目核心）

### 4.1 为什么要自己写 native 采集器

`resources/faults-engine/android/native/page_fault_collector.c` 开头的注释说明了理由：

> Android 的 arm64 内核不暴露 x86 的 `page_fault_user` tracepoint；但
> `PERF_COUNT_SW_PAGE_FAULTS_{MIN,MAJ}` 这两个软件事件在 `handle_mm_fault()` 之后触发，
> 两种架构都有，并且**在 perf 的 sample address 字段里带着触发缺页的虚拟地址**。

也就是说：现成的 `simpleperf` CLI 拿不到缺页地址，Perfetto 也给不了这个粒度。所以 mperf
自己用 `perf_event_open` 写了一个采集器，`sample_type` 包含
`PERF_SAMPLE_IP | TID | TIME | ADDR | CPU | PERIOD`（开 `--native-stacks` 时再加 `CALLCHAIN`），
同时消费 `PERF_RECORD_MMAP2` 记录时间戳化的文件映射。

这个 C 文件**不是预编译的**：`AndroidFaultCollector.buildCollector()` 在采集时用 host 上的
**Android NDK clang**（`-O2 -Wall -Wextra -Werror`）现场交叉编译，push 到设备再执行，
并把源码和二进制的 SHA-256 写进 `capture_metadata.json`。

> ⚠️ 这意味着 `mperf faults android` 需要本机装 Android NDK（README 的 Requirements 里没写）。
> 查找顺序：`ANDROID_NDK_HOME` → `ANDROID_NDK_ROOT` → `$ANDROID_HOME/ndk/` 下版本号最大的一个
> → `~/Library/Android/sdk/ndk/`。

### 4.2 Android 采集时序

`faults/AndroidFaultCollector.kt` 的 `collect()` 严格按这个顺序走，每一步都留证据：

```
 1. --reboot-before-collect ? 重启并等新 boot_id + 开机完成
 2. adb root；读 sdk / abi / 内核版本 / boot_id / uptime
 3. 校验内核页大小：getconf PAGESIZE 必须和 /proc/self/smaps 的 KernelPageSize 一致
    （x86_64 ps16k 镜像会在 4K 内核上模拟 16K 用户态 —— 这不算 16K 覆盖）
 4. NDK 编译 page_fault_collector.c → push；dwarf 模式再旁加载 simpleperf
 5. AOT 准备：--compilation speed-profile（默认）逐个校验已安装 APK 的真实 ART filter
 6. am force-stop，等进程真的没了
 7. 记录驻留基线 before_drop（mincore）
 8. sync → echo 3 > /proc/sys/vm/drop_caches → 对 app 文件逐个 fadvise 驱逐
 9. 记录 after_drop 驻留，严格校验（默认容忍 0 页，超了直接失败）
10. 启动 Perfetto（启动上下文） → 启动 native collector → 可选启动 DWARF 伴随录制器
    ★ 三者都在 app 进程创建之前就位，所以能覆盖启动最开头
11. 复查 app 文件集合没变（新出现的文件补驱逐），再验一次 before_launch 驻留
12. am start -W 启动；要求 "Status: ok"
13. 立刻抓 /proc/<pid>/maps、/proc/<pid>/stat、inode 清单
14. sleep --settle-ms（默认 750ms）
15. 停 collector / Perfetto / DWARF，pull 回所有产物
16. 完整性闸门：lost / integrity_errors / throttled / callchain_overflow 任一非 0 → 采集判定失败
```

第 16 步是这个项目的性格所在：**宁可让整次采集失败，也不产出一份看起来干净、实际丢了事件的报告。**

### 4.3 归因：从虚拟地址到「文件 + 偏移 + 方法」

`faults/AndroidFaultProcessor.kt` 在 host 侧做离线归因：

1. **地址 → 文件**：用时间戳化的 `PERF_RECORD_MMAP2` 事件 + `maps.txt` + inode 清单，
   找到缺页发生那一刻覆盖该地址的映射，算出文件内偏移。APK 内条目、整个 ODEX/VDEX 都能归到。
2. **窗口截断**：分析窗口结束于捕获进程主线程的第一个 `reportFullyDrawn*` marker；没有就用
   Perfetto 的首帧时间戳。这个截断点对 faults、page-cache 事件、DWARF 匹配、I/O 上下文**共用**，
   并写进 `capture_metadata.json` 的 `startup.end_marker` / `startup.first_frame_ts_end` 以便审计。
3. **VDEX/DEX 边界**（`AndroidArtifacts.kt`）：只有当 ART 里存的每一个 DEX location checksum
   都和 APK 条目对得上时，才标注原始 `classes*.dex` 边界；对不上就把 VDEX 当整文件处理，**不猜**。
4. **编译方法归因**（`AndroidOat.kt`）：用设备上的 `oatdump` 解析 ODEX，校验所有 DEX location
   checksum 和 APK/ODEX/VDEX 哈希后，定位缺页地址落在哪个编译方法里；同一页上的其他方法单独列出。
5. **调用栈**（`AndroidDwarf.kt`）：`--native-stacks` 给的是帧指针调用链（和缺页地址同一条 perf 记录，
   天然精确）。`--dwarf-stacks` 另开一个 system-wide simpleperf 录制器补 DWARF/ART 栈，
   **只有在 PID、TID、纳秒时间戳、指令地址、CPU 全部唯一精确匹配**，且时钟/boot/哈希校验通过、
   零丢失时才做 enrich —— 不做就近时间或序号拼接。匹配不上就明确告警并省略，不降级冒充。

### 4.4 iOS 侧

`faults/IosFaultCollector.kt`，基于 Instruments 的 **Virtual Memory Trace**：

- 先校验当前 Xcode 确实暴露了 `Virtual Memory Trace` 这个 instrument
- **启动顺序保证**：先起 `xctrace`，用 `--notify-tracing-started` + `/usr/bin/notifyutil` 等到
  录制器明确就绪的通知才启动 app；有界超时，超时就中止并回收录制器。host 单调时钟记录
  「录制器就绪」和「启动」两个时间戳，事后可审计。
- 每一行都按数字 PID + 进程身份过滤，防 PID 复用
- 栈帧归属按 **UUID 校验过的 Mach-O 镜像**（`IosMachO.kt`）判断，认 `__TEXT`/`__DATA` 等 section，
  而不是看调用者；只认装在 app bundle 根下的二进制（含内嵌 framework 和 `.appex`）
- **缓存驱逐**：Simulator 上清点 bundle 文件并在启动前用 `mincore` 验证驻留；`auto` 策略先试
  host `purge`，`--allow-host-pressure` 时可退化为有界内存压力。物理设备没有受支持的全局
  page cache drop，只能重启或用附带的签名压力助手，**做不到 Simulator/rooted Android 那种强保证**。

术语提醒：iOS 的 "major" 指 file-backed page-in 操作，"minor" 把 cache 命中、zero-fill、
COW、解压都归到一起。这是从 Instruments 操作类型推导的分析桶，**不是 Darwin 内核的 fault 标签**。

### 4.5 报告生成与离线 viewer

- `faults/FaultEngine.kt`（`BundledFaultEngine`）把 JAR 里 `faults-engine/manifest.txt` 声明的资产
  按**内容哈希**解压到 `~/.mperf/cache/faults-engine/<hash16>/`，带文件锁 + `.complete` 标记 +
  逐字节校验，多进程并发安全，版本变了自动换目录。
- `faults/SharedFaultReport.kt` 把 `plotly.min.js`、`report.js/css/html`、`model.js`、`stacks.js`、
  `context.js`、`perfetto.js` 和采集数据一起塞进 HTML 模板的占位符，产出**单文件自包含报告**。
- 构建期有 `verifyFaultResources` 任务：打包进 JAR 的 `faults-engine/**` 必须和 `manifest.txt`
  **完全一致**（多一个少一个都报错），`jar` 和 `shadowJar` 都依赖它。
- 前端逻辑（缩放、栈聚合、账目核对）有独立的 Node 测试：`./gradlew testFaultViewer`。

报告包含：全文件地址/时间散点、按文件时间线、顺序性分析、APK/DEX 与 VDEX/ODEX 归因、
major/minor 证据、**Capture health** 标签页（缓存校验、丢失、栈覆盖率、符号解析、I/O 可用性）、
**I/O context** 标签页（缺页 / cache 插入 / ART advice / 阻塞 / block 事件对齐到启动时钟）、
对比视图、有序调用栈列表。Android 报告还有一个 **Open in Perfetto** 按钮，把本次
`faults.pftrace` 通过 `postMessage` 递给 Perfetto UI（不上传）。

> 采集失败时，只要有自有证据，仍会写出 `capture-health.html`（CLI 依然以失败退出，
> 该报告不含缺页图表）。原始证据完整保留。

## 5. 外部二进制供应与安全

`tools/` 把所有外部二进制的获取集中管理，共同模式是 **版本钉死 + SHA-256 校验 + 本地缓存**：

| 组件 | 版本/提交 | 落地位置 |
|---|---|---|
| tracebox（Android < 10 的 Perfetto） | Perfetto v58.2 | 设备 `/data/local/tmp/tracebox` |
| simpleperf prebuilt | AOSP commit `829c235…` | 设备 `/data/local/tmp/simpleperf` |
| simpleperf host 脚本 | AOSP commit `fc2494a…`，**整棵树哈希**校验 | `~/.mperf/simpleperf/` |
| Perfetto trace_processor_shell | v58.2，size + SHA-256 | `~/.mperf/cache/trace-processor/` |

几个值得注意的加固点：

- 旁加载前先 `sha256sum` 设备上已有的文件，一致就跳过下载
- simpleperf 脚本 tarball 解包前检查**每一个条目**：不能是绝对路径、不能含 `..`、
  不能是符号链接或特殊文件；解包后再算整棵树的哈希比对
- `NativeTraceProcessor` 用文件锁 + 原子 move，避免并发下载互相踩踏
- googlesource 的 `?format=TEXT` 返回 base64，下载后先解码再校验

## 6. 贯穿全项目的「证据纪律」

读代码和读报告都要理解这条设计取向，它解释了很多看起来"过于严格"的行为：

| 原则 | 具体表现 |
|---|---|
| 相关 ≠ 因果 | I/O context 明说是时间对齐，不声称某个 block 请求导致了某次缺页 |
| 缺失 ≠ 零 | 某条流采不到就标为「不可用」，不记 0 |
| 不猜 | VDEX 校验不过就不标 DEX 边界；OAT 元数据损坏就留空而不丢弃整个 trace |
| 不降级冒充 | DWARF 栈流匹配不上时不会被重新标注成「精确页归因」 |
| 丢事件即作废 | lost/throttle/overflow 非 0 直接判采集失败，而不是产出一份"干净"的报告 |
| 冷即可验证的冷 | mincore 实测驻留页，默认容忍 0；Android 16 模拟器镜像有残留就直接失败，而不是自称冷启动 |
| 口径写清楚 | major fault 数 ≠ 存储读次数；page-cache 插入不计入缺页总数；Simulator ≠ 物理设备 |

## 7. 构建产物一览

一次 `mperf faults android` 的输出目录（默认 `artifacts/faults/android-<时间戳>/`）：

| 文件 | 内容 |
|---|---|
| `report.html` | 自包含交互报告（主产物） |
| `capture-health.html` | 失败时的诊断报告（仅在有自有证据时） |
| `capture_metadata.json` | 设备指纹、内核、页大小、采集器哈希、丢失计数、启动窗口、警告 |
| `fault_events.csv` / `mapping_events.csv` | 采集器原始缺页事件 / MMAP2 映射 |
| `fault_callchains.csv` / `resolved_fault_callchains.csv` | 原始 / 已符号化调用栈 |
| `faults.pftrace` | 同场 Perfetto trace（启动上下文） |
| `maps.txt` / `process_stat.txt` / `inodes.txt` / `launch.txt` | 进程与启动现场快照 |
| `cache_residency.csv` | 各阶段 mincore 驻留测量 |
| `artifacts.json` / `binary_sections.json` / `oatdump.json` | 拉回的 APK/ART 产物与解析结果 |
| `vdex_dex_boundaries.csv` | 校验通过的 DEX 边界 |
| `page_cache_events.csv` / `file_sizes.csv` | page-cache 插入证据 / 文件尺寸 |
| `page_fault_stats.json` / `major_page_fault_*.csv` | 聚合统计 |

## 8. 技术栈

Kotlin 2.4.10 / JVM toolchain 21 · Clikt 5.1（CLI）· Protobuf 4.36（Perfetto config）·
Jackson YAML（配置）· Gson · kotlinx-coroutines · Shadow（fat JAR）· ktlint · JMH（转换器基准）·
JUnit 5 + Mockito · Node `--test`（报告前端回归）· C（native 采集器）· Swift（iOS 压力助手）
