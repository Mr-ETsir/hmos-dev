# hmos-dev — 鸿蒙全栈开发技能

整合 5 大 HarmonyOS 开发能力于一体的智能路由技能。

## 能力矩阵

| 子能力 | 来源 Skill | 触发关键词 |
|--------|-----------|-----------|
| 项目构建与语法检查 | hmos-arkts-syntax-checker | 编译、构建、build、HAP、语法错误 |
| 废弃接口检查与迁移 | hmos-arkts-deprecated-interface-checker | 废弃、deprecated、API迁移 |
| ArkTS 知识检索 | hmos-arkts-knowledge-retriever | ArkTS语法、怎么用、文档 |
| ArkUI 开发 | hmos-arkui-develop-skill | ArkUI、组件、状态管理、MVVM |
| 多设备适配 | hmos-multidevice-scenario-entry | 多设备、折叠屏、适配、breakpoint |

## 安装位置

`~/.claude/skills/hmos-dev/`（全局安装）

## 使用方式

在 Claude Code 中，只需描述你的鸿蒙开发需求，技能会自动路由到对应能力：

- "帮我编译这个鸿蒙项目" → 构建工作流
- "检查项目中的废弃API" → 废弃API检查
- "ArkTS 怎么定义泛型类" → 知识检索
- "写一个带状态管理的任务列表页面" → ArkUI 开发
- "适配折叠屏悬停态" → 多设备适配

## 目录结构

```
hmos-dev/
├── SKILL.md                # 主入口（智能路由）
├── README.md               # 本文件
├── references/
│   ├── arkts-knowledge/    # ArkTS 知识索引
│   ├── arkui/              # ArkUI 开发参考（13个主题）
│   ├── build/              # 构建相关参考
│   ├── deprecated-api/     # 废弃 API 参考
│   └── multidevice/        # 多设备适配参考
├── assets/                 # 代码模板（7个分类）
└── scripts/                # 检索脚本
```

原始仓库：整合自 HarmonyOS_Skills 系列项目
