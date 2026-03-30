# 05 - 差距评估：从 Demo 到产品级

> 本文档回答：**从已有的可运行 Demo 到产品级的官方平台支持，还差什么？**
> 
> 前置阅读：[04-适配层分析](./04-adaptation-analysis.md)

---

## 1. 评估基准

以 **Tauri 官方 Android 平台支持**为参照基准，从以下维度评估 OHOS Demo 的差距：

```
                    Android 官方    OHOS Demo
代码完整性          ████████████    ████░░░░░░░░  (~35%)
功能覆盖率          ████████████    ██████░░░░░░  (~55%)
工具链支持          ████████████    ██████░░░░░░  (~50%)
测试覆盖            ████████████    ░░░░░░░░░░░░  (~0%)
文档完善度          ████████████    █░░░░░░░░░░░  (~10%)
插件生态            ████████████    ██░░░░░░░░░░  (~15%)
错误处理            ████████████    ████░░░░░░░░  (~30%)
构建/分发           ████████████    ██░░░░░░░░░░  (~15%)
```

---

## 2. 代码量对比

### 2.1 WRY WebView 层

| 组件 | Android 官方 | OHOS Demo | 比例 |
|------|:-----------:|:---------:|:----:|
| 核心逻辑 (mod.rs) | 530 行 | 279 行 | 53% |
| 绑定层 (binding.rs) | 483 行 | — | 0% |
| 消息管道 (main_pipe.rs) | 607 行 | — | 0% |
| **合计** | **1,620 行** | **279 行** | **17%** |

> WRY OHOS 代码量仅为 Android 的 17%。差距的原因：大量功能委托给了 `openharmony-ability` crate。这既是优势（代码简洁）也是风险（对底层控制力弱）。

### 2.2 TAO 窗口管理层

| 组件 | Android 官方 | OHOS Demo | 比例 |
|------|:-----------:|:---------:|:----:|
| 核心逻辑 (mod.rs) | 1,340 行 | 904 行 | 67% |
| 胶水层 (ndk_glue.rs / keycodes.rs) | 774 行 | 377 行 | 49% |
| **合计** | **2,114 行** | **1,281 行** | **61%** |

> TAO 层相对完整（61%），因为窗口管理和事件循环是必须的基础设施。

### 2.3 总计

| 层 | Android | OHOS | 比例 |
|----|:-------:|:----:|:----:|
| WRY | 1,620 | 279 | 17% |
| TAO | 2,114 | 1,281 | 61% |
| **合计** | **3,734** | **1,560** | **42%** |

---

## 3. 功能差距分析

### 3.1 WRY 功能矩阵（OHOS vs Android）

#### ✅ 已实现（与 Android 同等）

| 功能 | 说明 |
|------|------|
| URL/HTML 加载 | 基础页面加载 |
| JS 执行 + 回调 | `evaluate_script_with_callback` |
| IPC 通信 | `WebProxyBuilder` + `postMessage` |
| 自定义协议 | `custom_protocol_async` |
| 初始化脚本 | 注入 Tauri IPC bridge |
| Cookie 管理 | `cookies_with_url` |
| 透明度/背景色 | 构建时设置 |
| User Agent | 自定义 UA |
| DevTools | 编译时开关 |

#### ❌ 未实现（Android 已有）

| 功能 | 影响 | 补齐工作量 |
|------|------|:---------:|
| `navigation_handler` | 无法拦截/控制页面导航（安全相关） | 3-5 天 |
| `on_page_load_handler` | 无法监听页面加载状态 | 3-5 天 |
| `document_title_changed` | 窗口标题无法动态更新 | 2-3 天 |
| `destroy_webview()` | **内存泄漏风险** | 2-3 天 |
| `platform_webview_version()` | 版本检测不准确（硬编码 "1.0.0"） | 1 天 |
| Asset Loader | 无本地资源加载器 | 3-5 天 |
| 运行时 `set_background_color()` | 仅构建时设置，运行时为空实现 | 1-2 天 |

#### ⚠️ 架构差异

| 方面 | Android 官方 | OHOS Demo |
|------|-------------|----------|
| 消息管道 | `MainPipe`（完整的跨线程消息系统） | 无（委托给 openharmony-ability） |
| Handler 基础设施 | 5 个全局 static handler | 无 |
| 底层逃生舱 | `JniHandle`（直接 JNI 访问） | 无等效机制 |
| WebView 创建 | 异步（通过 MainPipe） | 同步（WebViewBuilder） |
| 资源清理 | 完整的 `destroy_webview()` | **无显式销毁逻辑** |

### 3.2 TAO 功能矩阵

TAO 层功能基本与 Android 同等，差距主要在 **PC 特有功能**：

| 功能 | 手机/平板 | PC (2in1) | 状态 |
|------|:---------:|:---------:|:----:|
| EventLoop | 需要 | 需要 | ✅ |
| Window 创建 | 需要 | 需要 | ✅ |
| 触摸事件 | 需要 | 可选 | ✅ |
| 键盘事件 | 可选 | 需要 | ✅ |
| IME 输入法 | 需要 | 需要 | ✅ |
| 多窗口 | 不需要 | **需要** | ❌ |
| 窗口装饰 | 不需要 | **需要** | ❌ |
| 菜单栏 | 不需要 | **需要** | ❌ |
| 系统托盘 | 不需要 | 可选 | ❌ |
| 鼠标事件 | 不需要 | **需要** | ❌ |
| 键盘快捷键 | 不需要 | **需要** | ❌ |
| 文件拖放 | 不需要 | 可选 | ❌ |

---

## 4. 工具链差距

### 4.1 CLI 命令对比

| 命令 | Android 官方 | OHOS |
|------|:-----------:|:----:|
| `tauri android init` | ✅ 自动生成项目 | ✅ 有项目模板 |
| `tauri android dev` | ✅ 热重载开发 | ⚠️ 基本可用 |
| `tauri android build` | ✅ 一键构建 APK | ✅ `cargo tauri ohos build` |
| 设备检测 | ✅ adb | ✅ hdc |
| 模拟器管理 | ✅ | ⚠️ 基本支持 |
| 自动签名 | ✅ | ❌ 需手动 |

### 4.2 依赖管理差距

**Android 官方**：依赖来自 crates.io 正式发布版本
```toml
[dependencies]
tauri = "2.x"
```

**OHOS Demo**：9 个 crate 全部通过 Git patch 覆盖
```toml
[patch.crates-io]
wry = { git = "https://github.com/richerfu/wry", rev = "814340e..." }
tao = { git = "https://github.com/richerfu/tao", branch = "feat-ohos-webview" }
# ... 还有 7 个
```

> 这意味着无法通过 `cargo update` 正常更新，版本管理完全依赖手动操作。

### 4.3 开发体验差距

| 方面 | Android 官方 | OHOS Demo |
|------|-------------|----------|
| 项目创建 | `tauri android init` 一键 | 需 clone demo + 手动配置 |
| 开发模式 | `tauri android dev` 热重载 | 基本可用但不够完善 |
| 构建 | `tauri android build` 一键 | `cargo tauri ohos build` 可用 |
| 部署 | 自动部署到设备 | 需在 DevEco Studio 中手动运行 |

---

## 5. 质量保证差距

### 5.1 测试

| 方面 | Android 官方 | OHOS Demo |
|------|:-----------:|:---------:|
| 单元测试 | ✅ WRY 有 `mod tests` | ❌ 无 |
| 集成测试 | ✅ 完整测试套件 | ❌ 无 |
| CI/CD | ✅ GitHub Actions | ❌ 无 |
| 代码审查 | ✅ PR 审查流程 | ❌ 个人 fork |

### 5.2 错误处理

| 方面 | Android 官方 | OHOS Demo |
|------|-------------|----------|
| 错误类型 | 专门的错误类型定义 | 统一映射为 2 种错误 |
| 超时机制 | MainPipe 10 秒超时 | 无超时机制 |
| 错误传播 | JNI 调用完整错误传播 | 部分操作直接 `unwrap()` |

### 5.3 文档

| 方面 | Android 官方 | OHOS Demo |
|------|:-----------:|:---------:|
| API 文档 | ✅ docs.rs | ❌ 无 |
| 使用指南 | ✅ tauri.app 官方文档 | ⚠️ README 简要说明 |
| 架构文档 | ✅ ARCHITECTURE.md | ❌ 无 |
| 变更日志 | ✅ CHANGELOG | ❌ 无 |

---

## 6. 插件生态差距

Tauri v2 官方插件在 OHOS 上的兼容性：

| 插件 | 类型 | Android | OHOS | 说明 |
|------|------|:-------:|:----:|------|
| tauri-plugin-fs | 纯 Rust | ✅ | ⚠️ | 可能部分工作 |
| tauri-plugin-http | 纯 Rust | ✅ | ⚠️ | 可能部分工作 |
| tauri-plugin-opener | 需原生 API | ✅ | ❌ | 需 OHOS 适配 |
| tauri-plugin-shell | 需原生 API | ✅ | ❌ | 需 OHOS 适配 |
| tauri-plugin-notification | 需原生 API | ✅ | ❌ | 需 OHOS 适配 |
| tauri-plugin-dialog | 需原生 API | ✅ | ❌ | 需 OHOS 适配 |
| tauri-plugin-clipboard | 需原生 API | ✅ | ❌ | 需 OHOS 适配 |

> **规律**：纯 Rust 实现的插件可能直接工作，需要原生 API 的插件都需要专门适配。

---

## 7. 差距优先级排序

按**对用户体验的影响程度**排序：

| 优先级 | 差距 | 影响 | 工作量 |
|:------:|------|------|:------:|
| **P0** | WebView 销毁逻辑缺失 | 内存泄漏风险 | 2-3 天 |
| **P0** | PC 窗口管理（多窗口、装饰） | PC 场景不可用 | 10-20 天 |
| **P1** | `navigation_handler` 未实现 | 无法控制导航（安全） | 3-5 天 |
| **P1** | `on_page_load_handler` 未实现 | 无法监听加载状态 | 3-5 天 |
| **P1** | `document_title_changed` 未实现 | 标题无法动态更新 | 2-3 天 |
| **P1** | CLI 工具链完善 | 开发体验不佳 | 5-10 天 |
| **P2** | 测试套件 | 无法保证质量 | 10-15 天 |
| **P2** | 插件生态适配 | 官方插件不可用 | 15-25 天 |
| **P2** | 依赖管理正规化 | 版本管理困难 | 5-10 天 |
| **P3** | 文档 | 开发者难以上手 | 5-10 天 |
| **P3** | CI/CD | 无自动化保证 | 3-5 天 |
| **P3** | 错误处理完善 | 调试困难 | 3-5 天 |

---

## 8. 风险评估

| 风险 | 等级 | 说明 | 缓解措施 |
|------|:----:|------|---------|
| richerfu fork 与上游不同步 | 🟡 中 | fork 可能落后于 Tauri 官方更新 | 定期 rebase，关注上游合并进度 |
| `openharmony-ability` 稳定性 | 🟡 中 | 第三方 crate，可能有 bug | 关注社区更新，准备 fork 自行维护 |
| PC 场景 API 差异 | 🟡 中 | 2in1 设备的窗口管理 API 可能与手机不同 | 尽早在 PC 真机上测试 |
| 上游 PR 长期未合并 | 🟡 中 | TAO PR #1128 仍未合并 | 维护独立 fork，争取贡献回上游 |
| OHOS PC 版本可用性 | 🟢 低 | module.json5 已声明支持 "2in1" | 关注鸿蒙 PC 版本发布 |
| N-API 性能瓶颈 | 🟢 低 | IPC 链路多一层运行时 | 性能关键路径可优化 |

---

## 9. 本章小结

### 一句话总结

> Demo 项目证明了技术可行性（核心 IPC + WebView + 自定义协议均可工作），但距离官方级别的平台支持，在 **PC 适配（0%）、测试（0%）、功能完整性（55%）、插件生态（15%）** 等方面仍有显著差距。

### 关键数字

| 指标 | 值 |
|------|:--:|
| 代码完成度（vs Android） | 42% |
| 功能覆盖率 | ~55% |
| PC 特有功能完成度 | 0% |
| 测试覆盖 | 0% |
| 预估补齐总工作量 | 37-73 人天 |

**下一步**：[实施方案](./06-implementation-plan.md) — 如何从当前状态到达目标状态。
