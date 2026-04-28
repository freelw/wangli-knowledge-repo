# k8s-chrome-daemon 查询链路与性能瓶颈

## 目的

这篇文档用于回答 4 个问题：

1. `browser-manager-reconciler` 为什么会把 `getContainerInfo()` 打成固定节奏的查询洪峰
2. `k8s-chrome-daemon` 的 `GET /chromium/{id}` 当前到底走了哪条链路
3. 这条链路的主要瓶颈在哪里
4. 后续应该按什么顺序优化

## 问题背景

此前出现过这类现象：

- `browser-manager/utils/docker.js` 中的 `getContainerInfo()` 在 10 秒后超时
- `browser-manager-reconciler` 一度会把这类超时误判成 session 失效并关闭数据库中的 session

后续虽然已经把“超时直接关 session”的动作去掉，但这只能避免误杀，不能说明底层查询链路本身没有性能问题。

真正需要分析的是：

- 为什么 `GET /chromium/{id}` 会在 10 秒内拿不到结果
- 这到底是 `browser-manager` 调用方式的问题，还是 `k8s-chrome-daemon` 本身承载能力的问题

## 当前查询链路

### 1. 触发方

当前触发查询的核心来源是：

- `browser-manager-reconciler`

其中：

- `reconcileActiveSessions()` 会拉取全部 active sessions
- 然后串行调用 `validateSessionContainer(dbSession)`
- 每个 session 都会调用一次 `getContainerInfo(containerId)`

也就是说，当前模型不是“按需零散查询”，而是：

- 每分钟对全部 active sessions 做一轮固定扫描

## 2. browser-manager 侧行为

`getContainerInfo()` 的当前特点是：

- 每次调用都会发起一个新的 HTTP 请求
- 超时时间固定为 10 秒
- 当前没有专门的 keepalive agent 复用连接

因此在 active sessions 较多时，会形成：

- 固定周期
- 大量短连接
- 集中打向同一个 daemon API

这对下游服务天然不友好。

## 3. k8s-chrome-daemon 侧行为

`GET /chromium/{id}` 当前在 `pkg/server/server.go` 中的实现很直接：

- 收到 REST 请求
- 调用 `s.Client.Get(ctx, client.ObjectKey{Name: id, Namespace: namespace}, instance)`
- 把查到的 `BrowserInstance` 直接编码返回

这里的关键点是：

- 这个 `Client` 不是 `mgr.GetClient()`
- 而是 `server.NewK8sClient(config, scheme)` 单独 new 出来的 client

这意味着当前 REST 查询链路是：

- 每个 `GET /chromium/{id}` 都直连 apiserver

而不是：

- 先读 controller-runtime 本地 cache

## 当前瓶颈判断

### 瓶颈 1：REST GET 直连 apiserver

这是当前最核心的问题。

当前 `k8s-chrome-daemon` 并不是一个拥有本地只读状态视图的查询服务，而更像是：

- 一个把浏览器实例查询请求转发给 apiserver 的薄代理层

这样带来的后果是：

1. daemon 自己不掌握稳定的本地状态副本
2. 每次查询都要承担 apiserver 的实时开销
3. 一旦 apiserver 抖动、网络波动、对象读取排队或序列化变慢，用户查询就直接受影响

因此 10 秒超时虽然在 `browser-manager` 侧暴露出来，但真正的风险放大点在：

- 查询路径过于依赖 apiserver 实时可用性

### 瓶颈 2：单副本承载 controller + API server

当前 `apps/chrome-daemon/manager/manager.yaml` 中：

- `replicas: 1`

并且同一个进程中同时承载：

- BrowserInstance controller
- Timeout controller
- PocketBase subscriber
- REST API server

资源限制也比较紧：

- request: `100m / 128Mi`
- limit: `500m / 512Mi`

这意味着：

- 控制面工作和查询面工作没有隔离
- 一个方向忙起来，会直接挤占另一个方向的资源

所以即使 `GET /chromium/{id}` 本身逻辑很短，只要：

- controller 正在忙
- timeout controller 正在批量处理
- PocketBase subscriber 正在消耗资源
- 或进程出现 GC / CPU 抖动

REST 查询延迟也会一起上升。

### 瓶颈 3：reconciler 的固定全量扫描模型

当前 `browser-manager-reconciler` 的模型是：

- 每 1 分钟全量扫描 active sessions
- 每个 session 查询一次 `GET /chromium/{id}`

这会形成一种很典型的压力模式：

- 平时没压力
- 到固定时刻突然打出一波查询洪峰

它不一定会把 daemon 打死，但很容易放大以下问题：

1. 短时间内大量新建连接
2. 单副本 API server 的瞬时排队
3. apiserver 查询延迟抖动

### 瓶颈 4：缺少 GET 观测指标

当前 `k8s-chrome-daemon` 已有的指标只有：

- create request duration
- wait ready duration
- delete request duration

缺少的恰恰是这次问题最关键的：

- get request duration
- get request result
- inflight query count
- handler 总耗时与 `Client.Get` 耗时拆分

所以现在看到“10 秒超时”时，无法立刻判断卡在：

1. HTTP server 接收前排队
2. `Client.Get`
3. JSON encode
4. CPU / 内存 / GC 抖动
5. 与 controller 的资源争用

这会导致优化只能靠猜。

## 当前不建议误判的方向

### 1. 不要把问题简单归结为 `browser-manager` 的 10 秒超时太短

10 秒只是暴露问题的阈值，不是根因。

如果只是把 timeout 从 10 秒调到 20 秒：

- 只能延后报错
- 不能消除查询高延迟
- 还会让问题更难被发现

### 2. 不要把问题简单归结为 “session 关闭逻辑错了”

关闭逻辑之前确实有 bug，但那只是误伤放大器。

即使已经改成“不因 timeout 直接关 session”，也仍然说明：

- `GET /chromium/{id}` 这条查询链在压力下不够稳定

### 3. 不要先把精力放在调大 controller 并发上

`BROWSERINSTANCE_MAX_CONCURRENT_RECONCILES` 现在默认已经是 30。

这类并发参数影响的是控制器处理 BrowserInstance 的能力，不直接等于：

- REST 查询延迟就会下降

如果查询链本身仍然是“每次直连 apiserver”，单纯调 controller 并发并不能从根上解决问题。

## 优化优先级

## 第一优先级：补 GET 观测

先补指标，不先盲改。

最低建议补：

- `k8s_chrome_daemon_get_request_duration_ms`
- `k8s_chrome_daemon_get_request_inflight`
- `k8s_chrome_daemon_get_request_result_total`

`result` 维度建议至少包括：

- `success`
- `not_found`
- `get_error`
- `encode_error`
- `context_done`

最好再拆两段耗时：

- `Client.Get` 耗时
- handler 总耗时

这样才能知道瓶颈是在：

- 下游 apiserver
- 还是 daemon 自身

## 第二优先级：把查询路径从“直连 apiserver”改成“优先本地读”

目标不是立刻做复杂架构，而是先收口一个核心原则：

- `GET /chromium/{id}` 不应该每次都打实时 apiserver 读取

可选路径有两类：

### 路线 A：优先复用 controller-runtime cache

先评估是否能让 REST API 查询直接复用 manager 侧的 cache client / reader。

优点：

- 改动相对小
- 语义仍然清晰

风险：

- 需要明确 cache 一致性边界
- 需要确认当前 server 生命周期和 manager 的耦合方式

### 路线 B：引入单独只读状态视图

如果希望查询面更可控，可以单独维护一个只读状态层，例如：

- 内存索引
- 轻量只读缓存

优点：

- 查询面和控制面可以更强隔离

代价：

- 设计和维护复杂度更高

第一阶段更建议先做路线 A 的可行性验证。

## 第三优先级：拆查询面和控制面

中期建议评估把：

- BrowserInstance controller
- REST 查询 API

从同一个单副本进程里拆开。

合理方向是：

1. controller 继续保留 leader election
2. query API 独立 deployment
3. query API 允许多副本横向扩展

这样可以把查询面从：

- 单点控制器附属能力

变成：

- 可独立扩展的只读查询服务

这是解决高并发查询瓶颈最稳的方向，但优先级应低于“补观测”和“先去掉直连 apiserver”。

## 第四优先级：优化 reconciler 扫描模型

`browser-manager-reconciler` 当前的全量扫描模型需要优化，但不建议作为第一刀。

可选方向包括：

### 1. 分批扫描

不要在同一时间点把全部 active sessions 都扫完。

### 2. 加并发上限

当前是串行扫，这虽然避免了瞬时更大的洪峰，但也会拉长整轮扫描时间。

后续可以考虑：

- 有上限的并发扫描
- 避免既全串行也无限并发

### 3. 按最近活跃度或风险分层扫描

不是所有 active sessions 都需要同样频率地确认状态。

### 4. 事件驱动 + 周期补偿

把“状态变化尽量靠事件感知”和“全量兜底扫描”结合起来，而不是只靠固定频率全量轮询。

## 建议的实施顺序

如果要把这件事拆成真正可执行的路线，建议顺序如下：

1. 给 `GET /chromium/{id}` 补指标
2. 在 office 观察一轮真实延迟分布
3. 验证 REST 查询改走 cache 的可行性
4. 再决定是否拆 query service
5. 最后优化 reconciler 扫描模型

原因是：

- 没指标时，后面每一步都缺少验证闭环
- 查询路径不改，单纯调副本或调 timeout 很容易只是掩盖问题

## 一句话结论

这次暴露出来的问题，本质不是：

- `browser-manager` timeout 设成了 10 秒

而是：

- `k8s-chrome-daemon` 当前把浏览器实例查询做成了“单副本进程上的实时 apiserver 代理”

在 `browser-manager-reconciler` 周期性全量扫描的流量模式下，这条链路会天然暴露出：

- 单点承载
- 实时读 apiserver
- 缺少 GET 观测

这三类瓶颈。
