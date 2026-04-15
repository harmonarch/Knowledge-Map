---
type: concept
status: active
created: 2026-04-15
tags:
  - concept
  - ai
  - gateway
---

# Cloudflare AI Gateway

## 一句话定义

专门为 AI 应用设计的 API 代理层，帮你记录日志、统计费用、缓存相同请求、设置限流。

## 它解决什么问题

做 AI 应用最头疼的四个问题：花了多少钱不透明、哪个 Prompt 效果好难对比、相同请求重复付费、缺乏限流保护。AI Gateway 一次性解决。

## 出现前提

你的应用调用了外部 AI API（OpenAI / Claude / Gemini 等），需要监控、缓存和限流。

## 核心机制

只需要改一个 baseURL，其他代码完全不变。所有请求经过 Cloudflare 节点转发，Dashboard 可看到 Token 用量、费用、响应时间、缓存命中。

支持语义缓存——相似的问题复用之前的答案。

## 收益

- 完全免费，不限请求次数
- 支持几乎所有主流 AI 服务（OpenAI、Anthropic、Gemini、Azure OpenAI、Mistral、Cohere、HuggingFace、Replicate）
- 语义缓存减少重复 API 费用
- 统一的费用和性能监控面板

## 代价

- 多一跳网络延迟（通常可忽略）
- 依赖 Cloudflare 的可用性

## 常见误区

- 以为接入很复杂 — 只改一行 baseURL
- 以为缓存只能精确匹配 — 支持语义缓存，相似问题也能命中

## 关联节点
- [[Cloudflare]]
- [[Cloudflare Workers]]
- [[AI Agent]]

## 适用场景
- 任何调用外部 AI API 的应用
- 需要监控 AI 费用的场景
- 高频相似请求需要缓存的场景

## 一个真实例子

```javascript
// 原来直接调用 OpenAI
const openai = new OpenAI({ apiKey: process.env.OPENAI_API_KEY });

// 改为通过 AI Gateway 代理——只改 baseURL
const openai = new OpenAI({
  apiKey: process.env.OPENAI_API_KEY,
  baseURL: 'https://gateway.ai.cloudflare.com/v1/{account_id}/{gateway_name}/openai',
});

// 调用方式完全一样
const response = await openai.chat.completions.create({
  model: 'gpt-4o',
  messages: [{ role: 'user', content: 'Hello!' }],
});
```

开启语义缓存：

```javascript
const response = await fetch(gatewayUrl, {
  method: 'POST',
  headers: { 'cf-aig-cache-ttl': '3600' }, // 缓存 1 小时
  body: JSON.stringify({ messages: [...] }),
});
```
