# 前端面试官：大文件分片上传与 MD5 校验机制

> 一道把「文件上传底层」讲透的工程化面试高频大题

## 一、完整回答框架：四层讲清分片上传

视频从一个面试现场切入：

- 今天面了个 4 年前端，面试官随口抛了个高频真题：**大文件怎么分片上传？MD5 校验底层机制是什么？**；
- 候选人就只会一句「把大文件切成小块慢慢传」，再深问为啥不能整包直接传、MD5 到底干啥用的、断点续传和秒传怎么实现——当场卡壳，面试止步；
- 现在好多前端日常业务用过文件上传，但从来没吃透分片逻辑、MD5 校验、断点续传底层原理，面试一问就露馅。

这道题看起来像后端的事，实则是**前端工程化能力**的核心。真正讲清楚的关键是四层递进——**为什么分片 → 完整流程（含秒传+断点续传）→ MD5 核心作用 → 前端代码实现**。

### 第 1 层：为什么要分片？

| 问题 | 分片怎么解决 |
| --- | --- |
| 大文件一次性提交容易**超时** | 分片小，单个请求不会超时 |
| 网络波动就得**从头重传** | 只重传失败的那个分片 |
| 会被**服务器请求体大小限制**（Nginx 默认 1MB） | 分片逐个请求，绕过限制 |
| 无法**并发提速** | 多个分片并行上传，速度翻倍 |

### 第 2 层：完整流程（8 步闭环，面试必背）

```
① 前端用 File.slice() 把大文件切固定大小分片（通常 2~5MB）
       │
② 用 SparkMD5 计算：整体文件 MD5 + 每个分片的 MD5
       │
③ 先发整体 MD5 给后端校验（秒传判断）
       │
④ 后端查到已有同 MD5 文件 → 直接秒传成功，不用再上传
       │
⑤ 没秒传的话，后端返回**已上传过的分片列表**（断点续传依据）
       │
⑥ 前端只上传**缺失的分片**，不做无用重复传输（可并发）
       │
⑦ 所有分片传完 → 通知后端自动合并文件
       │
⑧ 最后前后端比对整体 MD5 → 校验文件完整性，防止丢包 / 篡改
```

> 💡 视频画面里明确标了「**先把整体 MD5 发给后端校验 → 直接秒传成功不用再上传**」、「**前端只上传缺失分片，不做无用重复传输**」。

### 第 3 层：MD5 核心作用（4 点记住）

| 作用 | 说明 |
| --- | --- |
| **① 校验文件完整性** | 传输完毕后前后端比对整体 MD5，防止损坏 / 篡改 |
| **② 秒传** | 靠整体 MD5 查后端是否已有相同文件，有就直接返回成功 |
| **③ 断点续传** | 靠分片 MD5 标记每一块，后端返回已上传分片列表 → 前端跳过 |
| **④ 唯一标识文件** | 防止同一文件重复存储 |

**SparkMD5 为啥不用原生 MD5？** 大文件算 MD5 是**同步阻塞操作**，会卡死主线程。SparkMD5 可以分片算、还能丢 WebWorker 里异步算，不卡 UI。

### 第 4 层：前端代码实现（核心片段）

#### 切分片 + 算 MD5

```js
import SparkMD5 from 'spark-md5'

const CHUNK_SIZE = 2 * 1024 * 1024  // 2MB 一片

async function uploadFile(file) {
  // 1. 切分片
  const chunks = []
  let start = 0, index = 0
  while (start < file.size) {
    chunks.push({
      index,
      blob: file.slice(start, start + CHUNK_SIZE),
    })
    start += CHUNK_SIZE
    index++
  }

  // 2. 算整体 MD5（SparkMD5 支持分片增量计算）
  const spark = new SparkMD5.ArrayBuffer()
  for (const chunk of chunks) {
    const buf = await chunk.blob.arrayBuffer()
    spark.append(buf)
    chunk.md5 = spark.end()  // 先给每个分片也算上
  }
  const fileMd5 = spark.end()  // 整体 MD5
  // ...
}
```

#### 秒传 + 断点续传逻辑

```js
// 3. 先拿整体 MD5 去问后端
const { data } = await axios.post('/api/upload/check', { fileMd5 })

if (data.uploaded) {
  // 秒传：后端已有相同文件
  return { success: true, url: data.url }
}

// 4. 后端返回已上传分片列表 → 过滤出缺失的
const uploadedIndexes = new Set(data.uploadedChunks || [])
const pendingChunks = chunks.filter((c) => !uploadedIndexes.has(c.index))

// 5. 并发上传缺失分片（限制并发数，别打爆带宽）
await Promise.all(
  pendingChunks.map((c) =>
    axios.post('/api/upload/chunk', c.blob, {
      params: { fileMd5, index: c.index, chunkMd5: c.md5 },
      headers: { 'Content-Type': 'application/octet-stream' },
    }),
  ),
)

// 6. 通知后端合并
await axios.post('/api/upload/merge', { fileMd5, totalChunks: chunks.length })

// 7. 后端合并完会返回最终文件 MD5，前端可以再校验一次
```

---

## 二、面试满分总结化数（背下来就能讲）

> **金句**：大文件分片上传分四步——**切分片 → 算 MD5（整体+分片）→ 先发整体 MD5 问后端（秒传判断 + 返回已上传分片列表做断点续传）→ 只传缺失分片 → 全部传完通知合并 → 前后端比对整体 MD5 校验完整性**。MD5 四个作用：完整性校验、秒传、断点续传、文件唯一标识。用 SparkMD5 异步算不卡主线程，并发上传提速度。

**回答主线：**
为什么分片（超时/重传/大小限制/并发）→ File.slice 切 2~5MB 片 → SparkMD5 算整体+分片 MD5 → 先发整体 MD5：后端有就秒传、没有返回已上传分片列表 → 过滤缺失分片并发上传 → 通知合并 → 比对 MD5 防篡改 → MD5 四个作用（完整性/秒传/断点续传/去重）。

---

*素材来源：B 站视频《前端面试官：大文件分片上传与MD5校验机制》（BV1yFL66bEWE），基于视频逐字稿与画面文字整理为回答思路框架；视频后半段为资料推广内容，已略去。*
