# Go Runtime Cutover Summary — 2026-04-28

## 背景
目标是把原本落在 `browser-manager-reconciler` JS 服务里的 session runtime control 职责，迁移到 Go 控制面（`k8s-chrome-daemon` / `browser-operator`）里。

迁移后职责边界：
- Go：负责 runtime control
  - `active session cleanup`
  - `creating session reconcile`
  - 基于 `StateStore + APIReader` 做状态判断与补偿
- JS：退回 housekeeping
  - `downloads cleanup`
  - `closed session history cleanup`
  - `/metrics`
  - `/healthz`

---

## 涉及仓库
1. `k8s-chrome-daemon`
2. `demo-nodejs-backend`
3. `lexmount-k8s-manifests`

---

## 代码改动概览

### 1. `k8s-chrome-daemon`
分支：`wangli_dev_20260428_go_session_cleanup_main`
PR：<https://github.com/lexmount/k8s-chrome-daemon/pull/64>

主要改动：
- `cmd/main/main.go`
- `pkg/sessiondb/postgres.go`
- `pkg/sessioncleanup/runner.go`
- `pkg/sessionreconcile/creating_runner.go`

落地内容：
- 新增 leader-only `active session cleanup runner`
- 新增 leader-only `creating session reconcile runner`
- 通过 `database/sql + lib/pq` 读取 `session_history`
- `active` session：优先看 `StateStore`，必要时 fallback `APIReader`
- `creating` session：
  - `Running + podIP` 时请求 `/json/version`，生成 ws URL 并激活 session
  - `Failed / Error` 时删除实例并标记 `create_failed`
  - `context_id` 存在时绑定 session

额外修复：
- 修掉 `session_history` 可空字段导致的 `NULL -> string` 扫描错误
- 修复 commit：`b57934b`

---

### 2. `demo-nodejs-backend`
分支：`wangli_dev_20260428_go_session_cleanup_reference`
PR：<https://github.com/lexmount/demo-nodejs-backend/pull/128>

主要改动范围：
- `browser-manager-reconciler/reconciler.js`
- `browser-manager-reconciler/utils/reconciler-tasks.js`
- `browser-manager-reconciler/utils/reconciler-metrics.js`
- `browser-manager/utils/instance-helpers.js`
- `browser-manager/utils/docker.js`
- `browser-manager/utils/session-history-db.js`
- `grafana-dashboard-init/import_dashboards.py`

落地内容：
#### JS 职责收缩
- 删除 JS 侧 runtime/watch 逻辑
- 删除 `BrowserInstanceWatchManager`
- 删除旧的 `validateSessionContainer` / `reconcileActiveSessions` / `reconcileSessionById`
- 删除 `reconcileCreatingSessions`
- 删除仅服务于旧 JS reconcile 的 helper：
  - `getBrowserInstance()`
  - `getSessionsByStatuses()`

#### `browser-manager-reconciler` 重新定义为 housekeeping service
保留：
- `downloads cleanup`
- `closed session history cleanup`
- `/metrics`
- `/healthz`

语义对齐：
- 启动日志从 runtime/reconcile 口径调整为 housekeeping 口径
- `package.json` 描述同步调整

#### 新增最小 housekeeping metrics
新增指标：
- `browser_manager_reconciler_housekeeping_task_total{task,result}`
- `browser_manager_reconciler_housekeeping_task_duration_ms`
- `browser_manager_reconciler_housekeeping_items_total{task}`

#### Grafana 调整
`browser-manager-reconciler 自定义指标` dashboard 改成新的 housekeeping 口径：
- housekeeping task QPS
- housekeeping task 耗时 P95
- housekeeping task 处理量 QPS

---

### 3. `lexmount-k8s-manifests`
分支：`wangli_dev_20260428_go_runtime_cutover_refresh`
PR：<https://github.com/lexmount/lexmount-k8s-manifests/pull/357>

主要改动：
- 删除 JS runtime/watch 相关的 `ServiceAccount / ClusterRole / ClusterRoleBinding`
- 删除 `reconciler-service-account-patch.yaml`
- 为 `browser-operator` 增加 Go runner 配置和 DB/WSS 注入
- 同步 `office / qcloud / qcloud-hk` 需要的镜像 tag

关键配置：
- `ENABLE_GO_SESSION_CLEANUP_RUNNER`
- `GO_SESSION_CLEANUP_INTERVAL`
- `ENABLE_GO_CREATE_SESSION_RECONCILE_RUNNER`
- `GO_CREATE_SESSION_RECONCILE_INTERVAL`
- `GO_CREATE_SESSION_RECONCILE_TIMEOUT`
- `WSS_PREFIX`

---

## 重要 commit

### `k8s-chrome-daemon`
- `b57934b` — `fix: handle nullable session fields in runners`

### `demo-nodejs-backend`
- `b2d888f` — `refactor: remove stale reconciler watch code`
- `8c760c4` — `refactor: remove stale creating reconcile helpers`
- `18a79c1` — `chore: align reconciler housekeeping wording`
- `15a0aa8` — `feat: add reconciler housekeeping metrics`

### `lexmount-k8s-manifests`
- `089a460` — `chore: update office runtime cutover images`
- `0ee2861` — `chore: update runtime cutover images for office and qcloud`
- `7a72112` — `chore: align qcloud hk reconciler and grafana tags`
- `3237eea` — `chore: align qcloud hk operator tag`

---

## 镜像/tag

### office
- `browser-operator`
  - `code.lexmount.net/wangli/browser-operator:b57934b-20260428-163326`
- `browser-manager-reconciler`
  - `code.lexmount.net/wangli/browser-manager-reconciler:15a0aa8-20260428-165728`
- `grafana-dashboard-init`
  - `code.lexmount.net/wangli/grafana-dashboard-init:15a0aa8-20260428-170134`

### qcloud
- `browser-operator`
  - `lexmount.tencentcloudcr.com/cloud/browser-operator:b57934b-20260428-163326`
- `browser-manager-reconciler`
  - `lexmount.tencentcloudcr.com/cloud/browser-manager-reconciler:15a0aa8-20260428-165728`
- `grafana-dashboard-init`
  - `lexmount.tencentcloudcr.com/cloud/grafana-dashboard-init:15a0aa8-20260428-170134`

### qcloud-hk
- `browser-operator`
  - `lexmoun-tcr-hk.tencentcloudcr.com/cloud/browser-operator:b57934b-20260428-163326`
- `browser-manager-reconciler`
  - `lexmoun-tcr-hk.tencentcloudcr.com/cloud/browser-manager-reconciler:15a0aa8-20260428-165728`
- `grafana-dashboard-init`
  - `lexmoun-tcr-hk.tencentcloudcr.com/cloud/grafana-dashboard-init:15a0aa8-20260428-170134`

注：过程中用户明确要求不要由我主动推 HK 镜像；后续 tag 对齐是按用户最新明确指令继续更新 manifests 的。

---

## office 验证结果

### 1. `browser-operator`
已验证：
- rollout 成功
- `active session cleanup runner` 启动正常
- `creating session reconcile runner` 启动正常
- 之前的 `NULL -> string` 启动错误已消失

### 2. `browser-manager-reconciler`
已验证：
- rollout 成功
- 当前 `serviceAccount` 为 `default`
- 日志显示仅启动 housekeeping loops
- `/healthz` 返回 `200 ok`
- `/metrics` 已暴露 housekeeping 指标

### 3. `grafana-dashboard-init`
已验证：
- rollout 成功
- 手动执行 `python /app/import_dashboards.py`
- `browser-manager-reconciler 自定义指标` dashboard 返回 `status=success`

### 4. `browser-operator-api`
已验证：
- `GET /chromium/all` 返回 `200`
- header 带 `X-Chromium-State-Source: local`

### 5. 当前验证边界
已做：
- release-level smoke test
- 关键链路上线态验证

未做：
- 全量业务回归
- `creating -> active` 完整真实业务链路大范围冒烟
- `qcloud / qcloud-hk` 实际 rollout 验证

---

## 仍观察到的环境问题
office 环境里仍存在历史异常 `BrowserInstance`：
- `session-retryfix-1777286673-30006`
- 报错：`session_id is required in labels and cannot be empty`

这个是环境里已有脏对象，不是本轮迁移引入的问题。

---

## 当前结构性结论
迁移后的职责边界已经比之前清晰很多：
- Go 控制面：负责 session runtime lifecycle
- JS reconciler：只做 housekeeping
- manifests：不再给 JS runtime/watch 保留多余 RBAC
- metrics / dashboard：已从 runtime/watch 语义收口成 housekeeping 语义

---

## 后续可继续收的点
1. 把重复的 `wss-prefix -> browser-operator-config.WSS_PREFIX` replacement 上提到共享层
2. 补 `qcloud / qcloud-hk` 实际 rollout 验证
3. 补真实创建请求的完整业务闭环 smoke test
4. 把本轮总结同步到 `knowledge-repo`（如果需要沉淀成正式文档）
