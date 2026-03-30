# 03 - Tauri 架构与平台扩展机制

> 本文档回答：**Tauri 如何支持多平台？新增平台的扩展点在哪？**
> 
> 前置阅读：[02-鸿蒙平台技术基础](./02-harmonyos-platform.md)

---

## 1. Tauri 的本质：一个分层的平台抽象框架

Tauri 的核心设计思想是**分层抽象**——将"平台相关"和"平台无关"的代码严格分离。理解这个分层是理解适配工作的关键。

```
┌─────────────────────────────────────────────────────────┐
│                    Tauri Application                     │
├─────────────────────────────────────────────────────────┤
│  ┌─────────────────┐         ┌────────────────────┐    │
│  │   Frontend       │  IPC   │   Rust Backend      │    │
│  │   (HTML/CSS/JS)  │◄──────►│   (Commands/Plugins)│    │  ← 开发者代码
│  └────────┬─────────┘        └─────────┬──────────┘    │
├───────────┼────────────────────────────┼───────────────┤
│           │                            │               │
│  ┌────────┴────────────────────────────┴──────────┐    │
│  │              tauri (主 crate)                    │    │  ← 平台无关
│  │  App 生命周期 / IPC / 插件系统 / 安全权限         │    │
│  └──────────────────┬─────────────────────────────┘    │
│                     │                                   │
│  ┌──────────────────┴─────────────────────────────┐    │
│  │           tauri-runtime (抽象接口)               │    │  ← 抽象层
│  │  定义 Runtime trait / 窗口 trait / WebView trait  │    │
│  └──────────────────┬─────────────────────────────┘    │
│                     │                                   │
│  ┌──────────────────┴─────────────────────────────┐    │
│  │        tauri-runtime-wry (实现层)               │    │  ← 桥接层
│  │  实现 Runtime trait，桥接 TAO + WRY              │    │
│  └───────┬──────────────────────────────┬─────────┘    │
│          │                              │              │
│  ┌───────┴────────┐          ┌─────────┴──────────┐   │
│  │     TAO         │          │      WRY            │   │  ← 平台抽象
│  │  (窗口管理)     │          │  (WebView 渲染)     │   │
│  └───────┬─────────┘          └─────────┬──────────┘   │
│          │                              │              │
│  ┌───────┴──────────────────────────────┴──────────┐   │
│  │              OS Native APIs                      │   │  ← 平台实现
│  │  Windows: Win32 + WebView2                       │   │
│  │  macOS:   Cocoa + WKWebView                      │   │
│  │  Linux:   GTK + WebKitGTK                        │   │
│  │  Android: JNI/NDK + Android WebView              │   │
│  │  iOS:     UIKit + WKWebView                      │   │
│  │  OHOS:    N-API + ArkWeb  ← 新增                 │   │
│  └────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────┘
```

**关键洞察**：新增平台只需要在最底部两层（TAO + WRY）实现平台特定代码，上层通过抽象接口自动适配。这就是 Tauri 的**平台扩展机制**。

---

## 2. 核心 Crate 职责

### 2.1 平台无关层

| Crate | 职责 | 适配时需要改动？ |
|-------|------|:---------------:|
| `tauri` | 主框架：App 生命周期、IPC、插件、安全权限 | 少量（`cfg` 条件编译） |
| `tauri-build` | 编译时代码生成 | 少量 |
| `tauri-codegen` | 资源嵌入、配置解析 | 无 |
| `tauri-macros` | 过程宏 | 无 |
| `tauri-utils` | 通用工具 | 少量（平台检测） |
| `@tauri-apps/api` | 前端 JS API | 无 |

### 2.2 平台抽象层

| Crate | 职责 | 适配时需要改动？ |
|-------|------|:---------------:|
| `tauri-runtime` | 定义 Runtime/Window/WebView trait | 无（纯接口定义） |
| `tauri-runtime-wry` | 实现 Runtime trait，桥接 TAO+WRY | 少量（平台分支） |

### 2.3 平台实现层（适配重点）

| Crate | 职责 | 适配时需要改动？ |
|-------|------|:---------------:|
| **TAO** | 跨平台窗口管理（fork 自 winit） | **核心**（新增 `ohos/` 模块） |
| **WRY** | 跨平台 WebView 渲染 | **核心**（新增 `ohos/` 模块） |

### 2.4 工具链层

| Crate | 职责 | 适配时需要改动？ |
|-------|------|:---------------:|
| `tauri-cli` | CLI 工具（init/dev/build） | **需要**（新增 `ohos` 子命令） |
| `tauri-bundler` | 应用打包 | **需要**（新增 HAP 打包） |
| `cargo-mobile2` | 移动端脚手架 | **需要**（新增 OHOS 支持） |

---

## 3. 平台扩展的四个接入点

新增一个平台，需要在以下四个层次实现代码：

### 接入点 1：TAO — 窗口管理

TAO 通过 `#[cfg]` 条件编译选择平台实现：

```rust
// vendor/tao/src/platform_impl/mod.rs
#[cfg(target_os = "windows")]   → windows/mod.rs
#[cfg(target_os = "linux")]     → linux/mod.rs
#[cfg(target_os = "macos")]     → macos/mod.rs
#[cfg(target_os = "android")]   → android/mod.rs
#[cfg(target_os = "ios")]       → ios/mod.rs
#[cfg(target_env = "ohos")]     → ohos/mod.rs    // ← 新增
```

需要实现的核心接口：

| 接口 | 说明 | 鸿蒙映射 |
|------|------|---------|
| `EventLoop<T>` | 事件循环 | UIAbility 生命周期 + `poll_events()` |
| `Window` | 窗口创建和管理 | WindowStage |
| `MonitorHandle` | 显示器信息 | 屏幕 API |
| `EventLoopProxy<T>` | 跨线程事件发送 | `mpsc::Sender` |
| `raw_window_handle` | 原生窗口句柄 | `OhosNdkWindowHandle` |

### 接入点 2：WRY — WebView 渲染

WRY 同样通过 `cfg` 选择平台实现：

```rust
// vendor/wry/src/lib.rs
cfg(target_os = "windows")      → webview2/
cfg(target_os = "linux")        → webkitgtk/
cfg(target_vendor = "apple")    → wkwebview/
cfg(target_os = "android")      → android/
cfg(target_env = "ohos")        → ohos/          // ← 新增
```

需要实现的核心接口：

| 接口 | 说明 | 鸿蒙映射 |
|------|------|---------|
| `InnerWebView` | WebView 实例 | ArkWeb (Web 组件) |
| WebView 创建 | 创建并配置 WebView | `WebViewBuilder` |
| IPC 通道 | JS ↔ Rust 消息传递 | `javaScriptProxy` + `postMessage` |
| Custom Protocol | 自定义协议处理 | `onInterceptRequest` |
| JS 执行 | 在 WebView 中执行 JS | `runJavaScript` |

### 接入点 3：tauri-runtime-wry — Runtime 桥接

需要添加平台特定的条件编译分支：

```rust
// 窗口创建逻辑中的平台分支
#[cfg(target_env = "ohos")]
{
    // OHOS 特定的窗口初始化
}
```

### 接入点 4：tauri-cli — 构建工具

需要新增 OHOS 子命令：

```
tauri ohos init   → 初始化 OHOS 项目结构
tauri ohos dev    → 开发模式（热重载）
tauri ohos build  → 构建 HAP 包
```

---

## 4. 条件编译策略

Tauri 使用 Rust 的 `cfg` 系统实现多平台支持。对 OHOS 的关键配置：

### 4.1 平台检测

```rust
// ⚠️ 重要：OHOS 的 target_os 是 "linux"，不是 "ohos"
// 必须使用 target_env 来区分
#[cfg(target_env = "ohos")]           // ✅ 正确
#[cfg(target_os = "ohos")]            // ❌ 错误，不存在这个值
```

### 4.2 平台分组

Tauri 将平台分为桌面和移动两组，OHOS 归入移动组：

```rust
// 移动平台 = Android + iOS + OHOS
#[cfg(any(
    target_os = "android",
    target_env = "ohos",
    all(target_vendor = "apple", not(target_os = "macos"))
))]

// 桌面平台（排除 OHOS）
#[cfg(all(
    any(target_os = "linux", target_os = "windows", target_os = "macos"),
    not(target_env = "ohos")
))]
```

### 4.3 依赖管理

```toml
# OHOS 专属依赖
[target.'cfg(target_env = "ohos")'.dependencies]
openharmony-ability = { version = "0.2" }
openharmony-ability-derive = { version = "0.2" }

# 桌面特性排除 OHOS（菜单栏、系统托盘等）
[target.'cfg(all(any(target_os = "linux", target_os = "windows", target_os = "macos"), not(target_env = "ohos")))'.dependencies]
muda = { ... }      # 菜单栏
tray-icon = { ... } # 系统托盘
```

---

## 5. Android 适配作为参照

理解 Tauri 如何适配 Android，有助于理解 OHOS 适配的模式：

### 5.1 Android 的实现结构

```
TAO Android:
├── mod.rs          1340 行  (EventLoop + Window + 事件处理)
└── ndk_glue.rs      774 行  (NDK 生命周期胶水)
                    ─────
                    2114 行

WRY Android:
├── mod.rs           530 行  (核心逻辑 + Handler 注册)
├── binding.rs       483 行  (JNI 绑定层)
└── main_pipe.rs     607 行  (Rust↔Android UI 线程消息管道)
                    ─────
                    1620 行

Kotlin 胶水代码:
├── WryActivity.kt
├── RustWebView.kt
├── RustWebChromeClient.kt
├── RustWebViewClient.kt
└── Ipc.kt
```

### 5.2 Android 的关键设计模式

1. **MainPipe 消息管道**：Rust 线程和 Android UI 线程之间的异步消息传递
2. **Handler 注册机制**：5 个全局 static handler（IPC、REQUEST、TITLE_CHANGE 等）
3. **JNI 绑定宏**：`android_binding!` 宏自动生成 JNI 绑定代码
4. **JniHandle 逃生舱**：允许上层直接执行 JNI 代码

### 5.3 OHOS 适配的对应模式

| Android 模式 | OHOS 对应 | 差异 |
|-------------|----------|------|
| MainPipe | 无（委托给 openharmony-ability） | OHOS 实现更简洁 |
| Handler 注册 | 无（委托给 openharmony-ability） | 功能更少但更简单 |
| JNI 绑定宏 | N-API 绑定（napi-ohos） | API 风格不同 |
| JniHandle | 无等效机制 | 缺少底层逃生舱 |
| Kotlin 胶水 | ArkTS 胶水（@ohos-rs/ability） | 代码量更少 |

---

## 6. 运行时工作流程

### 6.1 编译时

```
tauri-build (build.rs)
    │
    ├─► 读取 tauri.conf.json
    ├─► tauri-codegen 嵌入前端资源
    ├─► tauri-macros 生成 IPC 处理器
    └─► Rust 编译 → .so (OHOS) / .so (Android) / .exe (Windows)
```

### 6.2 运行时

```
应用启动
  │
  ├─► OHOS: UIAbility.onCreate() → RustAbility 转发到 Rust
  │
  ├─► 初始化 Runtime (TAO EventLoop)
  │     └─► OHOS: OpenHarmonyApp.poll_events()
  │
  ├─► 创建窗口 (TAO Window)
  │     └─► OHOS: WindowStage.getMainWindow()
  │
  ├─► 创建 WebView (WRY)
  │     └─► OHOS: WebViewBuilder → ArkWeb
  │
  ├─► 注入 IPC 脚本
  │     └─► OHOS: javaScriptProxy("__TAURI_IPC__")
  │
  ├─► 加载前端内容
  │     ├─► 开发模式: devServer URL
  │     └─► 生产模式: tauri:// 自定义协议
  │
  └─► 进入事件循环
        ├─► 窗口事件 (TAO)
        ├─► WebView 事件 (WRY)
        ├─► IPC 消息 (Commands/Events)
        └─► 插件事件
```

---

## 7. 本章小结

| 问题 | 答案 |
|------|------|
| Tauri 如何支持多平台？ | 分层抽象：平台无关层 → 抽象接口层 → 平台实现层 |
| 新增平台需要改哪里？ | 主要改 TAO（窗口）和 WRY（WebView），少量改 tauri core 和 CLI |
| 改动量有多大？ | Android 参照：TAO ~2100 行 + WRY ~1600 行 + Kotlin 胶水 |
| OHOS 与 Android 适配模式相同吗？ | 模式相同，但 OHOS 大量委托给 `openharmony-ability`，代码更简洁 |
| 平台检测有什么坑？ | `target_env = "ohos"` 而非 `target_os = "ohos"` |

**下一步**：了解 [每一层的具体适配状态](./04-adaptation-analysis.md)——需要做什么、已经做了什么。
