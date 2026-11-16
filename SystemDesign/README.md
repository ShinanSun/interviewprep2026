---
layout:
  width: wide
  title:
    visible: true
  description:
    visible: false
  tableOfContents:
    visible: true
  outline:
    visible: true
  pagination:
    visible: true
  metadata:
    visible: true
  tags:
    visible: true
  actions:
    visible: true
  anchors:
    visible: true
---

# 📶System Design Networking 101

## System Design Networking 101

The OSI model (Open Systems Interconnection model) is a networking framework that explains how data travels from one computer to another over a network. It divides network communication into 7 layers, each with a specific responsibility.For your system design interview, think of it as a way to understand where networking problems happen and how protocols work together.

<figure><img src=".gitbook/assets/image (1).png" alt=""><figcaption></figcaption></figure>

The three layers to focus on for system design interviews

Layer 7 — ApplicationHTTP methods, REST APIs, DNS, request/response semantics, and application protocols.

Layer 4 — TransportTCP vs. UDP, ports, reliable delivery, connection establishment, retransmissions, and latency.

Layer 3 — NetworkIP addresses, routing, and how packets travel across networks.



#### SSE vs. WebSockets vs. polling <a href="#id-3.-sse-vs.-websockets-vs.-polling" id="id-3.-sse-vs.-websockets-vs.-polling"></a>

These three approaches are common ways to deliver updates to a client.

| Feature                | SSE                                     | WebSockets                                          | HTTP polling                  |
| ---------------------- | --------------------------------------- | --------------------------------------------------- | ----------------------------- |
| Communication          | Server → client                         | Both directions                                     | Client repeatedly asks        |
| Connection             | Long-lived HTTP response                | Persistent upgraded connection                      | Repeated HTTP requests        |
| Server pushes updates  | Yes                                     | Yes                                                 | No, client must ask again     |
| Automatic reconnection | Built into `EventSource`                | Must generally be implemented by the application    | Client schedules next request |
| Best use case          | Live feeds, notifications, AI streaming | Chat, multiplayer games, interactive real-time apps | Simple periodic refreshes     |

A useful distinction: SSE is one-way at the application level. The client can still send separate HTTP requests to the server, but it cannot send messages back through the SSE stream itself.

#### 4. When would you use SSE in system design? <a href="#id-4.-when-would-you-use-sse-in-system-design" id="id-4.-when-would-you-use-sse-in-system-design"></a>

​​AI response streamingDisplay generated text progressively as the model produces it, instead of waiting for the entire answer.​​Live dashboardsPush updated metrics, job progress, or status changes to a browser.​​Notifications and event feedsDeliver new alerts or activity updates without constant client polling.​

#### load balancing ALB & NLB <a href="#load-balancing-alb-and-nlb" id="load-balancing-alb-and-nlb"></a>

> **HTTP request 是“有语义的一封信”；TCP connection 是“传输字节的电话线”。** ALB 读得懂信的内容 **所以 ALB 像一个会读地址、主题和内容类型的邮件分拣中心。**&#x4E;LB 只负责把电话线接到某一台后端。

## Load Balancing: ALB vs. NLB

### 先理解 TCP 在转发什么

```
Browser / Client
        │
        │ TCP connection
        │ 建立一条可靠、双向、按顺序的 byte stream
        ▼
Load Balancer
        │
        │ 在该通道内传输 HTTP / HTTPS / gRPC / 自定义协议数据
        ▼
Backend Target
```

* **Layer 4: Transport layer** — TCP connection
* **Layer 7: Application layer** — HTTP, HTTPS, gRPC, and custom protocols

TCP 不是 request protocol。它的作用是在两个 `IP:port` 之间建立一条可靠、双向、按顺序的 byte stream。

当 NLB 接收到 TCP connection 后，会选定一个 target，并持续双向转发该 connection 的 bytes。它不需要理解这些 bytes 的含义。

对 NLB 而言，所有数据都是：

```
"This connection's bytes"
```

对于 TCP，NLB 使用 flow hash 选择 target；一条 TCP connection 在整个生命周期内会固定到同一个 target。

***

### ALB: Forward HTTP Requests

ALB 的入口只接受 HTTP/HTTPS，并理解 HTTP request 的结构，例如：

```
GET /v1/sessions HTTP/1.1
Host: api.example.com
Authorization: Bearer ...
```

因此，ALB 可以根据 request content 做 routing。

ALB listener rules 支持根据以下内容匹配：

* Host header
* HTTP method
* Path
* Source IP
* HTTP headers
* Query string

ALB 处理的是每一个 HTTP request，而不只是“这个 client 连上来了”。

可以把 ALB 想成一个会读取地址、主题和内容类型的邮件分拣中心。

***

### NLB: Forward TCP Connections

NLB 的 listener 可以使用：

* TCP
* TLS
* UDP
* QUIC

对于 TCP listener，NLB 只知道：

```
Client IP:port → NLB IP:port
```

它通常根据 connection 的 flow characteristics 选择一个 healthy target，然后将这条 connection 的流量持续转发给该 target。

如果把两个 HTTP services 放在同一个 NLB 的同一个 listener 后：

```
Client
  │
  ▼
NLB:443
  ├── Service A
  └── Service B
```

NLB 无法根据以下 HTTP 内容决定发送给 A 或 B：

```
/v1/sessions
/v1/detections
Host header
HTTP header
Query string
```

它只能在 connection 建立时，把整条 connection 选到 A 或 B 的其中一个 target。

若需要按 URL route，应使用：

```
NLB → ALB → Backend Services
```

或由后端的 proxy/router 处理。

***

### ELB 产品家族

```
Elastic Load Balancing (ELB)
├── ALB  — Layer 7; understands HTTP/HTTPS
├── NLB  — Layer 4; handles TCP/TLS/UDP/QUIC
├── GWLB — Layer 3; inserts network/security appliances transparently
└── CLB  — Legacy load balancer
```

***

### Target Groups

NLB 和 ALB 的 target 在概念上相同：都是 target group 中健康的后端，例如：

* EC2 instance
* ECS task 的 `IP:port`

但它们不是同一种 target group，不能把同一个 target group 同时挂到 ALB 和 NLB。

同一个 ECS task 或 EC2 instance 可以注册到两个不同的 target groups。

| Load Balancer | Supported target types          |
| ------------- | ------------------------------- |
| ALB           | Instance, IP, Lambda            |
| NLB           | Instance, IP, ALB               |
| NLB           | Does not support Lambda targets |

***

### When to Choose Which

| Use case                                                                          | Recommended load balancer |
| --------------------------------------------------------------------------------- | ------------------------- |
| HTTP APIs, browser traffic, path-based routing, multiple backend services         | ALB                       |
| TCP/UDP, fixed IP, source-IP preservation, very low latency, high connection rate | NLB                       |

***

### NLB → ALB Architecture

```
Client
  │
  ▼
NLB
  │ Fixed IP / Layer 4 ingress
  ▼
ALB
  │ HTTP-aware routing
  ├── /v1/sessions   → Session Service
  ├── /v1/detections → Detection Service
  └── api.example.com → API Service
```

在这个架构中：

* **NLB** 提供稳定的 Layer 4 network entry point 和 fixed IP。
* NLB 接收 connection、健康检查 target，并按 listener/port 转发 connection。
* NLB 的 target group 中注册的是一个 ALB，因此同一 listener 的流量基本都发送到该 ALB。
* **ALB** 根据 path、host、header 和 query string 等 HTTP 语义进行业务分流。

NLB 也可以按不同 listener port 做粗粒度分流：

```
NLB:443  → ALB for HTTPS traffic
NLB:5000 → TCP Service A
NLB:9000 → TCP Service B
```

但 NLB 不能根据 HTTP path 分流：

```
/v1/sessions   → Service A
/v1/detections → Service B
```

因为 path 属于 HTTP application-layer 内容，是 ALB 的 Layer 7 routing 能力。

**准确总结：**

> NLB 为 ALB 提供固定的 Layer 4 ingress IP 和 connection-level forwarding；ALB 保留并执行 HTTP-aware request routing.
