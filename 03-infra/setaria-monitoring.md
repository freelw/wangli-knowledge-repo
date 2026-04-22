# setaria 监控接入

## 背景

`setaria` 需要接入平台现有的 Prometheus + Grafana 监控链路，用于观察 deployment 内三个容器的资源消耗，以及对应 Pod 的网络流量。

这次接入分成两部分：

1. 在 `lexmount-k8s-manifests` 中补 Prometheus 抓取配置与 recording rules
2. 在 `demo-nodejs-backend/grafana-dashboard-init` 中补 dashboard 注入逻辑

## 涉及仓库

- `lexmount-k8s-manifests`
- `demo-nodejs-backend`

## Prometheus 抓取配置

位置：
- `monitoring/prometheus/prometheus-configmap.yaml`

接入方式：
- 通过 `role: pod` 做服务发现
- 用 `app=setaria` 过滤目标 Pod
- 抓取端口为 `8080`
- 抓取路径为 `/metrics`

这套写法与现有 `watermark-monitor`、`browser-replay-processor` 的模式一致。

## Recording Rules

位置：
- `monitoring/prometheus/prometheus-rules-configmap.yaml`

新增规则组：
- `setaria-resource-metrics`

当前规则：
- `setaria:container_cpu_usage:rate:cores`
- `setaria:container_memory_rss:bytes`
- `setaria:pod_network_receive_bytes:rate`
- `setaria:pod_network_transmit_bytes:rate`

指标口径：
- CPU / 内存是容器维度
- 网络流量是 Pod 维度

这个差异不是实现疏漏，而是 Kubernetes / cAdvisor 原始指标决定的：
- `container_cpu_usage_seconds_total` 与 `container_memory_rss` 可以稳定按容器聚合
- `container_network_receive_bytes_total` 与 `container_network_transmit_bytes_total` 本质上是 Pod 共享网络指标，不适合强行拆到容器维度

## Grafana Dashboard

位置：
- `demo-nodejs-backend/grafana-dashboard-init/import_dashboards.py`

新增 dashboard：
- `setaria Deployment 资源指标`

主要面板：
- `setaria 抓取状态`
- `setaria 抓取耗时`
- `容器 CPU`
- `容器内存 RSS`
- `Pod 入流量`
- `Pod 出流量`

## 镜像与多环境同步

`grafana-dashboard-init` 改动后，不能只停留在单环境验证，必须同步处理：

- `office`
- `qcloud`
- `qcloud-hk`

对应做法：

1. 构建并推送新的 `grafana-dashboard-init` 镜像
2. 更新 `apps/clusters/office/images-configmap.yaml`
3. 更新 `apps/clusters/qcloud/images-configmap.yaml`
4. 更新 `apps/clusters/qcloud-hk/images-configmap.yaml`

这次任务最终落地的镜像 tag 是：
- `078e52b-20260422-172059`

## 常见问题

### 1. Grafana dashboard UID 过长

如果 dashboard 注入时报：

```text
HTTP error 400: {"message":"uid too long, max 40 characters"}
```

先检查 `make_dashboard_uid(...)` 传入的 suffix 是否过长。

这次的实际修复方式是：
- 从 `setaria-resource-metrics` 缩短为 `setaria-res-metrics`

## 实施注意事项

1. 新增监控相关改动时，`manifests` 和 `grafana-dashboard-init` 都要基于最新 `origin/main` 拉分支。
2. Dashboard 里如果引用 recording rules，要保证对应 rules PR 同轮存在，避免 reviewer 误判为“无数据依赖未满足”。
3. `grafana-dashboard-init` 改动完成后，不要忘记把镜像 tag 同步到三边环境。

## 相关任务归档

- [2026-04-22 setaria 数据采集](../90-task-archive/2026/2026-04-22-setaria-metrics.md)
