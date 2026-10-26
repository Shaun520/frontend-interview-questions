# 前端面试官：讲一讲双 Token 与 Token 无感刷新

> 一道把「登录态保活」讲透的鉴权面试必答题

## 一、完整回答框架：五层讲清双 Token

视频从一个面试现场切入：

- 今天面了个 4 年前端，面试官问：**项目里怎么做双 Token 实现登录态无感刷新？完整流程和踩坑点说一下**；
- 不少开发只知道存一个 Token，过期就跳登录页，用户体验很差；
- 问到双 Token 设计、并发请求到底刷新互斥问题就答不上来。

这是中后台、移动端项目必备的**权限方案**，也是进阶前端面试高频考点。真正讲清楚的关键是五层递进——**基础概念 → 完整流程 → 代码实现 → 并发与安全踩坑 → 面试满分总结**。

### 第 1 层：两个 Token 是什么

| Token | 作用 | 有效期 | 特点 |
| --- | --- | --- | --- |
| **Access Token（访问令牌）** | 日常接口鉴权 | 短（1~2 小时） | 降低被盗用后的安全风险 |
| **Refresh Token（刷新令牌）** | **专门用来换新 Access Token** | 长（几天/几周） | 不参与业务接口鉴权，只走刷新接口 |

两个配合起来就是双 Token 方案——**短期令牌做业务、长期令牌做续期**。

### 第 2 层：完整执行流程（8 步闭环）

```
① 用户登录（账号密码校验）
   │
② 后端同时返回 Access Token + Refresh Token
   │
③ 前端持久化存储（localStorage / Cookie）
   │
④ 后续业务请求 → 请求头带 Access Token 鉴权
   │
⑤ Access Token 过期 → 后端返回 401（未授权）
   │
⑥ 前端拦截 401，带 Refresh Token 调刷新接口
   │
⑦ 后端校验 Refresh Token 合法 → 返回全新 Access Token
   │        （部分场景也同步更新 Refresh Token）
   │
⑧ 前端更新本地令牌，重新发起之前失败的请求
   │
   ⟶ 用户全程无感知 = 无感刷新
```

### 第 3 层：核心代码实现（axios 拦截器）

```js
// 响应拦截器里统一处理 401
import axios from 'axios'

let isRefreshing = false          // 全局刷新锁
let pendingRequests = []          // 等待队列：刷新期间进来的请求

function addPendingRequest(req) {
  return new Promise((resolve) => {
    pendingRequests.push((newToken) => {
      req.headers['Authorization'] = `Bearer ${newToken}`
      resolve(axios(req))
    })
  })
}

function refreshToken() {
  const refreshToken = localStorage.getItem('refresh_token')
  return axios.post('/api/auth/refresh', { refresh_token: refreshToken })
    .then(({ data }) => {
      localStorage.setItem('access_token', data.access_token)
      // 部分场景也同步更新 refresh_token
      if (data.refresh_token) {
        localStorage.setItem('refresh_token', data.refresh_token)
      }
      return data.access_token
    })
}

axios.interceptors.response.use(
  (res) => res,
  async (err) => {
    const originalReq = err.config

    if (err.response?.status === 401 && !originalReq._retry) {
      if (!isRefreshing) {
        isRefreshing = true
        originalReq._retry = true  // 标记已刷新，避免死循环
        try {
          const newToken = await refreshToken()
          // 刷新成功 → 遍历等待队列，依次重发
          pendingRequests.forEach((cb) => cb(newToken))
          pendingRequests = []
          // 重发当前这个失败请求
          return axios(originalReq)
        } catch (refreshErr) {
          // Refresh Token 也过期 → 清空令牌，强跳登录页
          localStorage.removeItem('access_token')
          localStorage.removeItem('refresh_token')
          router.push('/login')
          return Promise.reject(refreshErr)
        } finally {
          isRefreshing = false
        }
      } else {
        // 正在刷新中 → 把新请求塞进等待队列
        return addPendingRequest(originalReq)
      }
    }
    return Promise.reject(err)
  }
)
```

### 第 4 层：关键优化与避坑（面试加分项）

#### ① 并发请求互斥——为什么要加锁 + 队列

> 短时间多个接口同时 401，如果不控制会**重复调刷新接口、令牌被覆盖、请求丢数据**。

解决：全局 `isRefreshing` 标记 + `pendingRequests` 等待队列。刷新期间进来的 401 请求全部入队，刷新完毕统一执行。

#### ② Refresh Token 续期策略

| 方案 | 做法 | 优缺点 |
| --- | --- | --- |
| **固定有效期** | 到期直接失效 | 安全性高；缺点是长期不操作会被强制下线 |
| **滑动续期** | 每次刷新 Access Token 时，同步返回新 Refresh Token | 延长登录时长；适合常在线系统 |

两种都对，看业务场景选——网银类高安全用固定，IM/中后台用滑动。

#### ③ 安全防护（后端 + 前端）

| 防护 | 做法 | 原因 |
| --- | --- | --- |
| **Refresh Token 存 HttpOnly Cookie** | 不让前端 JS 读到 | 防止 XSS 窃取 |
| **Refresh Token 单点校验** | 一地登录后直接作废旧 Token | 防止多端同时用 |
| **限制刷新接口频率** | 后端限流 | 防止恶意刷接口暴力破解 |

#### ④ 主动登出逻辑

别只跳登录页——**前端清两个 Token + 调后端接口作废服务端令牌**。如果 Refresh Token 没在服务端作废，被偷了还是能刷新。

---

## 二、面试满分总结化数（背下来就能讲）

> **金句**：双 Token 分为短期有效的 **Access Token**（用于接口鉴权）和长期有效的 **Refresh Token**（专门用来刷新令牌）；Access Token 过期返回 401 时，前端拦截请求利用 Refresh Token 换取新令牌，通过 **`isRefreshing` 锁 + 请求队列** 处理并发、保证只执行一次刷新操作，再重发失败请求实现全程无感；若 Refresh Token 也失效则清空登录信息跳转登录页。兼顾安全性（短期令牌降低盗用风险、Refresh Token 存 HttpOnly Cookie）与用户体验（无感刷新不打断操作）。

**回答主线：**
两个 Token 分工（短期 Access 做业务 / 长期 Refresh 做续期）→ 登录时后端双发 + 前端持久化 → 业务请求带 Access → 401 拦截 + 锁 + 队列 → 刷新成功统一重发 / 刷新失败强跳登录 → 安全：HttpOnly Cookie、单点校验、接口限流 → 主动登出要前后端双清。

---

*素材来源：B 站视频《前端面试官：讲一讲双 Token 与 Token 无感刷新》（BV1gdEE6pETd），基于视频逐字稿与画面文字整理为回答思路框架；视频后半段为资料推广内容，已略去。*
