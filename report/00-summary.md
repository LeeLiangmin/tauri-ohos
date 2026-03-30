# Tauri 鸿蒙适配调研报告 — 总览

> **调研日期**：2026-03-28  
> **调研范围**：Tauri v2 框架对 HarmonyOS NEXT PC 平台的支持可行性  
> **源码版本**：Tauri v2 (dev 分支), TAO (dev 分支), WRY (main 分支)  
> **关键发现**：社区已有可运行原型 [richerfu/tauri-demo](https://github.com/richerfu/tauri-demo)

---

## 核心结论

**Tauri 适配鸿蒙已从"技术验证"进入"产品化完善"阶段。**

| 判断维度 | 结论 | 详情 |
|---------|------|------|
| 技术可行性 | ✅ 已验证 | 已有可运行 Demo，IPC/WebView/自定义协议均可工作 |
| 交叉编译 | ✅ 完全可行 | Rust 官方支持 OHOS target，Windows 可交叉编译 |
| 当前完成度 | ~80-85%（适配层） | 核心链路已通，剩余为完善和 PC 适配（代码量约为 Android 的 42%） |
| 剩余工作量 | 37-73 人天 | 2 人团队 1.5-3 个月 |
| 最大风险项 | PC 适配 | 窗口管理、菜单栏、多窗口等桌面特性 |

---

## 报告结构

本报告基于**第一性原理**组织，从底层问题出发，逐层推导：

```
问题本质 → 平台能力 → 框架机制 → 适配映射 → 差距量化 → 实施路径
```

| 序号 | 报告 | 回答的核心问题 |
|------|------|--------------|
| 01 | [第一性原理分析](./01-first-principles.md) | Tauri 适配鸿蒙的**本质问题**是什么？ |
| 02 | [鸿蒙平台技术基础](./02-harmonyos-platform.md) | 鸿蒙**提供了什么**能力来支撑 Tauri？ |
| 03 | [Tauri 架构与扩展机制](./03-tauri-architecture.md) | Tauri **如何支持**多平台？扩展点在哪？ |
| 04 | [适配层分析](./04-adaptation-analysis.md) | 每一层**需要做什么**、**已经做了什么**？ |
| 05 | [差距评估](./05-gap-assessment.md) | 从 Demo 到产品级**还差什么**？ |
| 06 | [实施方案](./06-implementation-plan.md) | **如何**从当前状态到达目标状态？ |

---

## 关键数据

### 工作量估算

| 工作项 | 乐观 | 悲观 | 说明 |
|--------|------|------|------|
| 环境搭建 + Demo 复现 | 2天 | 3天 | 按 tauri-demo 流程 |
| WRY 完善 | 5天 | 10天 | 补全空实现（bounds/visible/print 等） |
| TAO 完善 | 5天 | 10天 | 补全高级功能（多窗口、菜单等） |
| Tauri Core 完善 | 5天 | 10天 | 插件系统、权限管理等 |
| CLI 工具链完善 | 5天 | 10天 | dev 模式、热重载等 |
| **PC 适配** | **10天** | **20天** | 窗口管理、菜单栏、快捷键 |
| 测试和文档 | 5天 | 10天 | — |
| **合计** | **37天** | **73天** | |

### 团队建议

| 配置 | 人员 | 周期 |
|------|------|------|
| 最小团队 | 2 人（1 Rust + 1 OHOS） | 3 个月 |
| 推荐团队 | 2-3 人 | 1.5-2 个月 |
| 快速团队 | 3-4 人 | 1-1.5 个月 |

### 技术路线（3 阶段）

```
阶段 1 (1-2周)          阶段 2 (4-8周)           阶段 3 (3-5周)
复现 Demo ──────────► 完善功能 ──────────────► PC 适配 + 发布
```

---

## 立即行动

```bash
# 1. 安装工具
cargo install tauri-cli --git https://github.com/tauri-apps/tauri --branch feat/open-harmony
cargo install ohrs

# 2. 克隆 demo
git clone https://github.com/richerfu/tauri-demo.git

# 3. 安装依赖并构建
cd tauri-demo && pnpm install
cd src-tauri && cargo tauri ohos build

# 4. 在 DevEco Studio 中打开 src-tauri/gen/ohos 并运行
```

---

## 源码仓库

| 仓库 | 路径 | 说明 |
|------|------|------|
| tauri (官方) | `vendor/tauri/` | 含 feat/open-harmony 分支 |
| tao (官方) | `vendor/tao/` | 含 PR #1128 分支 |
| wry (官方) | `vendor/wry/` | 官方版，无 OHOS |
| **tauri-demo** | `vendor/tauri-demo/` | ⭐ 可运行的鸿蒙 Demo |
| **wry (fork)** | `vendor/wry-ohos/` | ⭐ 含 OHOS WebView 实现 |
| **tao (fork)** | `vendor/tao-ohos/` | ⭐ 含 OHOS 窗口管理 |
