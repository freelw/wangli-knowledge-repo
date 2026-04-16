# browser-manager 拆分现状与后续演进

## 文档定位

这篇文档不再记录“最初的拆分设想”，而是记录：

1. 当前代码里已经落地的拆分状态
2. 仍然保留在现有实现中的边界问题
3. 后续还可以继续演进的方向

也就是说，本文优先描述“现在已经是什么”，再描述“后面还要做什么”。

## 当前结论

`browser-manager` 相关拆分并不是停留在方案阶段，而是已经落地了一半以上。

当前更准确的系统结构是：

1. `browser-manager`
2. `browser-ws-gateway`
3. `browser-manager-reconciler`

其中：

1. `browser-manager` 当前实际承担 HTTP 控制面
2. `browser-ws-gateway` 当前实际承担 websocket 接入与 relay
3. `browser-manager-reconciler` 当前实际承担后台补偿和清理

但这不代表拆分已经完全完成，因为：

1. `browser-manager-api` 这个命名和实体还没有正式落地
2. `browser-ws-gateway` 还不是“纯 relay”
3. `/connection` 仍然带有一部分控制面逻辑

## 当前代码事实

## 1. `browser-manager` 当前已经是 HTTP 服务

当前文件：

`demo-nodejs-backend/browser-manager/server.js`

从当前实现看：

1. 启动的是 `express` + `http.createServer(app)`
2. 挂载的是各类 HTTP 路由
3. 包括：
   - `instance`
   - `contexts`
   - `extension`
   - `inner-interface`
   - `proxy`
   - `redirect-display`
4. 提供：
   - `/health`
   - `/healthz`
   - `/readyz`
   - `/metrics`

关键点是：

当前这个文件里已经没有再挂 websocket server。

这说明“HTTP 与 websocket 仍在同一个进程里”这句话对当前代码已经不成立。

## 2. `browser-ws-gateway` 已独立存在并承载 websocket relay

当前目录：

`demo-nodejs-backend/browser-ws-gateway`

关键文件：

1. `browser-ws-gateway/server.js`
2. `browser-ws-gateway/websocket.js`
3. `browser-ws-gateway/session-activity.js`
4. `browser-ws-gateway/cdp-tracking.js`

从当前实现看：

1. `browser-ws-gateway/server.js` 独立启动了自己的 HTTP server
2. `setupWebSocketServer()` 在这里完成挂载
3. 这个服务已经具备独立的：
   - 健康检查
   - readyz
   - draining 状态
   - 优雅退出逻辑
4. 收到 `SIGTERM` 后会进入 draining，再等待连接退出

这说明 relay 层不仅已经拆出来，而且已经开始有自己独立的运行期治理逻辑。

从 manifests 角度，这也不是“只有代码目录”，而是已经有独立部署形态。

当前已确认：

1. 有独立镜像
2. 有独立 Deployment / Service / PDB
3. `office`、`qcloud`、`qcloud-hk`、`aws`、`guoge` 等环境已把 `browser-ws-gateway` 纳入部署

## 3. `browser-manager-reconciler` 也已经独立

当前目录：

`demo-nodejs-backend/browser-manager-reconciler`

这说明后台补偿/清理这一层也已经从主控制面中独立出来。

因此，最初“三段拆分”的总体方向，当前至少已经在服务形态上落地了。

## 当前已落地架构

## 1. `browser-manager`

当前更接近：

1. HTTP 控制面
2. session / context / extension / downloads 相关能力
3. `/json`、`/json/version` 等 inspect 代理
4. 对外 websocket 地址生成

这部分和知识库里“控制面”定位是一致的。

## 2. `browser-ws-gateway`

当前已经实际负责：

1. websocket upgrade
2. `/devtools/*`
3. `/connection`
4. client <-> chrome 双向 relay
5. CDP activity 上报
6. CDP tracking
7. 优雅下线与 draining

这部分已经不是规划，而是现状。

## 3. `browser-manager-reconciler`

当前仍作为后台补偿和清理相关服务独立存在。

## 当前仍未完全落地的部分

虽然服务进程已经拆开了，但边界并不是完全纯净。

## 1. `browser-manager-api` 这个实体还没有真正落地

当前 HTTP 控制面服务实际仍然叫：

1. `browser-manager`

这意味着：

1. 文档不能写成“已经拆成 `browser-manager-api` + `browser-ws-gateway` + `browser-manager-reconciler`”这种完成时
2. 更准确的说法应该是：
   - 已拆出 `browser-ws-gateway`
   - 已拆出 `browser-manager-reconciler`
   - HTTP 控制面服务仍沿用 `browser-manager`

## 2. ws gateway 仍在直接复用 `browser-manager` 内部模块

从当前代码看，`browser-ws-gateway/websocket.js` 仍直接依赖了很多 `browser-manager` 内部实现，例如：

1. `session-history-db`
2. `connection-active-browser-db`
3. `instance-helpers.createSession`
4. `contexts-manager`
5. `browser-manager/chrome`
6. `browser-manager/utils/logger`

这说明：

1. 进程层已经拆开
2. 但代码层和业务边界仍然高度耦合

所以当前更像：

1. “服务已拆”
2. “模块边界未完全拆”

## 3. `/connection` 仍然带有较重的控制面逻辑

这是当前最明显的未完成项。

在 `browser-ws-gateway/websocket.js` 里，`/connection` 的握手阶段仍然会做这些事：

1. 校验 `project_id` / `api_key`
2. 查 `connection_active_browser`
3. 查 `session_history`
4. 必要时创建 context
5. 锁定 context
6. 创建 session
7. upsert 连接映射

也就是说：

虽然 websocket 服务已经拆出来了，但它还不是“纯 relay 层”。

它仍然承担了一部分控制面职责。

## 4. relay 层和控制面仍共享同一套业务库与数据模型

这本身不是错误，但意味着：

1. 后续改动时，两个服务仍容易被同一批业务逻辑牵动
2. 想进一步降低发布耦合度，还需要继续理清内部接口边界

## 当前拆分已经带来的实际收益

尽管边界还没完全纯化，当前拆分已经带来了明显收益。

### 1. 普通 HTTP 变更不再必然和 websocket 进程一起发布

这是当前最重要的改进。

过去的问题核心是：

1. HTTP 控制面和 websocket 长连接在同一个服务里
2. 一发布 HTTP，已有 websocket 就容易一起掉

现在至少在进程、服务和部署层已经分离了。

### 2. ws gateway 已有独立的优雅下线逻辑

当前 `browser-ws-gateway/server.js` 已经实现：

1. draining 状态
2. `readyz` 在 draining 时返回不可用
3. 等待连接自然退出
4. 到达超时后再关闭连接

这说明 websocket 层已经开始按“长连接服务”来治理，而不是只作为 `browser-manager` 的附属功能。

## 当前仍然存在的风险

## 1. `/connection` 还没演进成纯 relay 模式

当前 websocket 服务虽然拆了出来，但 `/connection` 仍然在握手阶段完成 session / context 相关动作。

这意味着：

1. websocket 服务仍会跟随部分业务逻辑变化
2. 还没有达到“薄 relay 层”的理想状态

## 2. 代码耦合仍然较高

当前 `browser-ws-gateway` 仍大量依赖 `browser-manager` 目录下的模块。

这意味着：

1. 服务边界已经存在
2. 但代码边界还没完全稳定

后续如果要长期维护这两个服务，最好继续把共享逻辑抽出来，避免 relay 层反向被控制面内部实现牵着走。

## 需要纠正的旧说法

以下说法对于当前系统已经不准确，不应继续作为现状写入文档：

### 1. “`browser-manager` 仍同时承载 HTTP 和 websocket”

这对当前代码已经不成立。

### 2. “普通 `browser-manager` rollout 会直接打断现有 websocket，因为还没拆分”

这也已经过时。

现在更准确的说法应该是：

1. 普通 HTTP 控制面和 websocket 承载已经分成两个 Deployment
2. 普通 `browser-manager` 发布不应再直接影响已建立 websocket
3. 当前更需要重点关注的是 `browser-ws-gateway` 自己发布或 draining 时的连接表现

### 3. “短期先拆服务”这种表述

这也已经落后于现状。

现在更准确的说法是：

1. 服务已经拆出来了
2. 当前剩余工作是继续纯化边界

## 更准确的后续演进方向

基于当前实现状态，后续不应该再描述成“先拆服务”，而应该描述成“服务已拆，继续收敛边界”。

更准确的下一步是：

### 1. 继续收敛 `/connection` 边界

理想方向仍然是：

1. HTTP 预创建 session
2. ws gateway 只接受 `session_id`
3. relay 层尽量不再承担 context / session 创建逻辑

### 2. 抽离共享业务模块

把当前 `browser-ws-gateway` 依赖的部分 `browser-manager` 内部模块抽成更稳定的共享层。

目标是让：

1. 控制面服务边界更清晰
2. relay 服务边界更清晰
3. 两边后续变更的相互影响更小

### 3. 继续强化 websocket 层的运行期治理

当前 draining 已经有了，但后续仍可继续补：

1. 更细的连接观测
2. 更明确的连接关闭原因
3. 长会话、多 tab、异常断连下的行为验证

## 当前结论

现在再描述这条链路，最准确的说法应该是：

1. `browser-manager`、`browser-ws-gateway`、`browser-manager-reconciler` 这三个服务形态已经存在
2. websocket relay 已经从 `browser-manager` 中独立出来，不应再写成“还没拆”
3. 当前真正还没完成的，不是“服务拆分”本身，而是 `/connection`、共享业务模块，以及控制面命名/边界的进一步纯化
