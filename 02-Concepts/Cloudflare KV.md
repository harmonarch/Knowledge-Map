---
type: concept
status: active
created: 2026-04-15
tags:
  - concept
  - cache
  - kv
---

# Cloudflare KV

## 一句话定义

分布式键值数据库，数据写入后同步到全球所有边缘节点，读取极快，最终一致性。

## 它解决什么问题

需要全球低延迟读取的简单数据存储：Session、配置、缓存、限流计数器。

## 出现前提

你有读多写少的数据，且不需要强一致性。

## 核心机制

写入后数据异步传播到全球边缘节点。读取从最近的节点返回，延迟极低。最终一致性模型，写入传播有延迟。

## 收益

- 1000 万次读取 / 天免费
- 10 万次写入 / 天免费
- 1 GB 存储免费
- 全球边缘读取，延迟极低
- 支持 TTL 过期

## 代价

- 最终一致性，写入传播有延迟
- 不适合频繁更新的数据
- 不支持复杂查询

## 常见误区

- 以为 KV 可以当数据库用 — 它只适合简单的键值读写，复杂数据用 D1
- 以为写入立即全球可见 — 有传播延迟，不适合需要强一致性的场景

## 关联节点
- [[Cloudflare]]
- [[Cloudflare Workers]]
- [[Cloudflare D1]]

## 适用场景
- 用户 Session Token
- 应用配置（Feature Flags）
- API 响应缓存
- 限流计数器

## 一个真实例子

```javascript
export default {
  async fetch(request, env) {
    // 写入，1小时后过期
    await env.MY_KV.put('user:123:session', 'token-xyz', {
      expirationTtl: 3600
    });
    // 读取
    const session = await env.MY_KV.get('user:123:session');
    return new Response(session);
  }
};
```
