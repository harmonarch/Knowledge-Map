---
type: concept
status: active
tags:
  - concept
  - storage
  - s3
---

# Cloudflare R2

## 一句话定义

Cloudflare 的对象存储服务，完全兼容 S3 API，核心卖点是出流量完全免费。

## 它解决什么问题

AWS S3 的"隐形成本"是出流量费（Egress Fee）——用户每次下载文件都按流量付钱。R2 消除了这个成本。

## 出现前提

你的应用有大量图片、视频或文件下载需求，或者需要一个 S3 兼容的对象存储。

## 核心机制

S3 兼容 API，原来写的 S3 代码改几行配置就能用。存储在 Cloudflare 的全球网络上。

## 收益

- 出流量完全免费
- 10 GB 免费存储
- 100 万次 A 类操作（写入）/ 月免费
- 1000 万次 B 类操作（读取）/ 月免费
- S3 API 兼容，迁移成本低

## 代价

- 没有 S3 的所有高级功能（如 S3 Select）
- 生态和工具链不如 AWS 成熟

## 常见误区

- 以为 R2 只是便宜版 S3 — 对于出流量大的场景，R2 可能比 S3 便宜一个数量级
- 以为需要重写代码 — S3 SDK 改个 endpoint 就能用

## 关联节点
- [[Cloudflare]]
- [[Cloudflare Workers]]

## 适用场景
- 用户上传的图片、文档
- AI 生成的图片（Stable Diffusion 等）
- 应用的静态资源（大文件）
- 数据库备份

## 一个真实例子

```bash
wrangler r2 bucket create my-files
```
