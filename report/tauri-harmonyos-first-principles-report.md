# Tauri 适配鸿蒙（HarmonyOS）全过程链路报告
> 报告日期：2026-03-30  
> 分析范围：Tauri v2 框架适配 HarmonyOS NEXT 全链路  
> 核心结论：**技术可行，核心链路已通，剩余为完善和 PC 适配**

---

## 一、第一性原理：Tauri 运行的本质需求

### 1.1 问题本质

**剥离表象，回到最基本的事实**：一个 Tauri 应用，无论运行在什么操作系统上，本质上只需要三样东西：

```
┌────────────────────────────────────────────────────────────┐
│                  Tauri 应用的本质需求                        │
│                                                            │
│   ① 窗口 (Window)      ② WebView        ③ IPC 桥梁         │
│   ┌──────────────┐    ┌─────────────┐   ┌──────────────┐   │
│   │  承载应用的   │    │ 渲染前端 UI │   │ 连接前端与   │   │
│   │  容器        │    │ HTML/CSS/JS │   │ Rust 后端    │   │
│   └──────────────┘    └─────────────┘   └──────────────┘   │
│                                                            │
│   TAO 负责              WRY 负责           Runtime 负责     │
└────────────────────────────────────────────────────────────┘
```

### 1.2 鸿蒙平台的能力映射

| 本质需求 | 鸿蒙提供了什么 | 能否满足 |
|---------|--------------|:-------:|
| **窗口** | Ability Kit（UIAbility + WindowStage） | ✅ 完全满足 |
| **WebView** | ArkWeb（基于 Chromium 的 Web 组件） | ✅ 完全满足 |
| **IPC 桥梁** | N-API（Rust↔ArkTS）+ javaScriptProxy（ArkTS↔JS） | ✅ 完全满足 |

**结论**：鸿蒙平台具备支撑 Tauri 运行的所有基础能力。

---

## 二、全过程链路：七层架构

Tauri 适配鸿蒙需要完成以下七层工作：

```
应用层
   │
   ├── Layer 7: 打包分发 ─────────── HAP 签名、应用商店
   │                                    ↑
   ├── Layer 6: CLI 工具链 ─────────── tauri ohos build/dev
   │                                    ↑
   ├── Layer 5: Tauri Core ─────────── 条件编译、平台抽象
   │                                    ↑
   ├── Layer 4: WRY WebView ────────── WebView + IPC + 自定义协议
   │                                    ↑
   ├── Layer 3: TAO 窗口管理 ───────── EventLoop + Window + 事件处理
   │                                    ↑
   ├── Layer 2: 系统胶水层 ─────────── Ability 生命周期、N-API 桥接
   │                                    ↑
   └── Layer 1: Rust 工具链 ────────── OHOS Target 支持
```

---

## 三、各层详细分析

### Layer 1: Rust 工具链 ✅ 已完成（0 人天）

**需要做什么**：Rust 编译器能生成 OHOS 平台的二进制代码。

**已完成**：Rust 官方已支持三个 OHOS target：

```bash
rustup target add aarch64-unknown-linux-ohos   # ARM64 (主流)
rustup target add armv7-unknown-linux-ohos      # ARM32
rustup target add x86_64-unknown-linux-ohos     # x86_64 (模拟器)
```

**技术要点**：
- `target_os` 是 `"linux"`，不是 `"ohos"`
- 平台检测必须使用 `cfg(target_env = "ohos")`

---

### Layer 2: 系统胶水层  90% 完成（5-10 人天）

**需要做什么**：提供 Rust 与鸿蒙系统之间的桥接，类似 Android 的 `android-activity` crate。

**已完成**：`openharmony-ability` + `@ohos-rs/ability` 提供了：

| 功能 | 状态 | 说明 |
|------|:----:|------|
| Ability 生命周期管理 | ✅ | onCreate/onForeground/onBackground/onDestroy |
| XComponent 原生渲染 | ✅ | 支持 NativeWindow |
| 输入事件（触摸、键盘） | ✅ | TouchEvent + KeyEvent |
| WebView 高层抽象 | ✅ | `WebViewBuilder` + `WebProxyBuilder` |
| 配置信息查询 | ✅ | 屏幕尺寸、DPI 等 |

**关键仓库**：
- Rust 侧：[harmony-contrib/openharmony-ability](https://github.com/harmony-contrib/openharmony-ability)
- ArkTS 侧：`@ohos-rs/ability` (npm 包，v0.4.0-beta.0)

**架构特点**：
```
Rust (.so) ←──N-API──→ ArkRuntime (ArkTS) ←──javaScriptProxy──→ V8 (Web JS)
```

鸿蒙特有的双运行时架构，比 Android 多一层运行时边界。

---

### Layer 3: TAO 窗口管理  90% 完成（5-10 人天）

**需要做什么**：实现 TAO 的 `EventLoop`、`Window`、`MonitorHandle` 等核心接口。

**已完成**：richerfu/tao fork（`feat-ohos-webview` 分支）实现了 **1,281 行**代码：

```
platform_impl/ohos/
├── mod.rs          904 行  (EventLoop + Window + 事件处理)
└── keycodes.rs     377 行  (键码映射表)
```

**功能矩阵**：

| 功能 | 状态 | 实现方式 |
|------|:----:|---------|
| EventLoop（事件循环） | ✅ | `OpenHarmonyApp.poll_events()` |
| Window（窗口管理） | ✅ | `OpenHarmonyApp.native_window()` |
| 触摸/键盘/IME 事件 | ✅ | `InputEvent` 系列 |
| raw_window_handle | ✅ | `OhosNdkWindowHandle` |
| **多窗口** | ❌ | **PC 需要** |
| **菜单栏** | ❌ | **PC 需要** |
| **系统托盘** | ❌ | **PC 需要** |

**与 Android 对比**：
- Android: 2,114 行 → OHOS: 1,281 行（**61%**）
- 核心功能完整，差距主要在 PC 特有功能

---

### Layer 4: WRY WebView 85% 完成（5-10 人天）

**需要做什么**：实现 WRY 的 `InnerWebView`，包括 WebView 创建、IPC、自定义协议、JS 执行。

**已完成**：richerfu/wry fork 实现了 **279 行**代码（仅为 Android 的 **17%**）：

**高度委托给 `openharmony-ability`**：
```
Android WRY:  Rust → MainPipe → JNI 绑定 → Kotlin → Android WebView  (1,620 行)
OHOS WRY:     Rust → openharmony-ability API → ArkTS → ArkWeb          (279 行)
```

**功能矩阵**：

| 功能 | 状态 | 实现方式 |
|------|:----:|---------|
| WebView 创建 | ✅ | `WebViewBuilder` |
| URL/HTML 加载 | ✅ | `webview.load_url/html` |
| **IPC 通信** | ✅ | `WebProxyBuilder` + `postMessage` |
| **自定义协议** | ✅ | `custom_protocol_async` |
| JS 执行 + 回调 | ✅ | `evaluate_script_with_callback` |
| 初始化脚本 | ✅ | `WebViewBuilder.initialization_scripts` |
| Cookie/UA/缩放 | ✅ | 完整支持 |
| **navigation_handler** | ❌ | **待实现** |
| **on_page_load_handler** | ❌ | **待实现** |
| **destroy_webview** | ❌ | **待实现（内存泄漏风险）** |

**关键差异**：
- Android: 命令式 UI (`new WebView(context)`)
- 鸿蒙: 声明式 UI (`Web({ src, controller })`)
- 解决方案：`WebViewBuilder` 封装了声明式系统的对接

---

### Layer 5: Tauri Core  80% 完成（5-10 人天）

**需要做什么**：在 Tauri 主框架中添加 OHOS 平台的条件编译和模块。

**已完成**：richerfu/tauri fork（`feat/open-harmony` 分支），改动规模：**361 files changed, 10,713 insertions(+), 15,508 deletions(-)**

**核心改动**：

| 文件/改动 | 说明 |
|-----------|------|
| `crates/tauri/src/ohos.rs` | OHOS 模块，管理 `OpenHarmonyApp` 全局状态 |
| 条件编译 | `cfg(target_env = "ohos")` 贯穿各 crate |
| 移动端分组 | OHOS 归入 mobile 类别 |
| 桌面特性排除 | 菜单栏、系统托盘在 OHOS 上不可用 |
| 环境变量 | `WRY_OHOS_PACKAGE`、`TAURI_OHOS_PROJECT_PATH` 等 |

**剩余工作**：
- 与最新 dev 分支同步
- 完善插件系统和权限管理

---

### Layer 6: CLI 工具链 ⚠️ 75% 完成（5-10 人天）

**需要做什么**：实现 `tauri ohos init/dev/build` 命令。

**已完成**：`crates/tauri-cli/src/mobile/open_harmony/` 模块：

| 文件 | 功能 |
|------|------|
| `mod.rs` | 平台入口，环境配置 |
| `dev.rs` | `tauri ohos dev` 命令 |
| `build.rs` | `tauri ohos build` 命令 |
| `project.rs` | DevEco Studio 项目生成 |

**命令对比**：

| 命令 | Android 官方 | OHOS Demo |
|------|:-----------:|:---------:|
| `init` | ✅ 自动生成 | ✅ 有项目模板 |
| `dev` | ✅ 热重载 | ⚠️ 基本可用 |
| `build` | ✅ 一键构建 | ✅ 可用 |
| 设备检测 | ✅ adb | ✅ hdc |
| **自动签名** | ✅ | ❌ **需手动** |

**剩余工作**：
- 完善 dev 模式热重载
- 实现自动化签名流程

---

### Layer 7: 打包分发  60% 完成（5-10 人天）

**需要做什么**：生成 HAP 包、签名、分发到应用商店。

**已完成**：
- ✅ DevEco Studio 项目模板生成
- ✅ hdc 设备检测和部署
- ✅ 基本的 HAP 构建流程

**未完成**：
- ❌ 自动化签名流程
- ❌ 应用商店分发适配
- ❌ CI/CD 集成

**HAP 包结构**：
```
entry.hap
├── module.json5          # 模块配置（声明 Ability、权限）
├── ets/modules.abc       # ArkTS 编译后的方舟字节码
├── resources/            # 资源文件
└── libs/arm64-v8a/
    └── libentry.so       # ← Rust 编译的动态库
```

---

## 四、完整依赖链

```
tauri-demo (应用层)
│
├── tauri (richerfu/tauri feat/open-harmony)        ← Layer 5
│   ├── tauri-runtime                                ← 抽象层
│   ├── tauri-runtime-wry                            ← 桥接层
│   │   ├── wry (richerfu/wry rev:814340e)           ← Layer 4
│   │   └── tao (richerfu/tao feat-ohos-webview)     ← Layer 3
│   └── ...
│
├── openharmony-ability                              ← Layer 2
├── napi-ohos (1.1)                                  ← Layer 1
│
ArkTS 侧:
├── @ohos-rs/ability (0.4.0-beta.0)                  ← Layer 2 (ArkTS)
└── libentry.so (N-API 原生模块)                      ← Layer 1
```

**关键配置**（Cargo.toml）：
```toml
[dependencies]
tauri = { git = "https://github.com/richerfu/tauri", branch = "feat/open-harmony" }
napi-ohos = { version = "1.1" }

[patch.crates-io]
wry = { git = "https://github.com/richerfu/wry", rev = "814340e" }
tao = { git = "https://github.com/richerfu/tao", branch = "feat-ohos-webview" }
openharmony-ability = { git = "https://github.com/harmony-contrib/openharmony-ability.git" }
```

> ⚠️ 9 个 crate 全部通过 Git patch 覆盖，版本锁定在特定 commit/branch。

---

## 五、与 Android 的对比分析

| 维度 | Android | OHOS | 说明 |
|------|---------|------|------|
| **窗口管理** | Activity + JNI | UIAbility + N-API | 模式相同，API 不同 |
| **WebView** | Android WebView | ArkWeb | 都基于 Chromium |
| **JS 桥接** | `addJavascriptInterface` | `javaScriptProxy` | API 名称不同 |
| **请求拦截** | `shouldInterceptRequest` | `onInterceptRequest` | API 名称不同 |
| **原生语言** | Kotlin/Java | ArkTS | 类型系统不同 |
| **FFI 机制** | JNI | N-API | 规范来源不同 |
| **包格式** | APK/AAB | HAP | 结构类似 |
| **构建工具** | Gradle | hvigor | 都是声明式构建 |
| **设备工具** | adb | hdc | 命令高度相似 |
| **UI 模型** | 命令式 | **声明式** | **核心差异** |
| **运行时** | 单一 JVM | **双运行时** | **鸿蒙特有复杂性** |

**代码量对比**：

| 层 | Android | OHOS | 比例 |
|----|:-------:|:----:|:----:|
| WRY | 1,620 行 | 279 行 | **17%** |
| TAO | 2,114 行 | 1,281 行 | **61%** |
| **合计** | **3,734 行** | **1,560 行** | **42%** |

---

## 六、关键差异点：鸿蒙不是"另一个 Android"

### 6.1 Target Triple 的特殊性

```
Android:  aarch64-linux-android       → target_os = "android"
OHOS:     aarch64-unknown-linux-ohos  → target_os = "linux", target_env = "ohos"
```

**陷阱**：`target_os` 是 `"linux"` 而非 `"ohos"`，平台检测必须用 `cfg(target_env = "ohos")`。

### 6.2 声明式 UI vs 命令式 UI

```typescript
// Android (命令式): 直接创建对象
val webView = WebView(context)
webView.loadUrl("https://example.com")

// 鸿蒙 (声明式): 在组件树中声明
Web({ src: 'https://example.com', controller: this.controller })
```

**解决方案**：`openharmony-ability` 提供了类似命令式的 `WebViewBuilder` API。

### 6.3 双运行时架构

```
Android:  Rust ←JNI→ JVM (单一 Java 运行时)
鸿蒙:     Rust ←N-API→ ArkRuntime ←JSBridge→ V8 (双运行时)
```

Tauri 的 IPC 需要跨越 `JS(V8) → ArkTS(ArkRuntime) → Rust(Native)` 三层。

### 6.4 PC 场景的额外需求

鸿蒙 PC（2in1 设备）相比手机/平板，需要额外支持：
- 多窗口管理（创建、关闭、最小化、最大化、拖拽调整大小）
- 菜单栏和系统托盘
- 键盘快捷键
- 鼠标事件（右键菜单、悬停、滚轮）
- 文件拖放

---

## 七、工作量评估

### 7.1 各层完成度

| 层次 | 完成度 | 核心贡献者 | 剩余工作 |
|------|:------:|-----------|---------|
| Layer 1: Rust 工具链 | 100% | Rust 官方 | 无 |
| Layer 2: 系统胶水层 | 90% | harmony-contrib | 补充 API |
| Layer 3: TAO 窗口管理 | 90% | richerfu | **PC 适配** |
| Layer 4: WRY WebView | 85% | richerfu | 事件回调、销毁逻辑 |
| Layer 5: Tauri Core | 80% | richerfu | 同步上游、插件系统 |
| Layer 6: CLI 工具链 | 75% | richerfu | 热重载、自动签名 |
| Layer 7: 打包分发 | 60% | 部分完成 | CI/CD、应用商店 |

**整体完成度：~80-85%**

### 7.2 剩余工作量估算

| 工作项 | 乐观 | 悲观 | 说明 |
|--------|:----:|:----:|------|
| 环境搭建 + Demo 复现 | 2天 | 3天 | 按 tauri-demo 流程 |
| WRY 完善 | 5天 | 10天 | 补全空实现 |
| TAO 完善 | 5天 | 10天 | 补全高级功能 |
| Tauri Core 完善 | 5天 | 10天 | 插件系统、权限管理 |
| CLI 工具链完善 | 5天 | 10天 | dev 模式、热重载 |
| **PC 适配** | **10天** | **20天** | **最大风险项** |
| 测试和文档 | 5天 | 10天 | — |
| **合计** | **37天** | **73天** | **1.5-3 个月** |



## 八、实施路线图

### 阶段 1：复现 Demo（1-2 周）

**目标**：在本地环境中成功运行 tauri-demo，理解完整架构和构建流程。

```bash
# 1. 安装工具
rustup target add aarch64-unknown-linux-ohos
cargo install tauri-cli --git https://github.com/tauri-apps/tauri --branch feat/open-harmony
cargo install ohrs

# 2. 克隆 demo
git clone https://github.com/richerfu/tauri-demo.git
cd tauri-demo && pnpm install

# 3. 构建并运行
cd src-tauri && cargo tauri ohos build
# 在 DevEco Studio 中打开 src-tauri/gen/ohos 并运行
```

### 阶段 2：完善功能（4-8 周）

**目标**：补齐 P0/P1 差距，达到功能完整。

| 优先级 | 任务 | 工作量 |
|:------:|------|:------:|
| **P0** | WebView 销毁逻辑 | 2-3 天 |
| **P0** | `navigation_handler` | 3-5 天 |
| P1 | `on_page_load_handler` | 3-5 天 |
| P1 | `document_title_changed` | 2-3 天 |
| P1 | CLI 热重载 | 3-5 天 |
| P2 | 与上游同步 | 3-5 天 |

### 阶段 3：PC 适配 + 发布（3-5 周）

**目标**：支持鸿蒙 PC（2in1 设备）的桌面特性。

| 任务 | 工作量 |
|------|:------:|
| 多窗口管理 | 3-5 天 |
| 窗口装饰 | 3-5 天 |
| 键盘快捷键 | 2-3 天 |
| 鼠标事件 | 2-3 天 |
| 测试套件 | 5-10 天 |
| 文档完善 | 3-5 天 |

---

## 九、已验证的运行效果

tauri-demo 项目已在鸿蒙设备上成功运行，验证了：

- ✅ **WebView 正常渲染** HTML/CSS/JS（Tauri + Vite + Vue 欢迎页面）
- ✅ **Rust IPC 通信正常**（输入 → 调用 Rust `greet` 命令 → 返回结果）
- ✅ **图片资源正常加载**
- ✅ **用户交互正常**（输入框、按钮点击）
- ✅ 支持设备类型：phone, tablet, 2in1（module.json5 声明）
- ✅ 最低 SDK：5.0.0 (API 12)

---

## 十、核心洞察总结

### 10.1 第一性原理推导的结论

1. **Tauri 适配任何平台的本质**是提供"窗口 + WebView + IPC"三大能力，鸿蒙完全具备这些基础。

2. **适配的本质是 API 映射**，与 Android 适配模式一致，只是目标 API 不同：
   ```
   Tauri 抽象层              鸿蒙具体实现
   ─────────────           ─────────────
   TAO EventLoop    ──→    UIAbility 生命周期 + 事件分发
   TAO Window       ──→    WindowStage + 窗口管理
   WRY WebView      ──→    ArkWeb (Web 组件)
   WRY IPC          ──→    javaScriptProxy + postMessage
   WRY Protocol     ──→    onInterceptRequest
   Tauri CLI build  ──→    hvigorw + HAP 打包
   ```

3. **社区已完成核心映射**（richerfu 的 fork），从"能不能做"变成了"做得好不好"。

4. **PC 适配是新增挑战**，手机端已验证的能力需要扩展到桌面场景。

5. **双运行时是鸿蒙特有的复杂性**，IPC 链路比 Android 多一层，但 `openharmony-ability` 已封装了这个复杂性。

### 10.2 关键数据

| 指标 | 数值 |
|------|:----:|
| 整体完成度 | **~80-85%** |
| 代码量（vs Android） | **42%**（1,560 / 3,734 行） |
| 核心链路 | ✅ **已通** |
| 预估剩余工作量 | **37-73 人天** |
| 推荐团队配置 | **2-3 人** |
| 预计周期 | **1.5-3 个月** |
| 最大风险项 | **PC 窗口管理** |

### 10.3 最终结论

> **Tauri 适配鸿蒙已从"技术验证"进入"产品化完善"阶段。**
> 
> 技术可行性已充分验证（核心 IPC + WebView + 自定义协议均可工作），交叉编译完全可行，Windows 可交叉编译。
> 
> 剩余工作主要是：
> 1. **补全功能缺口**（WebView 销毁、事件回调）
> 2. **完善工具链**（热重载、自动签名）
> 3. **PC 桌面适配**（多窗口、菜单栏、快捷键）
> 
> 建议基于现有 Demo 进行完善，不要从零开始。2-3 人团队可在 1.5-2 个月内完成手机/平板支持，3 个月内完成 PC 支持。

---

