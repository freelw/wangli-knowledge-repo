# 2026-05-08 Lexmount E2E Tool 工作总结

频道：`#e2e测试工具`

## 背景

本次工作的目标是建设一个类似 `browser-spider-bench` 的页面化 E2E 测试工具，用来集中跑 Lexmount SDK / quickstart 覆盖到的 demo。最终方向从“放在 quickstart 仓库里”调整为：

- E2E tool 放在 `demo-nodejs-backend` 仓库的独立目录 `lexmount-e2e-tool/`
- E2E tool 作为 Kubernetes 中的独立服务运行
- manifests 负责在指定环境上线服务、配置密钥、配置镜像和 Kong 路由
- qcloud / qcloud-hk 只准备配置，不由 Alice apply

## 相关 PR

- JS SDK async create / context lock：<https://github.com/lexmount/lexmount-js-sdk/pull/11>
- JS quickstart 依赖升级到 `0.5.3`：<https://github.com/lexmount/lexmount-js-sdk-quickstart/pull/8>
- E2E tool 服务实现：<https://github.com/lexmount/demo-nodejs-backend/pull/143>
- E2E tool manifests：<https://github.com/lexmount/lexmount-k8s-manifests/pull/404>

以上 PR 已在 2026-05-08 合入。

## SDK 与 quickstart 前置对齐

### JS SDK

JS SDK 先补齐 Python SDK 已有的 session create 能力：

- `sessions.create()` 默认走 `/instance/v2` async create
- 支持 polling 到 session `active`
- 支持 `asyncCreate:false` 回退旧 `/instance`
- 支持 `weakLock`
- 支持 `pollIntervalMs` / `pollTimeoutMs`

随后在 E2E 验证中发现 context lock 场景下，async create 会先返回 `session_id`，后续 polling 才返回 `create_failed`。旧 SDK 会抛普通 `APIError`，不会映射成 `ContextLockedError`。

最终修复为：

- `lexmount@0.5.3`
- async create polling 遇到 read-write context lock create failure 时抛 `ContextLockedError`
- `src/index.ts` 导出的 `VERSION` 同步为 `0.5.3`
- npm 已发布 `lexmount@0.5.3`

验证：

- `npm test`
- `npm run typecheck`
- `npm run lint`

### JS quickstart

JS quickstart 已升级依赖到：

```json
"lexmount": "^0.5.3"
```

同时补齐与 Python quickstart 对齐所需的 demo，并移除后续不再需要的 `new-page-repro`。

验证：

```bash
npx tsc --noEmit
```

## E2E Tool 服务设计

服务位置：

```text
demo-nodejs-backend/lexmount-e2e-tool/
```

服务形态：

- Node.js HTTP 服务
- 运行在 K8s 中
- 提供 HTML 页面
- 提供任务 API
- 串行执行选中的 demo
- 在页面展示进度、日志和结果
- 不挂载存储
- 不写截图、链接文件、summary JSON 等中间产物
- task state 只保存在内存中

关键环境变量：

- `PORT`
- `LEXMOUNT_BASE_URL`
- `LEXMOUNT_API_KEY`
- `LEXMOUNT_PROJECT_ID`
- `LEXMOUNT_REGION`
- `E2E_TOOL_TARGET_URL`

`E2E_TOOL_TARGET_URL` 当前默认值：

```text
https://www.bilibili.com/
```

不再使用：

- `E2E_TOOL_OUTPUT_DIR`
- volumeMount
- emptyDir
- `/app/runs`

## 当前 Demo 集合

当前 E2E tool 保留 11 个 demo，并且全部纳入默认选中：

- `catalog-info`
- `basic-demo`
- `light-demo`
- `session-list`
- `context-basic`
- `context-list-get`
- `context-lock-handling`
- `inspect-url`
- `session-targets`
- `connection`
- `extension-list-get`

已移除：

- `new-page`
- `Context Modes`
- `Proxy`
- `Extension Basic`

移除原因：

- `new_page_repro` 不再作为 quickstart / E2E 覆盖范围
- Context Modes / Proxy / Extension Basic 暂不放入 E2E tool 当前页面

默认选中验证：

```text
total=11
defaultSelected=11
notDefault=[]
```

## 镜像

E2E tool 最终镜像：

```text
code.lexmount.net/wangli/lexmount-e2e-tool:0418abd-20260508-212550
lexmount.tencentcloudcr.com/cloud/lexmount-e2e-tool:0418abd-20260508-212550
lexmoun-tcr-hk.tencentcloudcr.com/cloud/lexmount-e2e-tool:0418abd-20260508-212550
```

Kong init 最终镜像：

```text
code.lexmount.net/wangli/kong-init:8287913-20260508-215015
lexmount.tencentcloudcr.com/cloud/kong-init:8287913-20260508-215015
```

## Manifests

新增 K8s base：

```text
apps/lexmount-e2e-tool/
```

包含：

- Deployment
- Service
- ConfigMap
- Secret
- ServiceAccount
- image replacement

环境接入范围：

- `office-nanjing`：接入并已 apply
- `qcloud`：配置合入，不由 Alice apply
- `qcloud-hk`：配置合入，不由 Alice apply
- `office-beijing`：明确不接入 E2E tool

配置补充：

- qcloud-hk `LEXMOUNT_BASE_URL=https://api.lexmount.com`
- office-nanjing `LEXMOUNT_BASE_URL=https://apitest.local.lexmount.net`
- qcloud 使用默认 `https://api.lexmount.cn`
- 新 ServiceAccount 已补齐 imagePullSecrets

## Kong 路由

### office-nanjing public Kong

新增 route：

```text
host: e2e.local.lexmount.net
service: lexmount-e2e-tool.system.svc.cluster.local:80
```

office-nanjing 已 apply，并验证：

```text
https://e2e.local.lexmount.net/ -> 200 text/html
```

### qcloud internal Kong

新增 internal route：

```text
host: e2e.lexmount.cn
service: lexmount-e2e-tool.system.svc.cluster.local:80
route: lexmount-e2e-tool-internal
sni: e2e.lexmount.cn
target_tag: qcloud-internal
```

qcloud 的 `kong-init-image` 已配置为：

```text
lexmount.tencentcloudcr.com/cloud/kong-init:8287913-20260508-215015
```

qcloud 未由 Alice apply。

## office-nanjing 发布验证

office-nanjing 已发布：

- `deployment/lexmount-e2e-tool`
- `deployment/kong-init`

最终镜像：

```text
lexmount-e2e-tool: code.lexmount.net/wangli/lexmount-e2e-tool:0418abd-20260508-212550
kong-init: code.lexmount.net/wangli/kong-init:8287913-20260508-215015
```

验证结果：

- `deployment/lexmount-e2e-tool` rollout 成功
- `deployment/kong-init` rollout 成功
- `https://e2e.local.lexmount.net/` 返回 `200 text/html`
- `/api/demos` 返回 11 个 demo
- 11 个 demo 均为 `defaultSelected=true`
- 页面不再包含 Context Modes / Proxy / Extension Basic / new-page

## 环境边界

本次明确的环境边界：

- office-nanjing：Alice 负责 apply 和验证
- qcloud：只提交配置和镜像 tag，不 apply
- qcloud-hk：只提交配置和镜像 tag，不 apply
- office-beijing：不接入 E2E tool

## 后续建议

- 如果 qcloud / qcloud-hk 要正式上线，需要由环境 owner apply manifests 并执行对应 Kong init。
- E2E tool 当前 task state 在内存中，服务重启会丢失历史任务；如果后续需要审计或保留运行历史，再引入持久化。
- 当前 demo 串行执行，适合稳定 smoke；如后续 demo 数量明显增加，再考虑有限并发和项目配额保护。
- 如果后续新增 SDK demo，默认应同步加入 E2E tool，并明确是否纳入 default。
