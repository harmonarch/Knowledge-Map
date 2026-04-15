---
type: domain
status: active
created: 2026-04-15
tags:
  - domain
  - infrastructure
  - serverless
---

# Cloudflare

## 这个问题域解决什么问题

独立开发者做 AI 产品时，需要一整套基础设施：部署、存储、数据库、AI 模型调用。大公司有自己的服务器，小开发者需要一个免费额度慷慨、产品设计极简、全球边缘部署的平台。

## 为什么它重要

Cloudflare 从 CDN 和安全防护起家，现在已经是小开发者最喜欢的全栈基础设施平台。几乎零成本就能跑一个生产级的 AI 应用架构。

## 核心子问题
- 域名与网络层（DNS、CDN）
- 部署与运行（[[Cloudflare Pages]]、[[Cloudflare Workers]]）
- 存储（[[Cloudflare R2]]、[[Cloudflare KV]]、[[Cloudflare D1]]）
- AI 专属（[[Cloudflare AI Gateway]]、[[Workers AI]]、[[Vectorize]]）

## 产品地图

```
├── 域名 & 网络层
│   ├── DNS 管理（免费，全球最快之一）
│   └── CDN（静态资源加速，免费）
│
├── 部署 & 运行
│   ├── Pages（前端静态托管，类似 Vercel）
│   └── Workers（后端逻辑，Serverless 函数）
│
├── 存储
│   ├── R2（对象存储，S3 兼容，无出流量费）
│   ├── KV（键值存储，适合配置/缓存）
│   └── D1（SQLite 数据库，Serverless）
│
└── AI 专属
    ├── AI Gateway（AI API 代理，限速/缓存/监控）
    ├── Workers AI（边缘运行 AI 模型）
    └── Vectorize（向量数据库，RAG 必备）
```

## 典型全栈架构

```
用户浏览器
    │
    ▼
Cloudflare Pages（前端，React/Next.js）
    │
    ▼
Cloudflare Workers（API 层，处理请求逻辑）
    │
    ├──▶ AI Gateway ──▶ OpenAI / Claude（外部 AI 服务）
    ├──▶ Workers AI（边缘运行开源模型）
    ├──▶ Vectorize（语义搜索，RAG）
    ├──▶ D1（用户数据、内容存储）
    ├──▶ KV（Session、缓存）
    └──▶ R2（图片、文件存储）
```

初期流量下月费用接近 $0，中等规模（日活几百人）约 $5-20/月。

## 快速上手路线图

1. 注册账号，迁移域名 DNS
2. 安装 Wrangler CLI：`npm install -g wrangler && wrangler login`
3. 部署第一个 Worker：`npm create cloudflare@latest my-first-worker`
4. 按需添加存储（D1 / R2 / KV）
5. 接入 AI Gateway

## 相关概念
- [[Cloudflare Pages]]
- [[Cloudflare Workers]]
- [[Cloudflare R2]]
- [[Cloudflare D1]]
- [[Cloudflare KV]]
- [[Cloudflare AI Gateway]]
- [[Workers AI]]
- [[Vectorize]]

## 相关决策
- [[前端部署平台选型-Cloudflare Pages vs Vercel]]

## 相关案例
- 

## 与其他领域的连接
- [[后端]] — Workers + D1 + R2 构成完整后端方案
- [[AI Agent]] — AI Gateway + Workers AI + Vectorize 支撑 AI 应用

## 当前理解边界
- 我已经理解：各产品定位、免费额度、全家桶架构
- 我还不理解：生产环境下的性能瓶颈、Workers 的实际 CPU 限制对复杂业务的影响

## 信息来源

- [Mr Panda @PandaTalk8 推文](https://x.com/PandaTalk8/status/2043616557667078515) — AI CODING 必须要用的平台系列一：CLOUDFLARE（2026-04-13）
- [Cloudflare 官方文档](https://developers.cloudflare.com)
