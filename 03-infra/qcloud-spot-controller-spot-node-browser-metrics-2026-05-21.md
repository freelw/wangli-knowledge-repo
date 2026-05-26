# qcloud-spot-capacity-controller Spot 节点浏览器实例上报

## 背景

引入 spot 节点后，需要持续观察当前有多少浏览器实例真正运行在 spot 节点上，用于判断正常节点容量是否不足、spot 兜底是否被触发，以及后续容量策略是否需要调整。

本轮工作在 `qcloud-spot-capacity-controller` 中增加 Prometheus 指标，并在 Grafana 中新增展示面板。

## 目标

- 统计当前调度到 `lexmount/spot=true` 节点上的浏览器实例数量。
- 指标由 `qcloud-spot-capacity-controller` 暴露到 `/metrics`。
- Prometheus 采集该指标。
- Grafana 新增 dashboard 展示该指标。
- manifests 收敛成单个 PR，同步 controller 镜像、Prometheus scrape 配置和 Grafana dashboard-init 镜像。

## 最终 PR

代码：

- `qcloud-k8s-tool`: https://github.com/lexmount/qcloud-k8s-tool/pull/28
- `demo-nodejs-backend`: https://github.com/lexmount/demo-nodejs-backend/pull/192

manifests：

- `lexmount-k8s-manifests`: https://github.com/lexmount/lexmount-k8s-manifests/pull/518

说明：中间曾有独立 Grafana manifests PR #521，后续按要求收敛到 #518，#521 已关闭。

## 代码变更

### qcloud-k8s-tool

目录：`qcloud-spot-capacity-controller/`

新增指标：

```text
qcloud_spot_capacity_controller_spot_node_browser_instance_running{region="..."}
```

语义：当前 `Running` 状态、`app=browser-instance` 的 Pod 中，有多少个调度在带有 `lexmount/spot=true` 标签的节点上。

实现要点：

1. 通过 Kubernetes API 列出 `lexmount/spot=true` 的 Node。
2. 构建 spot node name 集合。
3. 列出 `app=browser-instance` 的 Pod。
4. 只统计 `pod.Status.Phase == Running` 且 `pod.Spec.NodeName` 命中 spot node 集合的 Pod。
5. 每轮 reconcile 刷新 Gauge。

### 分页处理

Review comments 指出最初只给 Pod list 加了分页，Node list 仍然是无界 list。最终修正为：

- Node list 使用 `Limit: 500` + `Continue` 分页。
- Pod list 使用 `Limit: 500` + `Continue` 分页。

原因：避免大集群下单次 list 过大，对 API Server / etcd 造成不必要压力。

### 指标命名

最初指标名为：

```text
qcloud_spot_capacity_controller_spot_node_browser_instance_running_total
```

Review comments 指出 `_total` 后缀应保留给 Counter。该指标是 Gauge，因此最终改为：

```text
qcloud_spot_capacity_controller_spot_node_browser_instance_running
```

## `/metrics` 鉴权

Review comments 指出 `/metrics` 和业务 API 在同一个 HTTP 端口，且最初未鉴权。

最终处理：

- `/metrics` 复用 `SPOT_TERMINATION_TOKEN` 的 Bearer auth。
- Prometheus scrape job 增加：

```yaml
authorization:
  credentials: qcloud-spot-termination-internal
```

当前 qcloud-nanjing / qcloud-beijing 的 secret patch 中已有：

```yaml
SPOT_TERMINATION_TOKEN: "qcloud-spot-termination-internal"
```

注意：这里使用静态明文 token 是当前 manifests 既有模式。后续如果 Secret 管理方式升级，需要同步改 Prometheus scrape 配置。

## Grafana

仓库：`demo-nodejs-backend`

变更点：`grafana-dashboard-init/import_dashboards.py`

新增 dashboard：

```text
qcloud spot capacity 指标
```

新增面板：

```text
Spot 节点上的浏览器实例数
```

查询：

```promql
qcloud_spot_capacity_controller_spot_node_browser_instance_running{}
```

同时在抓取状态 dashboard 中补充 `qcloud-spot-capacity-controller` 的 scrape health 面板，便于确认 Prometheus 是否成功抓取。

## manifests

最终 manifests PR #518 包含：

- `qcloud-nanjing` / `qcloud-beijing` 的 `qcloud-spot-capacity-controller-image`。
- `office-nanjing` / `qcloud-nanjing` / `qcloud-beijing` 的 `grafana-dashboard-init-image`。
- `monitoring/prometheus/prometheus-configmap.yaml` 中新增 `qcloud-spot-capacity-controller` scrape job。
- scrape job 增加 Bearer auth。

最终镜像：

```text
qcloud-spot-capacity-controller:0f27079-20260521-212657
grafana-dashboard-init:c456500-20260521-211952
```

具体镜像地址：

```text
lexmount.tencentcloudcr.com/cloud/qcloud-spot-capacity-controller:0f27079-20260521-212657
lexmount-bj.tencentcloudcr.com/cloud/qcloud-spot-capacity-controller:0f27079-20260521-212657
code.lexmount.net/wangli/grafana-dashboard-init:c456500-20260521-211952
lexmount.tencentcloudcr.com/cloud/grafana-dashboard-init:c456500-20260521-211952
lexmount-bj.tencentcloudcr.com/cloud/grafana-dashboard-init:c456500-20260521-211952
```

## 校验记录

代码侧：

```bash
cd qcloud-spot-capacity-controller
go test ./...
```

Grafana 初始化脚本：

```bash
python3 -m py_compile grafana-dashboard-init/import_dashboards.py
```

manifests：

```bash
kubectl kustomize apps/clusters/qcloud-nanjing
kubectl kustomize apps/clusters/qcloud-beijing
kubectl kustomize apps/clusters/office-nanjing
```

## 后续观察

发布后重点看两类信号：

1. Prometheus target：`job="qcloud-spot-capacity-controller"` 是否 up。
2. 指标值：`qcloud_spot_capacity_controller_spot_node_browser_instance_running{region="..."}` 是否随 spot 节点上的浏览器实例变化。

如果 Prometheus target 401，优先检查：

- `SPOT_TERMINATION_TOKEN` 实际值。
- Prometheus scrape job 的 `authorization.credentials` 是否与 token 一致。
- qcloud-nanjing / qcloud-beijing 是否应用了最新 manifests。
