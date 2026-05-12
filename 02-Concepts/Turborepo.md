---
type: concept
status: active
created: 2026-04-16
tags:
  - concept
  - tooling
  - monorepo
---

# Turborepo

## 一句话定义

Vercel 维护的 Monorepo 任务编排工具，在 yarn/pnpm workspaces 之上提供并行执行、增量构建和缓存能力。

## 出现前提

Monorepo 中多个子项目的构建、测试、类型检查等任务存在依赖关系，需要一个工具来自动分析依赖图、并行执行无依赖任务、缓存已完成的任务结果。

## 核心机制

- **任务依赖声明**：通过 `turbo.json` 定义任务之间的依赖关系（如 `build` 依赖 `^build` 表示先构建依赖包）
- **增量构建**：缓存每个任务的输入（源码 hash）和输出（构建产物），输入不变则跳过执行
- **远程缓存**：团队成员共享构建缓存，A 同事构建过的产物 B 同事直接复用
- **并行执行**：自动识别无依赖关系的任务并行运行

## 最小配置示例

```json
{
  "tasks": {
    "build": {
      "dependsOn": ["^build"],
      "outputs": [".next/**", "dist/**"]
    },
    "dev": {
      "persistent": true,
      "cache": false
    },
    "typecheck": {
      "dependsOn": ["^build"]
    },
    "lint": {}
  }
}
```

## 收益 vs 代价

| 收益 | 代价 |
|------|------|
| 配置极简，约定大于配置 | 功能不如 Nx 全面（无代码生成器、无依赖图可视化） |
| 与 Next.js 深度集成（同属 Vercel 生态） | 对非 JS/TS 项目支持有限 |
| 增量构建 + 远程缓存显著提升 CI 速度 | 远程缓存需要额外配置 |
| 不替代包管理器，与 yarn/pnpm workspaces 各司其职 | — |

## 与同类工具对比

- **Nx**：功能更全面（代码生成器、依赖图可视化、受影响项目检测），适合大型团队和超大型 Monorepo（数十个子项目），但配置复杂度更高
- **Lerna**：主要解决"多 npm 包发布"问题，适合开源库的版本管理和发布流程，不适合应用级 Monorepo

## 适用场景

- 三到十个子项目的 TypeScript Monorepo
- 包含 Next.js 应用的项目（一等公民级别集成）
- 追求极简配置、快速上手

## 关联概念
- [[Monorepo]]
- [[三端架构与Monorepo决策]]
