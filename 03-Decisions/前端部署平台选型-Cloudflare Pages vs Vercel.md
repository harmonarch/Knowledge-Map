---
type: decision
status: active
tags:
  - decision
  - hosting
  - frontend
---

# 前端部署平台选型-Cloudflare Pages vs Vercel

## 决策问题

前端项目部署到哪个平台？

## 触发信号

用 AI 写完一个前端应用，需要部署上线，Cloudflare Pages 和 Vercel 是最常见的两个选择。

## 评估维度

| 维度 | Cloudflare Pages | Vercel |
|------|-----------------|--------|
| 带宽 | 无限，免费 | 有限制（免费版 100 GB/月） |
| Next.js SSR | 支持但不如 Vercel 完整 | 最好，官方维护 |
| 后端集成 | Workers / R2 / D1 原生集成 | Serverless Functions |
| AI 生态 | AI Gateway + Workers AI + Vectorize | AI SDK |
| 构建次数 | 500 次/月（免费） | 6000 次/月（免费） |

## 为什么选 Cloudflare Pages

- 带宽无限免费，不用担心流量费
- 和 Workers / R2 / D1 集成顺滑，适合做 AI 应用全栈
- 整套 Cloudflare 全家桶的成本极低

## 为什么选 Vercel

- 对 Next.js 支持最好（Vercel 是 Next.js 的开发公司）
- SSR / ISR / Edge Functions 功能更完整
- 开发体验和预览部署更成熟

## 边界条件

- 如果项目重度依赖 Next.js 的 SSR 特性 → Vercel
- 如果项目是 AI 应用全栈，需要存储 + AI 服务 → Cloudflare
- 如果流量大但预算有限 → Cloudflare（无限带宽）

## 最终判断规则

Next.js SSR 为主 → Vercel；AI 全栈 + 成本敏感 → Cloudflare Pages + 全家桶。

## 涉及的概念
- [[Cloudflare Pages]]
- [[Cloudflare]]

## 信息来源

- [Mr Panda @PandaTalk8 推文](https://x.com/PandaTalk8/status/2043616557667078515)（2026-04-13）
