# SD - 现场5步模版

**Need → Things → Contract → Flow → Boxes → Risks**\
要什么 → 有什么对象 → 怎么调用 → 怎么流动 → 架构盒子 → 风险与优化

<figure><img src="../.gitbook/assets/image (1).png" alt=""><figcaption></figcaption></figure>

### Step 1: Requirements（约 5 分钟）

核心口诀：

> **What can users do? What must the system be good at? What scale matters?**\
> 用户能做什么？系统必须擅长什么？规模在哪里？

#### 1. Functional requirements：用户要做什么

<mark style="color:$warning;">只选最重要的 3 件事</mark>。句式固定为：

> Users should be able to ...

Short URL 例子：

1. Users should be able to create a short URL from a long URL.
2. Visitors should be able to open a short URL and be redirected to the original URL.
3. （可选）Users should be able to set an expiration time for a short URL.

面试现场可以说：

> I’ll focus on URL creation and fast redirection as the core flows. I’ll treat analytics, custom aliases, and abuse detection as follow-up features if time permits.

{% hint style="info" %}
Keep your requirements targeted! The main objective in the remaining part of the interview is to develop a system that meets the requirements you've identified -- so it's crucial to be strategic in your prioritization.
{% endhint %}

#### 2. Non-functional requirements：系统要“做得怎样”

<mark style="color:$warning;">只选三个</mark>。 不要说泛泛的 “highly scalable”。要说清楚：**哪个动作、达到什么标准、为什么重要。**

Short URL 例子：

* <mark style="background-color:$primary;">（先考虑CAP）</mark>Availability >> consistency：The system should be highly available: a short link should still work during partial failures.
* <mark style="background-color:$primary;">Uniqueness and Invariant (不变量）</mark>? Ensure uniqueness of short urls
* [<mark style="background-color:$primary;">Latency:</mark>](#user-content-fn-1)[^1]
  * Read latency on redirection (<100ms)， for example SLA with p99 under 100 ms.
  * Write Latency on url creation, should be reponsilve, can tolerance with p99 < 300-500ms
* <mark style="background-color:$primary;">Scalable</mark> to 1B urls and 100M DAU



**Only PICK 3 that are most relevant to your system.**

1. **CAP Theorem**: Should your system prioritize consistency or availability? Note, partition tolerance is a given in distributed systems.
2. **Environment Constraints**: Are there any constraints on the environment in which your system will run? For example, are you running on a mobile device with limited battery life? Running on devices with limited memory or limited bandwidth (e.g. streaming video on 3G)?
3. **Scalability**: All systems need to scale, but does this system have unique scaling requirements? For example, does it have bursty traffic at a specific time of day? Are there events, like holidays, that will cause a significant increase in traffic? Also consider the read vs write ratio here. Does your system need to scale reads or writes more?
4. **Latency**: How quickly does the system need to respond to user requests? Specifically consider any requests that require meaningful computation. For example, low latency search when designing Yelp.
5. **Durability**: How important is it that the data in your system is not lost? For example, a social network might be able to tolerate some data loss, but a banking system cannot.
6. **Security**: How secure does the system need to be? Consider data protection, access control, and compliance with regulations.
7. **Fault Tolerance**: How well does the system need to handle failures? Consider redundancy, failover, and recovery mechanisms.
8. **Compliance**: Are there legal or regulatory requirements the system needs to meet? Consider industry standards, data protection laws, and other regulations.

#### 3. Capacity estimation：只算会改变设计的数字

口诀：

> **Do math only when it changes a design choice.**

在我们的题目里，read-heavy 这个结论会直接导向：**cache / CDN + horizontally scalable storage**，所以值得算。

```
100M DAU
× 1 new URL per DAU per day
= 100M new URLs/day
≈ 1.2K write RPS

Read : Write = 100 : 1
= 10B redirects/day
≈ 116K redirect RPS
```

面试表达：

> We have about 1.2K average write RPS but roughly 116K average redirect RPS. This is a strongly read-heavy system, so the redirect path should be cache-first and horizontally scalable.

这一阶段结束时，你应该已经能清楚说出：

> **Build a durable URL mapping service where users create short links, and visitors are redirected with low latency and high availability at massive read scale.**

下一步我们可以把这个需求直接压缩成 **Core Entities** 的好记版本。





#### Non-functional Requirements

### Step 2: Core entities (2min)

> **Core Entities = 系统世界里的核心名词。为了描述这个产品，哪些“人、物、关系、记录”是不可缺少的名词？**

### Step 2: Core Entities（约 2 分钟）— 私人笔记模板

#### Thinking pattern

1. **Who is involved?**\
   谁参与其中？\
   `User / Admin / Seller / Driver / Viewer ...`
2. **What is the main thing being created, viewed, changed, or consumed?**\
   用户围绕什么核心对象活动？\
   `Post / Order / Message / File / Booking ...`
3. **What relationship connects two core things?**\
   哪种“关系”本身需要被记录？\
   `Follow / Membership / Like / Assignment / Permission ...`
4. **What important event or record must exist?**\
   有什么关键事件或记录需要保留？\
   `Payment / Delivery / Notification / AuditEvent ...`

#### Output format

```
Actors:
- [actor 1]
- [actor 2]

Core entities:
- [main object]
- [relationship, if it must be stored]
- [important record/event, only if required]
```

#### Rules

> **Start with nouns, not verbs.**\
> 先找名词，不讨论动作。

> **Start small: 2–5 entities.**\
> 先写 2–5 个，不要列完整数据模型。

> **Add entities later when a requirement demands them.**\
> 后面发现一个 requirement 需要新对象，再加。

#### Interview sentence

> Before jumping into the design, I’ll identify the core entities: the main actors, the primary resource, and any relationships or records that must be persisted. This is an initial model, and I’ll refine it as requirements become clearer.

### Step 3: API / System Interface（约 3 分钟

#### Goal

> **Turn each functional requirement into an external contract.**\
> 把每一条 functional requirement 变成客户端和系统之间的接口合同。

#### 1. Choose API type

```
Default: RESTful API via HTTP 
- External, resource-oriented APIs
- Standard CRUD operations

Use RPC when: Action-oriented protocol (like gRPC) that's faster than REST for service-to-service communication.
- You need a performance-focused internal service interface
```



**选择 REST：**

> For the external client-facing interface, I’ll use HTTP with a RESTful API design since the core operations are resource-oriented. I’ll now go through each functional requirement and define the minimum endpoint or endpoints required to support it, using our core entities as the request and response objects.

**选择 RPC：**

> For this latency-sensitive internal facing interface, the operation is primarily an action-oriented rather than resource-oriented, so I’d expose it as an RPC-style endpoint with explicit parameters and return values.

**需要实时推送时：**

> I’ll start with REST for the public API.  If later this feature requires the server to push subsequent updates to the client, so I’d add WebSockets or Server-Sent Events for the real-time channel. Or if the system requires a latency-critical internal call, I'd explicit with gRPC for server to server communication.&#x20;

#### 2. Fulfill each functional requirement

```
Create [Entity]      → POST /entities        /*用复数*/
Read [Entity]        → GET /entities/{id}
List/Search [Entity] → GET /entities?filters
Update [Entity]      → PATCH /entities/{id}
Delete [Entity]      → DELETE /entities/{id}
Business action      → POST /entities/{id}/action
```

举个例子：

For user management, we will assume user has logged in with email and passcode. For this POST operation, we will assume user identity comes from the verified `Authorization` header in the HTTP request.&#x20;

```
# user gives a long url and will return a short one. 
POST /urls
Authorization: Bearer <access_token>
Content-Type: application/json

{
  "longUrl": "https://longurl.com",
  "alias": "uniqueAlias",
  "expiresAt": "2026-02-13T00:00:00Z" # ISO timestamp
}

return 200 OK
{shortenurl: "shortenurl.com"}
```

The service looks up that code and returns `302 Found` with a `Location` header pointing to the original URL, so the browser redirects automatically.

```
GET /{shortcode}

HTTP/1.1 302 Found
Location: https://originallongurl.com   # Location is a standard HTTP response header
```

#### 3. Add comments only when needed

```
POST /v1/[resources]/{id}/[action]
# Explain what changes, whether it is synchronous,
# and what happens if the action was already performed.
```





### Step 4:





### Step 5:



[^1]: Define 就好了， 不要包含计算。 计算留给HLD. Define 的时候default 用P99
