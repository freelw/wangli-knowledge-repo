# websocket relay

## 一句话定义

`relay` 可以理解成“中转层”。

在这套平台里，它的含义是：

1. 客户端不直接连接底层 Chrome 实例
2. 客户端先连接平台提供的 websocket 地址
3. 平台 websocket 服务再去连接真实的 Chrome DevTools websocket
4. 双方消息由平台进行双向转发

## 当前 relay 在哪里

当前 websocket relay 逻辑主要位于：

1. `demo-nodejs-backend/browser-ws-gateway/server.js`
2. `demo-nodejs-backend/browser-ws-gateway/websocket.js`
3. `demo-nodejs-backend/browser-ws-gateway/session-activity.js`
4. `demo-nodejs-backend/browser-ws-gateway/cdp-tracking.js`

这说明当前 relay 已经不再挂在 `browser-manager` 进程里，而是由独立的 `browser-ws-gateway` 承载。

同时还需要结合：

1. `demo-nodejs-backend/browser-manager/routes/proxy.js`
2. `demo-nodejs-backend/browser-manager/config.js`

原因是：

1. `browser-manager` 仍负责 `/json`、`/json/version` 等 inspect 代理
2. `browser-manager` 仍负责对外 websocket 地址改写
3. `browser-ws-gateway` 负责真正的 websocket 长连接承载

## 当前核心流程

### 1. 接收客户端 websocket 连接

当前入口支持两类路径：

1. `/devtools/...`
2. `/connection?...`

### 2. 根据 `session_id` 或连接参数查找会话

当前 websocket 入口不只是“转发”，还会参与会话建立。

它会查找或使用：

1. `session_history`
2. `connection_active_browser`

在某些情况下还会做这些动作：

1. `createContext`
2. `lockContext`
3. `createSession`
4. `upsertConnectionActiveBrowser`

这意味着当前 relay 层并不纯粹，它还承担了一部分控制面职责。

### 3. 连接到底层 Chrome websocket

当前逻辑会根据 session 信息拼出底层 Chrome 地址，再建立连接。

本质上是连接：

```text
ws://<containerIp>:9222/devtools/...
```

### 4. 双向转发 CDP 消息

relay 的核心动作就是双向转发：

1. 客户端 -> Chrome
2. Chrome -> 客户端

这部分是典型的长连接承载逻辑。

## 为什么需要 relay

客户端不直接连接 Chrome，而是通过平台 relay，主要有 4 个原因。

### 1. 隐藏底层实例地址

客户端拿到的是平台地址，例如：

```text
wss://apitest.local.lexmount.net/devtools/...
```

而不是底层 Pod IP 或容器地址。

这样客户端不需要知道：

1. Pod IP
2. K8s 网络
3. 实例是否迁移
4. 底层部署细节

### 2. 做鉴权和会话校验

平台可以在 relay 前做：

1. `project_id` 校验
2. `api_key` 校验
3. `session_id` 有效性检查
4. session 是否 active 的判断

### 3. 屏蔽底层实例生命周期变化

底层浏览器实例由 K8s 动态创建和销毁，客户端不适合直接耦合这些变化。

relay 层负责把底层变化隐藏起来。

### 4. 在中间插入平台能力

当前 relay 层已经承载了部分业务逻辑，例如：

1. `Browser.setDownloadBehavior` 的下载路径改写
2. session CDP activity 上报
3. CDP tracking / timing 记录

这些都天然适合放在 relay 层。

## relay 不等于控制面

这里必须区分两个层次。

### 控制面

典型控制面动作包括：

1. `create session`
2. `delete session`
3. `lock context`
4. `list sessions`
5. `/json`
6. `/json/version`

这些本质上是“管理和查询”。

### relay 面

典型 relay 动作包括：

1. websocket upgrade
2. 建立到 Chrome 的 websocket 连接
3. 双向转发 CDP 消息
4. 记录连接与活动状态

这些本质上是“承载长连接流量”。

## 当前架构的核心问题

当前主要风险已经不再是“HTTP 控制面和 websocket relay 仍在同一个进程里”。

更准确的现状是：

1. `browser-manager` 负责 HTTP 控制面
2. `browser-ws-gateway` 负责 websocket relay
3. `browser-ws-gateway` 已经有独立的 `readyz` / draining / 优雅退出逻辑

当前仍然存在的问题是：

1. `/connection` 仍在握手阶段承载一部分控制面逻辑
2. `browser-ws-gateway` 仍直接复用 `browser-manager` 内部模块
3. websocket 稳定性风险更应关注 `browser-ws-gateway` 自己发布 / draining 时的连接表现

所以当前问题根因更准确地说是：

1. 服务形态已经拆开
2. 但 `/connection` 和共享模块边界还没有完全纯化

## 哪些说法已经过时

下面这些说法不应再写成当前现状：

1. “`browser-manager` 同时承载 HTTP API 和 websocket relay”
2. “relay 主逻辑还在 `browser-manager/websocket.js`”

这些只能作为历史背景，不应继续当作当前代码事实引用。

## 推荐拆分方向

推荐继续演进的方向仍然是把 relay 边界做得更纯：

### 1. HTTP 控制面继续收敛成更清晰的 `browser-manager-api` 形态

负责：

1. HTTP 控制面接口
2. session / context / extension / downloads 管理
3. `/json`、`/json/version`
4. 对外 websocket 地址生成

特点：

1. 可高频发布
2. 多副本
3. 不承载长连接状态

### 2. `browser-ws-gateway` 继续收敛成更薄的 relay 层

负责：

1. websocket 接入
2. `/devtools/*`
3. `/connection`
4. client <-> chrome 双向 relay
5. activity / tracking 记录

特点：

1. 尽量少发版
2. 多副本
3. 需要更长的优雅下线时间
4. readiness / draining 策略要独立设计

## `/connection` 的边界问题

当前 `/connection` 有两种可选方向。

### 方案 A：继续由 ws gateway 在握手阶段创建 session

优点：

1. 客户端协议几乎不变
2. 能兼容现有接入方式

缺点：

1. websocket 服务仍带控制面逻辑
2. relay 层难以长期保持稳定

### 方案 B：先通过 HTTP 预创建 session，再由 ws gateway 只做中转

做法：

1. 先由 API 服务完成 session / context 创建
2. websocket 服务只接收 `session_id`
3. ws gateway 只负责 relay

优点：

1. relay 层更纯粹
2. websocket 服务能更稳定
3. 后续维护成本更低

缺点：

1. 客户端或 SDK 需要补一层预创建流程

## 当前建议

从长期架构看，更推荐方案 B。

原因：

1. 它能把 relay 层真正收敛成“薄中转层”
2. API 和长连接承载面边界最清晰
3. 后续 HTTP 高频迭代不会反复影响 websocket 稳定性

如果短期更关注兼容性，也可以先以方案 A 过渡，再逐步演进到方案 B。

## 当前结论

如果只记住 relay 的当前事实和价值，可以记住这四点：

1. relay 负责隐藏底层 Chrome 实例细节
2. relay 负责承载 websocket 长连接流量
3. relay 当前已经由独立的 `browser-ws-gateway` 承载
4. relay 后续还需要继续把 `/connection` 和共享模块边界纯化
