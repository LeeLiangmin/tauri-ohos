# 01 - 第一性原理分析：Tauri 适配鸿蒙的本质问题

## 1. 什么是第一性原理分析

第一性原理（First Principles）要求我们**剥离表象，回到最基本的事实**，然后从这些事实出发重新推导。对于"Tauri 适配鸿蒙"这个问题，我们不应该从"Tauri 现在支持哪些平台、怎么加一个"开始，而应该问：

> **Tauri 应用在任何平台上运行，本质上需要什么？**

---

## 2. Tauri 应用运行的三个本质需求

一个 Tauri 应用，无论运行在什么操作系统上，本质上只需要三样东西：

```
┌─────────────────────────────────────────────────────────┐
│                  Tauri 应用的本质需求                      │
│                                                         │
│   ① 一个窗口          ② 一个 WebView        ③ 一个桥梁   │
│   ┌─────────┐        ┌─────────────┐      ┌─────────┐  │
│   │ 承载应用 │        │ 渲染前端 UI  │      │ 连接前端 │  │
│   │ 的容器   │        │ HTML/CSS/JS │      │ 与 Rust  │  │
│   │         │        │             │      │ 后端     │  │
│   └─────────┘        └─────────────┘      └─────────┘  │
│                                                         │
│   TAO 负责             WRY 负责             IPC 负责     │
└─────────────────────────────────────────────────────────┘
```

### ① 窗口（Window）— 应用的容器

应用需要一个操作系统级别的窗口来承载内容。这个窗口需要：
- 能被创建、显示、隐藏、销毁
- 能接收用户输入（触摸、键盘、鼠标）
- 能响应系统事件（前后台切换、配置变更）
- 能提供一个"画布"给 WebView 渲染

### ② WebView（渲染引擎）— UI 的渲染器

应用需要一个能解析和渲染 HTML/CSS/JS 的引擎。这个引擎需要：
- 能加载和渲染 Web 内容
- 能执行 JavaScript
- 能拦截自定义协议请求（加载嵌入的前端资源）
- 能注入 JavaScript 代码（建立 IPC 桥梁）

### ③ IPC 桥梁（进程间通信）— 前后端的连接

前端 JS 和后端 Rust 需要一个通信通道。这个通道需要：
- JS → Rust：前端调用后端命令（Commands）
- Rust → JS：后端向前端发送事件（Events）
- 双向流式传输（Channels）

---

## 3. 从本质需求推导适配问题

有了这三个本质需求，我们可以问：**鸿蒙平台能否提供这三样东西？**

| 本质需求 | 鸿蒙提供了什么 | 能否满足 |
|---------|--------------|---------|
| **窗口** | Ability Kit（UIAbility + WindowStage） | ✅ 完全满足 |
| **WebView** | ArkWeb（基于 Chromium 的 Web 组件） | ✅ 完全满足 |
| **IPC 桥梁** | N-API（Rust↔ArkTS）+ javaScriptProxy（ArkTS↔JS） | ✅ 完全满足 |

**结论：鸿蒙平台具备支撑 Tauri 运行的所有基础能力。** 这不是"能不能做"的问题，而是"怎么做"的问题。

---

## 4. "怎么做"的本质：平台 API 映射

既然鸿蒙有能力，那适配的本质就是**将 Tauri 的抽象接口映射到鸿蒙的具体 API**：

```
Tauri 抽象层                    鸿蒙具体实现
─────────────                  ─────────────
TAO EventLoop          ──→     UIAbility 生命周期 + 事件分发
TAO Window             ──→     WindowStage + 窗口管理
WRY WebView            ──→     ArkWeb (Web 组件)
WRY IPC                ──→     javaScriptProxy + postMessage
WRY Custom Protocol    ──→     onInterceptRequest
Tauri CLI build        ──→     hvigorw + HAP 打包
设备连接                ──→     hdc (替代 adb)
```

这与 Tauri 适配 Android 的模式完全一致——只是映射的目标 API 不同：

| 抽象层 | Android 映射 | 鸿蒙映射 |
|-------|-------------|---------|
| 窗口管理 | Activity + JNI | UIAbility + N-API |
| WebView | Android WebView + JNI | ArkWeb + N-API |
| IPC | addJavascriptInterface | javaScriptProxy |
| 协议拦截 | shouldInterceptRequest | onInterceptRequest |
| 原生语言 | Kotlin/Java | ArkTS |
| FFI 机制 | JNI | N-API |
| 包格式 | APK/AAB | HAP |
| 构建工具 | Gradle | hvigor |
| 设备工具 | adb | hdc |

---

## 5. 关键差异点：鸿蒙不是"另一个 Android"

虽然映射模式相似，但鸿蒙有几个**本质性差异**需要特别注意：

### 5.1 Target Triple 的特殊性

```
Android:  aarch64-linux-android     → target_os = "android"
OHOS:     aarch64-unknown-linux-ohos → target_os = "linux", target_env = "ohos"
```

鸿蒙的 Rust target 中 `target_os` 是 `"linux"` 而非 `"ohos"`，平台检测必须使用 `cfg(target_env = "ohos")`。这是一个容易踩坑的细节。

### 5.2 声明式 UI vs 命令式 UI

```
Android (命令式):  new WebView(context)  → 直接创建对象
鸿蒙 (声明式):     Web({ src, controller }) → 在组件树中声明
```

Android 可以在任意时刻命令式地创建 WebView，而鸿蒙的 ArkUI 是声明式的——WebView 是组件树的一部分。这意味着不能简单"翻译" Android 的实现方式。

**实际解决方案**：`openharmony-ability` crate 封装了这个差异，提供了类似命令式的 `WebViewBuilder` API，内部处理了与 ArkUI 声明式系统的对接。

### 5.3 双运行时架构

```
Android:  Rust ←JNI→ JVM (单一 Java 运行时)
鸿蒙:     Rust ←N-API→ ArkRuntime ←JSBridge→ V8 (双运行时)
```

鸿蒙上存在两个 JavaScript 运行时：
- **ArkRuntime**：执行 ArkTS 宿主代码
- **V8**（ArkWeb 内部）：执行 Web 前端代码

Tauri 的 IPC 需要跨越 `JS(V8) → ArkTS(ArkRuntime) → Rust(Native)` 三层，比 Android 的 `JS → Java(JVM) → Rust` 多了一层运行时边界。

### 5.4 PC 场景的额外需求

鸿蒙 PC（2in1 设备）相比手机/平板，需要额外支持：
- 多窗口管理（创建、关闭、最小化、最大化、拖拽调整大小）
- 菜单栏和系统托盘
- 键盘快捷键
- 鼠标事件（右键菜单、悬停、滚轮）
- 文件拖放

这些在手机端的 Demo 中不需要，但对 PC 场景是必须的。

---

## 6. 推导出的工作分解

基于以上分析，适配工作可以从本质需求出发分解为：

```
                        Tauri 适配鸿蒙
                             │
              ┌──────────────┼──────────────┐
              │              │              │
         ① 窗口适配     ② WebView 适配   ③ IPC 适配
         (TAO 层)       (WRY 层)        (Runtime 层)
              │              │              │
         ┌────┴────┐    ┌────┴────┐    ┌────┴────┐
         │ 已完成   │    │ 已完成   │    │ 已完成   │
         │ 905 行   │    │ 280 行   │    │ 集成在   │
         │ richerfu │    │ richerfu │    │ WRY 中   │
         │ /tao     │    │ /wry     │    │         │
         └────┬────┘    └────┬────┘    └─────────┘
              │              │
         ┌────┴────┐    ┌────┴────┐
         │ 待完善   │    │ 待完善   │
         │ PC 窗口  │    │ 事件回调 │
         │ 多窗口   │    │ 资源清理 │
         │ 菜单栏   │    │ 导航控制 │
         └─────────┘    └─────────┘
              │
              ├── ④ 工具链适配 (CLI 层) — 部分完成
              │
              └── ⑤ 打包分发 (Bundler 层) — 部分完成
```

---

## 7. 核心洞察总结

1. **Tauri 适配任何平台的本质**是提供"窗口 + WebView + IPC"三大能力，鸿蒙完全具备这些基础
2. **适配的本质是 API 映射**，与 Android 适配模式一致，只是目标 API 不同
3. **社区已完成核心映射**（richerfu 的 fork），从"能不能做"变成了"做得好不好"
4. **PC 适配是新增挑战**，手机端已验证的能力需要扩展到桌面场景
5. **双运行时是鸿蒙特有的复杂性**，IPC 链路比 Android 多一层，但 `openharmony-ability` 已封装了这个复杂性
