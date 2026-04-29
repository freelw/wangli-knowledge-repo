# 基础设施

这一层放网络、镜像同步和其他基础设施相关知识。

当前文档：

- `tencent-network-topology.md`
- `regions-and-routing.md`
- `image-sync.md`
- `setaria-monitoring.md`
- `k8s-chrome-daemon-query-path.md`

建议阅读顺序：

1. 先看 `regions-and-routing.md`
2. 再看 `image-sync.md`
3. 如果要看监控接入范式，再看 `setaria-monitoring.md`
4. 需要排查腾讯云网络时再看 `tencent-network-topology.md`
5. 需要分析 BrowserInstance 查询路径和 daemon 性能瓶颈时，再看 `k8s-chrome-daemon-query-path.md`
- [k8s-chrome-daemon-local-state-watch.md](./k8s-chrome-daemon-local-state-watch.md) - daemon phase-2 local state view, watch path, and office validation.
- [browser-manager-reconciler-3a-3d.md](./browser-manager-reconciler-3a-3d.md) - reconciler from metrics to watch-first validation and watch-event-triggered targeted reconcile.
- [browser-manager-go-runtime-cutover.md](./browser-manager-go-runtime-cutover.md) - Go runtime cutover from JS reconciler to Go control plane, including housekeeping shrink and office validation.
