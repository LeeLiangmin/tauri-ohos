# 04 - 适配层分析：需要做什么、已经做了什么

> 本文档回答：**Tauri 适配鸿蒙的每一层，需要做什么？社区已经做了什么？**
> 
> 前置阅读：[03-Tauri 架构与扩展机制](./03-tauri-architecture.md) 中的四个接入点

---

## 1. 适配全景图

```
层次                    需要做什么              谁做了             
────                   ──────────             ──────             
Layer 1: Rust 工具链    OHOS target 支持       Rust 官方           
Layer 2: 系统胶水层     Ability 生命周期管理    harmony-contrib     
Layer 3: TAO 窗口管理   EventLoop + Window     richerfu/tao        
Layer 4: WRY WebView   WebView + IPC          richerfu/wry        
Layer 5: Tauri Core    条件编译 + 模块适配      richerfu/tauri      
Layer 6: CLI 工具链     ohos build/dev 命令    richerfu/tauri     
Layer 7: 打包分发       HAP 打包 + 签名        部分完成        
```

---

## 2. Layer 1：Rust 工具链 — ✅ 已完成

**需要做什么**：Rust 编译器能生成 OHOS 平台的二进制代码。

**已完成**：Rust 官方已支持三个 OHOS target：

```bash
rustup target add aarch64-unknown-linux-ohos   # ARM64
rustup target add armv7-unknown-linux-ohos      # ARM32
rustup target add x86_64-unknown-linux-ohos     # x86_64 (模拟器)
```

**工作量**：0（已完成，直接使用）

---

## 3. Layer 2：系统胶水层 

**需要做什么**：提供 Rust 与鸿蒙系统之间的桥接，类似 Android 的 `android-activity` crate。

**已完成**：`openharmony-ability` + `@ohos-rs/ability` 提供了：

| 功能 | 状态 | 说明 |
|------|:----:|------|
| Ability 生命周期管理 | ✅ | onCreate/onForeground/onBackground/onDestroy |
| XComponent 原生渲染 | ✅ | 支持 NativeWindow |
| 输入事件（触摸、键盘） | ✅ | TouchEvent + KeyEvent |
| IME 输入法事件 | ✅ | 软键盘交互 |
| NativeWindow 句柄 | ✅ | 用于 WRY 创建 WebView |
| WebView 高层抽象 | ✅ | `WebViewBuilder` + `Webview` + `WebProxyBuilder` |
| 配置信息查询 | ✅ | 屏幕尺寸、DPI 等 |

**关键仓库**：
- Rust 侧：[harmony-contrib/openharmony-ability](https://github.com/harmony-contrib/openharmony-ability)
- ArkTS 侧：`@ohos-rs/ability` (npm 包，v0.4.0-beta.0)

**剩余工作**：可能需要根据实际使用补充少量 API（约 5-10 人天）

---

## 4. Layer 3：TAO 窗口管理

**需要做什么**：实现 TAO 的 `EventLoop`、`Window`、`MonitorHandle` 等核心接口。

**已完成**：richerfu/tao fork（`feat-ohos-webview` 分支）实现了 1281 行代码：

### 4.1 文件结构

```
vendor/tao-ohos/src/platform_impl/ohos/
├── mod.rs          904 行  (EventLoop + Window + 事件处理)
└── keycodes.rs     377 行  (键码映射表)
```

### 4.2 功能矩阵

| 功能 | 状态 | 实现方式 |
|------|:----:|---------|
| EventLoop（事件循环） | ✅ | `OpenHarmonyApp.poll_events()` |
| Window（窗口管理） | ✅ | `OpenHarmonyApp.native_window()` |
| MonitorHandle（显示器信息） | ✅ | 屏幕 API |
| 触摸事件 | ✅ | `InputEvent::TouchEvent` |
| 键盘事件 | ✅ | `InputEvent::KeyEvent` + keycodes 映射 |
| IME 输入法 | ✅ | `ImeEvent` |
| 窗口生命周期 | ✅ | `MainEvent::InitWindow/TerminateWindow` |
| 窗口大小/缩放 | ✅ | `content_rect()` + `config()` |
| EventLoopProxy（跨线程） | ✅ | `mpsc::Sender` |
| raw_window_handle | ✅ | `OhosNdkWindowHandle` |
| 多窗口 | ❌ | 未实现（PC 需要） |
| 菜单栏 | ❌ | 未实现（PC 需要） |
| 系统托盘 | ❌ | 未实现（PC 需要） |
| 光标/拖拽 | ❌ | 标记为 `NotSupported` |

### 4.3 入口点模式

```rust
#[cfg(target_env = "ohos")]
use openharmony_ability_derive::ability;

#[ability]
fn openharmony_app(app: OpenHarmonyApp) {
    // 构建 EventLoop 并运行
}
```

**剩余工作**：PC 适配（多窗口、菜单栏等），约 5-10 人天

---

## 5. Layer 4：WRY WebView

**需要做什么**：实现 WRY 的 `InnerWebView`，包括 WebView 创建、IPC、自定义协议、JS 执行。

**已完成**：richerfu/wry fork 实现了 279 行代码：

### 5.1 文件结构

```
vendor/wry-ohos/src/ohos/
└── mod.rs          279 行  (全部逻辑)
```

### 5.2 架构特点

与 Android 的 WRY 实现（1620 行，3 个文件）不同，OHOS 实现**高度委托**给 `openharmony-ability`：

```
Android WRY:  Rust → MainPipe → JNI 绑定 → Kotlin → Android WebView
OHOS WRY:     Rust → openharmony-ability API → ArkTS → ArkWeb
```

这使得 WRY OHOS 代码量仅为 Android 的 17%，但也意味着对底层的控制力较弱。

### 5.3 功能矩阵

| 功能 | 状态 | 实现方式 |
|------|:----:|---------|
| WebView 创建 | ✅ | `openharmony_ability::WebViewBuilder` |
| URL 加载 | ✅ | `webview.load_url()` |
| HTML 加载 | ✅ | `webview.load_html()` |
| 带 Headers 加载 | ✅ | `webview.load_url_with_headers()` |
| IPC 通信 | ✅ | `WebProxyBuilder` + `postMessage` |
| 自定义协议 | ✅ | `webview.custom_protocol_async()` |
| JS 执行 + 回调 | ✅ | `webview.evaluate_script_with_callback()` |
| 初始化脚本 | ✅ | `WebViewBuilder.initialization_scripts()` |
| Cookie 管理 | ✅ | `webview.cookies_with_url()` |
| 页面刷新 | ✅ | `webview.reload()` |
| 缩放控制 | ✅ | `webview.set_zoom()` |
| 透明度/背景色 | ✅ | `WebViewBuilder.transparent()` / `WebViewStyle` |
| User Agent | ✅ | `WebViewBuilder.user_agent()` |
| DevTools | ✅ | 编译时开关 |
| 焦点控制 | ✅ | `webview.focus()` |
| 清除浏览数据 | ✅ | `webview.clear_all_browsing_data()` |
| **navigation_handler** | **❌** | **未实现** |
| **on_page_load_handler** | **❌** | **未实现** |
| **document_title_changed** | **❌** | **未实现** |
| **WebView 销毁** | **❌** | **无显式销毁逻辑** |
| 打印 | ⚠️ | 空实现 |
| Bounds 设置 | ⚠️ | 空实现 |
| 可见性控制 | ⚠️ | 空实现 |
| WebView 版本查询 | ⚠️ | 硬编码 "1.0.0" |

**剩余工作**：补全事件回调、销毁逻辑、空实现，约 5-10 人天

---

## 6. Layer 5：Tauri Core 

**需要做什么**：在 Tauri 主框架中添加 OHOS 平台的条件编译和模块。

**已完成**：richerfu/tauri fork（`feat/open-harmony` 分支），改动规模：361 files changed, 10713 insertions(+), 15508 deletions(-)

### 6.1 核心改动

| 改动 | 说明 |
|------|------|
| `crates/tauri/src/ohos.rs` | OHOS 模块，管理 `OpenHarmonyApp` 全局状态 |
| 条件编译 | `cfg(target_env = "ohos")` 贯穿各 crate |
| 移动端分组 | OHOS 归入 mobile 类别 |
| 桌面特性排除 | 菜单栏、系统托盘在 OHOS 上不可用 |
| 环境变量 | `WRY_OHOS_PACKAGE`、`TAURI_OHOS_PROJECT_PATH` 等 |

### 6.2 OHOS 模块

```rust
// crates/tauri/src/ohos.rs
use std::sync::Mutex;
pub use openharmony_ability;
pub use openharmony_ability_derive;
pub static APP: Mutex<Option<openharmony_ability::OpenHarmonyApp>> = Mutex::new(None);
```

**剩余工作**：与最新 dev 分支同步、完善插件系统和权限管理，约 5-10 人天

---

## 7. Layer 6：CLI 工具链

**需要做什么**：实现 `tauri ohos init/dev/build` 命令。

**已完成**：feat/open-harmony 分支在 `crates/tauri-cli/src/mobile/` 下新增了 `open_harmony/` 模块：

| 文件 | 功能 |
|------|------|
| `open_harmony/mod.rs` | 平台入口，环境配置 |
| `open_harmony/dev.rs` | `tauri ohos dev` 命令 |
| `open_harmony/build.rs` | `tauri ohos build` 命令 |
| `open_harmony/project.rs` | DevEco Studio 项目生成 |
| `open_harmony/dev_eco_studio_script.rs` | DevEco Studio 集成脚本 |

同时集成了扩展的 `cargo-mobile2`：

```rust
use cargo_mobile2::{
    open_harmony::{
        config::Config as OpenHarmonyConfig,
        env::Env,
        hap,
        hdc,
        target::Target,
    },
};
```

**剩余工作**：完善 dev 模式热重载、优化构建流程，约 5-10 人天

---

## 8. Layer 7：打包分发

**需要做什么**：生成 HAP 包、签名、分发到应用商店。

**已完成**：
- ✅ DevEco Studio 项目模板生成
- ✅ hdc 设备检测和部署
- ✅ 基本的 HAP 构建流程

**未完成**：
- ❌ 自动化签名流程
- ❌ 应用商店分发适配
- ❌ CI/CD 集成

---

## 9. 完整依赖链

所有层次的依赖关系：

```
tauri-demo (应用层)
│
├── tauri (richerfu/tauri feat/open-harmony)        ← Layer 5
│   ├── tauri-runtime                                ← 抽象层
│   ├── tauri-runtime-wry                            ← 桥接层
│   │   ├── wry (richerfu/wry rev:814340e)           ← Layer 4
│   │   └── tao (richerfu/tao feat-ohos-webview)     ← Layer 3
│   ├── tauri-macros
│   └── tauri-utils
│
├── openharmony-ability                              ← Layer 2
├── openharmony-ability-derive                       ← Layer 2
├── napi-ohos (1.1)                                  ← Layer 1 (N-API 绑定)
└── napi-derive-ohos (1.1)                           ← Layer 1

ArkTS 侧：
├── @ohos-rs/ability (0.4.0-beta.0)                  ← Layer 2 (ArkTS)
└── libentry.so (N-API 原生模块)                      ← Layer 1
```

### Cargo.toml 配置

```toml
# tauri-demo/src-tauri/Cargo.toml
[dependencies]
tauri = { git = "https://github.com/richerfu/tauri", branch = "feat/open-harmony" }
napi-ohos = { version = "1.1" }
napi-derive-ohos = "1.1"

[patch.crates-io]
wry = { git = "https://github.com/richerfu/wry", rev = "814340e" }
tao = { git = "https://github.com/richerfu/tao", branch = "feat-ohos-webview" }
openharmony-ability = { git = "https://github.com/harmony-contrib/openharmony-ability.git" }
openharmony-ability-derive = { git = "https://github.com/harmony-contrib/openharmony-ability.git" }
tauri-runtime = { git = "https://github.com/richerfu/tauri", branch = "feat/open-harmony" }
tauri-utils = { git = "https://github.com/richerfu/tauri", branch = "feat/open-harmony" }
tauri-macros = { git = "https://github.com/richerfu/tauri", branch = "feat/open-harmony" }
tauri-runtime-wry = { git = "https://github.com/richerfu/tauri", branch = "feat/open-harmony" }
```

> ⚠️ 注意：9 个 crate 全部通过 Git patch 覆盖，版本锁定在特定 commit/branch。

---

## 10. 已验证的运行效果

tauri-demo 项目已在鸿蒙设备上成功运行，验证了：

- ✅ WebView 正常渲染 HTML/CSS/JS（Tauri + Vite + Vue 欢迎页面）
- ✅ Rust IPC 通信正常（输入 → 调用 Rust `greet` 命令 → 返回结果）
- ✅ 图片资源正常加载
- ✅ 用户交互正常（输入框、按钮点击）
- ✅ 支持设备类型：phone, tablet, 2in1（module.json5 声明）
- ✅ 最低 SDK：5.0.0 (API 12)

---

## 11. 本章小结

| 层次 | 完成度 | 核心贡献者 | 剩余工作 |
|------|:------:|-----------|---------|
| Rust 工具链 | 100% | Rust 官方 | 无 |
| 系统胶水层 | 90% | harmony-contrib | 补充 API |
| TAO 窗口管理 | 90% | richerfu | PC 适配 |
| WRY WebView | 85% | richerfu | 事件回调、销毁逻辑 |
| Tauri Core | 80% | richerfu | 同步上游、插件系统 |
| CLI 工具链 | 75% | richerfu | 热重载、优化 |
| 打包分发 | 60% | 部分完成 | 签名、CI/CD |

**整体完成度：~80-85%**，核心链路已通，剩余为完善和 PC 适配。

**下一步**：[差距评估](./05-gap-assessment.md) — 从 Demo 到产品级还差什么？
