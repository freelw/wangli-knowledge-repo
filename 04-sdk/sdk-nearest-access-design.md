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

## 9. 最小日志口径

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

## 10. 第一版明确不做的事

为了避免范围失控，第一版明确不做：

- 不暴露 K8s / zone / Pod 语义给 SDK
- 不做每个请求动态选路
- 不依赖复杂外部 IP 地理服务做核心决策
- 不把 internal/debug 混进正式默认路径
- 不直接对外暴露 `qcloud / office` 这类内部环境名作为正式 `region`
- 不在单个 session 生命周期中做跨 region 控制面漂移
- 不让某个 region 的 `browser-manager` 去直接跨 region 调别的 `k8s-chrome-daemon`

## 11. 当前任务拆分顺序

当前最合适的先后顺序已经收口：

### 第一优先级

- `提供 SDK 可消费的 public endpoint catalog`

原因：

- 它是 SDK 自动选路的输入面
- 如果这一层没定稳，SDK 只能继续硬编码或猜入口

### 第二优先级

- `收口 public / internal / debug 暴露边界`

原因：

- catalog 如果不和 scope 边界一起定，后面很容易把 `office` 或 internal/debug 入口混进去

### 第三优先级

- `收口多 region 控制面 bundle 与 region sticky 规则`

原因：

- 如果 region 只停留在入口命名层，后续很容易出现跨 region 控制面串流
- SDK 即使选对了入口，也无法保证 session 真正落在对应 region 的完整控制面内

### 之后再开 SDK 侧 3 条

- `落地就近接入统一配置语义与参数解析`
- `实现自动选路主流程`
- `实现缓存 / fallback / 最小日志口径`

## 12. 一句话总结

这件事的前提不是先写 SDK 选路代码，而是：

- 先把服务端 `public endpoint catalog` 做出来
- 再把 `public / internal / debug` 的边界定死
- 再把多 region 下的控制面 bundle 和 region sticky 规则定死
- 然后 SDK 再在稳定的 public 输入面上实现 `catalog -> filter -> probe -> select`

否则 SDK 仍然会被迫自己猜入口，`office` 也会持续存在被误混进正式默认路径的风险。
