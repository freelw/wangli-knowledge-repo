# Watermark Monitor 节点范围指标拆分

日期：2026-05-21

## 背景

引入腾讯云 spot 实例后，`browser-ready=true` 节点集合里同时包含正常节点和 `lexmount/spot=true` 节点。原来的 watermark-monitor 指标只上报一组 browser-ready 节点容量与使用率，无法区分：

- 全部 browser-ready 节点的总容量
- 排除 spot 后，正常 browser-ready 节点的可用容量

这会影响两个判断：

1. 容量告警和 Grafana 展示无法看出正常节点是否紧张。
2. 自动扩容或水位判断如果把 spot 节点也算入主池，会掩盖正常节点容量不足。

## 目标

同一套 watermark 采集逻辑需要上报两份节点范围：

1. `browser_ready_all`：所有 `browser-ready=true` 节点。
2. `browser_ready_non_spot`：`browser-ready=true` 且排除 `lexmount/spot=true` 的节点。

扩缩容或水位决策优先使用 `browser_ready_non_spot`，Grafana 同时展示两组指标。

## 代码改动

仓库：`qcloud-k8s-tool`

PR：<https://github.com/lexmount/qcloud-k8s-tool/pull/26>

主要改动：

- watermark-monitor 增加节点范围维度 `node_scope`。
- 同一轮采集分别计算：
  - `browser_ready_all`
  - `browser_ready_non_spot`
- 指标上报保留原始含义，但通过 `node_scope` 标签区分范围。
- 水位判断使用非 spot 节点集合，避免 spot 容量影响正常节点池判断。

## Grafana 改动

仓库：`demo-nodejs-backend`

PR：

- <https://github.com/lexmount/demo-nodejs-backend/pull/186>
- <https://github.com/lexmount/demo-nodejs-backend/pull/188>

主要改动：

- Grafana dashboard 增加 `node_scope` 维度展示。
- 后续根据反馈调整 legend，避免曲线都显示成 `total_cpu_cores` / `total_memory_bytes` 这种无区分度名称。
- 最终 legend 使用显式命名，例如：
  - `all_total_cpu_cores`
  - `non_spot_total_cpu_cores`
  - `all_total_memory_bytes`
  - `non_spot_total_memory_bytes`

这样在 Grafana 上可以直接看出指标属于全部节点还是非 spot 节点。

## Manifests 改动

仓库：`lexmount-k8s-manifests`

PR：

- <https://github.com/lexmount/lexmount-k8s-manifests/pull/509>
- <https://github.com/lexmount/lexmount-k8s-manifests/pull/513>

主要改动：

- 更新 watermark-monitor 镜像。
- 更新 grafana-dashboard-init 镜像。
- `grafana-dashboard-init-image` 同步到：
  - `office-nanjing`
  - `office-beijing`
  - `qcloud-nanjing`
  - `qcloud-hk`
- qcloud-hk 在用户明确要求后同步镜像 tag。

相关镜像：

- `k8s-watermark-monitor:c525983-20260521-135658`
- `grafana-dashboard-init:4f63eca-20260521-145728`

## 验证

已验证项：

- 代码侧测试通过。
- manifests kustomize 校验通过。
- office 环境发布并验证 Grafana 展示。
- Grafana 曲线 legend 已调整为包含节点范围前缀，避免同名曲线无法识别。

## 设计结论

后续涉及 browser worker 容量、水位、调度池容量统计时，不应只看一组 `browser-ready=true` 节点指标。至少要明确当前使用的是：

- 全部 browser-ready 节点容量，还是
- 排除 spot 后的正常节点容量。

对于“是否需要扩正常节点池”“正常节点池是否紧张”这类决策，应使用 `browser_ready_non_spot`。

对于“集群总浏览器承载能力”这类观察，可以同时参考 `browser_ready_all`。

## 后续注意事项

- 如果新增其他节点类型标签，需要继续扩展节点范围语义，而不是把所有节点混在一个指标里。
- Grafana legend 必须带上范围含义，避免多条曲线显示相同名称。
- 若 Prometheus 规则后续接入告警，告警表达式也要明确选择 `node_scope`。
