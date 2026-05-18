# 2026-05-18 Project ID Inspector mirror 查询能力总结

## 背景

原始需求是仿造 `project-id-inspector` 新增一个 `session-inspector`，通过 `session_id` 查询 `region_mirror_sessions`，并只在 `qcloud-nanjing`、`office-nanjing`、`qcloud-hk` 发布。

后续方案调整为不再新增独立服务，而是把 session、context、extension 的 mirror 查询能力集中到现有 `project-id-inspector` 页面中，前端增加输入框，复用同一个页面工具和部署链路。

## 任务目标

- 保留原有 `project_id` 查询用户详细信息的能力。
- 在同一个页面中支持通过 `session_id`、`context_id`、`extension_id` 查询 mirror 表。
- 后端查询 `region_mirror_sessions`、`region_mirror_contexts`、`region_mirror_extensions`。
- 部署范围限制在 `office-nanjing`、`qcloud-nanjing`、`qcloud-hk`。
- manifests 复用现有 `browser-manager-secret` 连接 browser-manager PostgreSQL，不新增只读账号。

## 涉及仓库

- `demo-nodejs-backend`
- `lexmount-k8s-manifests`

## 关键改动

### project-id-inspector

- 前端从单一 `project_id` 查询扩展为四类查询：
  - `project_id`
  - `session_id`
  - `context_id`
  - `extension_id`
- 原有 `project_id` 查询链路不变：
  - 通过 Kong consumer 查 API key / project 关联信息。
  - 继续查询 SaaS 用户信息。
- 新增 mirror 查询接口：
  - `GET /api/mirror/session/:id`
  - `GET /api/mirror/context/:id`
  - `GET /api/mirror/extension/:id`
- 后端按类型查询对应 mirror 表：
  - `region_mirror_sessions`
  - `region_mirror_contexts`
  - `region_mirror_extensions`
- mirror kind 使用 allowlist，并用 `Object.hasOwn(mirrorQueries, kind)` 做 own-property 检查，避免 `__proto__` 这类路径绕过。
- `project-id-inspector/Makefile` 增加 qcloud registry push，方便同一 tag 同步到 qcloud-nanjing 使用。

### manifests

- `apps/project-id-inspector/deployment.yaml` 增加 PostgreSQL 连接环境变量，来源复用 `browser-manager-secret`：
  - `POSTGRES_HOST`
  - `POSTGRES_PORT`
  - `POSTGRES_DB`
  - `POSTGRES_USER`
  - `POSTGRES_PASSWORD`
- 只更新目标环境的 `project-id-inspector-image`：
  - `apps/clusters/office-nanjing/images-configmap.yaml`
  - `apps/clusters/qcloud-nanjing/images-configmap.yaml`
  - `apps/clusters/qcloud-hk/images-configmap.yaml`

## 镜像与发布状态

最终镜像 tag：

- `project-id-inspector:81557fc-20260515-173511`

各环境 manifests 配置：

- `office-nanjing`: `code.lexmount.net/wangli/project-id-inspector:81557fc-20260515-173511`
- `qcloud-nanjing`: `lexmount.tencentcloudcr.com/cloud/project-id-inspector:81557fc-20260515-173511`
- `qcloud-hk`: `lexmoun-tcr-hk.tencentcloudcr.com/cloud/project-id-inspector:81557fc-20260515-173511`

实际 registry 检查结论：

- qcloud-nanjing tag 已存在，`docker manifest inspect` 通过。
- qcloud-hk manifests 已改为新 tag，但 HK registry 当时还没有该 tag，`docker manifest inspect` 返回 `no such manifest`。
- Alice 没有主动推送 HK 镜像；发布 qcloud-hk 前需要走正常 HK 镜像同步流程。

office-nanjing 已发布：

- 当前运行镜像：`code.lexmount.net/wangli/project-id-inspector:81557fc-20260515-173511`
- `deploy/project-id-inspector` rollout 成功。
- 当前 ready 状态为 `1/1`。
- Pod 内访问 `http://127.0.0.1:3000/` 返回 200。

## 验证方式

代码侧：

- `node --check project-id-inspector/project-id-inspector.js`
- `make -C project-id-inspector push`

manifests 侧：

- `kubectl kustomize apps/clusters/office-nanjing`
- `kubectl kustomize apps/clusters/qcloud-nanjing`
- `kubectl kustomize apps/clusters/qcloud-hk`

office 发布验证：

- `kubectl apply -k apps/clusters/office-nanjing`
- `kubectl rollout status deploy/project-id-inspector -n system`
- 校验 deployment 镜像为 `81557fc-20260515-173511`
- Pod 内 HTTP 访问首页返回 200。

## 分支与 PR

- demo-nodejs-backend PR: https://github.com/lexmount/demo-nodejs-backend/pull/174
- lexmount-k8s-manifests PR: https://github.com/lexmount/lexmount-k8s-manifests/pull/470
- demo-nodejs-backend merge commit: `f50ff1e`
- lexmount-k8s-manifests merge commit: `5be1561`

## 发布注意事项

- 这个能力依赖 browser-manager 的 mirror 表，因此 `project-id-inspector` 必须能访问 browser-manager PostgreSQL。
- 当前按要求复用 `browser-manager-secret`。如果后续要收敛权限，可以单独新增只读 DB user/secret，但本轮没有做。
- qcloud-hk 发布前必须确认 `lexmoun-tcr-hk.tencentcloudcr.com/cloud/project-id-inspector:81557fc-20260515-173511` 已同步到 HK registry。
- 发布范围是 `office-nanjing`、`qcloud-nanjing`、`qcloud-hk`，不包含 `office-beijing` 和 `qcloud-beijing`。

## 结论

- `session-inspector` 没有作为独立服务落地，能力已经并入 `project-id-inspector`。
- `project-id-inspector` 现在可以同时查询 project 用户信息，以及 session/context/extension 的 region mirror 信息。
- 两个 PR 已合入，office-nanjing 已发布验证。
- qcloud-nanjing manifests 和 registry tag 都已就绪；qcloud-hk manifests 已就绪，但发布前仍需要确认 HK registry 镜像同步。
