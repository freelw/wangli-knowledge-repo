# browser-manager 拆分方案

## 背景

当前 `browser-manager` 在一个服务里同时承担了两类职责：

1. HTTP 控制面
2. websocket relay

这会带来一个结构性问题：

只要发布 `browser-manager`，同 Pod 上承载的 websocket 连接就会随着 rollout 被中断。

因此，这不是一个简单的“多副本不够”问题，而是服务边界划分错误的问题。

## 当前现状

### 1. HTTP 和 websocket 在同一个进程里

当前 `browser-manager` 同时承载：

1. `express` HTTP 服务
2. websocket server

这意味着：

1. HTTP 代码频繁迭代
2. websocket 长连接也被绑在同一发布单元里
3. 每次发布都可能主动打断现有连接

### 2. websocket 握手阶段不只是 relay

当前 `/connection` 入口除了建立 websocket 外，还承担了控制面动作，例如：

1. 鉴权
2. 查询 `connection_active_browser`
3. 查询 `session_history`
4. 必要时创建 context
5. 锁定 context
6. 创建 session
7. 回写连接映射

这说明当前 websocket 入口并不是纯粹的中转层。

### 3. 平台对外 websocket 地址已被改写

当前对外返回给客户端的是平台 websocket 地址，而不是底层 Chrome 的原始地址。

这件事很重要，因为它意味着：

1. 客户端依赖的是平台地址
2. 将来只要平台侧 `WSS_PREFIX` 指向新服务
3. SDK 和客户端协议就有机会保持基本不变

### 4. reconciler 已经独立

当前系统里已经存在：

1. `browser-manager`
2. `browser-manager-reconciler`

说明后台补偿任务已经部分拆开，但前台接入层还没有完成职责拆分。

## 核心问题

当前故障根因不是“连接没有多副本承接”，而是：

1. 长连接流量和高频变更代码在同一个发布单元
2. rollout 时旧 Pod 会退出
3. kube-proxy 只影响新建连接，不会迁移旧连接
4. 旧连接因此断开

所以要解决的不是单点容量问题，而是架构边界问题。

## 拆分目标

拆分方案需要同时满足以下目标：

1. HTTP 控制面可以继续高频发布
2. websocket 服务尽量少发布
3. websocket 服务可以水平扩容
4. 客户端和 SDK 尽量无感
5. 后续演进时服务边界清晰

## 推荐目标结构

推荐拆成三个长期服务：

1. `browser-manager-api`
2. `browser-ws-gateway`
3. `browser-manager-reconciler`

其中 `browser-manager-reconciler` 保持现有职责不变。

## 服务职责划分

### 1. `browser-manager-api`

建议保留这些能力：

1. 所有 HTTP 控制面接口
2. session 管理
3. context 管理
4. extension 管理
5. downloads 管理
6. `/json`
7. `/json/version`
8. 对外 websocket 地址生成
9. session / context / connection 映射写库

特点：

1. 变更频率高
2. 可以高频发布
3. 不承载长连接状态

### 2. `browser-ws-gateway`

建议保留这些能力：

1. websocket upgrade
2. `/devtools/*`
3. `/connection`
4. client <-> chrome 双向 relay
5. CDP activity 上报
6. CDP tracking / timing
7. 下载路径改写等 relay 侧逻辑

特点：

1. 承载长连接
2. 需要尽量少发版
3. 需要更长的优雅下线时间
4. readiness / draining 策略必须单独设计

### 3. `browser-manager-reconciler`

职责保持不变：

1. creating session reconcile
2. cleanup
3. retention

## `/connection` 的两种实现路线

这是拆分过程里的关键边界点。

### 方案 A：保持现有客户端协议，ws gateway 在握手阶段创建 session

做法：

1. 保持现有 `/connection?project_id=...&api_key=...`
2. 由 `browser-ws-gateway` 在握手时继续执行 session / context 建立逻辑

优点：

1. 客户端几乎无改动
2. 过渡成本低
3. 能较快完成服务拆分

缺点：

1. websocket 服务仍然带有控制面逻辑
2. relay 层边界不纯
3. 后续稳定性目标会受影响

### 方案 B：通过 HTTP 预创建 session，ws gateway 只接 `session_id`

做法：

1. 新增一个 HTTP 预创建接口
2. 由 `browser-manager-api` 完成：
   - 鉴权
   - 查/建 context
   - 查/建 session
   - 回写 `connection_active_browser`
3. 返回 `session_id` 和对外 websocket 地址
4. `browser-ws-gateway` 只负责根据 `session_id` 建立 relay

优点：

1. relay 层边界最清晰
2. websocket 服务更稳定
3. 后续维护和扩容都更简单

缺点：

1. SDK 或客户端接入流程需要补一步
2. 改造成本高于方案 A

## 推荐路线

### 短期

如果目标是尽快把“HTTP 一发布就断 websocket”这个问题切开，可以先落方案 A。

理由：

1. 对客户端影响最小
2. 可以先完成服务拆分
3. 能先把发布单元错误的问题拆开

### 长期

长期更推荐方案 B。

理由：

1. 它能让 `browser-ws-gateway` 真正收敛为薄 relay 层
2. API 和长连接承载面的职责最清楚
3. 高频 HTTP 变更不会继续污染 websocket 层

因此，更合理的路线是：

1. Phase 1：先按方案 A 拆服务
2. Phase 2：再把 `/connection` 迁到 HTTP 预创建

## 拆分后的发布收益

拆分后，大多数普通 API 变更只需要发布 `browser-manager-api`。

这样可以带来几个直接收益：

1. 已建立 websocket 连接不再因为普通 API 改动而中断
2. websocket 服务可以更强调稳定性而不是快速迭代
3. 发布风险从“整套接入层一起抖动”变成“按职责隔离”

## 仍需注意的问题

即使拆分完成，也不能误以为 websocket 服务天然不会断连。

还必须额外设计：

1. ws gateway 尽量少发版
2. 收到 `SIGTERM` 后先停止接新连接
3. 给老连接保留更长的优雅退出时间
4. readiness 和 draining 策略独立配置

也就是说，拆分只是把问题从：

1. “HTTP 一发布，ws 必掉”

变成：

1. “只有 ws gateway 发布时，ws 才可能掉”

这是明显改善，但不是自动彻底解决。

## 当前结论

当前 `browser-manager` 的拆分方向可以总结为三句话：

1. 问题根因是 HTTP 控制面和 websocket relay 混在一个服务里
2. 推荐拆成 `browser-manager-api`、`browser-ws-gateway`、`browser-manager-reconciler`
3. 长期应把 `/connection` 演进为“HTTP 预创建 + ws 纯 relay”的模式
