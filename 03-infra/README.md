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
- [qcloud-spot-capacity-controller.md](./qcloud-spot-capacity-controller.md) - 腾讯云竞价实例询价控制器设计、配置、选型、Web plan 页面和运行注意事项。
- [watermark-node-scope-metrics-2026-05-21.md](./watermark-node-scope-metrics-2026-05-21.md) - watermark-monitor browser-ready all/non-spot node-scope metrics and Grafana legend updates.
- [qcloud-spot-capacity-controller-page-adjustment-2026-05-21.md](./qcloud-spot-capacity-controller-page-adjustment-2026-05-21.md) - capacity-controller Web page current active instances list and plan instance record separation.
- [kustomize-patch-compatibility.md](./kustomize-patch-compatibility.md) - Kustomize `patches` 兼容性注意事项：strategic merge patch 建议一文件一资源，避免旧版 `kubectl apply -k` 无法解析多文档 patch。
