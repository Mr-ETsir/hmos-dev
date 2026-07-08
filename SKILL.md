---
name: hmos-dev
description: 鸿蒙(HarmonyOS)全栈开发技能——智能路由到5大子能力：ArkTS语法检查与构建、废弃API检查与迁移、ArkTS知识检索、ArkUI开发（状态管理/组件/布局/动画/导航/MVVM）、多设备适配。触发词：鸿蒙开发、HarmonyOS、ArkTS、ArkUI、编译构建、语法检查、废弃API、多设备适配、折叠屏。
---

# 鸿蒙全栈开发技能 (hmos-dev)

## 概述

本技能整合了鸿蒙 HarmonyOS 开发的全部核心能力，根据用户意图自动路由到对应的子能力模块。支持从语法检查、API 迁移、知识检索、ArkUI 开发到多设备适配的完整开发流程。

## 意图路由

### 路由规则（按优先级）

收到用户请求后，按以下顺序匹配意图：

```
用户请求
  │
  ├─ 包含"编译/构建/build/HAP/APP/语法错误/编译错误/构建失败"
  │   └─ → [构建工作流] 静态语法检查 + 错误修复 + 循环构建
  │
  ├─ 包含"废弃/deprecated/API迁移/升级API/清理技术债务"
  │   └─ → [废弃API工作流] 检查废弃接口 + 生成迁移方案
  │
  ├─ 包含"ArkTS语法/怎么用/语法参考/文档/知识/API用法"
  │   └─ → [知识检索工作流] 检索ArkTS语言指南文档
  │
  ├─ 包含"多设备/适配/折叠屏/平板/响应式/breakpoint/避让区/横竖屏/外设"
  │   └─ → [多设备适配工作流] 场景路由 + 适配方案
  │
  ├─ 包含"ArkUI/组件/页面/布局/状态管理/动画/导航/MVVM/@Component/@State/列表/网格"
  │   └─ → [ArkUI开发工作流] 代码编写/审查/改进
  │
  └─ 默认 → 询问用户具体需求，展示能力概览
```

### 复合意图处理

当用户请求同时命中多个场景时，按优先级并行处理：
- "开发一个页面并编译" → 先走 ArkUI 开发，再走构建工作流
- "检查废弃API并修复编译错误" → 废弃API检查 + 构建修复并行
- "多设备适配后构建验证" → 多设备适配 + 构建工作流

---

## 工作流一：项目构建与语法检查

### 能力来源
整合自 `hmos-arkts-syntax-checker`

### 触发条件
- 编译 HarmonyOS 项目生成 HAP/App 产物
- 项目存在语法错误需要修复
- 自动化构建流程
- CI/CD 场景

### 执行流程

#### 0. MCP 工具依赖检查（前置必需）

本工作流依赖 CodeGenie MCP Server 工具：

| MCP 工具 | 用途 |
|---------|------|
| `mcp__deveco-mcp__check` | ETS 文件静态语法检查 |
| `mcp_codegenie-mcp_build_project` | 项目构建 |
| `mcp_codegenie-mcp_harmonyos_knowledge_search` | 知识搜索（可选） |

**若 MCP 工具不可用，提示用户安装：**

在 MCP 配置文件中添加：
```json
{
  "mcpServers": {
    "codegenie-mcp": {
      "command": "npx",
      "args": ["-y", "@deveco-codegenie/mcp@beta", "--registry=https://registry.npmjs.org"],
      "env": {
        "PROJECT_PATH": "${workspaceFolder}",
        "DEVECO_PATH": "path to deveco studio"
      }
    }
  }
}
```

#### 1. 项目分析
检查关键配置文件：
- `build-profile.json5` — 项目级构建配置
- `entry/build-profile.json5` — 模块级构建配置
- `module.json5` — 模块配置
- `oh-package.json5` — 依赖配置

#### 2. 获取源文件列表
```bash
# Glob 搜索所有 .ets 文件，排除 oh_modules/ build/ .preview/
**/*.ets
```

#### 3. 静态语法检查
使用 MCP 工具 `mcp__deveco-mcp__check` 检查文件。

**诊断信息分类：**

| 错误码 | 类型 | 说明 | 优先级 |
|--------|------|------|--------|
| 28007 | Warning | 权限警告 | P2 |
| 6133 | Warning | 未使用的变量/符号 | P2 |
| 6387 | Information | 使用了废弃的 API | P1 |
| addTryCatch | Warning | 需要异常处理 | P2 |
| addAsyncCatch | Warning | 异步函数需要异常处理 | P2 |
| 其他 Error | Error | 语法错误/类型错误 | P0 |

#### 4. 错误修复策略

**P0 — 必须修复（阻止编译）：**
- 语法错误：缺少分号、括号不匹配
- 类型错误：类型不匹配
- 未定义的变量/函数
- 导入错误

**P1 — 强烈建议修复：**
- 废弃 API：查找替代 API，应用迁移方案

**P2 — 可选优化：**
- 未使用变量：删除或使用下划线前缀
- 异常处理：添加 try-catch

> 详细修复示例参考 [error-fixing-examples.md](references/build/error-fixing-examples.md)

#### 5. 构建项目
```
mcp_codegenie-mcp_build_project
参数:
  - buildTarget: "hap" 或 "app"
  - buildMode: "debug" 或 "release"
```

#### 6. 循环修复机制

```
最大重试次数: 5

while (retryCount < 5):
  1. 静态检查 → 发现错误？
     ├─ P0 → 必须修复 → 返回检查
     ├─ P1 → 建议修复 → 返回检查
     └─ P2 → 可选修复 → 继续构建
  2. 构建项目 → 失败？
     ├─ 依赖问题 → 安装依赖 → 重新构建
     ├─ 签名问题 → 提示手动修复 → 终止
     ├─ 资源问题 → 检查资源 → 重新构建
     └─ 编译错误 → 返回静态检查
  3. 成功 → 输出产物路径
```

#### 7. 构建产物定位
```
entry/build/default/outputs/default/entry-default-signed.hap    # HAP包
build/outputs/default/{project-name}-default-signed.app          # APP包
```

> 详细输出示例参考 [output-examples.md](references/build/output-examples.md)

---

## 工作流二：废弃接口检查与迁移

### 能力来源
整合自 `hmos-arkts-deprecated-interface-checker`

### 触发条件
- 项目升级 HarmonyOS API 版本
- 清理技术债务
- 代码审查发现废弃 API 警告
- 迁移到新版本（API 10→11→12）

### 执行流程

#### 0. MCP 工具依赖检查
同工作流一，依赖 `mcp__deveco-mcp__check`

#### 1. 项目分析
- 检查 `build-profile.json5` 中的 `compileSdkVersion` 和 `compatibleSdkVersion`
- 确定当前和目标 API 版本

#### 2. 静态检查
使用 `mcp__deveco-mcp__check` 扫描全部 `.ets` 文件，重点关注：
- `code: 6387` — 废弃 API 使用
- `data: "depreciatedSymbol"` — 废弃符号标记

#### 3. 生成检查报告
```
📊 废弃接口检查报告
🔴 严重：X 个错误
🟡 警告：X 个警告
ℹ️  信息：X 个废弃接口提示

每个问题包含：
  📍 文件位置
  ⚠️  问题描述
  💡 替代方案
  📋 迁移代码示例
```

#### 4. 修复优先级
- **P0**：影响稳定性/兼容性的错误 → 必须立即修复
- **P1**：废弃 API 有明确替代方案 → 强烈建议修复
- **P2**：未使用变量等 → 可选优化

#### 5. 常见废弃 API 速查

| 废弃 API | 替代方案 | 说明 |
|----------|---------|------|
| `px2vp(value: number)` | 使用新签名 | 参数类型变更 |
| `vp2px(value: number)` | 使用新签名 | 参数类型变更 |
| `@ohos.fileio` | `@ohos.file.fs` | 模块迁移 |
| `AbilityContext` | `UIAbilityContext` | 类型重命名 |

> 完整参考 [deprecated-api-reference.md](references/deprecated-api/deprecated-api-reference.md)

---

## 工作流三：ArkTS 知识检索

### 能力来源
整合自 `hmos-arkts-knowledge-retriever`

### 触发条件
- 查找 ArkTS 语法规则和用法
- 检索标准库/常用库 API
- 查找并发编程、运行时相关文档
- 代码审查时验证语法
- 从 TypeScript/Java/Swift 迁移到 ArkTS

### 查询类型路由

| 查询类型 | 关键词示例 | 目标范围 |
|---------|-----------|---------|
| 语法规则 | 类、函数、接口、泛型 | 02-Basic-Syntax |
| 库使用 | JSON、XML、容器、Buffer | 03-Common-Library |
| 并发编程 | TaskPool、Worker、线程 | 04-Concurrency |
| 运行时 | 动态导入、模块加载、GC | 05-Runtime |
| 工具链 | 编译、混淆、字节码 | 07-Compilation-Toolchain |
| 迁移指南 | TS→ArkTS、Java→ArkTS | 09-Migration-Guide |

### 检索方式

**方法一：使用检索脚本**
```bash
python3 scripts/search_docs.py --query "<查询内容>" [--scope "<范围>"] [--top-k 5]
```

**方法二：直接查索引**
查阅 [doc_index.json](references/arkts-knowledge/doc_index.json) 定位文档。

### 检索结果格式
```markdown
📚 **参考文档**: [classes.md](references/...)
📍 **章节**: Declaring a Class
💡 **匹配原因**: 查询涉及类声明语法
📖 **指导**: 使用 `class` 关键字定义类，字段必须在类体中声明
✅ **验证状态**: snippet_validated / doc_only
```

### 验证级别
- `snippet_validated`：代码示例已通过 arkts-cli 验证，可信度高
- `doc_only`：仅文档说明，建议用户自行验证

### 主题别名扩展
检索系统内置中英文主题别名映射（见 [topic_aliases.json](references/arkts-knowledge/topic_aliases.json)），支持中文查询自动扩展为英文关键词，提高检索命中率。

---

## 工作流四：ArkUI 开发

### 能力来源
整合自 `hmos-arkui-develop-skill`

### 触发条件
- 构建新页面或组件
- 实现状态管理（V1/V2）
- 开发布局（列表/网格/瀑布流）
- 添加动画和手势
- 搭建导航路由
- 处理弹窗和模态页面
- MVVM 架构设计
- V1 迁移到 V2

### 核心规则

1. **新项目优先使用 V2 装饰器**（@Local、@Param、@Event、@ObservedV2/@Trace）
2. **V1 项目可继续使用 V1**，仅在需要时迁移
3. **不强制架构模式** — 复杂应用用 MVVM，简单需求用简单结构
4. **性能从一开始就很重要** — 使用 LazyForEach/Repeat，最小化更新
5. **同一组件中不混用 V1 和 V2 装饰器**
6. **保持组件专注和可测试** — 小型、单一职责

### 状态装饰器速查

#### V2（推荐新项目使用）

| 装饰器 | 使用场景 |
|--------|---------|
| `@Local` | 组件内部状态（替代 @State） |
| `@Param` | 父→子数据传递（只读） |
| `@Param @Once` | 父→子传递，允许本地更新 |
| `@Event` | 子→父通信 |
| `@ObservedV2` | 类需要深度观测 |
| `@Trace` | 标记 @ObservedV2 类中的可观测属性 |
| `@Provider/@Consumer` | 跨组件状态共享 |
| `@Monitor` | 监听特定属性变化 |
| `@Computed` | 派生/缓存计算值 |

#### V1（旧版兼容）

| 装饰器 | 使用场景 |
|--------|---------|
| `@State` | 组件内部状态 |
| `@Prop` | 父→子单向绑定 |
| `@Link` | 父↔子双向绑定 |
| `@Observed/@ObjectLink` | 嵌套对象观测（必须配合使用） |
| `@Provide/@Consume` | 跨组件状态（祖先↔后代） |
| `@Watch` | 观察状态变化 |

### V1 → V2 迁移对应

| V1 | V2 |
|----|----|
| @Component | @ComponentV2 |
| @State | @Local |
| @Prop | @Param |
| @Link | @Param + @Event |
| @ObjectLink | @Param |
| @Observed | @ObservedV2 + @Trace |
| @Watch | @Monitor |
| @Provide/@Consume | @Provider/@Consumer |

### 布局组件选择

| 布局类型 | 组件 | 使用场景 |
|---------|------|---------|
| 线性 | Row/Column | 简单水平/垂直排列 |
| 弹性 | Flex | 弹性尺寸、换行、对齐 |
| 层叠 | Stack | 重叠元素、分层 |
| 相对 | RelativeContainer | 复杂相对定位 |
| 列表 | List | 长列表滚动（用 LazyForEach/Repeat） |
| 网格 | Grid | 规则网格项 |
| 瀑布流 | WaterFlow | 不等高项 |
| 标签 | Tabs | 标签导航 |

### 正确性检查清单

**状态管理：**
- [ ] 新代码使用 V2 装饰器（@Local、@Param、@Event）
- [ ] @ObservedV2/@Trace 用于深度观测
- [ ] 同一组件中不混用 V1 和 V2 装饰器
- [ ] V1: @State 必须有默认值初始化
- [ ] V1: @Observed 必须配合 @ObjectLink 使用

**组件结构：**
- [ ] 使用 @Component 或 @ComponentV2 装饰器
- [ ] @Entry 用于页面入口组件
- [ ] build() 方法简洁纯净，无副作用

**列表渲染：**
- [ ] ForEach 仅用于小型静态列表
- [ ] LazyForEach/Repeat 用于大量数据
- [ ] 列表项使用稳定的 key（动态内容绝不用 index）

**导航路由：**
- [ ] 使用 Navigation 进行页面管理
- [ ] 使用 NavPathStack 进行编程式导航

### 常见代码模板

**基础组件（V2）：**
```typescript
@ComponentV2
struct MyComponent {
  @Local count: number = 0;
  @Param title: string = '';
  @Event onCountChange: (value: number) => void = () => {};

  build() {
    Column() {
      Text(this.title)
      Button(`计数: ${this.count}`)
        .onClick(() => {
          this.count++;
          this.onCountChange(this.count);
        })
    }
  }
}
```

**可观测类（V2）：**
```typescript
@ObservedV2
class TaskModel {
  @Trace name: string = '';
  @Trace isDone: boolean = false;

  toggle() {
    this.isDone = !this.isDone;
  }
}
```

**MVVM 目录结构：**
```
src/main/ets/
├── model/           # 数据结构 (XxxModel.ets)
├── viewmodel/       # 状态与逻辑 (XxxViewModel.ets)
├── view/            # UI组件 (XxxView.ets)
└── pages/           # 页面入口
```

### 主题路由表

根据当前任务查阅对应参考文档：

| 主题 | 参考文档路径 |
|------|------------|
| V2 状态管理 | `references/arkui/02-state-management/v2-state-management.md` |
| V1 状态管理 | `references/arkui/02-state-management/v1-state-management.md` |
| 状态管理最佳实践 | `references/arkui/02-state-management/best-practices.md` |
| 自定义组件 | `references/arkui/01-basic-development/custom-components.md` |
| 声明式 UI | `references/arkui/01-basic-development/declarative-ui.md` |
| 渲染控制 | `references/arkui/01-basic-development/rendering-control.md` |
| 基础布局 | `references/arkui/03-layout/basic-layout.md` |
| List 列表 | `references/arkui/03-layout/list.md` |
| Grid 网格 | `references/arkui/03-layout/grid.md` |
| WaterFlow 瀑布流 | `references/arkui/03-layout/waterflow.md` |
| UI 组件 | `references/arkui/04-ui-components/ui-components.md` |
| 动画 | `references/arkui/05-animation/animation.md` |
| 转场动画 | `references/arkui/05-animation/transition-animation.md` |
| 事件交互 | `references/arkui/06-events-interaction/events-interaction.md` |
| 手势交互 | `references/arkui/06-events-interaction/gesture-interaction.md` |
| 弹窗菜单 | `references/arkui/07-dialogs/dialogs-menus.md` |
| 模态页面 | `references/arkui/07-dialogs/modals-pages.md` |
| 导航路由 | `references/arkui/08-navigation-routing/navigation-routing.md` |
| Tabs | `references/arkui/08-navigation-routing/tabs.md` |
| 自定义节点 | `references/arkui/09-custom-nodes/custom-node.md` |
| 图形绘制 | `references/arkui/09-custom-nodes/graphics-drawing.md` |
| 国际化 | `references/arkui/11-i18n-adaptation/internationalization.md` |
| 主题适配 | `references/arkui/11-i18n-adaptation/theming-adaptation.md` |
| 无障碍 | `references/arkui/11-i18n-adaptation/accessibility.md` |
| 稳定性 | `references/arkui/12-stability-performance/stability.md` |
| 调试与性能 | `references/arkui/12-stability-performance/debugging-performance.md` |
| MVVM 架构 | `references/arkui/13-mvvm-architecture/MVVM_Architecture_Guide.md` |

### 代码模板文件

| 分类 | 模板文件 |
|------|---------|
| 基础组件 | `assets/01-basic-components/` |
| 容器组件 | `assets/02-container-components/` |
| 状态管理 | `assets/03-state-management/state-templates.ets` |
| 动画 | `assets/04-animation/animation-templates.ets` |
| 手势事件 | `assets/05-gesture-events/gesture-templates.ets` |
| 自定义组件 | `assets/06-custom-components/custom-component-templates.ets` |
| 弹窗 | `assets/07-dialogs/` |

---

## 工作流五：多设备适配

### 能力来源
整合自 `hmos-multidevice-scenario-entry`

### 概述

本工作流负责鸿蒙多设备适配的入口路由。先判定当前问题属于哪类多设备适配场景，再引导到对应场景文件展开处理。

### 适用范围
- 设备范围：phone / tablet / tv / 2in1 / wearable
- 问题范围：多设备布局、折展状态、避让区、交互方式、自然方向、硬件能力差异
- **不在范围**：分布式流转、账号体系、网络通信、纯业务功能设计

### 阶段识别

| 标签 | 阶段 | 关注点 |
|------|------|--------|
| `REQ` | 需求分析设计 | 问题边界、设备范围、主适配维度 |
| `DEV` | 开发 | 主代码落点、是否需要多场景联合 |
| `FIX` | 问题修复 | 根因适配域、连带回归域 |
| `VAL` | 功能验证 | 验证矩阵、交叉验证 |

### 场景索引

#### SCENE-01 布局与窗口尺寸
**意图信号：** 响应式、breakpoint、GridRow/GridCol、多栏、windowSizeChange、media query、窗口变化、平行视界、分屏、多设备适配、适配折叠屏/平板/大屏

**适用：** 主要问题是断点、结构切换或窗口尺寸变化

**不适用：** 存在折痕、悬停态、键盘遮挡、方向语义、输入设备等更强信号

#### SCENE-02 折展状态与折痕
**意图信号：** foldStatus、foldStatusChange、悬停态、HALF_FOLDED、折痕避让、开合连续性、内外屏切换、G态、Pura X

**适用：** 主要问题依赖设备折叠状态变化、折痕几何或悬停态分屏

**不适用：** 只是要将布局扩展到折叠屏设备但未提及折展动态行为 → 归入 SCENE-01

#### SCENE-03 系统区域与键盘避让
**意图信号：** safe area、状态栏、导航栏、挖孔、刘海、沉浸式、键盘遮挡、输入法

#### SCENE-04 多输入与焦点交互
**意图信号：** 鼠标、hover、右键、键盘焦点、shortcut、手写笔、拖拽、外接键鼠

#### SCENE-05 自然方向与旋转语义
**意图信号：** rotation值、orientation、自然竖屏/横屏、setPreferredOrientation、传感器方向

#### SCENE-06 硬件能力与外设
**意图信号：** canIUse、SysCap、相机、camera、传感器、GPS、NFC、蓝牙

#### SCENE-07 折叠设备多形态验证（仅 VAL 阶段）
**意图信号：** hidumper、折叠态/展开态/悬停态布局验证、分辨率阶梯、模拟折叠

> 完整验证流程参考 [multi-device-verification.md](references/multidevice/multi-device-verification.md)

### 路由执行流程（强制）

1. **路由判断** — 根据用户请求判定阶段和场景
2. **读取场景文档** — 必须 Read 对应场景的参考文档
3. **输出路由确认** — 告知用户路由结果和已读取的文档
4. **进入实现** — 基于文档约束进行开发

**禁止行为：**
- 跳步（路由→直接写代码）
- 凭记忆猜测 API 用法替代文档
- 静默忽略路由错误

### 常用命令速查

**hidumper 模拟折叠形态：**
```bash
HDC=${HDC:-hdc}
TARGET=<device_id>

# 开启调试
hdc -t "$TARGET" shell "param set dms.hidumper.supportdebug true"

# 切换形态
hdc shell "hidumper -s DisplayManagerService -a '-p'"   # 折叠态
hdc shell "hidumper -s DisplayManagerService -a '-z'"   # 悬停态
hdc shell "hidumper -s DisplayManagerService -a '-y'"   # 展开态
hdc shell "hidumper -s DisplayManagerService -a '-yy'"  # 三屏态
```

---

## 最佳实践

### ✅ 推荐做法
1. **开发前先检索** — 使用知识检索确认 API 用法
2. **新代码用 V2** — @ComponentV2 + @Local/@Param/@Event
3. **定期检查废弃 API** — 纳入开发流程
4. **渐进式迁移** — 优先修复 P0 级别问题
5. **构建前先检查** — 避免浪费时间在必然失败的构建上
6. **多设备从设计开始** — 不要事后适配

### ❌ 避免做法
1. 忽略废弃 API 警告
2. 在关键业务逻辑中使用废弃 API
3. 跳过静态检查直接构建
4. V1/V2 装饰器混用
5. 列表使用 index 作为 key
6. 凭记忆猜测 API 而不查文档

## MCP 工具依赖

本技能的核心工作流（构建、废弃API检查）依赖以下 MCP 工具：

| 工具 | 用途 |
|------|------|
| `mcp__deveco-mcp__check` | ETS 文件静态语法检查 |
| `mcp_codegenie-mcp_build_project` | HarmonyOS 项目构建 |
| `mcp_codegenie-mcp_harmonyos_knowledge_search` | 知识搜索（可选） |

安装方式：https://github.com/open-deveco/deveco-toolbox

## 资源索引

### 参考文档
| 目录 | 内容 |
|------|------|
| `references/arkts-knowledge/` | ArkTS 语言指南索引和检索工具 |
| `references/arkui/` | ArkUI 开发全部参考文档（13个主题） |
| `references/build/` | 构建错误修复示例和输出示例 |
| `references/deprecated-api/` | 废弃 API 参考和配置指南 |
| `references/multidevice/` | 多设备验证指南和远程加载说明 |

### 代码资产
| 目录 | 内容 |
|------|------|
| `assets/01-basic-components/` | 基础组件模板（Button/Text/Image等12个） |
| `assets/02-container-components/` | 容器组件模板（List/Grid/Tabs等6个） |
| `assets/03-state-management/` | 状态管理模板 |
| `assets/04-animation/` | 动画模板 |
| `assets/05-gesture-events/` | 手势事件模板 |
| `assets/06-custom-components/` | 自定义组件模板 |
| `assets/07-dialogs/` | 弹窗模板（Dialog/Toast/Snackbar等5个） |

### 脚本工具
| 文件 | 用途 |
|------|------|
| `scripts/search_docs.py` | ArkTS 文档检索脚本 |

## 外部资源

- [HarmonyOS 开发指南](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides/)
- [HarmonyOS API 参考](https://developer.huawei.com/consumer/cn/doc/harmonyos-references/)
- [ArkTS 语法指南](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides/arkts-get-started)
- [CodeGenie MCP Server](https://github.com/open-deveco/deveco-toolbox)
- [HarmonyOS 版本变更](https://developer.huawei.com/consumer/cn/doc/harmonyos-releases/)
