# Setaria report_usage 内部接口

## 背景

SetariaGateway 对外承接 `POST /v1/extract`，并把请求转发到 Setaria 内部服务的 `/internal/v1/extract`。

Setaria 内部服务完成一次 webfetch / extract 后，可以回调 SetariaGateway 的内部接口上报项目维度用量。SetariaGateway 再把这次上报转发给 `user-project-plan-service`。

当前链路：

```text
Setaria
  -> setaria-gateway /internal/setaria/report_usage
  -> user-project-plan-service /internal/user-project-plans/projects/:projectId/webfetch-usage
```

## 接口

```text
POST /internal/setaria/report_usage
Content-Type: application/json
```

集群内地址：

```text
http://setaria-gateway.system.svc.cluster.local:9250/internal/setaria/report_usage
```

这个接口只给 Setaria 内部服务调用，不是公网 API。

## 鉴权

请求必须携带内部 token。两种 header 均可：

```http
Authorization: Bearer <SETARIA_GATEWAY_INTERNAL_TOKEN>
```

或：

```http
x-internal-token: <SETARIA_GATEWAY_INTERNAL_TOKEN>
```

SetariaGateway 侧 token 配置来源：

```text
SETARIA_GATEWAY_INTERNAL_TOKEN
```

如果没有配置，会 fallback 到：

```text
REGION_DATA_PLANE_TOKEN
```

## 请求体

```json
{
  "project_id": "<project-id>",
  "request_id": "<request-id>",
  "usage_ms": 1200,
  "reason": "setaria_webfetch"
}
```

字段说明：

| 字段 | 必填 | 说明 |
| --- | --- | --- |
| `project_id` | 是 | 项目 ID；兼容 `projectId`。 |
| `request_id` | 建议必填 | 本次 webfetch / extract 请求的唯一 ID；兼容 `requestId`；未传时 fallback 到 `session_id`，再没有则由 gateway 生成 `setaria_<uuid>`。 |
| `usage_ms` | 否 | 本次使用时长，单位毫秒；兼容 `usageMs`；默认取 `SETARIA_GATEWAY_USAGE_MS_DEFAULT`，再默认 `0`。 |
| `reason` | 否 | 用量原因；默认 `setaria_webfetch`。 |

## 成功响应

SetariaGateway 会透传 `user-project-plan-service` 的响应。当前 `user-project-plan-service` 返回当前 project plan：

```json
{
  "data": {
    "project_id": "<project-id>",
    "plan": "...",
    "usage": {},
    "features": {}
  }
}
```

具体字段以 `getProjectPlan(projectId)` 的返回为准。

## 错误响应

| HTTP 状态码 | 响应 | 含义 |
| --- | --- | --- |
| `400` | `{"error":"project_id is required"}` | 缺少 `project_id` / `projectId`。 |
| `401` | `{"error":"unauthorized"}` | 内部 token 缺失或不匹配。 |
| `503` | `{"error":"SETARIA_GATEWAY_INTERNAL_TOKEN is not configured"}` | SetariaGateway 未配置内部 token。 |
| `503` | `{"error":"USER_PROJECT_PLAN_TOKEN is not configured"}` | SetariaGateway 无法调用 `user-project-plan-service`。 |
| `502` / `504` | `{"error":"report_usage_upstream_failed"}` | 调用 `user-project-plan-service` 失败或超时。 |

## curl 示例

```bash
curl -sS -X POST 'http://setaria-gateway.system.svc.cluster.local:9250/internal/setaria/report_usage' \
  -H 'Content-Type: application/json' \
  -H "Authorization: Bearer ${SETARIA_GATEWAY_INTERNAL_TOKEN}" \
  -d '{
    "project_id": "<project-id>",
    "request_id": "<request-id>",
    "usage_ms": 1200,
    "reason": "setaria_webfetch"
  }'
```

## 当前实现注意点

SetariaGateway 当前已经把 `request_id`、`usage_ms`、`reason` 转发给 `user-project-plan-service`。

但 `user-project-plan-service` 的 `recordProjectWebfetchUsage` 当前只校验 `request_id` 并返回 project plan，还没有像 browser usage 那样写 ledger 表。后续如果要真正扣 webfetch 配额，需要在 `user-project-plan-service` 里补持久化和扣量逻辑。

