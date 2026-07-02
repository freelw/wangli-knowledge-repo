# 基础设施

这一层放网络、镜像同步和其他基础设施相关知识。

当前文档：

- `tencent-network-topology.md`
- `regions-and-routing.md`
- `image-sync.md`
- `setaria-monitoring.md`
- `k8s-chrome-daemon-query-path.md`
- `daemon-timeout-pg-plan.md`
- `devtools-frontend-nginx/`

建议阅读顺序：

1. 先看 `regions-and-routing.md`
2. 再看 `image-sync.md`
3. 如果要看监控接入范式，再看 `setaria-monitoring.md`
4. 需要排查腾讯云网络时再看 `tencent-network-topology.md`
5. 需要分析 BrowserInstance 查询路径和 daemon 性能瓶颈时，再看 `k8s-chrome-daemon-query-path.md`
6. 需要推进 daemon 去 PocketBase、session timeout 改 PG 时，再看 `daemon-timeout-pg-plan.md`
- [k8s-chrome-daemon-local-state-watch.md](./k8s-chrome-daemon-local-state-watch.md) - daemon phase-2 local state view, watch path, and office validation.
- [browser-manager-reconciler-3a-3d.md](./browser-manager-reconciler-3a-3d.md) - reconciler from metrics to watch-first validation and watch-event-triggered targeted reconcile.
- [browser-manager-go-runtime-cutover.md](./browser-manager-go-runtime-cutover.md) - Go runtime cutover from JS reconciler to Go control plane, including housekeeping shrink and office validation.
- [daemon-timeout-pg-plan.md](./daemon-timeout-pg-plan.md) - daemon no longer depends on PocketBase for session timeout; lexhome writes PG and notifies daemon timeout controller refresh.
- [devtools-frontend-nginx/](./devtools-frontend-nginx/) - nginx image definition for serving a DevTools frontend artifact over HTTP.
- [dotcom-ip-verification-and-webfetch-routing.md](./dotcom-ip-verification-and-webfetch-routing.md) - Kong client IP forwarding, browser-manager geo blocking, ip2region service, and HK webfetch routes.
- [setaria-report-usage-api.md](./setaria-report-usage-api.md) - SetariaGateway `/internal/setaria/report_usage` 内部用量上报接口文档。
- [qcloud-spot-capacity-controller.md](./qcloud-spot-capacity-controller.md) - 腾讯云竞价实例询价控制器设计、配置、选型、Web plan 页面和运行注意事项。
- [watermark-node-scope-metrics-2026-05-21.md](./watermark-node-scope-metrics-2026-05-21.md) - watermark-monitor browser-ready all/non-spot node-scope metrics and Grafana legend updates.
- [qcloud-spot-capacity-controller-page-adjustment-2026-05-21.md](./qcloud-spot-capacity-controller-page-adjustment-2026-05-21.md) - capacity-controller Web page current active instances list and plan instance record separation.
- [kustomize-patch-compatibility.md](./kustomize-patch-compatibility.md) - Kustomize `patches` 兼容性注意事项：strategic merge patch 建议一文件一资源，避免旧版 `kubectl apply -k` 无法解析多文档 patch。
- [browser-scheduling-spot-fallback-2026-05-21.md](./browser-scheduling-spot-fallback-2026-05-21.md) - BrowserInstance scheduling policy: browser-ready hard constraint with non-spot preference and spot fallback.
- [qcloud-spot-controller-spot-node-browser-metrics-2026-05-21.md](./qcloud-spot-controller-spot-node-browser-metrics-2026-05-21.md) - qcloud-spot-capacity-controller 上报 spot 节点上的浏览器实例数量，并接入 Prometheus / Grafana。
- [k8s-node-resource-buffer-todo-2026-05-22.md](./k8s-node-resource-buffer-todo-2026-05-22.md) - TODO: 通过 kubelet reserved/eviction 和 limit/request 策略保障 node 预留约 5% CPU / 内存。
- [k8s-node-5-percent-resource-buffer-runbook-2026-05-25.md](./k8s-node-5-percent-resource-buffer-runbook-2026-05-25.md) - 操作手册：在线上 K8s 节点配置 kubelet reserved/eviction，给 node 预留约 5% CPU / 内存。
- [qcloud-resource-inventory-2026-07-02.md](./qcloud-resource-inventory-2026-07-02.md) - 通过 `tccli` 梳理 qcloud 南京、北京、香港的 CVM、CLB、VPC/NAT/EIP/安全组资源，并记录 PG/Redis 查询权限缺口。
