# SDK 就近接入与多 Region 管理面落地计划

## 阶段一：Region 注册与私网链路

- 定义内部 `region_id`、region 能力和服务端 endpoint 配置。
- 打通南京到北京的私网 / VPN / 专线访问链路。
- 北京侧提供 region data-plane gateway，不直接暴露 PG。
- 两地服务调用使用 mTLS 或内部 service token。

## 阶段二：全局 Project / API Key

- 南京保留唯一 `lexhome` 和全局 project / api_key 真值。
- 用户只维护一套 `project_id / api_key`。
- 同步器把 key 相关授权对象写入各 region 的 Kong Admin。
- 各 region Kong 使用本地 PG 认证，不跨地域访问南京 PG。

## 阶段三：Region 数据面鉴权

- 南京调用北京时必须携带 service credential。
- 北京侧校验 `project_id / region_id / session_id` 归属。
- 所有跨 region 调用记录审计日志。
- 按 project 维护可用 region 和同步状态。
- 北京 daemon 的 session timeout 配置读写也走北京 region data-plane，不直接跨 region 访问 PG。
- 南京官网修改北京 timeout 后，由北京 region data-plane 通知北京 daemon 刷新 timeout controller。

## 阶段四：Session 动态查询展示

- 南京增加 session gateway，对 `lexhome` 保持现有 sessions 接口兼容。
- 各 region 提供 session 查询数据面 API 给 session gateway。
- session gateway 动态查询南京和北京 browser-manager。
- session gateway 按 `region_id` 拼接、排序和展示 session。
- session 明细不从北京同步到南京。

## 阶段五：跨 Region 管理操作

- 南京 session gateway 根据 session 的 `region_id` 找到 owner region。
- close / inspect / reconnect 等操作由 session gateway 代理到 owner region。
- region 间管理调用使用服务间认证。
- 操作后重新查询 owner region 刷新页面状态。

## 阶段六：Region 内资源边界

- context / download / extension 不跨 region 共享。
- context 和 extension 按 `region_id` 查询和展示。
- download 按产生它的 session 所在 region 查询。
- 创建 session 时只能选择同 region 的 context / extension。

当前状态：

- SDK / API 创建 session 主链路已满足第一版边界：SDK 选定 region 后，请求进入该 region 的 Kong / browser-manager / daemon，session 和资源查询都发生在该 region 内。
- 如果在北京创建 session 时传入北京不存在的 `context_id`，北京本地查询失败，session 创建失败，因此不会使用南京 context 创建北京 session。
- 暂不单独推进阶段六后端改造；剩余工作主要是 `lexhome` 的 context / extension / download 展示和选择体验，后续和 `lexhome` region-aware UI 改造一起处理。

## 阶段七：跨 Region 监控聚合

- 每个 region 保留本地 Prometheus，只采集本 region。
- 南京 Grafana 作为统一观测入口。
- 第一版由南京 Grafana 直连北京 Prometheus 只读 datasource。
- 后续需要长期存储时，再演进到 remote_write + 集中式指标库。

当前状态：

- office 环境已完成第一版：office-beijing Prometheus 通过 NodePort 暴露只读访问，office-nanjing Grafana 增加 `prometheus-office-beijing` datasource。
- `grafana-dashboard-init` 已支持 dashboard 顶部 `Datasource` 变量，现有 dashboard 可以在 `prometheus` 和 `prometheus-office-beijing` 之间切换。
- office 使用 `node IP + NodePort` 仅用于模拟跨 region 私网访问；正式环境应替换为北京侧私网 LB / 固定 VIP / 内网域名。

## 阶段八：Public 入口目录

- 定义 public endpoint catalog，SDK 从主 region 拉取可选入口目录。
- 明确 `region_id` 命名，不把 `qcloud / office` 这类内部环境名暴露给正式 SDK。
- 每个 public region 对应一个独立 K8s 集群和一个外部入口。
- catalog 只描述可接入 region，不负责 region 内服务编排。

落地方式：

- `browser-manager` 暴露 `GET /v1/regions/catalog`，返回服务端配置的 public 入口目录。
- `browser-manager` 暴露 `GET /v1/region/probe`，用于 SDK 对候选 region 做轻量探测。
- 只有主 region 开启 catalog；从 region 不响应 catalog。
- catalog 返回 host 域名，SDK 直接通过 `https://<host>` / `wss://<host>` 访问对应 region。
- catalog 不返回 `priority`，SDK 不做测速选最快。
- catalog 必须有且只有一个 `default=true` 的 region，SDK 未指定 region 时优先选择 default。
- office 模拟环境中，office-nanjing 是主，catalog 返回 office-nanjing / office-beijing 两个接入点；office-beijing 不响应 catalog。
- 当前不改 qcloud / qcloud-hk 配置；它们现在是两套隔离主环境，等后续 qcloud 拆成 qcloud-nanjing / qcloud-beijing 后再接入 catalog。
- 正式环境第一版由主 region 返回正式入口目录；对外 `region_id` 使用产品语义名，例如 `cn-nanjing`、`hk`。

## 阶段九：SDK 选路能力

- 不做自动测速选最快，也不做测速结果缓存。
- SDK 使用 `base_url` 作为 catalog base URL。
- 如果用户显式传入 `region`，SDK 从 catalog 中选择该 region。
- 如果用户未传入 `region`，SDK 使用 catalog 中唯一 `default=true` 的 region。
- 如果 catalog / probe 不可用，SDK fallback 到原始 `base_url` 行为。

当前状态：

- Python SDK / Node.js SDK 已支持显式 `region`、default region 和 catalog / probe 不可用时的 fallback。
- 阶段九中的自动测速选最快、测速缓存不做。
- qcloud / qcloud-hk 当前仍是两套隔离主环境，暂不接入 catalog；等后续 qcloud 拆成 qcloud-nanjing / qcloud-beijing 后再接入。
## 2026-05-09 office 全链路验收状态

已验证通过：

- `lexmount-e2e-tool` 的 `session-list` 用例通过，能通过当前默认入口创建、查询和清理 session。
- office-nanjing 的 `session-gateway`、`region-data-plane-gateway`、`session-quota`、`kong-key-sync`、`browser-manager` deployment 均为 `Available`。
- office-beijing 的 `region-data-plane-gateway`、`browser-manager` deployment 均为 `Available`。
- `kong-key-sync` 日志显示 `sync-all-projects` 分页同步成功，多个 project 已同步到 `office-beijing`。
- `sync-all-kong-projects-backfill` Job manifest server-side dry-run 通过。
- office-nanjing Grafana 已配置 `prometheus-office-beijing` datasource。
- office-beijing Prometheus `http://10.3.217.198:30926/prometheus/-/ready` 返回 Ready，`up` 查询成功。
- dashboard-init 已注入 `Datasource` 变量，dashboard panel / target datasource 均指向 `$datasource`，可在 `prometheus` 和 `prometheus-office-beijing` 之间切换。

当前阻塞：

- `demo-nodejs-backend` PR #139 `Use host-only public region catalog` 仍是 OPEN，尚未合入 `main`。
- 当前 `demo-nodejs-backend/main` 和 office 部署的 browser-manager 镜像不包含 `/v1/regions/catalog`、`/v1/region/probe` 代码。
- 因此 `https://apitest.local.lexmount.net/v1/regions/catalog`、`https://apitest.local.lexmount.net/v1/region/probe`、`https://apitest-beijing.local.lexmount.net/v1/region/probe` 当前均返回 404。
- Python quickstart `catalog_info.py` 返回 `available=false status_code=404`。
- `lexmount-e2e-tool` 的 `catalog-info` 只能验证 fallback，显示 `catalog available=false regions=0`。
- `lexmount-e2e-tool` 的 `region-sdk-session` 失败，错误为 `catalog has no regions`。

处理建议：

1. 先合入并发布 `demo-nodejs-backend` PR #139。
2. 基于合入后的 `main` 重新构建并发布 office browser-manager 镜像。
3. 重新执行 catalog / probe、Python SDK、Node.js SDK、`lexmount-e2e-tool region-sdk-session` 验收。

