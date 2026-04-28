# browser-manager-reconciler 3A-3D 演进记录

日期：2026-04-28
来源频道：#daemon性能调优
关联任务：task #2 `Phase 2: k8s-chrome-daemon 本地状态视图与 watch 方案落地`

## 背景

这条线的目标，是把原来：

- `browser-manager-reconciler`
- `getContainerInfo()`
- `k8s-chrome-daemon GET /chromium/{id}`

这条高频轮询链路，逐步改造成：

- 有可观测性
- daemon 查询本地优先
- reconciler 优先依赖 watch 状态
- 最终收口成事件驱动为主、低频轮询兜底

## 3A：先补观测

目标：
- 先补 `browser-manager-reconciler` 侧的观测能力
- 让 Prometheus / Grafana 能看到 scan、validation、container-info 相关指标

交付：
- JS 代码 PR：<https://github.com/lexmount/demo-nodejs-backend/pull/124>
- manifests PR：<https://github.com/lexmount/lexmount-k8s-manifests/pull/347>
- dashboard PR：<https://github.com/lexmount/demo-nodejs-backend/pull/123>

主要改动：
- `browser-manager-reconciler` 暴露 `/metrics` 和 `/healthz`
- 新增 reconciler 侧指标：
  - `browser_manager_reconciler_scan_duration_ms`
  - `browser_manager_reconciler_active_sessions_per_scan`
  - `browser_manager_reconciler_session_validation_total`
  - `browser_manager_reconciler_container_info_duration_ms`
  - `browser_manager_reconciler_container_info_inflight`
  - `browser_manager_reconciler_scan_inflight`
- Prometheus 增加 `browser-manager-reconciler` scrape job
- Grafana 增加 daemon / reconciler 相关 dashboard

## 3B：引入 BrowserInstance watch

目标：
- 在 JS 侧先引入 `BrowserInstance watch`
- 让稳定 Running 的 session 不再每轮都落回 `getContainerInfo()`

交付：
- 在 PR #124 / manifests PR #347 上继续迭代
- office 验证镜像：
  - `code.lexmount.net/wangli/browser-manager-reconciler:b10db7d-20260427-173952`

主要改动：
- `browser-manager-reconciler` 增加 `BrowserInstanceWatchManager`
- 本地维护 `session_id -> BrowserInstance` 状态索引
- watch ready 且 phase=`Running` 时：
  - 直接走 `watch_running_skip`
  - 不再调用 `getContainerInfo()`
- 增加 watch 相关指标：
  - `browser_manager_reconciler_browserinstance_watch_events_total`
  - `browser_manager_reconciler_browserinstance_watch_state_entries`
  - `browser_manager_reconciler_browserinstance_watch_ready`
  - `browser_manager_reconciler_browserinstance_watch_restarts_total`

## 3B 跟进修复：创建链路 500 与 stdout

交付：
- Go 代码 PR：<https://github.com/lexmount/k8s-chrome-daemon/pull/63>
- manifests 继续复用 PR #347
- office 验证镜像：
  - `code.lexmount.net/wangli/browser-operator:d54cc0d-20260427-184254`

主要改动：
- `k8s-chrome-daemon waitForPodReady()` 遇到短暂 `BrowserInstance not found`
  - 不再立刻返回 500
  - 改成按可重试状态继续等待
- `office` 的 `browser-manager` 改为 `stdout` 日志，便于直接用 `kubectl logs` 排查

## 3C：watch-first validation

目标：
- 让 reconciler 校验主路径更明确地“优先依赖 watch 状态”
- 只有缺状态、watch 未就绪或到达补偿窗口时，才退回 `getContainerInfo()`

交付：
- PR：<https://github.com/lexmount/demo-nodejs-backend/pull/125>
- 分支：`wangli_dev_20260428_reconciler_phase3c`
- commit：`423f28f`
- office 镜像：
  - `code.lexmount.net/wangli/browser-manager-reconciler:423f28f-20260428-093840`

主要改动：
- `validateSessionContainer()` 新增 watch 决策分支：
  - `watch_running_skip`
  - `watch_terminal_close`
  - `watch_deleted_close`
  - `watch_fallback_skip`
  - `fallback_poll`
- 新增 `recentlyDeletedBySessionId`
- 引入 `RECONCILER_CONTAINER_INFO_FALLBACK_INTERVAL_MS`

## 3D：watch event -> targeted reconcile

目标：
- 让 `BrowserInstance watch` 事件直接触发 targeted reconcile
- 把全量 active session scan 降成低频补偿

交付：
- PR：<https://github.com/lexmount/demo-nodejs-backend/pull/126>
- 分支：`wangli_dev_20260428_reconciler_phase3d`
- commit：`90848f0`
- office 镜像：
  - `code.lexmount.net/wangli/browser-manager-reconciler:90848f0-20260428-100238`

主要改动：
- `BrowserInstanceWatchManager` 增加 subscriber 能力
- watch 事件触发 targeted reconcile 队列
- 新增 targeted reconcile 指标：
  - `browser_manager_reconciler_targeted_reconcile_total`
  - `browser_manager_reconciler_targeted_reconcile_batch_size`
  - `browser_manager_reconciler_targeted_reconcile_queue_size`
- 全量 active session scan 改成低频补偿：
  - `RECONCILER_ACTIVE_SESSION_SCAN_INTERVAL_MS`
  - 默认 `5m`
- 新增 `RECONCILER_EVENT_RECONCILE_DEBOUNCE_MS`
  - 默认 `1000ms`

## office 验证结论

到 2026-04-28 为止，office 已经验证到：

- daemon 查询面已有本地状态视图
- reconciler 已有可观测性
- reconciler 已接入 BrowserInstance watch
- Running session 已能走 watch 主路径
- watch 事件已能触发 targeted reconcile 逻辑
- 全量扫描已降成低频补偿

说明：
- 3D smoke test 期间没有刻意制造 `DELETED` / 非 `Running` BrowserInstance 事件
- 所以 targeted reconcile 计数还没有样本值
- 但代码路径、镜像和 office 服务都已验证通过

## 配套发布 / manifests 跟进

这条线过程中还产生了几条发布与清理 PR：

- PR #349
  - <https://github.com/lexmount/lexmount-k8s-manifests/pull/349>
  - `chore: update office browser manager reconciler image`
- PR #350
  - <https://github.com/lexmount/lexmount-k8s-manifests/pull/350>
  - `chore: add image pull secrets for reconciler service account`
- PR #351
  - <https://github.com/lexmount/lexmount-k8s-manifests/pull/351>
  - `chore: align qcloud-hk grafana dashboard image`

其中 PR #350 对应一条新的发布规范沉淀：
- 每当创建新的 `ServiceAccount`
- 都要同步补齐 `imagePullSecrets`

## 后续建议

1. 先补一轮真实事件样本验证
   - 人工制造 `DELETED` / `Failed` / 非 `Running` BrowserInstance 事件
   - 验证 `targeted_reconcile_total` 的实际样本值
2. 再补 Grafana 面板
   - 把 3D 新增的 targeted reconcile 指标接进 dashboard
3. 再看是否要进一步下调低频补偿扫描
   - 根据线上命中率和事件覆盖率决定
4. 长期再评估是否继续 Go 化 `browser-manager-reconciler`
