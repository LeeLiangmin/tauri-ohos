# 02 - 鸿蒙平台技术基础

> 本文档回答：**鸿蒙提供了什么能力来支撑 Tauri？**
> 
> 前置阅读：[01-第一性原理分析](./01-first-principles.md) 中推导出 Tauri 需要"窗口 + WebView + IPC"三大能力

---

## 1. 鸿蒙技术栈全景

HarmonyOS NEXT 的技术栈可以分为四层，从上到下依次是：

```
┌─────────────────────────────────────────────────────────┐
│  应用层     ArkTS 应用 (.ets)  /  Web 应用 (HTML/JS)     │
├─────────────────────────────────────────────────────────┤
│  框架层     Ability Kit │ ArkUI │ ArkWeb │ 系统服务 Kit   │
├─────────────────────────────────────────────────────────┤
│  运行时层   ArkRuntime (方舟运行时) + ArkCompiler          │
├─────────────────────────────────────────────────────────┤
│  系统层     HarmonyOS 系统 API / N-API / 微内核           │
└─────────────────────────────────────────────────────────┘
```

对 Tauri 适配而言，我们只需要关注其中**与三大本质需求直接相关**的组件：

| Tauri 需求 | 鸿蒙组件 | 作用 |
|-----------|---------|------|
| **窗口** | Ability Kit + WindowStage | 应用生命周期 + 窗口管理 |
| **WebView** | ArkWeb | 基于 Chromium 的 Web 渲染 |
| **IPC** | N-API + javaScriptProxy | 原生桥接 + JS 桥接 |
| 构建/打包 | hvigor + HAP | 构建工具 + 包格式 |
| 设备连接 | hdc | 设备调试工具 |

---

## 2. Ability Kit — 应用的骨架

### 2.1 核心概念

Ability Kit 是鸿蒙应用的**必选基座**，类似 Android 的 Activity/Service 框架。核心概念：

| 概念 | Android 类比 | 说明 |
|------|-------------|------|
| **UIAbility** | Activity | 带 UI 的应用组件，是 Tauri 应用的入口 |
| **WindowStage** | Window | 窗口管理器，在此加载 UI 页面 |
| **AbilityContext** | Context | 应用上下文，获取信息、请求权限 |
| **Want** | Intent | 组件间通信载体 |

### 2.2 生命周期

UIAbility 的生命周期是 Tauri 必须正确对接的：

```
onCreate → onWindowStageCreate → onForeground ←→ onBackground → onDestroy
                  │
                  └─→ 在此处初始化 WebView 和 Rust 后端
```

**对 Tauri 的意义**：`onWindowStageCreate` 是创建窗口和 WebView 的时机，对应 TAO 的 `EventLoop` 初始化和 WRY 的 `WebView` 创建。

### 2.3 在 Tauri Demo 中的实际使用

```typescript
// EntryAbility.ets — tauri-demo 的入口
import { RustAbility } from '@ohos-rs/ability'

export default class EntryAbility extends RustAbility {
  public moduleName: string = "tauri_demo_lib"  // 指向 Rust cdylib
  public defaultPage: boolean = true;

  async onWindowStageCreate(windowStage) {
    const window = windowStage.getMainWindowSync();
    await window.setWindowLayoutFullScreen(false);
    super.onWindowStageCreate(windowStage);  // 转发到 Rust 层
  }
}
```

关键点：继承 `RustAbility`（来自 `@ohos-rs/ability`），生命周期事件自动转发到 Rust 层。

---

## 3. ArkWeb — WebView 引擎

### 3.1 核心特性

ArkWeb 是鸿蒙内置的 Web 渲染引擎，基于 Chromium：

| 特性 | 说明 | Tauri 用途 |
|------|------|-----------|
| W3C 标准支持 | HTML5/CSS3/ES2020+ | 渲染前端 UI |
| `javaScriptProxy` | 向 JS 注入原生对象 | 建立 IPC 桥梁 |
| `onInterceptRequest` | 拦截 Web 请求 | 自定义协议（`tauri://`、`asset://`） |
| `runJavaScript` | 执行 JS 代码 | Rust → JS 通信 |
| DevTools | 远程调试 | 开发调试 |

### 3.2 ArkWeb 在 ArkUI 中的使用方式

```typescript
@Component
struct TauriWebView {
  controller: WebviewController = new webview.WebviewController()

  build() {
    Web({ src: 'tauri://localhost', controller: this.controller })
      .javaScriptAccess(true)
      .javaScriptProxy({
        object: this.tauriBridge,
        name: "__TAURI_IPC__",           // Tauri IPC 入口
        methodList: ["postMessage"],
        controller: this.controller
      })
      .onInterceptRequest((event) => {
        // 拦截 tauri:// 和 asset:// 协议
        return this.handleCustomProtocol(event)
      })
  }
}
```

### 3.3 ArkWeb vs Android WebView

| 维度 | Android WebView | ArkWeb |
|------|----------------|--------|
| 内核 | Chromium (系统 WebView) | Chromium (ArkWeb) |
| JS 桥接 | `addJavascriptInterface` | `javaScriptProxy` |
| 请求拦截 | `shouldInterceptRequest` | `onInterceptRequest` |
| JS 执行 | `evaluateJavascript` | `runJavaScript` |
| UI 模型 | 命令式（`new WebView(ctx)`） | 声明式（`Web({ src, controller })`） |
| 进程模型 | 多进程 | 多进程 |

**核心差异**：ArkWeb 使用声明式 UI 模型，WebView 是组件树的一部分，不能像 Android 那样命令式创建。`openharmony-ability` crate 封装了这个差异。

---

## 4. N-API — Rust 与鸿蒙的桥梁

### 4.1 什么是 N-API

N-API（Node-API）是鸿蒙提供的**原生代码桥接接口**，兼容 Node.js N-API 规范。它允许 C/C++/Rust 代码与 ArkTS 代码互调。

```
Rust 代码 (.so 动态库)
    │
    │  N-API (napi-ohos crate)
    │
    ▼
ArkRuntime (执行 ArkTS 代码)
    │
    │  javaScriptProxy
    │
    ▼
ArkWeb V8 (执行前端 JS 代码)
```

### 4.2 N-API vs JNI

| 维度 | JNI (Android) | N-API (鸿蒙) |
|------|--------------|--------------|
| 规范来源 | Java 标准 | Node.js 标准 |
| 调用方式 | 反射 + 方法签名 | 函数注册 + 回调 |
| 类型系统 | Java 类型 | napi_value (动态类型) |
| 线程模型 | JNI 线程绑定 | 线程安全函数 (ThreadSafeFunction) |
| Rust 绑定 | `jni` crate | `napi-ohos` crate |
| 异步支持 | 需手动管理 | 原生 Promise 支持 |

### 4.3 在 Tauri 中的角色

N-API 在 Tauri OHOS 适配中承担两个关键角色：

1. **Rust ↔ ArkTS 通信**：Tauri 的 Rust 后端通过 N-API 调用 ArkTS 代码（如创建 WebView、操作窗口）
2. **动态库入口**：Rust 编译为 `.so` 动态库，通过 N-API 导出函数供 ArkTS 调用

---

## 5. 构建与打包体系

### 5.1 HAP — 鸿蒙的应用包格式

HAP（HarmonyOS Ability Package）是鸿蒙的部署单元，类似 Android 的 APK：

```
entry.hap
├── module.json5          # 模块配置（声明 Ability、权限）
├── ets/
│   └── modules.abc       # ArkTS 编译后的方舟字节码
├── resources/            # 资源文件
└── libs/
    └── arm64-v8a/
        └── libentry.so   # ← Rust 编译的动态库
```

### 5.2 构建工具链

| 工具 | Android 类比 | 说明 |
|------|-------------|------|
| **hvigor** | Gradle | 鸿蒙构建工具 |
| **DevEco Studio** | Android Studio | 鸿蒙 IDE |
| **hdc** | adb | 设备连接和调试 |
| **ohrs** | cargo-ndk | Rust OHOS 构建辅助 |

### 5.3 Tauri OHOS 的完整构建流程

```
tauri ohos build
    │
    ├─► 前端构建 (Vite/Webpack → dist/)
    │
    ├─► Rust 交叉编译
    │     rustc --target aarch64-unknown-linux-ohos
    │     → libentry.so
    │
    ├─► ArkTS 编译 (ArkCompiler → .abc)
    │
    └─► HAP 打包 (hvigor → entry.hap)
          └─► 签名 → 部署到设备 (hdc)
```

---

## 6. 交叉编译支持

### 6.1 Rust OHOS Target

Rust 官方已支持三个 OHOS target：

```bash
rustup target add aarch64-unknown-linux-ohos   # ARM64 (主流)
rustup target add armv7-unknown-linux-ohos      # ARM32
rustup target add x86_64-unknown-linux-ohos     # x86_64 (模拟器)
```

### 6.2 Windows 交叉编译配置

```toml
# .cargo/config.toml
[target.aarch64-unknown-linux-ohos]
linker = "path/to/ohos-sdk/native/llvm/bin/aarch64-unknown-linux-ohos-clang"
```

```bash
# 环境变量
set OHOS_NDK_HOME=path/to/ohos-sdk/native
set CC_aarch64_unknown_linux_ohos=%OHOS_NDK_HOME%/llvm/bin/aarch64-unknown-linux-ohos-clang.exe
set AR_aarch64_unknown_linux_ohos=%OHOS_NDK_HOME%/llvm/bin/llvm-ar.exe
```

### 6.3 可行性结论

| 环节 | Windows 上可行？ | 说明 |
|------|:---------------:|------|
| Rust 交叉编译 | ✅ | 官方支持 OHOS target |
| 前端构建 | ✅ | 平台无关 |
| ArkTS 编译 | ✅ | DevEco Studio 支持 Windows |
| HAP 打包 | ✅ | hvigor 支持 Windows |
| 设备部署 | ✅ | hdc 支持 Windows |

**结论：Windows 开发 + 交叉编译到 OHOS 完全可行**，与 Android 开发流程高度相似。

---

## 7. 开发环境要求

```
必需组件：
├── Rust 工具链 (rustup)
│   └── target: aarch64-unknown-linux-ohos
├── OHOS SDK / NDK
│   ├── native/ (C/C++ 工具链)
│   └── toolchains/ (构建工具)
├── DevEco Studio
│   ├── hvigorw (构建工具)
│   ├── hdc (设备连接)
│   └── 模拟器
├── Node.js (前端构建)
└── Tauri CLI (feat/open-harmony 分支)

可选组件：
├── 鸿蒙真机 (USB 调试)
└── ohrs (Rust OHOS 构建辅助)
```

---

## 8. 本章小结

| 问题 | 答案 |
|------|------|
| 鸿蒙能提供窗口吗？ | ✅ Ability Kit (UIAbility + WindowStage) |
| 鸿蒙能提供 WebView 吗？ | ✅ ArkWeb (Chromium 内核) |
| 鸿蒙能提供 IPC 桥梁吗？ | ✅ N-API + javaScriptProxy |
| 能在 Windows 上交叉编译吗？ | ✅ Rust 官方支持 + OHOS NDK |
| 与 Android 的差异大吗？ | 中等 — API 不同但模式相似，声明式 UI 是主要差异 |

**下一步**：了解 [Tauri 的架构和扩展机制](./03-tauri-architecture.md)，看它如何将这些平台能力接入框架。
