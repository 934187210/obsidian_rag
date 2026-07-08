---
title: httpx：requests 的现代异步升级版
date: 2026-07-03
tags:
  - Python
  - HTTP客户端
  - 异步
  - 语法沉淀
---

# httpx：requests 的现代异步升级版

## 一句话总结
httpx 是 `requests` 的现代异步升级版，由同一团队（Encode）出品，同时支持 sync 和 async，并原生支持 HTTP/2，API 风格与 requests 几乎一致。

## 默认行为
- `httpx.get(url)` / `httpx.post(url)`：同步调用，写法跟 requests 几乎一样
- `httpx.AsyncClient()`：异步客户端，需配合 `async with` 上下文管理
- HTTP/2 默认**不开启**，需安装 `httpx[http2]` extras 后才能用
- 有默认 timeout（较短，不像 requests 那样不设超时），调 LLM 这类慢接口必须显式覆盖

## 显式语法
```python
import httpx

# 同步（用法跟 requests 几乎一样）
with httpx.Client() as client:
    r = client.get("https://api.example.com/v1/chat")
    data = r.json()

# 异步
async with httpx.AsyncClient(timeout=httpx.Timeout(60.0)) as client:
    r = await client.post("https://api.example.com/v1/chat", json=payload)
    data = r.json()
```

## 核心差异对比表

| 维度 | `httpx` | `requests` | `aiohttp` |
|---|---|---|---|
| 同步 | ✅ | ✅ | ❌ |
| 异步 | ✅ | ❌ | ✅ |
| HTTP/2 | ✅（需 `httpx[http2]`） | ❌ | ❌ |
| 类型注解 | 完整 | 缺失 | 部分 |
| 与 requests API 兼容 | ✅ | — | ❌ |
| 适用场景 | sync/async 通吃，迁移友好 | 纯同步经典 | 纯异步重型项目 |

## 注意点
- httpx 有默认 timeout（很短），不像 requests 默认无超时；调慢接口必须显式传 `timeout=`
- HTTP/2 需额外装 `httpx[http2]`，否则降级 HTTP/1.1
- async 路径下 `response.json()` / `response.text` 是同步方法，**不需要 await**，只有网络调用本身需要 await
- 异步客户端必须用 `async with` 关闭，否则连接池泄漏

## 选型观察
`backend/app/agent/intent_parser.py:136` import httpx，`LlmIntentParser` 大概率用 `httpx.AsyncClient` 调 LLM 提供方（OpenAI 兼容 API 或自托管模型）。选 httpx 而非 `requests` 的核心原因：
- LLM 调用是 I/O 密集型（几百毫秒到几秒），同步会阻塞事件循环
- async 路径下多个 intent 可以并发推理，整条管线吞吐直接上去
- httpx 的 async 接口跟 requests 写法几乎一样，迁移成本低

判断当前文件走的是 sync 还是 async：搜 `AsyncClient`。出现就是异步路径，只有 `httpx.get(...)` 或 `httpx.Client()` 才是同步。

## 官方文档
- [httpx 官方文档](https://www.python-httpx.org/)
- [GitHub 仓库（encode/httpx）](https://github.com/encode/httpx)
- [requests 文档（对比参考）](https://requests.readthedocs.io/)