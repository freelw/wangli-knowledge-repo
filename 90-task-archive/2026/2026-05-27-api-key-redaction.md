# 2026-05-27 API Key 脱敏改造

## 背景

本轮工作目标是收敛 Lexmount 云平台中 `api_key` 的存储、传输、日志暴露面，避免明文 key 在数据库、URL、接口返回、Kong 日志、应用日志中继续出现。

核心原则：

- `api_key` 不允许作为长期数据存储，只能存 hash。
- `api_key` 不通过普通业务接口返回，只有官网 API Key 页面可以在创建/查看场景按产品语义获取。
- 新链路必须做到“直接不落明文”，不能先写明文再异步清理。
- 历史数据通过补偿机制持续清理，直到旧明文列全部置空；列可以保留，但值要清空。

## 范围

涉及仓库：

- `demo-nodejs-backend`
- `lexmount-k8s-manifests`
- `lexmount-js-sdk-quickstart`
- `lexmount-python-sdk-quickstart`
- `lex-home`
- `k8s-chrome-daemon`

重点关注场景：

- SDK 创建 session、查询 session、下载文件。
- session-gateway / region-data-plane-gateway 的跨 region 转发。
- browser-manager / browser-operator 对 key 的存储、查询、鉴权。
- Kong `key-auth`、自定义 project id 校验、access log 暴露面。
- 数据库中历史明文 key 的自动补偿清理。

## 设计决策

### 只保留 hash，不保留明文

服务端新增统一 api-key hash helper，业务查询、鉴权、关联关系都基于 hash，不再把明文 key 当作业务字段长期保存。

旧明文列允许继续存在，目的是降低迁移风险和避免复杂 schema 变更；但值会被补偿机制逐步清空。

### 补偿机制自动收敛

补偿机制负责不断扫描旧数据，把仍然存在的明文 key 转换为 hash 后清空明文列。

要求：

- 默认开启。
- 可以通过开关关闭，便于完全迁移后停用。
- 使用 `setTimeout` 串行调度，避免上一轮还未结束时下一轮重复进入。
- 补偿失败不应破坏主链路，但必须记录可观测日志。

### SDK 和 gateway 改为 header 传输

新的 API key 传输方式统一走 header：

- `X-API-Key`
- `X-Project-ID`

不再通过 query/body 传输 `api_key`。尤其是 downloads GET 接口不能继续把 `api_key` 放在 query 中，因为 URL 容易进入：

- 浏览器历史。
- 代理和网关 access log。
- Kong 日志。
- 监控采样。
- 用户复制链接和错误日志。

### 保留 session pod 的 `hashed_api_key` label

session pod 上的 `hashed_api_key` label 保留。

它的作用是作为脱敏后的 key 标识，用于排查、过滤、统计和 session 关联；它不是鉴权凭据，也不是明文 key。

## 实现内容

### 后端和数据层

- 增加统一 api-key hash helper。
- browser-manager / browser-operator 改为基于 hash 读写 key 关联数据。
- 增加 api-key redaction compensator。
- compensator 支持默认开启和后续关闭开关。
- compensator 调度改为 `setTimeout` 串行执行。
- 保证新写入路径不再先落明文。

### SDK 与 E2E

- JS SDK 依赖升级并验证 header 传输。
- Python SDK quickstart 依赖升级。
- lexmount-e2e-tool 升级到新 JS SDK，并覆盖关键回归用例。

### Gateway 与 Kong

- session-gateway / region-data-plane-gateway 适配 header 传输。
- downloads 路径移除 query `api_key`。
- Kong `key-auth` 增加 header key name 支持：`x-api-key`。
- `api-browser-instance` 路由的 project id 校验兼容 header 场景，不再强依赖 body/query 中的 `project_id`。
- CORS allow headers 统一加入新 header。
- Kong log 配置检查方向明确：不能继续记录 query 中的 `api_key`。

## PR

### backend

- https://github.com/lexmount/demo-nodejs-backend/pull/203

### manifests

- https://github.com/lexmount/lexmount-k8s-manifests/pull/555

### quickstart

- https://github.com/lexmount/lexmount-js-sdk-quickstart/pull/12
- https://github.com/lexmount/lexmount-python-sdk-quickstart/pull/28

## 镜像

最终发布和验证使用的镜像：

```text
browser-manager:24e6e3f-20260527-175029
browser-operator:43eab7a-20260527-155313
session-gateway:893a34f-20260527-180620
region-data-plane-gateway:24e6e3f-20260527-175138
lexmount-e2e-tool:29e0b9a-20260527-175644
kong-init:43da18a-20260527-182119
```

说明：`lex-home` 镜像由人工单独更新，不在本次 manifests 自动更新范围内。

## office 发布

已发布组件：

- office-nanjing：
  - browser-manager
  - session-gateway
  - region-data-plane-gateway
  - lexmount-e2e-tool
  - kong-init
- office-beijing：
  - browser-manager
  - region-data-plane-gateway
  - kong-init

两个 office 集群均已执行 `kong-init` 脚本，确认 Kong 配置生效。

## 验证结果

### 直接 SDK 验证

在 office-nanjing 环境通过 JS SDK 直接创建并关闭 session：

```text
created session_1779877682237_1rldfv5sg office-nanjing
closed
```

### E2E 验证

lexmount-e2e-tool 选择用例 7/7 通过：

- `catalog-info`
- `region-probe-info`
- `session-downloads`
- `region-sdk-session`
- `session-gateway-apis`
- `session-gateway-capacity-create`
- `session-gateway-list-dedupe`

### Kong 配置验证

已确认：

- `api-browser-instance` 的 `key-auth` 支持 `x-api-key` header。
- `api-browser-instance` 的 `projectid-consumer-check.allow_missing_projectid=true`，用于兼容 project id 从 header 传入。
- wss 创建路由仍保留更严格的 project id 校验策略。

## 后续工作

1. 观察 compensator 日志，确认旧明文 key 持续被清空。
2. 抽查数据库：旧列可以存在，但值应逐步为空；hash 字段应完整。
3. 复扫接口返回、Kong access log、应用日志、pod env、ConfigMap、DB 表，确认没有明文 `api_key`。
4. lex-home 镜像更新后，验证官网 API Key 页面仍符合产品语义。
5. office 稳定后，再推进 qcloud-nanjing / qcloud-hk / qcloud-beijing 发布。
6. 数据完全迁移并稳定后，关闭 compensator 开关，并保留关闭前审计记录。

## 注意事项

- `hashed_api_key` 可以作为脱敏标识保留，但不能作为可逆凭据使用。
- query 传 api_key 的路径必须视为高风险，不应新增。
- 如果后续出现认证失败，优先检查 Kong 是否已经执行最新 `kong-init`。
- 如果出现 `Missing project id`，优先检查 SDK 是否发送 `X-Project-ID`，以及 Kong 自定义 project id 校验是否兼容 header 场景。
