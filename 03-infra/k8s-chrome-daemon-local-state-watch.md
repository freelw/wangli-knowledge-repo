# daemon Phase 2 本地状态视图与 watch 落地记录

日期：2026-04-27
频道：#daemon性能调优
任务：task #2 `Phase 2: k8s-chrome-daemon 本地状态视图与 watch 方案落地`

## 背景

前一阶段已经确认：
- `browser-manager-reconciler` 会持续调用 `getContainerInfo()`
- `k8s-chrome-daemon` 的 `GET /chromium/{id}` 之前是同步直查 apiserver
- 这条路径存在 10s timeout 风险
- timeout 被上游放大后，会带来错误的 session 生命周期判断风险

因此第二阶段目标不是单纯“把 GET 再调快一点”，而是把查询链改成：
- 本地状态视图 / cache
- `BrowserInstance` / `Pod` 事件驱动更新
- 查询优先读本地
- 本地缺失时再 fallback 到 `APIReader`

## 设计收口

本次实现口径：
- 在 `k8s-chrome-daemon` 内新增 `StateStore`
- `StateStore` 挂在 controller-runtime manager 上，跟随进程启动
- watch 两类对象：
  - `BrowserInstance`
  - `Pod`
- 本地状态视图以 `BrowserInstance` 为主键，叠加关联 Pod 的运行态信息
- `GET /chromium/{id}` 和 `GET /chromium/all` 改成：
  - 优先本地状态
  - 缺失时 fallback 到 `APIReader`
  - fallback 结果回填本地状态
- 为了方便排障，GET 响应头增加：
  - `X-Chromium-State-Source: local|fallback`

## 代码改动

仓库：`/home/lexmount/project/backend/k8s-chrome-daemon`
分支：`wangli_dev_20260427_daemon_local_state_watch`
PR：<https://github.com/lexmount/k8s-chrome-daemon/pull/62>

关键提交：
- `a237547` `feat: add daemon local state store for chromium queries`
- `bf4bc3c` `chore: refresh daemon module sums`

关键文件：
- `cmd/main/main.go`
- `pkg/server/server.go`
- `pkg/server/state_store.go`
- `pkg/server/server_test.go`
- `go.mod`
- `go.sum`

### 1. StateStore

新增 `pkg/server/state_store.go`：
- 维护 `BrowserInstance` 本地快照
- 维护按 `BrowserInstance` owner 归并的 `Pod` 本地状态
- 在读路径里把 Pod 的 `phase/podName/podIP` 叠加到返回的 `BrowserInstance.Status`

### 2. 启动方式

`cmd/main/main.go` 中：
- 不再为 REST API 单独创建一套新的 k8s client 作为主查询路径
- 改成复用 manager 提供的：
  - `mgr.GetClient()`
  - `mgr.GetAPIReader()`
  - `mgr.GetCache()`
- `StateStore` 通过 `mgr.Add(stateStore)` 注册为 runnable

### 3. GET 查询语义

`pkg/server/server.go` 中：
- `getBrowserInstance()`
- `listBrowserInstances()`

收口后的行为：
- 如果 `StateStore.IsSynced()` 且本地命中：直接返回 `local`
- 如果本地未命中或尚未 sync：走 `APIReader`
- fallback 查询成功后回填本地状态
- 对外在 header 里显式暴露来源：`local` 或 `fallback`

## 测试

新增单测覆盖：
- 本地视图能叠加 Pod 状态到 BrowserInstance 返回值
- `GET` 在本地已同步且命中时走 `local`
- 本地缺失时会 fallback 到 `APIReader`
- fallback 成功后会回填本地 `StateStore`
- `list` 路径本地前缀过滤正确
- `list` fallback 回填正确
- not-found 路径返回语义正确

执行结果：
- `go test ./...` 通过

## office 环境验证

### 测试镜像

- office: `code.lexmount.net/wangli/browser-operator:bf4bc3c-20260427-160553`
- qcloud: `lexmount.tencentcloudcr.com/cloud/browser-operator:bf4bc3c-20260427-160553`
- qcloud-hk: `lexmoun-tcr-hk.tencentcloudcr.com/cloud/browser-operator:bf4bc3c-20260427-160553`

### office 实际验证

已在 office 做实际 rollout 和 smoke test：
- `browser-operator-controller-manager` rollout 成功
- deployment 实际运行镜像已切到 `bf4bc3c-20260427-160553`

通过 port-forward 实测：
- `GET /chromium/all` 返回 `200`
- `GET /chromium/session-1777277114333-fi50scsux` 返回 `200`
- 两个响应头都带：
  - `X-Chromium-State-Source: local`

### 重启恢复验证

额外做了一次 deployment 重启验证：
- `rollout restart deploy/browser-operator-controller-manager`
- 新 pod ready 后再次请求上述两个 GET
- 结果仍为 `200`
- 响应头仍是 `X-Chromium-State-Source: local`

结论：
- 新实现能在 office 正常运行
- deployment 重启后服务能恢复
- cache / watch 热起来后，读路径能走本地状态视图

说明：
- 这次 smoke test 没有刻意卡“冷启动未热完”的瞬间窗口
- 但代码路径上 fallback 到 `APIReader` 的兜底仍然保留

## manifests 与发布跟进

仓库：`/home/lexmount/project/backend/lexmount-k8s-manifests`
分支：`wangli_dev_20260427_daemon_local_state_release`
PR：<https://github.com/lexmount/lexmount-k8s-manifests/pull/346>

本次 PR 只做一件事：
- 对齐 `office / qcloud / qcloud-hk` 三边 `browser-operator-image`

未夹带其他 manifests 变更。

## 当前交付物

1. Go 代码 PR：<https://github.com/lexmount/k8s-chrome-daemon/pull/62>
2. manifests PR：<https://github.com/lexmount/lexmount-k8s-manifests/pull/346>
3. office 已完成实测
4. 三边镜像 tag 已对齐

## 后续建议

如果继续往下一阶段推进，建议顺序：
1. 补充本地状态视图的命中率/回退率指标
2. 评估是否要给 fallback 路径单独打监控
3. 再看 `browser-manager-reconciler` 的查询频率和扫描模型是否继续下调
