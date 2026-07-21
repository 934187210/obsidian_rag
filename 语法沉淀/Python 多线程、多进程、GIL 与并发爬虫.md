---
title: Python 多线程、多进程、GIL 与并发爬虫
date: 2026-07-21
tags:
  - Python
  - 并发
  - 多线程
  - 多进程
  - GIL
  - 爬虫
  - 语法沉淀
---

# Python 多线程、多进程、GIL 与并发爬虫

## 一句话结论

CPython 支持多线程。GIL 限制的是同一进程中同一时刻通常只有一个线程执行 Python 字节码，不妨碍多个线程同时发出 HTTP 请求、等待网络响应；因此线程适合 I/O 密集型爬取，纯 Python 的重计算通常应使用多进程。

## 多线程和多进程分别解决什么问题

多线程让一个进程中的多个任务交替推进，它们共享内存，创建和通信成本较低。最典型的收益来自等待：一个线程等待网络、磁盘或数据库时，其他线程可以继续工作。

多进程让多个独立的进程工作，每个进程都有自己的内存空间。它可以使用多个 CPU 核心真正并行执行，更适合图像处理、大规模解析、数值计算等 CPU 密集型任务；代价是进程创建和进程间通信更重。

| 场景 | 优先选择 | 原因 |
| --- | --- | --- |
| HTTP 请求、文件读写、数据库查询 | 多线程或异步 I/O | 时间主要花在等待外部系统 |
| 纯 Python 的大循环、复杂计算 | 多进程 | 可利用多个 CPU 核心，避开传统 GIL 的限制 |
| 需要较强故障隔离 | 多进程 | 一个子进程异常通常不会直接破坏其他进程的内存 |

## GIL 到底限制了什么

以 4 个线程抓取同一网站的不同页面为例：

```text
线程 1 -> https://site.example/news/1 -> 等待响应
线程 2 -> https://site.example/news/2 -> 等待响应
线程 3 -> https://site.example/news/3 -> 等待响应
线程 4 -> https://site.example/news/4 -> 等待响应
```

每个线程发出请求时需要执行少量 Python 代码；随后大部分时间在等待网络数据。等待 socket I/O 时，CPython 不会让该线程持续占用 GIL，其他线程可以运行并发出自己的请求。因此，4 个请求可以同时在网络中传输，也可能同时被对方网站处理。

当响应返回后，线程需要执行 Python 代码读取、清洗或解析页面内容，此时传统 CPython 仍会受 GIL 约束：同一时刻通常只有一个线程执行这部分 Python 字节码。爬虫通常是 I/O 等待远多于轻量解析，所以多线程仍然有效；如果 HTML 后处理是很重的纯 Python 计算，线程不会带来等比例的 CPU 加速。

GIL 不是业务数据的互斥锁。即使有 GIL，"检查 URL 是否已经存在"和"记录该 URL"也可能在两个线程之间交错，仍需用 `Lock` 把这两个动作保护为一个整体。

## 并发爬同一网站，不重复处理 URL

要同时满足并发和去重，使用两层机制：

1. **任务队列**：线程安全地分发待抓取 URL。一个 URL 被某个工作线程取走后，不会再同时交给另一个线程。
2. **已发现集合**：URL 入队前先检查并登记，只有第一次发现时才能入队。检查和登记必须放在同一把锁内。

简化骨架如下。`fetch`、`extract_links` 和 `normalize_url` 由具体项目实现：

```python
from queue import Queue
from threading import Lock

url_queue = Queue()
seen_urls = set()
seen_lock = Lock()


def add_url(raw_url):
    url = normalize_url(raw_url)

    with seen_lock:
        if url in seen_urls:
            return False
        seen_urls.add(url)
        url_queue.put(url)
        return True


def worker():
    while True:
        url = url_queue.get()
        try:
            html = fetch(url)
            for link in extract_links(html, url):
                add_url(link)
        finally:
            url_queue.task_done()
```

启动前只把入口 URL 交给 `add_url`。启动多个 `worker` 后，队列负责把不同任务交给不同线程；`seen_urls` 防止同一 URL 被重复入队。

## URL 去重前先规范化

同一内容可能以多种 URL 形式出现，例如：

```text
https://site.example/a
https://site.example/a/
https://site.example/a#comments
https://site.example/a?utm_source=campaign
```

是否属于同一页面取决于网站规则，但通常应在 `normalize_url` 中至少移除片段标识（`#...`），并按业务需要统一域名、尾部斜杠和无意义的追踪参数。规范化必须发生在访问 `seen_urls` 之前，否则形式不同的同一页面仍会被重复抓取。

## 去重的边界

内存中的 `seen_urls` 只在当前进程的一次运行内有效。以下情况需要共享且可持久化的去重记录：

- 爬虫重启后不想重新抓取；
- 使用多个进程；
- 多台机器共同抓取。

这时可以用数据库的唯一约束，或使用具备原子“仅首次写入”语义的共享存储。关键仍然是：去重判断和登记必须是一个原子动作。

## 实际运行约束

能并发请求同一个网站，不代表应该无限提高线程数。网站可能按 IP、账号或接口限流，可能返回 `429 Too Many Requests`，也可能依据 `robots.txt` 限制抓取范围。并发数、超时、重试和请求间隔应依据目标网站规则与实际响应情况设定。

