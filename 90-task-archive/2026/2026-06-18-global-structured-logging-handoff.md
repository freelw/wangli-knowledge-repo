# 2026-06-18 全局结构化日志第一阶段交接

## 背景

云平台排障越来越依赖腾讯云 CLS 和跨服务日志串联。当前 `browser-manager`、`session-gateway`、`browser-ws-gateway`、`region-data-plane-gateway`、`k8s-chrome-daemon`、`qcloud-spot-capacity-controller`、`proxypool` 等服务日志格式不统一，主要问题是：

- Node 服务混用 `console.log`、winston 文本日志、`event|k=v` 字符串和手写 JSON 片段。
- Go 服务混用 controller-runtime zap、`log.Printf`、`slog.TextHandler`。
- `session_id`、`project_id`、`region_id`、`event`、`result`、`error_code` 等字段没有统一命名。
- 排查单个 session 时，需要跨多个服务手工拼日志，难以稳定通过 CLS 字段检索或 SQL 聚合。
- 部分错误日志存在输出上游原始 body / object 的风险，不适合长期作为统一排障日志。

本次任务的结论是：不要继续新增分隔符格式日志，应统一服务端 stdout 为单行 JSON Lines，并把常查字段提升为 top-level key；`detail` 只用于低频扩展字段。

## 任务目标

第一阶段目标是做一个可 review、可逐步扩展的最小闭环：

1. 定义全局日志 schema 和字段规范。
2. 在 Node 服务中提供无依赖结构化 stdout logger。
3. 先迁移 `browser-manager` 和 `session-gateway` 的 create session 主链路关键事件。
4. 修复 review 中指出的日志安全和稳定性问题。
5. 构建 `browser-manager` / `session-gateway` 镜像，并更新 manifests。

本阶段不追求一次性覆盖所有服务，避免 PR 过大、review 成本过高。

## 涉及仓库

- `demo-nodejs-backend`
- `lexmount-k8s-manifests`
- `knowledge-repo`

## 设计决策

统一格式为单行 JSON Lines：

```json
{
  "ts": "2026-06-18T11:40:00.123Z",
  "level": "info",
  "schema": "lex_log_schema_v1",
  "service": "browser-manager",
  "region_id": "qcloud-nanjing",
  "session_id": "session_xxx",
  "project_id": "project_xxx",
  "event": "session.create.request",
  "result": "success",
  "error_code": "",
  "status_code": 200,
  "latency_ms": 123,
  "detail": {
    "browser_mode": "normal"
  }
}
```

字段原则：

- `ts`、`level`、`schema`、`service`、`event` 由 logger 控制，调用方不能覆盖。
- `session_id`、`project_id`、`region_id`、`request_id`、`user_id` 等常查字段应放 top-level。
- `detail` 只放低频扩展字段，不放需要经常 SQL 查询或告警的字段。
- `api_key`、token、password、完整上游响应 body 不允许进入日志。
- `event` 使用点分层命名，例如 `session.create.request`、`session.create.route.response`、`region.capacity.schedule`。

## 代码改动

### demo-nodejs-backend PR

PR：

```text
https://github.com/lexmount/demo-nodejs-backend/pull/249
```

分支：

```text
wangli_dev_20260618_log_schema
```

最新关键提交：

```text
1498966
```

主要改动：

- 新增 `docs/logging-schema.md`，定义 `lex_log_schema_v1`、字段规范、event 命名、`detail` 边界和脱敏规则。
- 新增 `browser-manager/utils/structured-log.js`，提供无依赖结构化 stdout logger。
- `browser-manager/utils/logger.js` 增加 `eventInfo`、`eventWarn`、`eventError`，保持原 winston logger 兼容。
- `browser-manager` 迁移 `/instance`、`/instance/v2` create session 主链路关键事件。
- `session-gateway` 迁移 create session 路由选择链路关键事件。
- Dockerfile 补充复制新增 logger 文件。

补充覆盖的 create 链路日志：

- `session-gateway` 的 `region.capacity.schedule`：single region、active session fallback、registry fallback、active_sessions_per_total_cpu。
- `browser-manager` v1 `/instance`：request start、createSession 返回错误、成功 response、顶层 catch。
- `browser-manager` v2 `/instance/v2`：request start、reserve 失败、reserved、responded、async start、extension mount、context weak unlock、context lock、finalize、done、unhandled。
- `reserveCreatingSessionSlot`：plan concurrency 加载失败、global quota 拒绝、DB reserve 插入失败、本地 project quota 拒绝。
- `finalizeSessionCreationFromReserved`：bind context 失败。
- `assertBrowserInstance`：缺 containerIp / port / containerId / ws URL 时输出结构化错误，不再 `console.error` 整个 BrowserInstance。

## Review 修复

最终提交 `1498966` 处理了三类 review 问题：

1. 调用方字段不能覆盖 logger 核心字段。
   - `ts`、`level`、`schema`、`service`、`event` 等始终由 logger 控制。
   - 避免业务调用误传同名字段导致日志语义被篡改。

2. `JSON.stringify` 失败不能导致进程崩溃。
   - 对循环引用或不可序列化 detail 做 fallback。
   - fallback 事件为 `log.serialize.error`。

3. `session-gateway` 不再记录 raw upstream error body。
   - 只保留安全摘要字段：`status_code`、`code`、`message`、`region_id`。
   - 避免上游 body 中潜在敏感字段进入统一日志。

## 镜像与 manifests

manifests PR：

```text
https://github.com/lexmount/lexmount-k8s-manifests/pull/677
```

分支：

```text
wangli_dev_20260618_log_schema_images
```

最新关键提交：

```text
7779a1e
```

最终镜像：

```text
browser-manager:1498966-20260618-194017
digest: sha256:4fbf6e6794040a0835361aff7f91a0f5649110cea03460b5ca7b3c0cbf17d68f

session-gateway:1498966-20260618-194132
digest: sha256:98519eeb372e72d2875acb717923234b115ffd16e0b3645b6e5cfa574319b333
```

镜像推送范围：

- 已推送：office/code registry。
- 已推送：qcloud-nanjing registry。
- 已推送：qcloud-beijing registry。
- qcloud-hk：只同步 manifests tag，不推送 HK registry。

manifests 改动：

- 更新各 app cluster 中已有的 `browser-manager-image`。
- 更新各 app cluster 中已有的 `session-gateway-image`。
- 保持各环境 registry prefix 不变，只替换 tag。
- qcloud-hk 只做 tag 对齐。

## 验证结果

代码验证：

```bash
node --check browser-manager/utils/structured-log.js
node --check browser-manager/utils/logger.js
node --check browser-manager/utils/instance-helpers.js
node --check browser-manager/routes/instance.js
node --check session-gateway/server.js
```

logger 行为验证：

- 直接执行结构化 logger 输出样例，确认每条日志是一行 JSON。
- 验证 reserved field override 不生效。
- 验证循环引用 detail 不会导致进程崩溃，会降级输出 `log.serialize.error`。

manifests 验证：

```bash
kubectl kustomize apps/clusters/office-nanjing
kubectl kustomize apps/clusters/office-beijing
kubectl kustomize apps/clusters/qcloud-nanjing
kubectl kustomize apps/clusters/qcloud-beijing
kubectl kustomize apps/clusters/qcloud-hk
kubectl kustomize apps/clusters/guoge
kubectl kustomize apps/clusters/office-private
git diff --check
```

镜像验证：

```bash
docker manifest inspect <code/browser-manager image>
docker manifest inspect <qcloud-nanjing/browser-manager image>
docker manifest inspect <qcloud-beijing/browser-manager image>
docker manifest inspect <code/session-gateway image>
docker manifest inspect <qcloud-nanjing/session-gateway image>
docker manifest inspect <qcloud-beijing/session-gateway image>
```

已知非本次问题：

- `office-catl-private` kustomize 仍失败，错误与 `/metadata/namespace` replacement 有关。
- 该问题已在当前 `origin/main` 复现，不是 PR #677 引入。

## 当前交付状态

- 代码 PR #249 已完成第一阶段实现和 comments 修复。
- manifests PR #677 已更新到最新镜像 tag。
- HK 遵循团队要求：只同步 tag，不主动 push HK registry。
- 本次工作已补入 Notion `browser 云平台专项信息`：
  - `1.3 服务端开发 / 统一日志 / Trace / 审计`
  - 页面：`https://app.notion.com/p/383e7e998fb281648873e37bd2bad205`

## 未覆盖范围

本阶段刻意没有覆盖所有日志点。后续接手时，不要误判为遗漏发布。

明确未覆盖：

- `session-gateway` 非 create 链路：session list / get / delete、json debug route、downloads route、startup logs。
- `browser-manager` 非 create 链路：session list / get / close、context / extension 相关非主链路。
- `browser-ws-gateway`。
- `region-data-plane-gateway`。
- `k8s-chrome-daemon`。
- `qcloud-spot-capacity-controller`。
- `spot-termination-monitor`。
- `proxypool`。
- `session-quota`。
- `user-project-plan-service`。
- `lex-home` 登录、支付、审计相关链路。
- Kong access log JSON 化。

## 后续建议

### 第二阶段优先级

建议下一批优先处理：

1. `browser-ws-gateway`
   - 重点事件：frontend connect、CDP connect、disconnect、verify failed、upstream close、browser closed。
   - 关键字段：`session_id`、`project_id`、`context_id`、`connection_id`、`close_code`、`error_code`。

2. `region-data-plane-gateway`
   - 重点事件：proxy start / response / error。
   - 关键字段：`source_region_id`、`target_region_id`、`path`、`status_code`、`latency_ms`、`session_id`、`project_id`。

3. `session-gateway` downloads / json debug route
   - 重点事件：downloads list / file / archive / delete、json / json/version / activate。
   - 这几条路径历史上出现过 404、跨 region 路由、DevTools 调试问题，排障价值高。

4. `k8s-chrome-daemon`
   - Go 侧需要统一 production JSON logger。
   - 重点事件：BrowserInstance reconcile、pod create/delete、pod ready、timeout、browser container exit。

### CLS 配置建议

结构化日志上线后，需要在 CLS 侧配合：

- 采集规则使用 JSON 格式。
- 对常查 top-level 字段开启键值索引和统计。
- 优先字段：`schema`、`service`、`event`、`level`、`result`、`region_id`、`session_id`、`project_id`、`request_id`、`status_code`、`error_code`。
- 查询模板按 `session_id`、`project_id`、`region_id`、`service` 沉淀到 runbook。

### 审计边界建议

统一日志不是审计系统本身。后续需要明确三类日志边界：

- 排障日志：服务运行状态、错误、耗时、路由选择。
- 业务审计：登录、资源变更、权限变更、敏感操作。
- 计费审计：订阅、支付、用量、账单、发票、退款。

不要把计费审计完全依赖普通 stdout 日志；stdout 可作为辅助排障入口，但最终需要可追溯的数据库审计表或事件流。

## 交接注意事项

- 继续改 logger 时，不要允许调用方覆盖核心字段。
- 新增日志点前先确认字段是否需要 top-level 查询；不要把常查字段塞进 `detail`。
- 不要记录 raw api_key、token、password、完整上游 body。
- JSON Lines 必须单行输出，不能把 stack 或 detail 打成多行。
- error 级结构化事件也走 stdout，避免 stdout / stderr 双通道导致采集规则分裂。
- HK 镜像发布继续按团队约定处理：未明确要求时不要主动 push HK registry。

## 分支与 PR

```text
demo-nodejs-backend
branch: wangli_dev_20260618_log_schema
PR: https://github.com/lexmount/demo-nodejs-backend/pull/249
latest commit: 1498966

lexmount-k8s-manifests
branch: wangli_dev_20260618_log_schema_images
PR: https://github.com/lexmount/lexmount-k8s-manifests/pull/677
latest commit: 7779a1e
```

## 结论

- 第一阶段已建立 `lex_log_schema_v1` 和 Node 结构化日志基础设施。
- `browser-manager` / `session-gateway` create session 主链路已经有样例级结构化事件。
- review 中的核心安全和稳定性问题已经修复。
- 镜像和 manifests 已更新，验证通过。
- 后续重点是把同一 schema 推广到 websocket、region proxy、daemon、spot、proxypool、计费审计等链路，并同步完善 CLS 索引和查询模板。
