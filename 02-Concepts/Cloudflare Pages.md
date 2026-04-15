---
type: concept
status: active
created: 2026-04-15
tags:
  - concept
  - serverless
  - hosting
---

# Cloudflare Pages

## 一句话定义

Cloudflare 的静态网站托管服务，类似 Vercel / Netlify，推代码到 GitHub 自动构建部署到全球 CDN。

## 它解决什么问题

前端项目需要一个部署平台，能自动构建、全球加速、绑定自定义域名，且不按流量收费。

## 出现前提

你有一个前端项目（React / Vue / Next.js 静态导出 / Astro），需要上线给用户访问。

## 核心机制

连接 GitHub 仓库 → 选择构建命令和输出目录 → 自动构建并部署到全球 CDN。也可以用 Wrangler CLI 本地部署：`wrangler pages deploy ./dist`

## 收益

- 无限带宽，不按流量收费
- 每月 500 次构建（免费）
- 无限站点数
- 和 Workers / R2 集成顺滑

## 代价

- SSR 能力不如 Vercel 对 Next.js 的支持完整
- 构建环境的自定义程度有限

## 常见误区

- 以为 Pages 只能托管纯静态 HTML — 实际上可以配合 Workers 做 SSR
- 以为要在 Cloudflare 买域名才能用 — 只需要把 DNS 迁过来即可

## 关联节点
- [[Cloudflare]]
- [[Cloudflare Workers]]
- [[前端部署平台选型-Cloudflare Pages vs Vercel]]

## 适用场景
- 文档站、博客、落地页、工具站
- React / Vue / Astro 项目
- 任何"前端为主"的项目

## 一个真实例子

```bash
# 方法一：连接 GitHub（推荐）
# Dashboard → Pages → 创建项目 → 连接 GitHub 仓库

# 方法二：Wrangler CLI
npm install -g wrangler
wrangler pages deploy ./dist
```
