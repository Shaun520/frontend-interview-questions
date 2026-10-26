# 前端面试官：告别无效轮询，SSE 流式实时通信方案

> 一道把「服务端单向推送」讲透的实时通信面试必答题

## 一、完整回答框架：五层讲清 SSE

视频从一个面试现场切入：

- 今天面了个 4 年前端，面试官问：**需要服务端实时推送消息，除了轮询、WebSocket，还有什么方案？讲讲 SSE**；
- 候选人只会死磕定时器轮询、WebSocket，SSE 原理、适用场景、优缺点一概说不清——实时场景选型直接失分；
- 传统定时器轮询频繁发请求、空请求浪费带宽、后端压力大；而 SSE 这种单工流式推送，现在大量用在**日志打印、AI 对话、大屏实时数据**里。

这题看起来是背 API，实则是**实时通信选型**的核心。真正会用的前端会把 SSE 分成 **5 层**来讲：是什么 → 基础使用 → 对比轮询 / WebSocket → 适用场景 → 进阶踩坑。

### 第 1 层：SSE 是什么

- **SSE = Server-Sent Events（服务端推送事件）**；
- 基于 **HTTP 协议**，客户端发起一次 `GET` 长连接；
- 服务端持续**单向**下发数据流，浏览器自动解析消息；
- **单向**：只能后端推前端，前端不能主动发数据（这是和 WebSocket 的核心区别）。

### 第 2 层：基础使用（前后端都要写）

#### 前端

```js
const es = new EventSource('/api/sse')

es.onmessage = (e) => { /* 接收普通消息 */ }
es.onerror   = ()  => { /* 异常断开，浏览器自带重试机制 */ }
es.onopen    = ()  => { /* 连接成功 */ }

// 不需要时手动关闭
es.close()
```

#### 后端（以 Node/Express 为例）

```js
// 响应头固定配置，缺一不可
res.set({
  'Content-Type': 'text/event-stream',  // 告诉浏览器这是流式响应
  'Cache-Control': 'no-cache',           // 别缓存
  'Connection': 'keep-alive',            // 保持长连接
})

// 数据格式固定：以 \n\n 结尾，一次一条
// 服务端循环分片输出，别一次写完
let id = 0
setInterval(() => {
  res.write(`id: ${++id}\n`)
  res.write(`event: message\n`)
  res.write(`data: ${JSON.stringify({ time: Date.now() })}\n\n`)
}, 1000)
```

| 头 / 格式 | 作用 |
| --- | --- |
| `Content-Type: text/event-stream` | 让浏览器进入 SSE 解析模式 |
| `Cache-Control: no-cache` | 防止中间层（CDN、代理）缓存 |
| `Connection: keep-alive` | 保持长连接 |
| `data: xxx\n\n` | 消息体，`\n\n` 是消息分隔符 |

### 第 3 层：对比轮询 vs WebSocket

| 维度 | 轮询 | **SSE** | WebSocket |
| --- | --- | --- | --- |
| 通信方向 | 单向（客户端主动问） | **单向（服务端主动推）** | 双向 |
| 请求量 | 定时反复 HTTP 请求，大量无效空包 | **一次建联，持续推送，无多余请求** | 一次握手，持久连接 |
| 带宽 | 高（轮询间隔内消息全是空包） | **低（只推有数据的）** | 中 |
| 协议 | 普通 HTTP | **原生 HTTP，无需改造** | WS 新协议，复杂 |
| 服务端改造 | 无 | **无（HTTP 响应头改一下）** | 要单独处理 WS 协议 |
| 自动重连 | 需自己写定时器重建 | **浏览器自带断线自动重连** | 需自己写 |
| 缺点 | 无效请求浪费 | **只能后端推，前端不能发** | 实现复杂、兼容性麻烦 |

### 第 4 层：适用业务场景

✅ **适合（服务端 → 前端 单向推送）**：
- AI 流式打字输出（ChatGPT、豆包的逐字输出）
- 实时日志打印
- 大屏指标刷新
- 消息公告、订单状态变更

❌ **不适合（双向交互）**：
- 聊天 IM —— 前端也要主动发消息，用 WebSocket

### 第 5 层：进阶优化 & 踩坑

#### ① 自定义 event 字段——区分不同类型消息

```js
// 后端
res.write(`event: notice\ndata: ${msg}\n\n`)
res.write(`event: alarm\ndata: ${msg}\n\n`)

// 前端
es.addEventListener('notice', (e) => { /* 处理公告 */ })
es.addEventListener('alarm',  (e) => { /* 处理告警 */ })
// onmessage 只接收没有 event 字段的默认消息
```

#### ② `id:` 字段 + `Last-Event-ID`——重连补发

```js
// 后端写 id
res.write(`id: ${msgId}\ndata: ${data}\n\n`)

// 前端断线重连时，浏览器会自动带上 Last-Event-ID 请求头
// 后端据此补发丢失的数据
```

#### ③ 跨域 & Token 场景

- **跨域**：后端配置 CORS（`Access-Control-Allow-Origin`），EventSource 行为和 fetch 一样受 CORS 约束；
- **EventSource 不支持自定义请求头**——没法在请求头里塞 `Authorization`；
- **戴 Token 的两条路**：
  - 简单方案：把 token 拼进 URL 参数（`new EventSource('/api/sse?token=xxx')`），但会暴露在 URL 日志里；
  - 更稳方案：**使用 fetch + ReadableStream 手动封装 SSE**，可以自定义请求头。

#### ④ Nginx 代理要注意

如果 SSE 接口走了 Nginx 代理，Nginx 默认会缓冲响应——**`proxy_buffering off`** 要关掉，不然浏览器永远收不到流式数据。

---

## 二、面试满分总结化数（背下来就能讲）

> **金句**：SSE 是基于 HTTP 的**服务端单向流式推送**方案，一次建立长连接后端持续下发数据；相比轮询减少大量无效请求，对比 WebSocket 实现简单、无需新协议、浏览器自带断线重连；适合 AI 流式输出、大屏实时数据等**单向推送场景**，双向交互场景再选用 WebSocket。

**回答主线：**
单向推送 → EventSource + 三个回调 → 后端三件套头 + data 格式 → 对比轮询（无无效空包）/ WebSocket（实现简单、但单向）→ 适合 AI 输出/大屏/日志、不适合聊天 IM → 进阶：自定义 event 分类型、id + Last-Event-ID 补发、跨域 CORS、Token 拼 URL 或 fetch 手动封装。

---

*素材来源：B 站视频《前端面试官：告别无效轮询，SSE 流式实时通信方案》（BV1mwEJ6EEge），基于视频逐字稿与画面文字整理为回答思路框架；视频后半段为资料推广内容，已略去。*
