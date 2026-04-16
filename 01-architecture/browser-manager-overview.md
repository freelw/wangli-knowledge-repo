# browser-manager 总览

## 文档定位

这篇文档从服务视角说明当前 `browser-manager` 的职责、边界、与其他服务的关系，以及它为什么仍然是主链路里的核心控制面。

## 当前定位

当前 `browser-manager` 的准确定位是：

1. 浏览器云平台的 HTTP 控制面
2. session / context / extension / downloads 的主入口
3. inspect 代理和对外 websocket 地址生成器

它已经不是 websocket relay 承载服务，但仍然是大多数前台业务逻辑的中心。

## 当前职责

从当前代码看，`browser-manager/server.js` 主要挂载这些 HTTP 路由：

1. `instance`
2. `contexts`
3. `extension`
4. `inner-interface`
5. `proxy`
6. `redirect-display`

同时提供：

1. `/health`
2. `/healthz`
3. `/readyz`
4. `/metrics`

这说明它当前的职责可以概括成下面几类。

### 1. session 控制

主要包括：

1. 创建 session
2. 查询 session
3. 删除 session
4. 异步创建 session 的保留与后续补偿入口

### 2. context 控制

主要包括：

1. 创建 context
2. 锁定 / 解锁 context
3. 查询 context
4. 删除 context

### 3. extension 与 downloads

主要包括：

1. extension 挂载相关逻辑
2. session downloads 的同步、查询、归档、删除

### 4. inspect 代理

主要包括：

1. `/json`
2. `/json/version`

这些接口会把底层浏览器的原始 websocket 地址改写成平台对外地址。

## 为什么它仍然是主链路核心

当前主链路里，虽然 websocket relay 已经拆到 `browser-ws-gateway`，但 `browser-manager` 仍然有几个不可替代的位置。

### 1. 大多数业务入口仍在这里

无论是 SDK、客户端还是内部系统，真正发起 session / context / extension / download 控制请求时，当前还是先到 `browser-manager`。

### 2. 它连接了“控制逻辑”和“实例编排”

`browser-manager` 不直接创建 Kubernetes 资源，但它负责组织出会话创建请求，再驱动底层实例编排。

### 3. 它仍是对外地址转换中心

当前平台发给客户端的 inspect / websocket 地址，仍是通过 `browser-manager` 的逻辑改写出来的。

## 它和其他服务的关系

## 1. 和 `browser-ws-gateway`

关系是：

1. `browser-manager` 负责控制面
2. `browser-ws-gateway` 负责长连接承载面

但两者目前还没有完全纯化边界。

当前仍然存在：

1. websocket 层复用 `browser-manager` 内部模块
2. `/connection` 仍带一部分控制面逻辑

所以这两者当前是：

1. 服务形态已分离
2. 代码边界仍在演进

## 2. 和 `browser-manager-reconciler`

关系是：

1. `browser-manager` 负责前台请求触发
2. `browser-manager-reconciler` 负责后台补偿和清理

这意味着很多 session 生命周期动作并不在一次 HTTP 请求里完全结束。

## 3. 和 `k8s-chrome-daemon`

关系是：

1. `browser-manager` 发起和组织会话创建逻辑
2. `k8s-chrome-daemon` 负责真正把浏览器实例编排出来

可以理解成：

1. `browser-manager` 是控制面
2. `k8s-chrome-daemon` 是执行面

## 当前不是它职责的部分

为了避免再次把旧架构写错，下面这些不应再写成 `browser-manager` 当前职责。

### 1. 不再直接初始化 websocket server

当前代码里 websocket server 已经不在 `browser-manager/server.js` 里初始化。

### 2. 不再是唯一前台接入进程

现在前台接入已经至少拆成：

1. `browser-manager`
2. `browser-ws-gateway`

### 3. 不应再把“HTTP 一发布，ws 必掉”写成它当前的直接结构缺陷

当前普通 HTTP 变更和 websocket 承载已经拆到不同服务。

现在更应关注的是：

1. `browser-ws-gateway` 自己发布 / draining 时的连接表现

## 当前剩余问题

虽然 `browser-manager` 已经从 websocket relay 中拆出，但它仍有几个边界问题没有完全收敛。

### 1. `/connection` 还没完全从控制面抽走

当前 websocket 层握手仍在做部分 session / context 创建与锁定逻辑。

### 2. 共享模块边界还没完全纯化

当前 `browser-ws-gateway` 仍复用很多 `browser-manager` 内部模块。

### 3. 命名上还没有完成“browser-manager-api”化

现在文档上可以把它理解为 HTTP 控制面，但代码和部署实体仍然叫 `browser-manager`。

## 适合什么时候优先看它

当遇到以下问题时，应优先从 `browser-manager` 入手：

1. HTTP 接口异常
2. session 创建 / 删除逻辑异常
3. context 锁定 / 解锁问题
4. extension 挂载问题
5. downloads 管理问题
6. inspect 地址返回异常

## 当前结论

如果只用一句话描述当前 `browser-manager`：

> 它现在已经不再是 websocket relay 服务，但仍然是浏览器云平台最核心的 HTTP 控制面，负责把 session、context、extension、downloads 和 inspect 这些对外能力组织起来。
