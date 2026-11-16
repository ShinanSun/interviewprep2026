# 常见 practices



Cache / active working data → RAM

* Read < 1ms
* Write < 1ms
* Throughput for read/write: 100k QPS per instance for in-memory caches like ElastiCache Redis.
* Memory-bound (up to 1TB)

\
Database files / durable data → SSD

* Read \~5ms || Read throughput  \~50k TPS
* Write \~10ms | | Write throughput \~10k TPS
* 64 TiB+ storage capacity



App Server compute:\
\- 1k RPS per instance&#x20;





### 1. 如何保证生产Global unique ID ？ Snowflake-style ID

Snowflake-style ID 最早来自 **Twitter（现在的 X）在 2010 年公开的 Snowflake 项目**。Twitter 当时需要为大量 Tweet 等对象生成 ID，但传统数据库的 auto-increment ID 很难在多台机器、多个数据中心同时扩展：大家都去请求同一个 counter，会形成中心化瓶颈和单点风险。 Annotation 1 ([blog.x.com](https://blog.x.com/en/engineering/announcing-snowflake), [github.com](https://github.com/twitter-archive/snowflake))

它的核心想法很漂亮：

> **不要先请求中央系统拿号码；让每台机器在本地就能生成一个全局不重复的数字。**

把下面三件事打包进一个 64-bit number：

```
timestamp + worker/machine ID + sequence number
```

Twitter 的早期实现正是用 timestamp、worker number 和 sequence number；worker ID 在启动时通过协调服务分配，以确保不同 generator 不会拿到同一编号。([blog.x.com](https://blog.x.com/en/engineering/announcing-snowflake), [github.com](https://github.com/twitter-archive/snowflake/blob/b3f6a3c6ca/src/main/scala/com/twitter/service/snowflake/IdWorker.scala))

它成为常见实践，不是因为 “Snowflake” 这个名字本身，而是因为它解决了一个很普遍的 trade-off：

| 方案                      | 优点                                  | 问题                             |
| ----------------------- | ----------------------------------- | ------------------------------ |
| Database auto-increment | 最简单、严格唯一                            | central DB/counter 可能成为瓶颈      |
| Random UUID             | 不需协调、容易生成                           | 很长，不适合直接做短链接 code              |
| Random Base62           | code 短、难猜                           | 需要 collision check + retry     |
| Snowflake-style ID      | 本地高速生成、全局唯一、较短、roughly time-ordered | 要管理 worker ID 和 clock rollback |

Snowflake ID 很适合：Tweet ID、订单 ID、消息 ID、短链接内部 ID 等。它的好处是 ID generator 可以有多台机器并发运行；每台机器本地生成，不需要每个请求都同步访问中央数据库。([blog.x.com](https://blog.x.com/en/engineering/announcing-snowflake), [github.com](https://github.com/twitter-archive/snowflake/blob/b3f6a3c6ca/src/main/scala/com/twitter/service/snowflake/IdWorker.scala))

但在短链接面试中，不要因为它“常见”就强行用。更成熟的表达是：

> For an initial design, I can generate random Base62 codes and use a conditional write to detect collisions. If ID generation itself becomes a high-throughput bottleneck, I can use a Snowflake-style distributed ID generator and Base62-encode the result.

也就是说：**规模化方案**：Snowflake numeric ID → Base62

* **最终安全网**：数据库仍拒绝重复 `shortCode`

这样你展示的是：先选简单可行的方案，再在规模需要时升级，而不是背诵某个技术名词。
