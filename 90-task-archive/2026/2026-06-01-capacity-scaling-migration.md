# qcloud 扩容逻辑迁移到 qcloud-spot-capacity-controller

## 背景

原来的正常实例扩容链路由 `k8s-watermark-monitor` 计算水位线并触发 SCF。随着 qcloud spot 实例控制器逐步承担竞价实例购买、实例补偿、节点回收和页面展示能力，扩容逻辑继续分散在 `watermark-monitor` / SCF / spot controller 三处会导致职责不清、指标口径分裂、跨地域扩展困难。

本轮把正常实例扩缩容、水位线计算、Prometheus 指标和 Grafana 展示集中到 `qcloud-spot-capacity-controller`，后续 `watermark-monitor` 不再作为扩容触发入口。

## 任务目标

- 将正常实例扩缩容逻辑从 `watermark-monitor` / SCF 迁入 `qcloud-spot-capacity-controller`。
- `qcloud-spot-capacity-controller` 同时负责：spot 实例购买、正常实例扩缩容、水位线采集、节点数量 / CPU / 内存 request 使用率指标上报。
- Prometheus / Grafana 改为消费 `qcloud_spot_capacity_controller_*` 指标。
- qcloud 环境启用水位线与正常缩容能力；在真实 Launch Template 未配置前，禁止正常扩容，避免 API 错误重试。
- office 环境以 dry-run 方式部署 controller，只用于验证指标和容量接口，不执行真实购买或扩缩容。

## 涉及仓库

- `qcloud-k8s-tool`
- `demo-nodejs-backend`
- `lexmount-k8s-manifests`

## 分支与 PR

- `qcloud-k8s-tool` PR #36：<https://github.com/lexmount/qcloud-k8s-tool/pull/36>
- `demo-nodejs-backend` PR #192：<https://github.com/lexmount/demo-nodejs-backend/pull/192>
- `lexmount-k8s-manifests` PR #556：<https://github.com/lexmount/lexmount-k8s-manifests/pull/556>

关键镜像：

- `qcloud-spot-capacity-controller:3fa240b-20260601-210331`
- `region-data-plane-gateway:7098ab9-20260601-202721`
- `grafana-dashboard-init:893fad3-20260601-204600`

## 关键改动

### qcloud-spot-capacity-controller

新增正常实例扩缩容链路：

- 根据非 spot 的 `browser-ready=true` 节点池计算正常节点水位线。
- 高水位或正常节点数低于下限时触发正常实例扩容。
- 低水位且节点数高于下限时触发正常实例缩容。
- 正常实例扩容使用 Tencent Cloud Launch Template，join Kubernetes 后不打 `lexmount/spot=true` 标签。
- 正常实例缩容会先标记 scale-in、等待 browser instance pod 释放、cordon/drain、销毁 CVM、删除 Kubernetes Node。

拆分并行流程：

- 前置处理仍串行：托管实例状态补偿、NotReady spot node 补偿、spot node browser instance 指标更新。
- `reconcileCapacityScaling` 和 `reconcileSpotCapacity` 并行执行，避免正常实例扩缩容阻塞 spot 询价购买流程。

新增或调整关键开关：

- `capacity_scaling_enabled`：是否启用正常实例扩缩容与水位线控制。
- `disable_scale_out`：单独禁止正常实例扩容。
- `disable_scale_in`：单独禁止正常实例缩容。
- `spot_purchase_enabled`：单独控制 spot 购买流程，office dry-run 环境关闭。
- `event_retention_days`：`spot_capacity_events` 保留天数，默认 60 天。

修复 review 发现的问题：

- `drainNode` 捕获 Node Update 返回值，避免旧 `resourceVersion` 导致后续打 label 409。
- `getClusterResourceRequestUsage` 复用已 list 的 nodes，避免 pod loop 内 N+1 Node Get。
- `pods_drained_total` 去掉 `node_name` / `cvm_instance_id` 高基数标签，只保留 `region`。
- pod request 计算采用 kube-scheduler 语义：`max(sum(app containers), max(init containers))`。
- 跳过 `NodeName == ""` 的 Pending pod，避免无关未调度 pod 被计入水位线。
- `findNodeInScaleIn` 查询失败后直接返回，避免无法确认已有 scale-in 节点时继续扩缩容。
- `RunKubeadmJoin` 改名为 `RunKubeadmJoinSpot`，语义上只用于 spot 节点 join。

### Prometheus / Grafana

水位线指标改为由 `qcloud-spot-capacity-controller` 暴露：

- `qcloud_spot_capacity_controller_cluster_worker_nodes{region,node_scope}`
- `qcloud_spot_capacity_controller_cluster_total_cpu_cores{region,node_scope}`
- `qcloud_spot_capacity_controller_cluster_total_memory_bytes{region,node_scope}`
- `qcloud_spot_capacity_controller_cluster_used_cpu_request_cores{region,node_scope}`
- `qcloud_spot_capacity_controller_cluster_used_memory_request_bytes{region,node_scope}`
- `qcloud_spot_capacity_controller_cluster_cpu_request_percent{region,node_scope}`
- `qcloud_spot_capacity_controller_cluster_memory_request_percent{region,node_scope}`
- `qcloud_spot_capacity_controller_pods_drained_total{region}`

Grafana `qcloud capacity 水位线` dashboard 改为查询上述指标，并使用 `max by (region, node_scope)` 聚合，避免 rollout 期间旧 pod 的 0 时序影响面板显示。

### region-data-plane-gateway

`/v1/region/capacity` 改为读取 `qcloud_spot_capacity_controller_cluster_*{node_scope="browser_ready_all"}`，不再依赖 `watermark_monitor_*`。

### manifests

qcloud 三地：

- `capacity_scaling_enabled=true`
- `disable_scale_in=false`
- `disable_scale_out=true`
- qcloud-nanjing `min_worker_node_count=3`
- qcloud-beijing `min_worker_node_count=1`
- qcloud-hk `min_worker_node_count=2`

注意：`disable_scale_out=true` 是当前安全保护。三地 `normal_launch_template_id` / `normal_launch_template_version` 仍是 placeholder，因此不能开启正常扩容，否则会触发 Tencent Cloud `RunInstances` 启动模板错误并持续重试。待真实 Launch Template 配置完成后，再把 `disable_scale_out` 调回 `false`。

office 环境：

- office-nanjing / office-beijing 都部署 `qcloud-spot-capacity-controller`。
- `dry_run=true`。
- `spot_purchase_enabled=false`。
- `capacity_scaling_enabled=true`。
- `disable_scale_out=true` / `disable_scale_in=true`。
- `ignore_kong_node_for_browser_capacity=true`，因为 office 唯一 `browser-ready=true` 节点同时带 `lexmount/kong=true`，否则水位线会被过滤成 0。

## 验证方式

代码验证：

- `qcloud-spot-capacity-controller` 执行 `go test ./...` 通过。
- `grafana-dashboard-init/import_dashboards.py` 执行 Python 编译检查通过。

manifests 验证：

- `kubectl kustomize apps/clusters/qcloud-nanjing` 通过。
- `kubectl kustomize apps/clusters/qcloud-beijing` 通过。
- `kubectl kustomize apps/clusters/qcloud-hk` 通过。
- `kubectl kustomize apps/clusters/office-nanjing` 通过。
- `kubectl kustomize apps/clusters/office-beijing` 通过。
- `kubectl kustomize apps/clusters/office-private` 通过。
- `kubectl kustomize apps/clusters/guoge` 通过。

环境验证：

- office-nanjing 发布后，`qcloud-spot-capacity-controller` 正常 rollout。
- office-nanjing Prometheus `up{job="qcloud-spot-capacity-controller"}=1`。
- office-nanjing `/v1/region/capacity` 返回 200，容量来自新指标。
- office-beijing 发布后，`qcloud-spot-capacity-controller` 和 `region-data-plane-gateway` 正常 rollout。
- office-beijing Prometheus `up{job="qcloud-spot-capacity-controller"}=1`。
- office-beijing 水位线显示 `worker_nodes=1`，CPU request 约 48%，Memory request 约 47.393%。
- office-beijing `/v1/region/capacity` 返回 200。
- office-nanjing Grafana dashboard 通过 `grafana-dashboard-init` 手动同步成功。

## 关键经验

- 扩容逻辑要分清两个维度：spot 购买和正常实例扩缩容。它们可以共享水位线输入，但执行流程应并行，避免互相阻塞。
- qcloud 正常实例扩容依赖真实 Launch Template。只打开 `capacity_scaling_enabled` 不够，还必须确认 `normal_launch_template_id/version` 已配置，否则应保持 `disable_scale_out=true`。
- office 环境的 `browser-ready=true` 节点可能同时是 Kong 节点。用于验证水位线时需要配置 `ignore_kong_node_for_browser_capacity=true`，否则唯一节点会被过滤。
- Prometheus 指标要控制 label cardinality。节点名、实例 ID 不适合放在长期 counter 标签上。
- 统计 pod request 时不能简单把 init container request 加到 app container request 上，应按 kube-scheduler 语义取 max。
- 未调度 pod 没有 `NodeName`，不能纳入 capacity node 水位线，否则会把全局 Pending pod 当作 browser-ready 节点负载。

## 后续待办

- 为 qcloud-nanjing / qcloud-beijing / qcloud-hk 填入真实 `normal_launch_template_id` 和 `normal_launch_template_version`。
- 完成 Launch Template 验证后，再将 qcloud 三地 `disable_scale_out=false`。
- 逐步下线 `watermark-monitor` 的扩容触发职责，避免双写/双触发。
- 根据 qcloud 实际运行情况，复核 `min_worker_node_count`，尤其是 qcloud-beijing 当前为 1。

## 结论

- 扩容职责已经从 `watermark-monitor` / SCF 迁到 `qcloud-spot-capacity-controller`。
- 水位线指标、region capacity API、Grafana dashboard 已切到新指标链路。
- office-nanjing / office-beijing 已验证新链路可采集和展示。
- qcloud 三地当前启用水位线和正常缩容，正常扩容因 Launch Template 仍未配置而显式关闭。
