# session 生命周期

## 文档定位

这篇文档说明当前平台里一个 session 从创建、激活、使用到关闭的大致生命周期。

重点不是穷举所有字段，而是说明主链路上各服务分别在什么阶段参与。

## 参与角色

当前 session 生命周期主要涉及这些组件：

1. `browser-manager`
   - HTTP 控制面
2. `browser-ws-gateway`
   - websocket 接入与 relay
3. `browser-manager-reconciler`
   - creating session 补偿与清理
4. `k8s-chrome-daemon`
   - 实际浏览器实例编排

## 生命周期总览

可以把当前 session 生命周期理解成下面几段：

1. 请求创建 session
2. 记录 creating 状态或直接创建
3. 实际浏览器实例启动
4. session 激活并挂上 websocket 地址
5. 客户端通过 HTTP 或 websocket 使用 session
6. session 被删除、超时关闭或被补偿任务清理
7. closed history / downloads 等后台清理继续收尾

## 1. 创建入口

当前最常见的创建入口在：

`browser-manager/routes/instance.js`

### 入口 A：`POST /instance`

这是同步创建路径。

它会做这些事：

1. 校验 `project_id` / `api_key` / `browser_mode`
2. 处理 context 参数和锁定逻辑
3. 解析扩展挂载和代理配置
4. 调用 `createSession(...)`

### 入口 B：`POST /instance/v2`

这是异步创建路径。

它的特点是：

1. 先快速保留一个 `session_id`
2. 以 `202 Accepted` 先返回
3. 再异步继续补做创建过程

这条路径更适合“先拿到 session_id，再等创建完成”的流程。

## 2. context 绑定与锁定

如果请求里带了 context，创建 session 前通常会发生：

1. 查找 context
2. 根据模式决定是否加锁
3. 在必要时删除弱锁残留

当前这部分逻辑主要在：

1. `browser-manager/routes/instance.js`
2. `browser-manager/utils/contexts-manager.js`

关键点是：

session 生命周期和 context 生命周期并不是完全独立的。

context 是否被锁定、是否复用，直接影响 session 能否创建。

## 3. `createSession(...)` 之后发生什么

当前 `createSession(...)` 在：

`browser-manager/utils/instance-helpers.js`

它负责把 session 送入真正的创建流程。

从整体设计看，这一步通常会涉及：

1. 写入 `session_history`
2. 调起浏览器实例创建
3. 在实例准备好后补齐 websocket 调试地址
4. 将 session 状态切到可用

## 4. session 激活

当底层浏览器实例成功启动后，session 会进入 active 状态。

active session 的关键信息通常包括：

1. `session_id`
2. `browser_mode`
3. `container_id` / `pod`
4. `container_ip`
5. `port`
6. `websocket_debugger_url`
7. `websocket_debugger_url_internal`

在当前实现里，active session 还会对外暴露：

1. `inspect_url`
2. `ws`
3. `ws_internal`

这部分格式化逻辑当前在：

`browser-manager/utils/instance-helpers.js`

## 5. websocket 使用路径

当客户端走 websocket 接入时，当前主要经过：

`browser-ws-gateway/websocket.js`

这里有两类路径：

### 路径 A：`/devtools/*`

客户端直接带 `session_id` 访问已有 session。

### 路径 B：`/connection`

这是当前更复杂的一条路径。

在握手阶段，它仍会做一部分控制面动作，包括：

1. 校验 `project_id` / `api_key`
2. 查询 `connection_active_browser`
3. 查询已有 active session
4. 必要时创建 context
5. 锁定 context
6. 创建 session
7. 回写连接映射

这也是为什么当前文档里一直强调：

1. websocket 服务已经独立
2. 但 `/connection` 还不是“纯 relay”

## 6. creating session 的后台补偿

当前 `browser-manager-reconciler` 会周期性运行后台补偿任务。

入口在：

`demo-nodejs-backend/browser-manager-reconciler/reconciler.js`

当前至少包括这些循环任务：

1. `reconcileActiveSessions`
2. `reconcileCreatingSessions`
3. `expireOldDownloads`
4. `cleanupClosedSessionHistory`

其中与 session 生命周期直接相关的是：

1. creating session 的补偿
2. active session 的清理
3. closed history 的清理

这说明 session 生命周期并不只靠前台请求驱动，也有后台回收和修正逻辑。

## 7. session 关闭

当前 active session 的关闭至少有几类来源：

1. 主动删除
2. 批量删除
3. reconciler 检测到异常后关闭
4. 其他清理逻辑触发关闭

在 `instance-helpers.js` 里可以看到：

1. `deleteActiveSession`
2. `deleteAllActiveSessions`
3. `closeSession(...)`

这些逻辑通常会做：

1. 停掉底层浏览器实例
2. 把 session 状态从 active 更新为 closed
3. 记录 `closed_at`
4. 触发后续收尾逻辑

## 8. session 关闭后的收尾

session 关闭后并不代表生命周期完全结束。

后续还会有这些收尾动作：

1. downloads 清理
2. closed session history 清理
3. 相关上下文和指标收尾

因此更准确的理解是：

1. “session 对外不可用”早于
2. “session 所有后台痕迹都被清理完”

## 9. 当前最重要的边界

### 1. session 生命周期跨多个服务

不要把 session 生命周期理解成只在 `browser-manager` 内部完成。

它实际上跨越：

1. `browser-manager`
2. `browser-ws-gateway`
3. `browser-manager-reconciler`
4. `k8s-chrome-daemon`

### 2. `/connection` 仍然把一部分创建逻辑放在 websocket 层

这意味着：

1. session 的创建入口并不完全收敛在 HTTP 控制面
2. 当前仍处于“边界继续纯化”的阶段

### 3. session 生命周期和 context 生命周期强相关

context 的创建、锁定、解锁、复用，都会直接影响 session 生命周期。

## 当前结论

如果只用一句话概括当前实现：

> session 生命周期当前是一个跨控制面、websocket 网关、后台补偿和实例编排服务的联合流程；服务已经拆分，但创建与连接逻辑还没有完全收敛到单一入口，因此后续演进重点不是再发明新的生命周期，而是继续把现有边界纯化和文档化。
