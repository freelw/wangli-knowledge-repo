# 2026-04-28 Go Runtime Cutover

长期知识文档：
- [03-infra/browser-manager-go-runtime-cutover.md](../../03-infra/browser-manager-go-runtime-cutover.md)

本次任务完成的核心事项：
- 将 `active session cleanup` 与 `creating session reconcile` 从 JS 迁入 Go 控制面
- 将 `browser-manager-reconciler` 收口为 housekeeping service
- 清理 JS 侧遗留的 watch / runtime / creating reconcile 代码
- 为新的 housekeeping 形态补最小 metrics 和 Grafana 面板
- 完成 `office` smoke test，并同步 `office / qcloud / qcloud-hk` 对应 manifests tag

涉及仓库：
- `k8s-chrome-daemon`
- `demo-nodejs-backend`
- `lexmount-k8s-manifests`

相关 PR：
- `k8s-chrome-daemon` #64
- `demo-nodejs-backend` #128
- `lexmount-k8s-manifests` #357
