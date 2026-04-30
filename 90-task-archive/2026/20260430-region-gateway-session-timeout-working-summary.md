# 2026-04-30 多 Region 数据面与 Session Timeout 临时工作总结

> 状态：临时记录，相关 PR 尚未完全 review，暂不合入。
>
> 目标：记录当前 office-nanjing / office-beijing 模拟多 region 的设计、代码改动、部署验证结果、已知问题和后续 review 重点，避免上下文丢失。

## 相关 PR

- `demo-nodejs-backend`: <https://github.com/lexmount/demo-nodejs-backend/pull/131>
  - 当前分支：`flash1_region_gateway_phase1_20260430`
  - 当前关键提交：`dcbe4c1`
- `k8s-chrome-daemon`: <https://github.com/lexmount/k8s-chrome-daemon/pull/65>
  - 当前分支：`flash1_remove_pocketbase_timeout_20260430_095349`
  - 当前关键提交：`97b8a98`
- `lexmount-k8s-manifests`: <https://github.com/lexmount/lexmount-k8s-manifests/pull/367>
  - 当前分支：`flash1_region_gateway_phase1_20260430`
  - 当前关键提交：`b0f4cf8`

## 当前环境约定

- `office-nanjing` 模拟南京 region。
- `office-beijing` 模拟北京 region。
- 两个 region 是两个独立 K8s 集群。
- `lexhome` 只部署在南京。
- `session-gateway` 只部署在南京。
- `region-data-plane-gateway` 每个 region 都部署。
- 北京不部署 `lexhome`，也不部署 `session-gateway`。

## 总体调用链

### 官网 session 查询 / 操作

```text
nanjing lexhome
  -> nanjing session-gateway
  -> target region region-data-plane-gateway
  -> target region browser-manager / daemon
```

- `session-gateway` 是南京侧的跨 region 聚合 / 路由入口。
- `region-data-plane-gateway` 是每个 region 内部的数据面 adapter，只访问本 region 的 browser-manager / daemon。
- 跨 region 列表只由南京 `session-gateway` 读取；北京不需要 region registry。

### SDK 创建浏览器实例

正式路径仍是：

```text
SDK
  -> target region Kong
  -> target region browser-manager
  -> target region k8s-chrome-daemon
```

本次主要处理官网 / 管理面通过南京访问多 region session 的链路，不改变 SDK 正式入口选路规则。

## 已实现内容

### 1. Region Data Plane Gateway

仓库：`demo-nodejs-backend`

新增服务：`region-data-plane-gateway`

职责：

- 部署在每个 region。
- 对外提供 region 数据面 API。
- 校验内部 token。
- 代理到本 region 的 browser-manager / daemon。
- 不访问其它 region。
- 不直接访问 PG。

当前主要接口：

```text
GET  /healthz
GET  /readyz
GET  /v1/region
POST /v1/sessions
POST /v1/instance
POST /v1/instance/v2
POST /v1/session
POST /v1/session/close
DELETE /v1/session
GET  /v1/session-timeout/configs
POST /v1/session-timeout/configs
PUT  /v1/session-timeout/configs
DELETE /v1/session-timeout/configs
POST /v1/session-timeout/refresh
```

安全/稳定性补充：

- `secureEquals` 已改为 padding 后 constant-time compare，避免 token 长度 timing leak。
- JSON body 增加 1MiB 限制，超限返回 `413`。
- `/readyz` 不再返回内部 `browser_manager_base_url`。

### 2. Session Gateway

仓库：`demo-nodejs-backend`

新增服务：`session-gateway`

职责：

- 只部署在南京。
- 对 `lexhome` 保持 browser-manager sessions 相关接口兼容。
- 根据 region registry 路由 / fan-out 到各 region data-plane gateway。
- 聚合多 region session 查询结果。
- 单 session 操作按 `region_id` 或 session 查询结果路由到 owner region。

当前主要接口：

```text
GET  /healthz
GET  /readyz
GET  /v1/regions
POST /instance/sessions
POST /instance/v2/sessions
POST /instance
POST /instance/v2
POST /instance/session
DELETE /instance
GET  /v1/session-timeout/configs
POST /v1/session-timeout/configs
PUT  /v1/session-timeout/configs
DELETE /v1/session-timeout/configs
POST /v1/session-timeout/refresh
```

重要兼容修复：

- `lexhome` 创建 session 使用 `POST /instance`。
- 初版 `session-gateway` 未实现该路径，导致创建 session 返回 `404`。
- 已补 `POST /instance` 和 `POST /instance/v2`。
- 默认创建到 `LOCAL_REGION_ID`，也支持请求 body 携带 `region_id` 指定目标 region。

当前没有强制给 `session-gateway` 入站请求加内部 token。

原因：

- 当前 `lexhome` 调用 `BROWSER_BASE_URL` 不携带内部 token header。
- 如果直接强制鉴权，会导致官网 sessions / timeout / create session 路径 `401`。
- 后续如果要做，需要先改 `lexhome` 请求层统一带 `X-Internal-Token`，再开启 `session-gateway` 入站鉴权。

### 3. Daemon Session Timeout 从 PocketBase 迁移到 PG

仓库：`k8s-chrome-daemon`

目标：

- daemon 不再依赖 PocketBase 维护 session timeout。
- timeout 真值由 PG / 南京管理面维护。
- daemon 使用本地 cache 控制 timeout controller。

已实现：

- 新增 PG 表 `session_timeout_configs`。
- API key 不直接存明文，存 hash 后结果。
- 增删改查内部 HTTP 接口：

```text
GET    /internal/session-timeout/configs
POST   /internal/session-timeout/configs
PUT    /internal/session-timeout/configs
DELETE /internal/session-timeout/configs
POST   /internal/session-timeout/refresh
```

- 内部接口用 `X-Internal-Token` / bearer token 鉴权。
- token compare 使用 Go `subtle.ConstantTimeCompare`。
- 增加 PG `LISTEN/NOTIFY`，多 daemon 实例收到变更后刷新 cache。
- 保留周期性 full reload，作为事件丢失兜底。

### 4. 北京 Timeout 真值读取南京代理

用户明确后的最终约束：

- timeout 数据本身不和 `region_id` 绑定。
- timeout 真值在南京侧。
- 北京 daemon 在 timeout 场景下不直接读 / 写本地 PG。
- 北京 daemon 通过南京代理服务读取真值。
- 南京修改 timeout 后，触发北京 daemon refresh；北京 refresh 时仍从南京代理拉取真值。

已实现：

- daemon 支持远程 timeout 配置源：

```text
REMOTE_SESSION_TIMEOUT_CONFIG_URL
REMOTE_SESSION_TIMEOUT_CONFIG_TOKEN
```

- 如果配置了 remote URL：
  - daemon 不创建 / 使用本地 timeout PG 表作为真值。
  - daemon 不启动 PG `LISTEN/NOTIFY`。
  - `ListConfigs` 和 `RefreshProject` 从远程 URL 拉取。
  - `UpsertConfig` / `DeleteConfig` 变成只读错误。
- 北京 office daemon 配置：

```text
REMOTE_SESSION_TIMEOUT_CONFIG_URL=http://10.3.3.176:30924/v1/session-timeout/configs
REMOTE_SESSION_TIMEOUT_CONFIG_TOKEN=lexmount-region-data-plane-internal
```

- 南京 office daemon 保持本地 PG 真值。
- 默认 placeholder 使用明确字样：

```text
REMOTE_SESSION_TIMEOUT_CONFIG_URL=PLACEHOLDER_DISABLED_REMOTE_SESSION_TIMEOUT_CONFIG_URL
REMOTE_SESSION_TIMEOUT_CONFIG_TOKEN=PLACEHOLDER_OPTIONAL_REMOTE_SESSION_TIMEOUT_CONFIG_TOKEN
```

- daemon 识别 `PLACEHOLDER_DISABLED_REMOTE_SESSION_TIMEOUT_CONFIG_URL` 为未启用 remote source。

### 5. Manifests / Office 部署结构

仓库：`lexmount-k8s-manifests`

当前 office 结构：

```text
apps/clusters/office-nanjing
apps/clusters/office-beijing
```

主要结构：

- `office-nanjing`
  - 部署 `lexhome`
  - 部署 `session-gateway`
  - 部署 `region-data-plane-gateway`
  - 保留 `region-registry-configmap.yaml`
- `office-beijing`
  - 部署 `region-data-plane-gateway`
  - 不部署 `lexhome`
  - 不部署 `session-gateway`
  - 已删除无用 `region-registry-configmap.yaml`

NodePort：

- 北京 `region-data-plane-gateway`: `30923`
- 南京 `region-data-plane-gateway`: `30924`

NFS / PVC：

- 已处理静态 PVC 不能扩容的问题。
- 新 PV/PVC 使用 `browser-user-data-pvc-v2`，容量 `2048Gi`。
- 老 PVC 若无引用，可手动删除。

### 6. Region Registry 去重

之前重复：

- `apps/clusters/office-nanjing/region-registry-configmap.yaml`
- `apps/clusters/office-nanjing/session-gateway-config-patch.yaml`

两处都写了 region 列表，容易产生 split-brain。

已处理：

- 删除 `session-gateway-config-patch.yaml`。
- 保留 `region-registry-configmap.yaml` 为唯一 source of truth。
- 用 kustomize replacement 注入：

```text
region-registry.data.local-region-id -> session-gateway-config.data.LOCAL_REGION_ID
region-registry.data.regions.json    -> session-gateway-config.data.REGION_REGISTRY_JSON
```

### 7. Lexhome `BROWSER_BASE_URL` 环境区分

最终约束：

- qcloud / qcloud-hk / guoge 仍指向原 browser-manager。
- office-nanjing 指向 session-gateway，用于模拟多 region 官网聚合。

当前实现：

- `apps/lex-home/lex-home-configmap.yaml` base 保持：

```text
BROWSER_BASE_URL=http://browser-manager.system.svc.cluster.local:9222/
```

- `office-nanjing` 单独用 Deployment env patch 覆盖：

```text
BROWSER_BASE_URL=http://session-gateway.system.svc.cluster.local:9231/
```

- patch 只影响 `BROWSER_BASE_URL` 一个环境变量。
- 不再整段覆盖 `/app/.env`。
- `DATABASE_URL`、`BETTER_AUTH_SECRET`、OAuth / SMTP / Storage 等继续由 `lex-home-secret` 管理。

## 当前镜像

### Gateway

```text
code.lexmount.net/wangli/region-data-plane-gateway:dcbe4c1-20260430-181713
lexmount.tencentcloudcr.com/cloud/region-data-plane-gateway:dcbe4c1-20260430-181713
code.lexmount.net/wangli/session-gateway:dcbe4c1-20260430-181713
lexmount.tencentcloudcr.com/cloud/session-gateway:dcbe4c1-20260430-181713
```

### Browser Operator / Daemon

```text
code.lexmount.net/wangli/browser-operator:97b8a98-20260430-164548
lexmount.tencentcloudcr.com/cloud/browser-operator:97b8a98-20260430-164548
```

说明：

- 没有推送 HK 镜像。
- HK 只补 tag，HK 镜像由负责人后续推送。

## Office 当前验证结果

已在 office-nanjing / office-beijing apply 并验证：

- `region-data-plane-gateway` rollout 通过。
- `session-gateway` rollout 通过。
- `browser-operator-controller-manager` rollout 通过。
- `lex-home` rollout 通过。

关键 smoke：

1. 南京 `session-gateway` 查询 session：
   - `POST /instance/v2/sessions`
   - 返回 `regions=[office-nanjing, office-beijing]`

2. 创建 session 路由：
   - 使用 invalid `browser_mode` 验证路由，不创建真实实例。
   - `POST /instance` 不再 404。
   - 默认路由到 `office-nanjing`，返回 browser-manager 的 400。
   - body 指定 `region_id=office-beijing` 时，路由到北京，返回 browser-manager 的 400。

3. Lexhome 创建路径：
   - office-nanjing Pod 中 `process.env.BROWSER_BASE_URL` 是 session-gateway。
   - `/app/.env` 仍保持 browser-manager base URL。
   - lexhome 代码读取 `process.env.BROWSER_BASE_URL`。
   - 从 lexhome Pod 请求 `POST /instance` 不再 404。

4. Timeout remote truth：
   - 南京写 target `office-beijing` timeout。
   - 北京 daemon 通过南京 `region-data-plane-gateway` remote URL 读取到配置。
   - 南京删除后，北京 daemon list 为空。

5. Body limit：
   - 超过 1MiB JSON body 返回 `413 {"error":"request body too large"}`。

6. Region data-plane readyz：
   - `/readyz` 不再泄露 browser-manager 内部 URL。

## 当前已知 review / 风险点

### 1. `session-gateway` 入站鉴权尚未启用

- 当前没有强制要求 lexhome -> session-gateway 携带内部 token。
- 原因是 lexhome 当前请求 `BROWSER_BASE_URL` 没有统一注入 internal token header。
- 如果后续 review 要求必须鉴权，需要新增 lexhome 请求层配置，例如：

```text
BROWSER_BASE_HEADERS_JSON={"X-Internal-Token":"..."}
```

然后 session-gateway 才能强制校验。

### 2. Office secret 仍有明文问题

自动 review 指出 office runtime secret / lexhome secret 有明文提交问题。

当前本轮未彻底处理，因为：

- office 环境本身已有多处明文 secret。
- 改为 external-secrets / sealed-secrets 需要单独设计。

后续合入前需要决定：

- office 是否接受明文模拟配置。
- 还是必须改为 out-of-band secret。

### 3. 跨集群 HTTP 明文链路

当前 office 模拟使用：

```text
http://10.3.217.198:30923
http://10.3.3.176:30924
```

风险：

- token 通过明文 HTTP 传输。
- office 内网模拟可以临时接受，但正式环境需要 mTLS / VPN / HTTPS / service mesh 或其它安全链路。

### 4. `BROWSER_BASE_URL` 环境变量覆盖依赖应用读取优先级

当前 office-nanjing 使用 Deployment env 覆盖 `BROWSER_BASE_URL`。

验证结果：

- Pod env 是 session-gateway。
- `/app/.env` 仍是 browser-manager。
- 当前 lexhome 代码读取 `process.env.BROWSER_BASE_URL`，所以生效。

后续如果 lexhome 启动逻辑改成强制用 `.env` 覆盖 process env，需要再次验证。

### 5. 创建 session 默认 region

当前 `POST /instance` 未传 `region_id` 时，默认进入 `LOCAL_REGION_ID`，也就是 office-nanjing。

后续如果官网需要用户可选 region，需要：

- lexhome UI / action 增加 `region_id` 选择。
- create session 请求 body 携带 `region_id`。
- context / extension 也要按 region 过滤，不能跨 region 使用。

### 6. Context / Extension / Download 跨 region 尚未完整改造

目前明确约束：

- context / download / extension 数据不能跨 region 共享。

本轮主要打通 session / timeout 链路。

后续仍需补：

- lexhome context 列表按 region 查询。
- extension 列表按 region 查询。
- 创建 session 时只能选择同 region context / extension。
- download 按 session owner region 查询。

## 后续建议顺序

1. 完成三个 PR review，先解决阻塞性安全 / 架构 comment。
2. 明确 `session-gateway` 入站鉴权策略：是否先改 lexhome header 支持。
3. 决定 office secret 明文是否接受，或单独做 external secret 方案。
4. 对 create session 增加官网 region selector。
5. 补 context / extension / download 的 region 数据面。
6. 梳理正式 qcloud / qcloud-hk 是否引入 session-gateway，以及引入时的差异配置。
7. 合入前重新跑 office-nanjing / office-beijing 全量 smoke。

## 当前不要合入的原因

- PR comments 还没有完全 review 完。
- 部分安全问题需要产品 / 架构取舍：尤其是 `session-gateway` 入站鉴权和 secret 管理。
- context / extension / download 还没有完整 region 化。
- 当前 office 是模拟环境，正式 qcloud 的切换策略还需要单独确认。
