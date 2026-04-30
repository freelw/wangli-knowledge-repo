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

## 阶段七：跨 Region 监控聚合

- 每个 region 保留本地 Prometheus，只采集本 region。
- 南京 Grafana 作为统一观测入口。
- 第一版由南京 Grafana 直连北京 Prometheus 只读 datasource。
- 后续需要长期存储时，再演进到 remote_write + 集中式指标库。

## 阶段八：Public 入口目录

- 定义 public endpoint catalog，SDK 只消费 `scope=public` 的入口。
- 明确 `region_id` 命名，不把 `qcloud / office` 这类内部环境名暴露给正式 SDK。
- 每个 public region 对应一个独立 K8s 集群和一个外部入口。
- catalog 只描述可接入 region，不负责 region 内服务编排。

## 阶段九：SDK 选路能力

- SDK 启动时拉取 catalog，按手动 region 或自动模式筛选候选入口。
- 对候选入口做轻量 probe，选择最快可用 region。
- 缓存测速结果，避免每次启动都测速。
- 参数优先级固定为 `base_url > manual region > auto`。
