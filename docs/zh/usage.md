# mperf 使用方法

> 参数全集见 [`docs/cli.md`](../cli.md)（由 `./gradlew generateDocs` 自动生成）。
> 实现原理见 [核心功能与实现原理](architecture.md)。

## 1. 环境准备

| 依赖 | 必需性 | 说明 |
|---|---|---|
| **Java 21+** | 必需 | `.java-version` 要求 21；Gradle 会自动下载 toolchain，但直接 `java -jar` 需要 PATH 上是 21 |
| **adb**（Android SDK Platform-Tools） | Android 必需 | 需在 PATH 上 |
| **Android NDK** | **`faults android` 必需** | 采集器 C 源码在采集时现场交叉编译。README 未列出此项 |
| **完整 Xcode** | iOS 必需 | 需要 `xctrace`，Command Line Tools 单独装**不带**它；`sudo xcode-select --switch /Applications/Xcode.app` |
| **Python 3** | 部分场景 | 安装脚本、Simpleperf→Firefox 转换、可选 trace server。**`faults` 两个命令都不需要** |
| **Node** | 仅开发 | 跑报告前端回归测试 |
| `tar` / `gzip` | 安装时 | |

首次运行 `faults android` 会下载钉死的 Perfetto v58.2 trace processor（校验 size + SHA-256 后
缓存到 `~/.mperf/cache/`），**需要一次联网**。

## 2. 安装

### 方式一：安装脚本（推荐）

```bash
curl -fsSL https://raw.githubusercontent.com/benjaminromano/mperf/refs/heads/main/scripts/install.sh | bash
```

装完 PATH 上多三个命令：`mperf`、`aperf`（= `mperf android`）、`iperf`（= `mperf ios`）。

卸载：

```bash
curl -fsSL https://raw.githubusercontent.com/benjaminromano/mperf/refs/heads/main/scripts/uninstall.sh | bash
```

### 方式二：手动装 JAR

从 [GitHub Releases](https://github.com/benjaminromano/mperf/releases) 下 `mperf-<version>-all.jar`，
在 `~/.zshrc` / `~/.bashrc` 加：

```bash
MPERF_JAR="$HOME/tools/mperf/mperf-<version>-all.jar"
if [ -f "$MPERF_JAR" ]; then
  alias mperf="java -jar \"$MPERF_JAR\""
  alias aperf='mperf android'
  alias iperf='mperf ios'
fi
```

### 方式三：源码运行（开发用）

```bash
./gradlew run --args "android start -p com.example.app"
# 或先打包
./gradlew shadowJar && java -jar build/libs/mperf-*-all.jar --help
```

## 3. 配置文件

首次运行自动生成 `~/.mperf/config.yml`。配上以后就不用每次敲 `-p` / `-i` / `-b`：

```yaml
android:
  package: com.example.app
  instrumentationRunner: com.example.macrobenchmark/androidx.test.runner.AndroidJUnitRunner
ios:
  bundleIdentifier: com.example.app
  deviceId: <SIMULATOR_UDID>
traceHostUrl: https://myserver.com/trace     # 可选：trace 上传分享端点
perfettoUrl: https://perfetto.example.com    # 可选：自建 Perfetto UI
```

| 字段 | 作用 |
|---|---|
| `android.package` | `android start / collect` 的默认包名 |
| `android.instrumentationRunner` | Macrobenchmark 默认 runner |
| `ios.bundleIdentifier` | `ios start` 默认 bundle id |
| `ios.deviceId` | 未传 `--device` 时的默认设备/模拟器 UDID |
| `traceHostUrl` | 接受 `POST /trace`（multipart，返回 `{"id":...}`）并提供 `GET /trace/<id>` |
| `perfettoUrl` | 自建 Perfetto UI；公网 HTTPS trace 用官方 UI 即可，不必设置 |

## 4. 常规性能采集

### Android

```bash
# Perfetto ad-hoc（默认格式），按任意键停止
aperf start -p com.example.app

# Simpleperf 采样 → 默认在 Firefox Profiler 打开
aperf start -f simpleperf -p com.example.app

# Simpleperf 进阶：off-CPU + 4kHz + native 符号 + R8 反混淆 + 噪声帧过滤
aperf start -f simpleperf -p com.example.app \
  --simpleperfArgs "-e task-clock -g -f 4000 --trace-offcpu" \
  --symfs "$HOME/Android/Symbols" \
  --mapping app/build/outputs/mapping/release/mapping.txt \
  --remove-method "^io\.reactivex.*$" \
  --remove-method "^\[DEDUPED\].*$" \
  --no-show-art-frames

# ART Method Trace
aperf start -f method -p com.example.app

# 换查看器
aperf start -f simpleperf -p com.example.app --ui perfetto
aperf start -p com.example.app -f method --ui perfetto
```

**单次 Macrobenchmark 迭代：**

```bash
aperf collect -p com.example.app \
  -i com.example.macrobenchmark/androidx.test.runner.AndroidJUnitRunner \
  -t LoginBenchmark#loginByIntent

# method 格式：先跑完配置的测量迭代，再额外跑一次带 method tracing 的采集迭代
aperf collect -p com.example.app -f method -t SomeBenchmark#case

# simpleperf 格式：走 AndroidX 栈采样，产出 Perfetto trace，因此默认在 Perfetto UI 打开
aperf collect -f simpleperf -p com.example.app -t SomeBenchmark#case
```

> 省略 `--device` / `--instrumentation` 时，CLI 会列出候选让你交互式选择。

### iOS

```bash
# Time Profiler，结果在 Instruments 里打开
iperf start -b com.example.app --template "Time Profiler" --ui instruments

# 多个 instrument，导出到 Perfetto
iperf start -b com.example.app --instrument "Time Profiler" --instrument "Core Animation" --ui perfetto

# 已有 .trace 转 Gecko 给 Firefox Profiler
iperf convert --input MyTrace.trace --output my-trace.gecko.json --app MyApp
```

> 交互式设备选择器**只列已 boot 的模拟器**。先 `xcrun simctl boot <UDID>` 或用 Xcode 启动。

### 默认输出位置

不传 `-o` 时落在 `artifacts/trace_out/<format>-<yyyy-MM-dd-HH-mm>.<后缀>`：
Perfetto → `.perfetto-trace`，Simpleperf → `.json.gz`，Method → `.trace`。

## 5. 启动缺页分析（`faults`）

### Android

```bash
# 最简
mperf faults android -p com.example.app --reboot-before-collect

# 带调用栈的完整采集
mperf faults android \
  --package com.example.app \
  --device emulator-5554 \
  --reboot-before-collect \
  --native-stacks

# 再加 DWARF/ART 栈（更慢、更侵入，大 app 可能丢样本）
mperf faults android -p com.example.app --native-stacks --dwarf-stacks --reboot-before-collect

# 附带 block I/O 与调度证据
mperf faults android -p com.example.app --io-evidence

# 重放已有采集（不碰设备）
mperf faults android --skip-collect -o artifacts/faults/android-20260920-101500

# 两次采集对比
mperf faults android --skip-collect -o <capture-A> --compare <capture-B> \
  --label "优化前" --compare-label "优化后"
```

**前置条件**：能以 root 运行采集器的模拟器或设备（推荐 `userdebug` / `eng` 构建）。

**关键参数**：

| 参数 | 默认 | 说明 |
|---|---|---|
| `--settle-ms` | 750 | 首帧显示后继续采集的时长。app 的 `reportFullyDrawn` 较晚时要调大，如 `--settle-ms 3000` |
| `--max-resident-pages` | 0 | 启动前允许的 app 文件驻留页上限。严格默认 0，超了直接失败 |
| `--compilation` | `speed-profile` | 校验真实 ART filter；`as-is` 用于刻意的重置/全 AOT 对比 |
| `--reboot-before-collect` | 关 | 等新 boot ID + 开机完成后再准备 |
| `--reclaim-mapped-apks` | 关 | 对其他进程持有的只读 APK 映射做有界 page-out 建议（会影响其他进程，且不保证驱逐成功） |
| `--native-stacks` | 关 | 帧指针调用链，和缺页地址在同一条 perf 记录里 |
| `--dwarf-stacks` | 关 | 额外的 system-wide DWARF 录制器，仅精确匹配时用于补充 |
| `--dwarf-kernel-pages` | 4096 | 每 CPU 内核 ring 页数（2 的幂，64–16384）。内核环丢失只能靠加它，加用户态缓冲没用 |
| `--dwarf-user-buffer-mb` | 256 | 用户态缓冲 MiB（16–2048）。越大越占目标机内存，可能改变被测负载 |
| `--io-evidence` | 关 | 同场录制 block/调度事件，导出 ART advice、block 事件、线程状态 CSV |
| `--skip-collect` | 关 | 重放已有采集（包名从采集数据里读，不需要本地配置） |
| `--compare` | — | 第二个采集目录，出对比视图 |
| `--allow-incomparable` | 关 | provenance 不一致时仍允许探索性对比 |

### iOS

```bash
# Simulator
mperf faults ios --app path/to/MyApp.app --require-cold-cache --allow-host-pressure

# 完整形式
mperf faults ios \
  --app path/to/MyApp.app \
  --device booted \
  --cache-policy auto \
  --allow-host-pressure \
  --require-cold-cache
```

| 参数 | 说明 |
|---|---|
| `--cache-policy` | `auto`（先试 host `purge`）/ `purge` / `pressure` / `reboot` / `none` |
| `--require-cold-cache` | 除非 Simulator 驻留检查确认已驱逐，否则拒绝本次运行 |
| `--allow-host-pressure` | 允许 `auto` 退化为有界内存压力 |
| `--allow-unconfirmed-cache` | mincore 无法确认驱逐时仍继续 |
| `--residency-threshold` | 动作后允许的 app 文件驻留比例（默认 0.05） |
| `--settle-seconds` | 分析的启动窗口秒数（默认 3.0） |
| `--time-limit` | 最大录制时长（秒），不设则不限 |
| `--pressure-fraction` / `--pressure-megabytes` / `--pressure-hold-seconds` | 物理设备内存压力助手参数（默认 0.15 / 无 / 6s） |

### 报告怎么看

默认写到 `artifacts/faults/<platform>-<时间戳>/report.html`，自包含单文件。

- 报告固定在浏览器视口内，源列表、图表、底部「Selected fault」栏各自独立滚动
- **缩放**：捏合或 `Ctrl+滚轮`（围绕指针）；**平移**：WASD（不改变选中）；**重置**：Esc
- 栈图默认显示所有匹配的缺页，**调用者在上、触发缺页的帧在下**
- 缺页列表每个事件一行；点蓝色帧名查看该事件的完整调用栈
- 拖动「Selected fault」上方的分隔条调整栈面板高度；分隔条获得焦点后 ↑/↓ 调整、Home/End 取最小/最大
- **Capture health** 标签页：缓存校验、丢失、栈覆盖率、符号解析、I/O 可用性
- **I/O context** 标签页：显示整个采集窗口，独立于源/线程过滤
- Android 的 **Open in Perfetto** 按钮会把本次 `faults.pftrace` 递给 Perfetto UI；
  想要一键加载，需要把采集目录和报告放在一起并**用 HTTP 服务**起来（本地 `file://` 只能手动打开）

### 读数注意

- Android 的每行缺页来自 perf 子系统**实际投递**的用户态事件，不是从
  `/proc/<pid>/stat` 的 `min_flt`/`maj_flt` 反推的，所以行数可能少于那些聚合内核计数器
- page-cache tracepoint 是另一回事：它记录的是缓存**插入**，不是 minor fault，**不计入缺页总数**
- major fault 数 ≠ 存储读次数，也不等于读入的全部页数（readahead、显式读、ART advice 会提前填充很多页）
- 现代 VDEX 可能完全不含 DEX payload，原始代码仍在 APK 的 DEX 条目里
- Simulator 上 Instruments 观察的是 macOS host 进程，其存储行为**不能当作物理设备行为**

## 6. Trace 分享

配了 `traceHostUrl` 后，每次采集完 CLI 会 multipart `POST` 到该端点，期望返回含 `id` 的 JSON，
然后回显 `GET /trace/<id>` 的完整 URL 供分享。没配就从本地磁盘打开。

本地起一个参考实现：

```bash
pip install --upgrade fastapi uvicorn
python3 scripts/trace_server.py          # 持久化到 /tmp/mperf，监听 8080
```

然后把 `traceHostUrl` 设成 `http://127.0.0.1:8080/trace`。

> Perfetto v54 起官方 UI 能直接打开公网 HTTPS trace URL，前提是该 host 允许未认证 GET 并对
> `https://ui.perfetto.dev` 开放 CORS。本地文件走 9001 端口的临时 loopback 服务（浏览器可能
> 提示授予本地网络权限）。**两条路径都不上传 trace。**

## 7. 验证与测试

### 本机就能全跑（不需要设备）

```bash
./gradlew build                 # 编译 + ktlint + JUnit + 报告前端回归 + 资源清单校验
./gradlew shadowJar             # fat JAR
./gradlew generateDocs          # 刷新 docs/cli.md
git diff --exit-code -- docs/cli.md   # CI 会这样校验文档没过期
./gradlew jmhClasses            # 编译基准
scripts/test-install.sh         # 安装/卸载脚本
node --test src/test/javascript/android_compatibility.test.cjs   # CI runner 安全检查
./gradlew testFaultViewer       # 仅跑报告前端回归（已含在 build 里）
```

单项命令（对应 AGENTS.md 的约定）：

| 命令 | 作用 |
|---|---|
| `./gradlew test` | 完整 JUnit 5 套件 |
| `./gradlew ktlintCheck` / `ktlintFormat` | 风格检查 / 自动修 |
| `./gradlew jmh` | Instruments→Gecko 转换基准（macOS + Xcode） |

**本机实测基线**（macOS / Apple Silicon，2026-09-20）：`./gradlew build` 成功，
48 个 suite / 256 tests / 0 failures / 10 skipped——10 个 skipped 全是下面这些设备门控的集成测试。

### 需要真机或模拟器

```bash
# iOS Simulator 端到端：自建 fixture app，跑 launch + attach 两种采集，
# 校验 trace 目录结构，转换 Time Profiler trace；结束后还原模拟器 boot 状态并删除 fixture
./gradlew test -Dmperf.integration.ios.enabled=true \
  --tests com.bromano.mobile.perf.integration.IosProfilerIntegrationTest
# 指定模拟器
#   -Dmperf.integration.ios.device=<SIMULATOR_UDID>

# Android 模拟器端到端（profilers 套件）
bash scripts/run-android-integration.sh emulator-5554 35 4096 emulator

# Android 模拟器端到端（faults 套件：严格零驻留、as-is 编译、APK 回收、I/O 证据、DWARF 覆盖）
bash scripts/run-android-integration.sh emulator-5554 35 4096 emulator faults
```

脚本签名：`run-android-integration.sh SERIAL EXPECTED_API EXPECTED_PAGE_SIZE [emulator|physical] [profilers|faults]`。
它会强制校验实际 API 级别、`getconf PAGE_SIZE`、`/proc/self/smaps` 的内核页大小，对不上直接失败——
**不会**静默降低覆盖面。

设 `MPERF_INTEGRATION_ARTIFACTS` 可保留本地集成产物。

### CI 分工

| Workflow | 触发 | 内容 |
|---|---|---|
| **CI - Build** | push main / PR | 构建 + 测试 + lint + 打包 + CLI 冒烟 + 文档新鲜度；外加单台 API 35 模拟器和 iOS Simulator 两个 job |
| **Android compatibility** | 每周 + 手动 | API 29/33/35/36 × 4KB 内核页；额外跑 `AndroidFaultCaptureIntegrationTest` |
| **Release** | 推 SemVer tag | 见 §9 |

失败的 job 会保留测试报告和 fixture 产物 7 天；iOS 重试会同时保留两次尝试。

> 托管的 x86_64 矩阵**不声称**覆盖 16KB 内核：`google_apis_ps16k` x86_64 镜像是在 4KB 内核上
> 模拟 16KB 用户态，不能验证本采集器的内核页索引和 perf ring 计算。真正的 16KB 测试需要
> ARM64 + 16KB 内核的目标。

## 8. 常见问题

**`java -jar` 报 UnsupportedClassVersionError**
PATH 上的 java 版本低于 21。`./gradlew` 不受影响（foojay 会自动用下载好的 JDK 21，
通常在 `~/.gradle/jdks/`），但直接跑 JAR 需要把 JDK 21 放进 PATH。

**`Android NDK not found; install one or set ANDROID_NDK_HOME`**
`faults android` 要现场编译 native 采集器。装一个 NDK，或设 `ANDROID_NDK_HOME`。
（CI 用的是钉死的 29.0.14206865。）纯重放 `--skip-collect` 不需要 NDK。

**冷缓存校验失败，但我确实清了缓存**
部分 Android 16 模拟器镜像在全局 drop + 文件级驱逐后仍会保留一小撮可重复的 APK 页。
mperf 选择失败而不是谎称冷启动。确认这点残留可接受时，再用一个明确的小容忍值：
`--max-resident-pages <N>`。

**`xctrace` 找不到 / Virtual Memory Trace 不存在**
需要完整 Xcode，不是 Command Line Tools。多个 Xcode 时用
`sudo xcode-select --switch /Applications/Xcode.app` 选中目标版本。

**iOS Simulator 采集产物里有别的进程**
这是已知的 Xcode 限制导致的兼容回退（见[原理文档 §3.5](architecture.md#35-instrumentsios)）。
转换后的 Firefox/Perfetto 输出已按目标 PID 过滤，但原始 `.trace` 是 host 级数据。

**采集报「integrity failure」**
`lost` / `integrity_errors` / `throttled` / `callchain_overflow` 任一非 0 就会判定整次采集失败。
DWARF 场景下内核环丢失要加 `--dwarf-kernel-pages`（只加 `--dwarf-user-buffer-mb` 修不了）。

**报告里的 major fault 数和 `/proc/<pid>/stat` 对不上**
预期行为，口径不同。见 §5「读数注意」。

## 9. 发布

- 推 SemVer tag（`v1.2.3` 或 `v1.2.3-rc.1`）触发发布
- 用仓库的 `$release-mperf` skill（[`.codex/skills/release-mperf`](../../.codex/skills/release-mperf/SKILL.md)）
  跑 preflight、打 tag、验证结果
- 本地单独跑 preflight：`.codex/skills/release-mperf/scripts/preflight.sh 1.2.3`
- 发布 workflow 要求**该 tag 对应的确切源码 commit** 在 `main` 上已通过 build / Android 模拟器 / iOS Simulator
  三个 CI job；设备测试在 CI 跑，不在发版时重跑
- 产物：`mperf-<version>-all.jar` + `.sha256`，带 GitHub artifact provenance 证明；
  带 prerelease 后缀的版本发为 prerelease
- 需要 `gh` 已认证且有 tag push 权限；仓库/组织策略要允许 `contents`、`id-token`、`attestations` 写权限
