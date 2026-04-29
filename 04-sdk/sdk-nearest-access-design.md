# SDK 就近接入整体设计

## 1. 设计目标

这份设计要解决的问题不是“让 SDK 猜到底层节点在哪”，而是：

- 让 SDK 在多个正式可用的区域入口之间自动选择更合适的接入点
- 让用户可以显式指定入口或区域，不被自动逻辑覆盖
- 让自动选择结果可缓存、可回退、可排障
- 让服务端和 SDK 对“哪些入口可以被正式消费”使用同一套边界定义

## 2. 设计边界

### 2.1 SDK 选择的对象

SDK 选择的是：

- `region endpoint`

SDK 不选择的是：

- K8s zone
- Pod
- 节点 IP
- 具体浏览器实例
- 云厂商内部资源名

换句话说，SDK 处理的是“区域入口选择”，不是“基础设施调度”。

### 2.2 正式默认可见范围

正式 SDK 默认只消费：

- `scope=public`

不进入正式默认消费路径的包括：

- `scope=internal`
- `scope=debug`
- `office`

这里的核心规则已经收口：

- `office` 不进入默认 public catalog
- `office` 不参与正式自动选路
- `office` 不混进正式 fallback 路径

如果未来需要让内部调试用户访问 `office`，应该通过额外开关或单独入口控制，而不是把它混进正式默认流程。

## 3. 统一模型

### 3.1 region 的语义

`region` 不是底层机房信息，也不是自然语言地理文本，而是：

- 面向 SDK 用户暴露的“逻辑区域 ID”

它的职责是：

- 让用户表达“只想接这个区域”
- 让 SDK 在多个正式区域入口之间选择

它不应该直接暴露为这类名字：

- `qcloud`
- `office`
- `qcloud-hk`

原因是这些名字表达的是：

- 云厂商
- 历史环境
- 内部部署口径

而不是用户能直观看懂的地域。

所以对外更合理的方向应是地域型稳定 ID，例如：

- `cn-south`
- `cn-east`
- `hk`

或者：

- `china-south`
- `china-east`
- `hongkong`

结论是：

- 对外 `region` 应该是地域型稳定 ID
- 内部 `qcloud / qcloud-hk / office` 应留在服务端映射层，不直接暴露成正式 SDK 参数

### 3.2 region vs zone

这里需要明确区分两层概念：

- `region`
- `zone`

`region` 是：

- SDK 可感知、可配置、可切换的逻辑接入区域

`zone` 是：

- 某个 region 内部的基础设施可用区划分

两者职责不同：

- SDK 只理解 `region`
- 服务端基础设施可以在 region 内部继续用 zone / topology 做高可用

这条边界要写死，原因是：

1. SDK 做的是入口与控制面 bundle 选择，不是底层节点调度
2. K8s 原生 zone / topology 机制只能处理 region 内高可用，不能自动完成 region 级控制面隔离
3. 如果把 zone 语义暴露给 SDK，用户会把基础设施细节误当成产品能力接口

结论：

- 对外只暴露 `region`
- zone 只留在服务端实现层

### 3.3 endpoint 的语义

第一期里，`endpoint` 表示某个 region 对应的唯一正式入口。

也就是说当前前提是：

- 一个 `region` 对应一个 `public endpoint`
- 不存在同 region 多个 public endpoints 让 SDK 再次择优

因此第一期不需要把 `endpoint` 抽成独立选择层。

服务端内部仍然可以保留 endpoint 概念，但对 SDK 而言当前可简化成：

- `region_id`
- `base_url`
- `wss_prefix`
- `scope`
- `status`
- `priority`

结论：

- SDK 第一阶段最终选中的就是 `region`
- `region` 当前一一映射到对应的正式入口与控制面

### 3.4 catalog 的语义

`catalog` 是服务端维护的一份官方入口清单，用来告诉 SDK：

- 当前有哪些 public regions 可以选
- 每个 region 对应哪个正式入口
- 哪些入口已经停用、draining，或者不该接新流量

第一阶段建议由服务端维护一份官方 catalog，SDK 只消费，不拥有真值来源。

## 4. 服务端最小配合面

### 4.1 public endpoint catalog

这是第一优先级最高的前置项。

第一版 catalog 至少需要包含这些字段：

- `region_id`
- `display_name`
- `base_url`
- `wss_prefix`
- `scope`
- `status`
- `priority`

如果允许再加两个字段，建议补：

- `public_api`
- `tags`
- `probe_path`

### 4.2 probe / health 端点

SDK 不应该用业务接口测速，服务端需要提供统一 probe 入口。

probe 最低要求：

- 返回 `200 OK`
- 返回 JSON

最低字段建议：

- `status`
- `region_id`
- `scope`
- `timestamp`

如果再补两项会更好：

- `public_api`
- `accept_new_sessions`

这样 SDK 可以判断：

- 入口是否真的属于正式 public 消费面
- 当前是否还能接新流量

### 4.3 scope 边界

服务端需要明确收口：

- `public`
- `internal`
- `debug`

这不是注释层面的说明，而是要真的落到 endpoint 元数据上。

否则会出现两个问题：

1. SDK 只能自己猜哪些入口能用
2. `office` 容易被混进正式默认自动选路

这里建议再补一条发布规则：

- 只有 `scope=public` 的 endpoint 才允许进入对 SDK 开放的 catalog

也就是说：

- `internal` / `debug` 可以存在于服务端治理数据里
- 但不应该出现在正式 SDK 默认拉取的 public catalog 里
- `office` 如果要保留，也应通过独立开关或独立 catalog 暴露，而不是混进默认 public 视图

## 5. SDK 配置语义

### 5.1 参数集合

第一版建议统一支持：

- `selection_mode`
- `region`
- `base_url`
- `allowed_regions`
- `forbid_regions`
- `cache_ttl`
- `disable_probe`
- `force_refresh`

### 5.2 参数优先级

这条已经收口成固定规则：

- `base_url > manual region > auto`

具体解释：

1. `base_url`
- 最高优先级
- 一旦设置，直接跳过 catalog / cache / probe / select

2. `selection_mode=manual + region`
- 只在指定 region 内选
- 第一版不做跨 region 自动切换

3. `auto`
- 才允许使用缓存、probe、测速和 fallback

### 5.3 各参数的建议语义

#### `selection_mode`

- 含义：手动还是自动
- 建议取值：`auto` / `manual`
- 默认：`auto`

#### `region`

- 含义：指定逻辑区域 ID
- 默认：未设置
- 行为：直接限制 SDK 只选择该 region 对应的正式入口

#### `base_url`

- 含义：指定具体控制面入口
- 默认：未设置
- 行为：直接固定入口，跳过自动逻辑

#### `allowed_regions`

- 含义：自动模式下允许选择的 region 白名单
- 默认：未设置，表示允许全部 public regions

#### `forbid_regions`

- 含义：自动模式下禁止选择的 region 黑名单
- 默认：未设置

#### `cache_ttl`

- 含义：自动选路结果缓存有效期
- 默认建议：6 小时

#### `disable_probe`

- 含义：禁用主动探测，只依赖缓存或静态优先级
- 默认：`false`

#### `force_refresh`

- 含义：忽略缓存，强制重测

## 6. 多 Region 控制面流量隔离

这件事里还有一个容易被漏掉的关键点：

- SDK 选中的不只是“入口域名”
- 还隐含决定了后续控制面请求会进入哪个 region 的 `kong -> browser-manager -> k8s-chrome-daemon` 链路

如果多 region 只是“入口多了几个域名”，但云端控制面没有做严格隔离，就会出现：

- SDK 进了 A region 的 Kong
- 请求又被转发到 B region 的 `browser-manager`
- `browser-manager` 再去调 B 或 C region 的 `k8s-chrome-daemon`

这样会导致 region 语义被打穿。

### 6.1 需要隔离的不是单点，而是整条控制面链路

多 region 下真正需要隔离的是：

- region ingress: `kong`
- region control plane: `browser-manager`
- region orchestrator: `k8s-chrome-daemon`

也就是说，region 不只是 SDK 入口选择标签，而应该对应一整套：

- 入口
- 控制面
- 浏览器实例编排面

### 6.2 核心原则：入口选中哪个 region，后续控制面就锁定在哪个 region

正式设计里应明确一条规则：

- 一旦 SDK 选中了某个 `region`，后续控制面链路必须 region sticky

至少包括：

1. session 创建请求进入该 region 的 `kong`
2. `kong` 只把流量转给该 region 的 `browser-manager`
3. 该 region 的 `browser-manager` 只调用本 region 的 `k8s-chrome-daemon`
4. 该 session 的后续查询、关闭、状态同步，也都应继续回到同一 region

否则会出现几类问题：

1. session metadata 在 A region，真实实例在 B region
2. 一边创建在 A，一边 reconcile/close 在 B
3. region 级容量治理和故障隔离失效
4. 跨 region 调用把延迟和故障面一起放大

### 6.3 建议的 region 绑定对象

如果要让这条链路稳定工作，建议至少把以下对象显式绑定 region：

- session
- browser-manager deployment
- `k8s-chrome-daemon` API

第一版可以先做到：

1. catalog 里明确 `region_id -> base_url` 的唯一映射
2. session 创建成功后，把 `region_id` 写入 session 元数据
3. `browser-manager` 内部所有对 `k8s-chrome-daemon` 的调用，只走本 region 配置的 `CONTAINER_SERVICE_URL`

这样至少可以保证：

- 一个 session 从创建开始就有明确 region 归属

session 元数据第一版建议至少包含：

- `session_id`
- `region_id`
- `selected_by`

其中：

- `region_id` 用来表达 session 归属的逻辑区域
- `selected_by` 可取 `manual` / `auto` / `fallback`

当前设计下，`region_id` 就隐含表达了它绑定的是哪套 region 控制面。

这样后续在查询、关闭、reconcile、排障时，服务端和 SDK 都可以直接按 `region_id` 把 session 路由回同一套控制面。

### 6.4 服务端执行链路需要补齐到 node

目前文档里如果只写到 `kong -> browser-manager -> k8s-chrome-daemon` 还不够，因为真正被创建出来的浏览器实例最终会落到具体 node。

所以服务端实际需要锁定的是这条完整链路：

1. SDK 选定 `region`
2. 请求进入该 `region` 的 `kong`
3. `kong` 只转发给该 `region` 的 `browser-manager`
4. `browser-manager` 只调用该 `region` 的 `k8s-chrome-daemon`
5. `k8s-chrome-daemon` 只在该 `region` 的 node 池内创建浏览器实例

这里更准确的实现前提应写成：

- 所有 node 都需要带 `region` 标签
- 该 `region` 的控制面服务只应部署或只应路由到带同样 `region` 标签的 node
- 浏览器实例也只能创建在带同样 `region` 标签的 node 上

这条规则的关键不是“最终是哪一台 node”，而是：

- node 选择必须发生在该 region 内部
- 不能在 region 已经选定后，再把实例调度到别的 region 的 node

也就是说：

- SDK 决定 region
- 服务端在该 region 内完成实例落点选择
- node 是 region 内部调度结果，不是跨 region 再次选路

### 6.5 node 归属约束

服务端需要明确一条约束：

- 一个 session 一旦绑定 `region_id`，它创建出来的浏览器实例也必须属于同一个 `region`

这里建议在实现层至少满足下面三条：

1. `browser-manager` 只持有本 region 的 daemon 配置
2. `k8s-chrome-daemon` 只面向本 region 的 node 池做调度
3. session 元数据和实例元数据都能反查出它们属于哪个 `region`

如果再写得更落地一点，可以直接补成四条：

1. 所有 node 必须带 `region=<region_id>` 标签
2. `kong` / `browser-manager` / `k8s-chrome-daemon` 只处理本 `region` 的流量
3. 浏览器实例只允许调度到 `region=<region_id>` 一致的 node
4. session 所属 `region` 与实例所在 node 的 `region` 必须一致

如果这三条不成立，就会出现表面上 session 在 A region，但实例实际跑在 B region node 上的问题。

### 6.6 region 内部调度与 region 级选路的边界

这里也要明确两层调度边界：

- region 级选路：由 SDK 和 catalog 决定
- region 内实例调度：由服务端在该 region 内完成

region 内部可以继续基于：

- node 余量
- zone
- topology
- 污点/容忍
- 机型差异

做调度，但这些都只能在已选定 region 的边界内发生。

换句话说：

- K8s 可以帮我们决定“region A 里落哪台 node”
- 但不能替我们决定“是不是改落到 region B”

### 6.7 `browser-manager` 不应跨 region 调度浏览器实例

这里建议明确禁止一类设计：

- 某个 region 的 `browser-manager` 根据负载情况，把实例调度到别的 region 的 `k8s-chrome-daemon`

原因很直接：

1. 会打穿 region 语义
2. 会让 session 和实例的位置脱钩
3. 会让调试、清理、reconcile、发布和容量分析都变复杂

如果未来真的需要“跨 region 借容量”，那应该作为单独的调度体系设计，而不是悄悄混进当前主路径。

第一版更合理的口径是：

- region 内自治
- region 间切换只发生在 SDK 入口选择或显式 fallback 时
- 不在单个 session 生命周期中做跨 region 漂移

### 6.8 fallback 不能只切入口，还要切整条控制面

如果 primary region 不可用，fallback 的真实语义不应该是：

- 换一个外层域名继续打

而应该是：

- 换到另一套完整 region 控制面

也就是说 fallback 的粒度是：

- `region`

而不是：

- 单个公网入口地址

这里的 `region` 隐含包含：

- Kong endpoint
- browser-manager endpoint
- `k8s-chrome-daemon` dependency chain

所以 fallback 的执行语义应明确为：

1. 放弃当前 primary region
2. 重新选择下一个候选 public region
3. 后续请求整体切到该 region 对应的完整控制面

而不是：

1. 沿用原有 session 归属
2. 只替换外层入口域名

否则会出现入口在 B、session 仍指向 A、控制面又回到 A 的不一致状态。

### 6.9 服务端 catalog 里需要显式表达 region 隔离边界

如果服务端只返回：

- `base_url`

还不够。

至少还需要让 catalog 能表达：

- 这个入口属于哪个 `region_id`
- 这个 `region_id` 对应哪套正式 public control plane

否则 SDK 选路只能知道“打哪个 URL”，却不知道这个 URL 背后的控制面是否真的是独立 region。

第一版即使不把内部拓扑直接暴露给 SDK，也建议在服务端治理层明确维护：

- `region_id -> control plane` 的一一映射

### 6.10 故障隔离目标

多 region 设计最终要达到的不是“名字上有 region”，而是下面几条真的成立：

1. region A 的 `browser-manager` 故障，不应拖垮 region B 的 session 创建
2. region A 的 `k8s-chrome-daemon` 延迟抖动，不应影响 region B 的编排路径
3. region A 的 reconcile 洪峰，不应传导到 region B 的控制面
4. region A 发布或回滚，不应影响 region B 的稳定性

只有这几条成立，SDK 层的 region 选择才是真的有意义。

## 7. 自动选路主流程

主流程固定为：

- `catalog -> filter -> probe -> select`

### 7.1 catalog

SDK 拉取官方 public endpoint catalog。

要求：

- 默认只消费 `scope=public`
- 不把 `office`、internal、debug 混进候选集

### 7.2 filter

依次应用：

- `allowed_regions`
- `forbid_regions`
- 显式 `region`

目标是把候选集收窄到：

- 合法的 public regions

### 7.3 probe

第一版不建议做复杂网络推断，只做：

- 最小 HTTP 健康探测
- 小样本 RTT 测试

如果后续服务端支持 ws probe，再补：

- 轻量 ws 握手探测

评分建议先简单：

- 先看可用性
- 再看 RTT
- 再看 catalog priority

### 7.4 select

从 probe 通过的 public regions 中选出：

- primary region

同时记录：

- 次优 regions 列表

供后续 fallback 使用。

## 8. 缓存与 fallback

### 8.1 缓存内容

最少记录：

- `region_id`
- `base_url`
- `selected_at`
- `ttl`

建议再补：

- `candidate_results`
- `last_probe_result`
- `network_fingerprint`

### 8.2 缓存层次

建议两层：

1. 进程内缓存
2. 本地持久缓存

比如可以写到用户目录下的 cache 文件。

### 8.3 何时使用缓存

只在以下条件同时满足时直接复用：

- `selection_mode=auto`
- 未设置 `force_refresh`
- 缓存未过期
- region 过滤条件一致
- 网络指纹未明显变化

### 8.4 fallback 规则

primary 失败后，正式默认路径只有两种动作：

1. 切到次优 public region
2. 触发重测

不能做的事情：

- 把 `office`
- `internal`
- `debug`

混进正式 fallback 路径。

这里要强调：

- fallback 切换的是 region，不是只换 `base_url`
- 一旦切换成功，新 session 应写入新的 `region_id`
- 已经创建成功的 session 不建议在生命周期中跨 region 漂移

### 8.5 失败判定

HTTP 失败示例：

- DNS 失败
- TLS 失败
- 连接失败
- 超时
- 5xx

WS 失败示例：

- 握手失败
- 认证失败
- 建连后立即断开

建议 HTTP 和 WS 失败分开记分，不要混成一个桶。

## 9. 跨 Region 共享数据与统一管理面

如果后续从南京扩到北京，不能只讨论 SDK 选路，还要把“全局管理数据”和“区域运行态数据”拆开。

这里建议采用一条核心原则：

- 南京保留统一管理面
- 北京只承载本 region 的运行面
- 两地不共享一套在线 PG

换句话说：

- `lexhome` 可以先只部署在南京
- 北京集群有自己独立的 Kong / browser-manager / k8s-chrome-daemon / PG
- 南京 `lexhome` 通过 session gateway 动态查询区域 API 看到北京 session，而不是直接读北京本地业务库

### 9.1 数据类型拆分

这件事里至少有三类数据，不能混成一套同步模型。

#### 全局身份与授权数据

包括：

- user
- project
- api_key
- project 与 api_key 的绑定关系
- project 的 region 权限

这类数据应该有一个全局真值来源。

第一版建议：

- 南京作为全局管理面主站
- `project_id / api_key` 在南京创建和维护
- 同一对 `project_id / api_key` 在所有被授权的 public regions 内都应该可用
- 每个 public region 只消费同步后的授权快照

这里的产品语义要明确：

- 用户只维护一套 `project_id / api_key`
- SDK 通过 `region` 决定在哪个地域创建实例
- 不同 region 不能各自生成一套割裂的 project / key
- 但 context / download / extension 不属于全局授权数据，不能跟着 project / key 跨 region 共享

#### 区域运行态数据

包括：

- session
- browser instance
- pod / node / region 归属
- session 状态
- session close / reconnect / heartbeat
- context
- download
- extension

这类数据应该由创建它的 region 负责。

如果 SDK 选择了北京：

- session 创建发生在北京 Kong
- browser-manager 在北京
- k8s-chrome-daemon 在北京
- 浏览器实例在北京 K8s 集群内
- 这个 session 的权威运行态也属于北京

这里需要明确一条边界：

- context / download / extension 不能跨 region 共享
- 北京创建的 context 只属于北京
- 北京 session 产生的 download 只属于北京
- 北京上传的 extension 只属于北京
- 南京同名 context / extension 和北京同名 context / extension 也应视为不同 region 资源

原因是这些数据通常依赖 region 内的对象存储、文件系统、browser-manager 元数据或实例运行态。

如果把它们跨 region 共享，会引入：

- 大文件跨地域复制
- 生命周期和清理策略不一致
- session 引用到不存在或未同步完成的资源
- 用户以为同一个 `context_id / extension_id` 在所有 region 都可用

#### 区域 Session 数据面

南京 `lexhome` 需要看到北京 session，但不需要把北京 session 具体数据同步到南京。

当前网站层已经通过 `BROWSER_BASE_URL` 直接访问 browser-manager 的 sessions 接口，例如：

- `POST /instance/v2/sessions`
- `POST /instance/stop`
- `DELETE /instance`

为了降低 `lexhome` 改造成本，建议在 `lexhome` 和各 region browser-manager 之间加一层 session gateway。

session gateway 对 `lexhome` 继续暴露相同的 sessions 接口，但它的数据源不再是单个 browser-manager，而是多个 region 的 browser-manager。

也就是说：

- `lexhome` 仍然调用一个 `BROWSER_BASE_URL`
- `BROWSER_BASE_URL` 从单 region browser-manager 切到南京 session gateway
- session gateway 并发请求南京 browser-manager 和北京 browser-manager
- session gateway 在服务端按 `region_id` 拼接、排序、分页或分组
- session gateway 返回给 `lexhome` 的结构尽量保持和现有 `/instance/v2/sessions` 一致

每个 region browser-manager 至少需要提供：

- session list
- session detail
- close session
- inspect / reconnect
- region health

这样 session 真值始终留在创建它的 region，南京只做统一网站入口和 session gateway 请求聚合，不保存北京 session 明细副本。

### 9.2 Kong key 数据不能直接依赖跨 Region PG

Kong 依赖 PG 存 consumer / credential / plugin / route 等数据。

北京不能直接依赖南京 PG，原因是：

1. 跨地域数据库延迟和故障面不可控
2. 北京 region 故障隔离会被南京 PG 打穿
3. Kong 的运行面不应该因为跨地域链路抖动而无法认证请求

所以更合理的模型是：

- 每个 region 的 Kong 都使用本 region 本地 PG
- Kong 本地 PG 里的 key 数据不是全局真值
- Kong 本地 PG 只保存同一套全局 project / api_key 数据同步后的 region 副本
- Kong 的 route / service / plugin 配置由各 region 本地维护，不参与跨 region key 同步

同步方式第一版建议不要做 PG 级双向复制。

更推荐：

- 南京管理面维护 project / api_key 真值
- 通过同步任务或发布管道把同一个 project 对应的 consumer / key-auth credential 写入各 region Kong Admin API
- region Kong 只负责本地认证和转发
- route / service / plugin 仍然从各 region 自己的 Kong PG 读取

也就是说：

- 同步的是“key 相关授权对象”
- 不是让多个 Kong 共用或互相复制一套 PG
- 用户看到的是同一对 `project_id / api_key`，不是 region 私有凭证
- 路由信息不跨 region 同步，避免把北京和南京的入口拓扑绑死在一起

#### 当前 `lexhome` 与 Kong 在 key 上的交互

当前 `lexhome` 不自己维护 API key 真值，而是作为 Kong Admin API 的管理 UI。

现有调用链是：

1. `lexhome` 服务端通过 `KONG_ADMIN_BASE_URL` 连接 Kong Admin API
2. 用户进入 API key 页面时，`lexhome` 读取登录态里的 `user.id`
3. `lexhome` 调用 `PUT /consumers/{user.id}`，请求体为 `{ username: user.id }`
4. Kong 返回 consumer 对象，`lexhome` 把返回的 `consumer.id` 展示为 `project_id`
5. `lexhome` 用 `GET /consumers/{consumer.id}/key-auth` 列出该 consumer 的 key
6. 创建 key 时，`lexhome` 调用 `POST /consumers/{consumer.id}/key-auth`，由 Kong 生成 `api_key`
7. 删除 key 时，`lexhome` 调用 `DELETE /consumers/{user.id}/key-auth/{keyAuthId}`

运行时请求的校验链路是：

1. SDK / 用户请求携带 `api_key` 和 `project_id`
2. Kong `key-auth` 插件先用 `api_key` 解析出 consumer
3. `projectid-consumer-check` 插件再读取请求里的 `project_id`
4. 插件默认把 `project_id` 和 `consumer.id` 对比
5. 因此当前语义下，`project_id` 实际上就是 Kong consumer id

这个现状带来的多 region 结论是：

- 当前 key 真值在 Kong PG 里，而不在 `lexhome` 自有业务库里
- 南京创建出来的 consumer / key-auth credential 只存在南京 Kong PG
- 北京如果使用独立 Kong PG，就必须额外写入同一份 consumer / key-auth credential
- 多 region 第一版应把 key 管理从“直接操作单个 Kong”抽出来，变成南京管理面产生全局 key 变更，再同步到各 region Kong Admin API
- 各 region Kong 只保存 key 相关授权副本，route / service / plugin 仍读取本 region 本地 PG

这里的“同一份 key”指的是 key-auth credential 的 `key` 字段必须一致，不要求 Kong credential 自己的 `id` 一致。

例如南京创建出来的是：

- `project_id = consumer.id`
- `api_key = abc123`
- 南京 Kong key-auth credential id = `cred-nanjing-001`

同步到北京时，目标语义应该是：

- 北京 Kong 里能用同一个 `project_id`
- 北京 Kong 里能用同一个 `api_key = abc123`
- 北京 Kong 自己生成或保存的 credential id 可以是 `cred-beijing-999`

不能在北京再调用一次空 body 的 `POST /consumers/{consumer.id}/key-auth` 让 Kong 自动生成新 key。

否则北京会得到另一个 `api_key`，用户手里原来的南京 `api_key` 就不能在北京使用，和“一套 project_id / api_key 跨 region 可用”的目标冲突。

所以同步器写北京 Kong 时，必须确认当前 Kong Admin API 支持写入指定 `key` 值；如果当前版本或配置不支持，就不能用 Kong 自动生成 key 的方式做跨 region 同步，需要改成由管理面生成 key，再分别写入各 region。

还有一个需要顺手收口的代码边界：

- 当前 `listApiKeys` / `createApiKey` 接收客户端传入的 `consumerId`，服务端只校验登录态，没有在 action 内强制校验该 `consumerId` 归属当前用户
- 多 region 改造时应取消对客户端 `consumerId` 的信任，由服务端根据登录用户查全局 project，再决定允许操作哪些 key 和 region

### 9.3 `project_id / api_key` 的推荐流向

推荐流向如下：

1. 用户在南京 `lexhome` 创建 project / api_key
2. 南京管理面写入全局项目库
3. 同步器把同一个 project 和同一个 api_key 对应的 Kong consumer / key-auth credential 分发到南京 Kong 和北京 Kong
4. SDK 选择北京 region 后，使用同一组 `project_id / api_key` 访问北京 Kong
5. 北京 Kong 用本地 PG 中的 key 副本完成认证，并用北京本地 PG 中的 route / service / plugin 完成路由
6. 北京 browser-manager 创建 session，并把 session 数据保留在北京 region
7. 南京 session gateway 按 region 动态查询南京和北京的数据面，再把结果返回给 `lexhome`

这里的关键点是：

- 用户只维护一套 `project_id / api_key`
- 同一对 `project_id / api_key` 可以在不同 region 创建实例
- 每个 region 都有本地可用的授权副本
- region 内认证不依赖跨地域在线查询
- region 内路由也不依赖南京 Kong 的配置

### 9.4 授权同步器的状态模型

为了让“同一对凭证跨 region 可用”可运维，授权同步器需要有明确状态。

建议至少维护：

- `project_id`
- `api_key_id`
- `target_region`
- `sync_status`
- `last_sync_at`
- `last_error`
- `credential_version`

`sync_status` 可以先收敛成：

- `pending`
- `synced`
- `failed`
- `revoking`
- `revoked`

创建或轮换 api_key 时，建议流程是：

1. 南京写入全局 project / api_key 真值
2. 为每个目标 region 创建同步任务
3. 同步器调用各 region Kong Admin API 写入 consumer / key-auth credential
4. 每个 region 成功后标记 `synced`
5. 只有目标 region 全部 `synced` 后，南京 `lexhome` 才显示该 key 在这些 region 可用

这样用户仍然只看到一对 `project_id / api_key`，但后台能清楚知道它在每个 region 的分发状态。

如果某个 region 同步失败：

- 不应该影响已同步 region 的使用
- `lexhome` 应展示该 region 暂不可用或同步失败
- SDK 自动选路时不应把该 project 选到尚未同步成功的 region

所以 public catalog 只告诉 SDK 哪些 region 存在还不够，最终还需要结合 project 的 region 授权状态。

### 9.5 session 数据如何被南京 `lexhome` 展示

南京 `lexhome` 想看到北京 session，不应该把北京 session 明细同步到南京，也不应该直接读北京 browser-manager 的内部库或本地 PG。

建议引入南京 session gateway。

它对上保持现有 browser-manager sessions 接口兼容，对下访问各 region browser-manager。

对上接口：

- `POST /instance/v2/sessions`
- `POST /instance/stop`
- `DELETE /instance`

对下数据源：

- 南京 browser-manager
- 北京 browser-manager
- 后续其他 public region browser-manager

北京 region 至少需要提供给 session gateway：

- `list sessions`
- `get session`
- `close session`
- `inspect session`
- `region health`

南京 `lexhome` 的展示逻辑保持简单：

1. 用户打开 session 页面
2. `lexhome` 继续请求 `BROWSER_BASE_URL /instance/v2/sessions`
3. 南京 session gateway 根据用户 project 权限确定可查询的 regions
4. session gateway 分别请求南京和北京 browser-manager
5. 各 region 返回自己本地的 session 列表
6. session gateway 按 `region_id` 拼接结果并返回给 `lexhome`

这个模型里：

- session 明细不从北京同步到南京
- 南京不维护全局 session index
- SDK / API 层按 `region` 字段直接访问对应 region 的运行面
- `lexhome` 网站层通过 session gateway 获得多 region 聚合结果，尽量不感知各 region browser-manager 细节

为了避免页面被慢 region 拖死，session gateway 查询各 region 时应做并发请求和超时控制。

如果北京 region 查询失败：

- 南京 session 仍然正常展示
- 北京区域展示为暂不可用或查询失败
- 不影响用户操作其他 region 的 session

### 9.6 南京如何访问北京数据面

南京访问北京 browser-manager 不能直接走公网裸奔，也不能直接访问北京 PG。

推荐链路是：

```text
lexhome
  -> nanjing session gateway
  -> private network / VPN / 专线 / 云联网
  -> beijing region data-plane gateway
  -> beijing browser-manager
```

这里建议北京侧不要直接把 browser-manager 暴露给南京，而是在北京集群内放一个 region data-plane gateway。

原因：

- browser-manager 仍保持 region 内服务
- 对外只暴露受控的 session 查询 / 管理 API
- 鉴权、限流、审计、字段裁剪都放在 gateway 层
- 后续北京内部 browser-manager 结构变化时，不影响南京 session gateway

南京到北京至少需要三层约束：

1. 网络连通：VPC 对等连接、VPN、专线或云联网，优先私网链路
2. 服务认证：mTLS 或内部 service token，不能只靠 IP 白名单
3. 权限校验：请求必须带 `project_id / region_id / session_id`，北京侧确认 session 属于该 project 和本 region

第一版可以先简化成：

- 北京暴露一个内网 HTTPS endpoint
- 南京 session gateway 配置北京 endpoint
- 请求带内部 service token
- 北京 region data-plane gateway 校验 token 和 project/session 归属
- 所有请求记录审计日志，包括 `project_id / region_id / operation / status`

### 9.7 Region 数据面 API 边界

北京提供给南京 `lexhome` 的不是数据库访问权限，而是一组稳定 API。

推荐边界：

- 只暴露查询和管理 session 所需字段
- 不暴露 region 内部 PG 表结构
- 不让南京直接访问北京 K8s API
- 不让外部用户直接访问 browser-manager 管理接口
- region API 必须带服务间认证
- 请求里必须带 `project_id` 和 `region_id`

region API 返回的 session 数据至少包含：

- `session_id`
- `project_id`
- `region_id`
- `status`
- `created_at`
- `updated_at`
- `browser_type`
- `inspect_url`
- 必要的连接信息或可反查 token

南京 session gateway 需要完成用户权限校验，然后用内部 service credential 调 region API。

北京 region 收到南京调用后，还需要校验：

- service credential 是否可信
- `project_id` 是否允许查询
- 请求目标是否属于本 region
- 操作的 `session_id` 是否属于该 `project_id`

### 9.8 管理操作如何路由到对应 Region

展示只是第一步，用户还会在南京 `lexhome` 上做管理操作，例如：

- close session
- reconnect
- inspect
- view logs

这些操作不能在南京本地处理掉，因为 session 真正归属北京。

所以页面和接口都必须携带 `region_id`。

当用户在南京 `lexhome` 关闭北京 session 时：

1. 页面上的 session 已带 `region_id=beijing`
2. 用户点击关闭
3. 南京 `lexhome` 调用南京 session gateway 的兼容 stop 接口
4. session gateway 根据 `region_id=beijing` 调用北京 region data-plane gateway
5. 北京 data-plane gateway 调用北京 browser-manager 关闭实例
6. 南京 `lexhome` 重新查询 sessions 接口并刷新页面展示

这里南京做的是“管理面代理”，不是直接操作北京 K8s。

这里还需要一条权限规则：

- 南京管理面对 owner region 的管理调用，必须携带服务间认证

不能让外部用户直接拿 `api_key` 调北京 browser-manager 管理接口。

推荐做法：

- 用户请求只到南京 `lexhome` / 管理 API
- 南京完成用户权限校验
- 南京 session gateway 用内部 service credential 调 owner region 管理 API
- owner region 再校验该 service credential 和 `project_id / session_id / region_id` 是否匹配

这样外部 API key 仍只用于用户侧 SDK 创建 session，不扩大成跨 region 管理凭证。

### 9.9 Context / Download / Extension 的 Region 边界

context / download / extension 不进入全局同步，也不应由南京统一保存北京副本。

它们的归属规则是：

- context 属于创建它的 region
- extension 属于上传它的 region
- download 属于产生它的 session 所在 region
- session 创建时引用的 `context_id / extension_id` 必须和目标 `region_id` 一致

如果 SDK 选择北京 region：

- 只能使用北京已有的 context
- 只能使用北京已上传的 extension
- 下载产物保留在北京
- 南京 `lexhome` 如需展示或下载，只能通过北京 region data-plane gateway 代理读取

如果用户希望同一个 extension 在南京和北京都可用，第一版应要求用户分别上传到两个 region，或者后续再提供显式的“复制到其他 region”动作。

这里不建议做隐式复制，原因是：

- 上传文件可能很大
- 复制状态会影响 session 创建成功率
- 失败补偿和清理成本高
- 用户很难判断某个 `extension_id / context_id` 到底在哪些 region 可用

`lexhome` 展示这些资源时，也应带上 `region_id`。

推荐 UI / API 语义：

- context 列表按 region 查询和展示
- extension 列表按 region 查询和展示
- download 列表按 session 的 region 查询
- 创建 session 时，只允许选择同 region 的 context / extension
- 如果用户切换 region，context / extension 选择项也随 region 切换

### 9.10 一致性模型

这套设计不应该追求强一致全局数据库。

建议明确成：

- 授权数据：最终一致，但创建 / 轮换 api_key 时需要等待各 region 同步完成后再对外声明可用
- session 展示：南京 session gateway 实时查询各 region 数据面，不做 session 明细同步
- session 操作：按 `region_id` 路由到 owner region，操作结果以 owner region 返回为准
- context / download / extension：region 内可见，不做跨 region 共享
- region 运行态：region 内强一致，跨 region 不做强一致

如果北京到南京的链路短暂不可用：

- 北京 region 已同步过的 api_key 仍可继续创建 session
- 北京本地 session 运行不应受影响
- 南京 `lexhome` 暂时查询不到北京 session
- 南京 `lexhome` 应把北京 region 标记为查询失败或暂不可用
- 链路恢复后，南京 `lexhome` 下一次查询即可看到北京当前状态

### 9.11 推荐的第一版架构

第一版推荐这样做：

1. 南京保留唯一 `lexhome`
2. 南京维护全局 project / api_key 真值
3. 南京到各 region 只同步 Kong key 相关数据
4. 每个 region 保留本地 Kong PG
5. 每个 region 的 route / service / plugin 配置由本地 Kong PG 管理
6. 每个 region 独立运行 browser-manager / k8s-chrome-daemon / K8s 集群
7. 南京增加 session gateway，对 `lexhome` 保持现有 sessions 接口兼容
8. 每个 region 提供 session 查询和管理数据面 API
9. 南京 session gateway 动态查询各 region 数据面并拼接
10. 管理操作按 `region_id` 代理到 owner region
11. context / download / extension 只在所属 region 内查询和使用

这套模型里，“共享数据”不是共享所有数据库，而是共享两类经过治理的数据：

- key 相关授权数据：从南京同步到各 region
- session 查询 / 管理能力：由各 region 暴露 API 给南京 session gateway

同时明确不共享：

- context
- download
- extension

### 9.12 落地阶段拆分

建议不要一次性做完整多 region 平台，可以拆成多个阶段。

#### 阶段一：region 注册与私网链路

目标：

- 先让南京能够识别并安全访问北京 region

要做：

- 定义内部 `region_id`、region 能力和服务端 endpoint 配置
- 打通南京到北京的私网 / VPN / 专线访问链路
- 北京侧提供 region data-plane gateway，不直接暴露 PG
- 两地服务调用使用 mTLS 或内部 service token

#### 阶段二：授权全局化

目标：

- 用户只维护一套 `project_id / api_key`
- 南京和北京 Kong 都能认证同一对凭证

要做：

- 全局 project / api_key 真值表
- Kong consumer / key-auth credential 同步器
- 每个 region 的同步状态表
- `lexhome` 展示 key 在各 region 的同步状态

明确不做：

- 不同步 route
- 不同步 service
- 不同步 plugin 配置

#### 阶段三：region 数据面鉴权

目标：

- 南京访问北京数据面时有明确认证、鉴权和审计边界

要做：

- 南京调用北京时必须携带 service credential
- 北京侧校验 `project_id / region_id / session_id` 归属
- 所有跨 region 调用记录审计日志
- 按 project 维护可用 region 和同步状态

#### 阶段四：session 动态查询展示

目标：

- 南京 `lexhome` 能看到北京 session，但不把北京 session 明细同步到南京

要做：

- 南京增加 session gateway，并保持现有 sessions 接口兼容
- 北京提供 session 查询数据面 API
- session gateway 并发查询南京 / 北京 session 数据
- session gateway 按 `region_id` 拼接展示数据
- 查询失败时按 region 展示降级状态

#### 阶段五：跨 region 管理操作

目标：

- 南京 `lexhome` 能关闭 / 查询 / 管理北京 session

要做：

- owner region 管理 API
- 南京 session gateway 按 `region_id` 路由
- 服务间认证
- 操作后重新查询 owner region 刷新展示
- 失败重试和错误展示

#### 阶段六：region-scoped 资源展示

目标：

- context / download / extension 按 region 展示和使用

要做：

- context 列表增加 region 维度
- extension 列表增加 region 维度
- download 通过 session 所属 region 查询
- 创建 session 时只能选择同 region 的 context / extension

### 9.13 不建议的方案

第一版不建议：

- 让北京 Kong 直接连南京 PG
- 两地 Kong PG 做双主复制
- 南京 `lexhome` 直接查北京业务库
- 把北京 session 明细同步到南京
- 跨 region 共享 context / download / extension
- session 运行态在两地互相写
- SDK 选择北京后，控制面请求又回南京创建实例

这些方案都会让 region 隔离失效，或者让故障面跨地域扩散。

### 9.14 跨 Region 监控数据聚合

北京 Prometheus 采集到的数据，需要能被南京 Grafana 读取。

这属于观测面聚合，不应该和业务数据同步、Kong key 同步混在一起。

推荐模型是：

- 每个 region 保留本地 Prometheus
- 每个 region 的 Prometheus 只负责 scrape 本 region 内的服务和节点
- 南京 Grafana 作为统一观测入口
- 南京 Grafana 通过数据源读取南京 Prometheus 和北京 Prometheus
- 指标必须带 `region_id` label，避免 dashboard 聚合时混淆来源

第一版可以选择两种落地方式。

#### 方式一：南京 Grafana 直连北京 Prometheus

链路：

1. 北京 Prometheus 在北京集群内部采集指标
2. 北京暴露一个只读 Prometheus query endpoint
3. 南京 Grafana 增加北京 Prometheus datasource
4. dashboard 查询时按 datasource 或 `region_id` 区分北京 / 南京

优点：

- 实现最快
- 不需要引入新的指标存储组件
- 各 region Prometheus 仍然独立

约束：

- 北京 Prometheus query endpoint 必须只对南京 Grafana 或南京内网开放
- 需要服务间认证或网络白名单
- 北京到南京链路断开时，南京 Grafana 无法实时查看北京指标，但北京本地采集不受影响

#### 方式二：Prometheus remote_write 到南京长期指标库

链路：

1. 北京 Prometheus 本地 scrape
2. 北京 Prometheus 通过 `remote_write` 把指标写到南京的集中式指标后端
3. 南京 Grafana 读取集中式指标后端

集中式指标后端可以是：

- VictoriaMetrics
- Thanos Receive
- Mimir
- 其他兼容 Prometheus remote_write / PromQL 的存储

优点：

- 南京 Grafana 查询路径稳定
- 更适合长期存储和跨 region 聚合查询
- 北京 Prometheus 短期不可达时，可以通过 remote_write 队列和重试做缓冲

约束：

- 需要额外维护集中式指标存储
- 需要处理指标量、保留周期、租户隔离和写入鉴权

#### 第一版建议

第一版建议先用方式一：

- 北京保留本地 Prometheus
- 南京 Grafana 增加北京 Prometheus datasource
- datasource 走内网专线 / VPN / 私网互通
- query endpoint 做只读访问控制
- 所有指标统一补 `region_id`

后续如果需要跨 region 统一告警、长期存储、全局容量报表，再演进到方式二。

这里的边界是：

- Prometheus 指标可以跨 region 被南京 Grafana 读取
- 但 Prometheus 不参与 session 查询 / 管理链路
- Prometheus 不作为 `lexhome` session 列表的数据源
- Prometheus 不替代各 region 的 session 数据面 API

## 10. 最小日志口径

默认至少应记录：

- 当前 `selection_mode`
- 是否命中缓存
- 是否触发 `force_refresh`
- 候选 regions 数量
- 过滤后剩余数量
- 最终选中的 `region_id / base_url`
- 是否发生 fallback
- fallback 后切到哪里

建议日志分层：

- `info`
  - 是否命中缓存
  - 最终选中了哪个入口
  - 是否 fallback
- `debug`
  - 每个 endpoint 的 probe 摘要
  - RTT 细节

## 11. 第一版明确不做的事

为了避免范围失控，第一版明确不做：

- 不暴露 K8s / zone / Pod 语义给 SDK
- 不做每个请求动态选路
- 不依赖复杂外部 IP 地理服务做核心决策
- 不把 internal/debug 混进正式默认路径
- 不直接对外暴露 `qcloud / office` 这类内部环境名作为正式 `region`
- 不在单个 session 生命周期中做跨 region 控制面漂移
- 不让某个 region 的 `browser-manager` 去直接跨 region 调别的 `k8s-chrome-daemon`
- 不让 region Kong 在线依赖其他地域的 PG
- 不让 `lexhome` 直接跨地域读取 region 本地运行态数据库，只能通过 region 数据面 API 查询

## 12. 当前任务拆分顺序

当前最合适的先后顺序已经收口：

### 第一优先级

- `定义全局 project / api_key 真值来源`
- `实现 Kong key 相关授权数据向各 region 同步`

原因：

- 这是多 region 能否共用一套凭证的前提
- 如果 key 不能跨 region 使用，SDK 选到北京也无法完成认证

### 第二优先级

- `实现各 region 的 session 查询 / 管理数据面 API`
- `让 session gateway 动态查询各 region 并拼接`

原因：

- 南京 `lexhome` 不保存北京 session 明细
- 北京必须提供受控数据面，南京 session gateway 才能展示和操作北京 session

### 第三优先级

- `让 lexhome 管理操作按 region_id 路由到 owner region`

原因：

- close / inspect / reconnect 都必须由 session 所在 region 执行
- 南京只做用户入口和管理面代理

### 第四优先级

- `让南京 Grafana 读取北京 Prometheus`

原因：

- 这是观测面聚合，不影响 session 运行态
- 第一版可以先通过 Grafana datasource 直连北京 Prometheus

### 之后再开 SDK / catalog 侧

- `提供 SDK 可消费的 public endpoint catalog`
- `收口 public / internal / debug 暴露边界`
- `落地就近接入统一配置语义与参数解析`
- `实现自动选路主流程`
- `实现缓存 / fallback / 最小日志口径`

## 13. 一句话总结

这件事的前提不是先写 SDK 选路代码，而是先把多 region 管理面打通：

- 先让一套 `project_id / api_key` 能在南京和北京 Kong 使用
- 再让北京提供 session 查询 / 管理数据面给南京 `lexhome`
- 再让南京 session gateway 动态查询和拼接各 region session，并对 `lexhome` 保持 sessions 接口兼容
- 再补齐跨 region 观测入口
- 最后 SDK 再基于稳定的 public catalog 实现 `catalog -> filter -> probe -> select`

否则 SDK 即使选到了北京，也会遇到凭证不可用、网站看不到 session、南京无法管理北京实例的问题。
