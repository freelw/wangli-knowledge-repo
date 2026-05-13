# qcloud 多 Region 正式发布总结（2026-05-13）

## 1. 发布结论

2026-05-12，qcloud 多 Region 架构已完成第一版正式发布，核心能力从 office 模拟环境推进到正式 qcloud 环境：

- `qcloud-nanjing` 作为主 region，保留官网 `lexhome`、catalog、session 聚合、Kong key 同步和全局并发配额真值入口。
- `qcloud-beijing` 作为从 region，提供独立 K8s 集群、独立 Kong / browser-manager / browser-operator / region-data-plane-gateway，用于创建和管理北京 region 的浏览器实例。
- SDK 指定 `region=qcloud-beijing` 后，能够通过南京 catalog 发现北京入口 `api-beijing.lexmount.cn`，并将 session create 请求切到北京 Kong。
- `project_id / api_key` 仍以南京为真值，通过 `kong-key-sync` 同步到北京 Kong，本次 backfill 已同步 34 个 project，失败数为 0。
- qcloud-beijing 的 session timeout、session quota、session 管理数据面均通过南京 / 北京之间的 region gateway 链路打通，不直接跨 region 访问南京 PG。

## 2. 当前拓扑

### 2.1 Region 角色

| Region | 角色 | 入口 | 主要职责 |
| --- | --- | --- | --- |
| `qcloud-nanjing` | 主 region | `api.lexmount.cn` | catalog、lexhome、session-gateway、kong-key-sync、session-quota、南京本地 browser 控制面 |
| `qcloud-beijing` | 从 region | `api-beijing.lexmount.cn` | 北京本地 Kong、browser-manager、browser-operator、region-data-plane-gateway、浏览器实例 |

### 2.2 请求链路

SDK 创建北京 session 的目标链路是：

```text
SDK(region=qcloud-beijing)
  -> https://api.lexmount.cn/v1/regions/catalog
  -> https://api-beijing.lexmount.cn/v1/region/probe
  -> https://api-beijing.lexmount.cn/instance...
  -> qcloud-beijing Kong
  -> qcloud-beijing browser-manager
  -> qcloud-beijing browser-operator / browser pod
```

南京官网展示 / 操作多 region session 的目标链路是：

```text
lexhome (qcloud-nanjing)
  -> session-gateway (qcloud-nanjing)
  -> region-data-plane-gateway (owner region)
  -> owner region browser-manager / browser-operator
```

其中，北京侧不应该成为 catalog 真值，也不应该提供 lexhome 聚合入口。

## 3. 已落地能力

### 3.1 Public endpoint catalog

`browser-manager` 已提供：

- `GET /v1/regions/catalog`
- `GET /v1/region/probe`

正式 qcloud 当前配置：

- `qcloud-nanjing`：`REGION_CATALOG_ENABLED=true`，catalog 返回南京和北京两个 region。
- `qcloud-beijing`：`REGION_CATALOG_ENABLED=false`，只提供本 region probe，不提供 catalog。
- 北京入口 host：`api-beijing.lexmount.cn`。

相关提交：

- `demo-nodejs-backend` PR #139：host-only catalog / probe 基础能力。
- `demo-nodejs-backend` PR #163：qcloud-nanjing / qcloud-beijing Kong 增加 `/v1/region/probe` route。
- `lexmount-k8s-manifests` PR #446：qcloud catalog / quota / region registry 基础配置。
- `lexmount-k8s-manifests` PR #447：更新 qcloud `kong-init` 镜像 tag。

### 3.2 SDK region 选择

Python / Node.js SDK 已支持：

- 显式 `region` 选择。
- 未指定 region 时选择 catalog 中唯一 `default=true` 的 region。
- catalog / probe 不可用时 fallback 到原始 `base_url` 行为。
- host-only catalog：SDK 根据 catalog 中的 `host` 构造 `https://<host>` 访问 region，不再依赖 endpoint IP + Host header。

验证过的 qcloud 行为：

```python
from lexmount import Lexmount

lm = Lexmount(region="qcloud-beijing")
print(lm.region_info())
```

期望结果：

```text
selected_region = qcloud-beijing
endpoint_base_url = https://api-beijing.lexmount.cn
host = api-beijing.lexmount.cn
```

已确认 `https://api-beijing.lexmount.cn/v1/region/probe` 返回：

```json
{"region_id":"qcloud-beijing","display_name":"QCloud Beijing"}
```

相关提交：

- `lexmount-python-sdk` PR #92 / #93：region selection、host-only catalog，版本进入 `0.5.x`。
- `lexmount-js-sdk` PR #9 / #12：region selection、session region 信息暴露，版本进入 `0.5.x`。

### 3.3 Kong key 同步

南京保留 project / api_key 真值，北京 Kong 使用本地 PG 鉴权。

同步链路：

```text
qcloud-nanjing kong-key-sync
  -> 读取 qcloud-nanjing Kong Admin consumer / key-auth credential
  -> 调 qcloud-beijing region-data-plane-gateway
  -> 写入 qcloud-beijing Kong Admin
```

qcloud-nanjing 已引入：

- `kong-key-sync` deployment / service。
- `sync-all-kong-projects-backfill` 手动 Job。

上线 backfill 结果：

```text
source_region_id = qcloud-nanjing
target_regions = [qcloud-beijing]
page_project_count = 34
processed_count = 34
ok_count = 34
failed_count = 0
```

执行方式：

```bash
kubectl -n system delete job sync-all-kong-projects-backfill --ignore-not-found
kubectl -n system apply -f apps/clusters/qcloud-nanjing/manual-jobs/sync-all-kong-projects-job.yaml
kubectl -n system logs -f job/sync-all-kong-projects-backfill
```

相关提交：

- `demo-nodejs-backend` PR #134：Kong key sync 与 region data-plane hardening。
- `demo-nodejs-backend` PR #135：sync-all 分页 backfill。
- `lexmount-k8s-manifests` PR #448：qcloud-nanjing 引入 `kong-key-sync` 和 backfill Job。

### 3.4 Session quota 全局并发

`session-quota` 服务部署在南京，用于维护跨 region 的 project 并发租约。

当前配置：

- qcloud-nanjing browser-manager：使用本地 `session-quota.system.svc.cluster.local:9233`。
- qcloud-beijing browser-manager：通过 `http://10.206.0.89:30925` 访问南京 session-quota。
- lease TTL：`3600000ms`（1 小时）。

调度要求：

- `session-quota` 强制调度到 `lexmount/kong=true` 节点。
- 增加软反亲和和 topology spread，尽量分散，但节点不足时不阻塞调度。

相关提交：

- `demo-nodejs-backend` PR #133：全局 session quota lease 服务。
- `lexmount-k8s-manifests` PR #446：qcloud-nanjing 部署 session-quota，qcloud-beijing 接入南京 quota。

### 3.5 Region data-plane gateway

`region-data-plane-gateway` 是跨 region 管理面入口，不是业务公网入口。

用途：

- 南京 session-gateway 动态查询 / 操作北京 session。
- 南京 kong-key-sync 写北京 Kong key。
- 北京 daemon 从南京读取 timeout 配置真值。

当前端口：

- qcloud-nanjing：NodePort `30924`。
- qcloud-beijing：NodePort `30924`。

当前 registry：

```text
qcloud-nanjing local data plane = http://region-data-plane-gateway.system.svc.cluster.local:9230
qcloud-beijing data plane = http://172.21.0.2:30924
```

调度要求：

- 强制调度到 `lexmount/kong=true` 节点。
- 增加软反亲和和 topology spread，尽量均匀分布。

相关提交：

- `demo-nodejs-backend` PR #131：region data-plane gateway。
- `lexmount-k8s-manifests` PR #446：qcloud region registry、NodePort、调度策略。

### 3.6 Session gateway 与 lexhome

南京 `lexhome` 不应直连南京 browser-manager，否则只能看到南京本地 session。

qcloud-nanjing 当前目标配置：

```text
lexhome BROWSER_BASE_URL = http://session-gateway.system.svc.cluster.local:9231/
```

session-gateway 读取 `region-registry` 后，动态查询南京和北京，并按 `region_id` 聚合 session。

注意：

- 修改 `region-registry` 后，需要 rollout `session-gateway`，否则旧 Pod 仍只知道旧 region 列表。
- 修改 `lexhome` 的 `BROWSER_BASE_URL` 后，需要 rollout `lex-home`。

常用命令：

```bash
kubectl -n system rollout restart deploy/session-gateway
kubectl -n system rollout status deploy/session-gateway
kubectl -n frontend rollout restart deploy/lex-home
kubectl -n frontend rollout status deploy/lex-home
```

### 3.7 Session timeout 配置

`session_timeout_configs` 的真值在南京。

北京 daemon 不直接读南京 PG，而是通过北京配置的远端 timeout URL 拉取南京真值：

```text
qcloud-beijing REMOTE_SESSION_TIMEOUT_CONFIG_URL = http://10.206.0.89:30924/v1/session-timeout/configs
```

南京修改 timeout 后，通过 region data-plane 通知目标 region 刷新；北京 daemon refresh 时仍然从南京真值源读取。

相关提交：

- `k8s-chrome-daemon` PR #65：session timeout 从 PG 读取。
- `k8s-chrome-daemon` PR #68：remote session timeout config source。
- `demo-nodejs-backend` PR #136：local truth by default。
- `lexmount-k8s-manifests` PR #446：qcloud-beijing remote timeout patch。

### 3.8 跨 Region 监控

office 阶段已验证跨 region Grafana 读取 Prometheus 的模式：

- 每个 region 保留本地 Prometheus。
- 主 region Grafana 增加远端 Prometheus datasource。
- dashboard-init 支持 datasource 变量，在 dashboard 顶部切换数据源。

相关提交：

- `demo-nodejs-backend` PR #145：Grafana dashboard datasource 变量。
- `lexmount-k8s-manifests` PR #411：office 跨 region Prometheus datasource 验证。

qcloud 正式环境后续如需统一观测，应沿用“主 region Grafana + 远端 Prometheus datasource”的模式，或进一步演进到 remote_write + 集中指标库。

## 4. 发布中处理过的问题

### 4.1 DNS 已通但 HTTPS 不通

现象：

```bash
ping api-beijing.lexmount.cn
curl https://api-beijing.lexmount.cn/v1/region/probe
```

DNS 解析到 `152.136.231.37`，但 curl 超时。

结论：

- 问题不在 SDK，也不在 Kong route。
- TCP 端口不通，原因是 CLB 安全组默认未放通。

处理：

- 放通 CLB 安全组后，`/v1/region/probe` 返回 200。

### 4.2 创建北京 session 却落到南京

可能原因：

1. SDK 传入的 region ID 不匹配 catalog。
   - 正确值是 `qcloud-beijing`，不是 `beijing`。
2. 北京域名 / probe 不通。
   - SDK probe 失败会 fallback 到原始 `base_url=https://api.lexmount.cn`。
3. 南京 browser-manager 未 rollout catalog。
   - catalog 没包含 qcloud-beijing 时，SDK 会 fallback。
4. qcloud-beijing Kong 未配置 `/v1/region/probe` route。
   - 已通过 `demo-nodejs-backend` PR #163 修复。

排查命令：

```bash
curl -i https://api.lexmount.cn/v1/regions/catalog
curl -i https://api-beijing.lexmount.cn/v1/region/probe
```

Python SDK 排查：

```python
from lexmount import Lexmount
lm = Lexmount(region="qcloud-beijing")
print(lm.catalog_info())
print(lm.region_info())
```

### 4.3 lexhome 看不到北京 session

原因链路：

- lexhome 如果仍直连南京 browser-manager，只能看到南京 session。
- lexhome 必须指向南京 session-gateway。
- session-gateway 必须读取包含 qcloud-beijing 的 region-registry。
- 改 ConfigMap 后必须 rollout lex-home 和 session-gateway。

### 4.4 qcloud-beijing 不应该部署 session-gateway

设计上：

- session-gateway 是主 region 的聚合入口。
- qcloud-beijing 只需要 region-data-plane-gateway，供南京访问。

当前 main 在发布过程中曾引入 qcloud-beijing session-gateway。后续清理 PR：

- `lexmount-k8s-manifests` PR #449：Remove qcloud-beijing session-gateway。

该 PR 合入后，qcloud-beijing 应只保留本地控制面和 data-plane gateway。

## 5. 发布涉及的主要 PR / 提交

### demo-nodejs-backend

- PR #131：Add region data-plane gateway。
- PR #133：Add global session quota leases。
- PR #134：Add Kong key backfill and region gateway hardening。
- PR #135：Page Kong key sync backfill。
- PR #136：Read timeout configs from local truth by default。
- PR #139：Use host-only public region catalog。
- PR #142：Route session management operations by owner region。
- PR #145：Add Grafana datasource variable to dashboards。
- PR #153：Add qcloud-nanjing kong init configs。
- PR #161：Add qcloud-beijing kong init config。
- PR #163：Add qcloud region probe Kong routes。

### lexmount-k8s-manifests

- PR #367：office region data-plane gateway。
- PR #385：office session quota。
- PR #411：office cross-region monitoring aggregation。
- PR #424：qcloud-nanjing gateway catalog rollout。
- PR #434：qcloud-beijing cluster manifests。
- PR #446：qcloud region quota / registry / NodePort / timeout groundwork。
- PR #447：qcloud kong-init image tag。
- PR #448：qcloud-nanjing kong-key-sync。
- PR #449：qcloud-beijing session-gateway cleanup（后续清理项）。

### k8s-chrome-daemon

- PR #65：timeout config 改为 PG。
- PR #68：remote session timeout config source。
- PR #69：session create 阶段 `/json/version` connection refused 作为 transient 状态处理。
- PR #70 / #71：browser pod resource requests / limits。

### SDK

- `lexmount-python-sdk`：`v0.5.0` region selection，`v0.5.1` host-only region selection。
- `lexmount-js-sdk`：`v0.5.1` host-only region selection，`v0.5.4` session region info。

## 6. 当前运行检查清单

发布 / 变更后建议按以下顺序检查：

```bash
# 1. catalog / probe
curl -i https://api.lexmount.cn/v1/regions/catalog
curl -i https://api-beijing.lexmount.cn/v1/region/probe

# 2. SDK region resolution
python - <<'PY'
from lexmount import Lexmount
lm = Lexmount(region="qcloud-beijing")
print(lm.region_info())
PY

# 3. Kong key backfill
kubectl -n system logs -f job/sync-all-kong-projects-backfill

# 4. session-gateway config
kubectl -n system get cm session-gateway-config -o yaml | grep -A30 REGION_REGISTRY_JSON
kubectl -n system rollout status deploy/session-gateway

# 5. lexhome base URL
kubectl -n frontend exec deploy/lex-home -- printenv | grep BROWSER_BASE_URL

# 6. qcloud-beijing timeout remote source
kubectl -n system get cm browser-operator-config -o yaml | grep REMOTE_SESSION_TIMEOUT_CONFIG
```

## 7. 长期边界

- SDK 只选择 region，不选择 node / pod / zone。
- 每个 region 是独立 K8s 集群和独立控制面 bundle。
- 用户只维护一套 `project_id / api_key`，但每个 region Kong 使用本地 PG 完成本地鉴权。
- Session 明细不从北京同步到南京；lexhome 展示时由南京 session-gateway 动态查询各 region。
- Context / download / extension 不跨 region 共享；创建 session 时只能使用目标 region 本地存在的资源。
- 主 region 响应 catalog，从 region 不响应 catalog。
- qcloud-beijing 不承载 lexhome，也不需要 session-gateway。
