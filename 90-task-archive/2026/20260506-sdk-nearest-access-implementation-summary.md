# SDK 就近接入 2026-05-06 工作总结

## 背景

本轮工作继续推进 SDK 就近接入 / 多 region 控制面的第一阶段落地。当前 office 环境用于模拟双 region：

- `office-nanjing`：模拟南京，承载官网 `lex-home`、聚合入口、主控数据。
- `office-beijing`：模拟北京，承载独立 K8s 集群、北京 Kong、browser-manager、daemon、浏览器实例。

核心目标是：用户只维护一套 `project_id` / `api_key`，SDK 指定 region 后可以在对应 region 创建实例，同时南京官网能查看和操作多 region session。

## 今日合入的 PR

- `demo-nodejs-backend`：region data plane / session gateway / kong-key-sync。
- `lexmount-k8s-manifests`：office-nanjing / office-beijing 部署配置。
- `k8s-chrome-daemon`：去 PocketBase timeout 依赖，改为 PG + HTTP 接口 + refresh 通知链路。

三个 PR 已经基于最新 `main` 处理 review comments 后合入。

## Kong Key 同步

新增独立服务 `kong-key-sync`，放在 `demo-nodejs-backend` 中。

同步方式：

1. 以 `project_id` 为同步粒度。
2. 从南京 Kong Admin 读取该 project 对应的 consumer 和 key-auth credentials。
3. 通过北京 `region-data-plane-gateway` 的内部 Kong Admin 代理接口写入北京 Kong。
4. 同步采用 project 级 reconciliation：
   - 南京有、北京没有的 key：在北京创建。
   - 北京有、南京没有的 key：从北京删除。
   - credential id 可以不同，但 key 值必须一致。

`lex-home` 已接入触发逻辑：

- create api key 后触发当前 project 的同步。
- delete api key 后触发当前 project 的同步。
- 如果没有配置 `KONG_KEY_SYNC_BASE_URL`，则不会触发同步。

当前只有 `office-nanjing` 配置了 `KONG_KEY_SYNC_BASE_URL`，`qcloud` / `qcloud-hk` 不配置，因此发布新 lex-home 镜像对它们基本无感。

## Region Data Plane Gateway

`region-data-plane-gateway` 用作跨 region 的内部数据面代理。北京侧部署该服务，南京侧通过它访问北京 region 内部能力。

本轮已扩展内部 Kong Admin 代理能力，包括：

- 查询 / 创建 / 更新 Kong consumer。
- 查询 / 创建 key-auth credential。
- 删除 key-auth credential。

这些接口只用于内部链路，需要通过内部 token 鉴权。

## Session Gateway

南京部署 `session-gateway`，作为官网访问 browser-manager sessions 接口的中间层。

设计原则：

- 不把北京 session 明细同步落库到南京。
- 南京官网展示时，由 `session-gateway` 动态读取南京和北京的 browser-manager sessions 数据并拼接。
- session 运行态真值仍归属创建它的 region。
- 对北京 session 的关闭等操作，通过北京数据面代理转发到北京 region 内执行。

本轮 office 环境已验证：

- 可以通过北京 Kong 创建实例。
- 南京 `session-gateway` 可以看到北京 session。
- 南京侧可以操作北京 session。

## Timeout 配置链路

`k8s-chrome-daemon` 已去掉对 PocketBase 的 timeout 配置依赖，改为 PG + HTTP 接口。

当前原则：

- timeout 配置不按 `region_id` 区分。
- 南京维护 timeout 真值。
- 北京 daemon 不直接读北京 PG 获取 timeout 真值。
- 北京 daemon 需要通过南京代理数据面读取 timeout 配置。
- 官网修改 timeout 后，需要通知北京 daemon refresh cache。

本轮已完成 daemon 侧基础接口和 refresh 链路，后续仍需结合跨 region 代理链路继续补齐生产化细节。

## Office 双 Region 配置验证

当前 live office 环境确认：

### Kong PG

- `office-nanjing`: `database=k8s_kong`, `user=k8s_kong`
- `office-beijing`: `database=k8s_kong_beijing`, `user=k8s_kong_beijing`

### Browser Manager / Session PG

- `office-nanjing`: `POSTGRES_DB=browser_sessions_office`, `POSTGRES_USER=browser_sessions_office`
- `office-beijing`: `POSTGRES_DB=browser_sessions_office_beijing`, `POSTGRES_USER=browser_sessions_office_beijing`

因此 Kong 和 browser-manager/session 主业务 PG 已经按 region 隔离。

注意：`office-runtime-secrets.yaml` 中 replay processor / cleaner 以及 backend health monitor 的 DSN 目前仍是共享配置。如果后续要求 office-beijing 所有 PG 连接都完全隔离，这几项还需要继续拆。

## 已验证结果

- office-nanjing / office-beijing 双 context 可同时操作。
- 北京 Kong route 修正后，北京 region 可以创建 browser instance。
- 南京 session-gateway 可以聚合展示北京 session。
- lex-home 新增 key 后，可以触发 project 级 Kong key 同步到北京。
- lex-home 删除 key 后，可以触发 project 级 reconciliation，从北京删除对应 key。
- 北京 Kong key 查询可通过北京 `region-data-plane-gateway` NodePort 验证。

## 后续 TODO

### 1. 上线前 backfill

需要补一个 `sync-all-kong-projects` backfill 任务：

- 枚举南京 Kong 中已有的所有 project / consumer。
- 按 project 调用现有 `kong-key-sync`。
- 记录失败项目并支持重试。
- 用于正式上线前把历史 project 的 key 补齐到目标 region。

### 2. 监控汇聚

需要让北京 Prometheus 采集到的数据能被南京 Grafana 读取。

### 3. Public Endpoint Catalog

需要继续实现 SDK 自动选路依赖的 public endpoint catalog：

- 默认只暴露 `scope=public` 的正式 region。
- `office` 不进入正式默认 catalog。
- catalog 需要包含 region、endpoint、scope、健康状态等信息。

### 4. SDK 自动选路

SDK 主流程仍按以下链路推进：

`catalog -> filter -> probe -> select`

还需要实现：

- SDK 启动时枚举 endpoint。
- 探测延迟 / 可用性。
- 选择最快 region。
- 缓存测速结果，避免每次启动都测速。
- 用户显式 `base_url` 优先级最高，其次 manual region，最后 auto。

### 5. Region 资源边界

后续需要继续明确 context / download / extension 等数据的跨 region 边界。这些数据不能默认跨 region 共享。
