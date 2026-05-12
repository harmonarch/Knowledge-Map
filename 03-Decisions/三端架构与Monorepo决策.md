---
type: decision
status: active
created: 2026-04-16
tags:
  - decision
  - architecture
  - monorepo
  - ai-agent
---

# 三端架构与Monorepo决策

## 要决策的问题

AI 伴侣产品需要哪些子系统？它们应该放在一个代码仓库（Monorepo）还是多个独立仓库（Multirepo）？

## 触发这个决策的信号

完成了技术选型（CloudFlare Workers + Hono.js + Next.js）后，需要将这些技术组合成一个可维护、可协作、可持续演进的工程项目。

## 可选方案

### 方案 A：Monorepo（单仓库）
- 所有子系统放在一个代码仓库
- 使用 Turborepo + yarn workspaces 管理
- 共享类型定义和业务逻辑

### 方案 B：Multirepo（多仓库）
- 每个子系统独立仓库
- 通过 npm 包共享代码
- 独立的版本管理和发布流程

## 评估维度

| 维度 | Monorepo | Multirepo |
|------|----------|-----------|
| 类型共享 | 直接 import，零延迟 | 需要发包、安装、版本对齐 |
| API 变更同步 | 原子提交，一个 PR 搞定 | 多仓库协调，存在时间差 |
| 工具链配置 | 一份配置，全局统一 | 多份配置，容易漂移 |
| 本地开发 | 一条命令启动所有服务 | 多终端、多仓库、需要 link |
| CI/CD | 一套流水线，按变更范围触发 | 多套流水线，跨仓库触发复杂 |
| 代码审查 | 一个 PR 看到完整变更 | 跨仓库 PR 关联困难 |
| 仓库体积 | 较大（但可接受） | 各自较小 |
| 权限控制 | 较粗（需要 CODEOWNERS） | 天然隔离 |

## 为什么选择方案 A（Monorepo）

### 1. 类型共享是第一驱动力

Hono.js 的 RPC 类型安全调用要求前端能直接 import 后端的类型定义：

```typescript
// 服务端定义路由
const route = app.post('/chat',
  zValidator('json', chatSchema),
  async (c) => {
    const body = c.req.valid('json')
    return c.json({ reply: '...', emotion: 'happy' })
  }
)

// 导出类型
export type AppType = typeof route

// 客户端直接 import 服务端类型
import type { AppType } from '@ai-companion/server'
import { hc } from 'hono/client'

const client = hc<AppType>('/api')
const res = await client.chat.$post({ json: { content: '你好' } })
```

Multirepo 下需要手动导出类型到 npm 包，每次 API 变更都要重新发布、重新安装，破坏了 Hono RPC 的核心价值。

### 2. 共享业务逻辑

三个子系统之间存在大量需要保持一致的业务逻辑：

- **Zod Schema 复用**：服务端用 Zod 做运行时参数验证，客户端用同一份 Schema 做表单验证，后台用同一份 Schema 验证配置输入
- **错误码定义**：服务端抛出的错误码，客户端需要据此展示对应的用户提示，后台需要据此做错误分类统计
- **工具函数**：日期格式化、文本截断、ID 生成等通用逻辑

### 3. 原子提交保证版本一致

API 变更时，Monorepo 可以一次 commit 同时修改 server、shared、web、admin，要么全部更新，要么都不更新。

Multirepo 下需要分四步操作，存在时间差，容易出现版本不匹配的问题。

### 4. 统一工具链与开发体验

- 一份 tsconfig.json、一份 Biome 配置、一份 yarn.lock
- 本地开发：克隆一个仓库，运行一条命令，三个子系统同时启动
- 修改 server 的 API，web 的类型检查立刻报错，反馈环路是即时的

## 为什么不选择方案 B（Multirepo）

Multirepo 的优势（仓库体积小、权限天然隔离）在 AI 伴侣项目的场景中并不关键：

- 代码量不会大到 Monorepo 无法承受
- 权限控制可以通过 GitHub 的 CODEOWNERS 文件解决

而 Multirepo 的劣势（类型共享困难、版本同步复杂、开发体验割裂）恰恰是我们最需要避免的。

## 边界条件

- 项目规模：三个子系统（Hono 服务端、Next.js 客户端、Next.js 后台管理）
- 团队规模：小型团队，不需要复杂的权限隔离
- 技术栈：TypeScript 全栈，需要端到端类型安全
- 开发模式：快速迭代，频繁的 API 变更

## 最终判断规则

**当满足以下条件时，选择 Monorepo：**
1. 前后端使用相同语言（TypeScript）
2. 需要端到端类型安全
3. 子系统之间有大量共享逻辑
4. 团队规模较小，不需要复杂的权限隔离
5. 追求快速迭代和即时反馈

**当满足以下条件时，选择 Multirepo：**
1. 子系统使用不同语言/技术栈
2. 团队规模大，需要严格的权限隔离
3. 子系统独立演进，很少共享代码
4. 发布节奏完全独立

## 三个子系统的职责划分

### 1. Hono 服务端 — AI 对话的中枢神经

- **对话管线**：接收消息 → 安全检查 → 身份验证 → 记忆检索 → 情绪状态读取 → Prompt 组装 → LLM 调用 → 响应解析 → 记忆写回 → 情绪状态更新 → 流式返回
- **用户认证与鉴权**：JWT 签发与验证、用户注册登录、会话管理
- **记忆系统 CRUD**：围绕 D1、Vectorize、KV 构建的记忆读写接口
- **情绪状态管理**：维护 AI 伴侣的情绪状态机
- **后台管理接口**：为 admin 系统提供数据查询和系统配置的 API

### 2. Next.js 客户端 — 用户直接触达的产品界面

- **聊天界面**：实时对话窗口，支持流式消息展示、消息气泡、时间戳、情绪标签展示
- **用户注册与登录**：账号体系的前端部分
- **个人设置**：自定义 AI 伴侣的性格特征、设置记忆偏好、管理对话历史
- **SSR/SSG 落地页**：产品介绍页、功能说明页、定价页（SEO 友好）
- **记忆回顾**：查看 AI 伴侣记住了哪些关于自己的信息

### 3. Next.js 后台管理系统 — 运营与维护的控制台

- **用户数据总览**：活跃用户数、日均对话轮次、记忆使用量、用户留存率等核心指标
- **对话审计与内容安全**：查看用户与 AI 的对话记录（脱敏后），识别可能的不当内容
- **Prompt 模板管理**：在线编辑 AI 伴侣的系统 Prompt、人设描述、情绪响应模板，支持版本管理和 A/B 测试
- **系统配置**：LLM 模型切换、记忆策略调整、情绪状态机参数调优
- **运维监控**：API 请求延迟、错误率、Workers CPU 用量、D1 查询性能等技术指标

### 为什么客户端和后台要分成两个 Next.js 应用

- **用户群体完全不同**：客户端面向终端用户（数万到数百万访问量），后台面向几个到几十个管理员
- **安全边界**：后台管理系统包含敏感数据，必须有独立的访问控制
- **构建与部署独立性**：客户端更新频繁，后台相对稳定
- **包体积控制**：后台会引入大量图表库、表格组件、富文本编辑器等重型依赖，不应该出现在客户端的 bundle 中

## Monorepo 工具选型：Turborepo

### 为什么选 Turborepo

- **与 Next.js 同属 Vercel 生态**：天然理解 Next.js 的构建输出、开发服务器、以及增量构建
- **极简配置**：一个 turbo.json 文件定义任务依赖关系就可以工作
- **增量构建与远程缓存**：缓存每个任务的输入输出，只改了 apps/web 的代码，apps/admin 和 apps/server 的构建会直接命中缓存
- **与 yarn workspaces 天然配合**：yarn workspaces 负责依赖安装和包之间的链接，Turborepo 负责任务的并行执行和缓存

### 与其他方案的对比

- **Nx**：功能更全面，适合大型团队和超大型 Monorepo（数十个子项目），但对于三个子项目的规模，学习成本和配置复杂度是过度的
- **Lerna**：主要解决"多 npm 包发布"的问题，我们的三个子项目是应用（不需要发布到 npm），不是库

## 关联概念
- [[Monorepo]]
- [[Turborepo]]
- [[Hono.js]]
- [[Cloudflare Workers]]

## 关联案例
- 

## 信息来源

- [usehook - 三端架构与 Monorepo 决策](https://aicompanion.usehook.cn/11-monorepo-architecture-decision)（2026-04-16）
