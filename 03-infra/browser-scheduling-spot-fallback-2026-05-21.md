# 浏览器实例调度策略：正常节点优先，Spot 节点兜底

日期：2026-05-21

## 背景

`k8s-chrome-daemon` 创建 BrowserInstance Pod 时，原本通过 `browser-ready=true` 把浏览器实例调度到浏览器工作节点。

引入腾讯云 spot 实例后，spot node 也带有：

- `browser-ready=true`
- `lexmount/spot=true`

这会导致一个问题：如果只使用硬性 `nodeSelector: browser-ready=true`，正常节点和 spot 节点在调度层面没有优先级差异。浏览器实例可能在正常节点仍可调度时就被放到 spot 节点上。

业务期望是：

1. 正常 browser-ready 节点优先。
2. 只有所有正常节点无法调度时，才使用 `lexmount/spot=true` 且 `browser-ready=true` 的 spot 节点。
3. 原有 `browser-worker-node=true:NoSchedule` taint 容忍逻辑不变。

## Kubernetes 实现方式

Kubernetes 原生支持这种需求，方式是：

- 用硬约束保证 Pod 只能进入浏览器工作节点。
- 用软约束表达“优先不要进入 spot 节点”。

具体策略：

1. `nodeSelector` 或 required node affinity：要求 `browser-ready=true`。
2. `preferredDuringSchedulingIgnoredDuringExecution`：偏好 `lexmount/spot` 不存在或不等于 `true` 的节点。
3. 保留原有 toleration：容忍 `browser-worker-node=true:NoSchedule`。

这样 scheduler 会优先选择正常节点；当正常节点资源不足、不可调度或不满足条件时，仍然可以 fallback 到 spot 节点。

## 代码改动

仓库：`k8s-chrome-daemon`

PR：<https://github.com/lexmount/k8s-chrome-daemon/pull/73>

主要改动：

- BrowserInstance Pod 保持硬性要求 `browser-ready=true`。
- 增加调度偏好，优先选择非 spot 节点。
- 保留 `browser-worker-node` taint 容忍：

```go
Tolerations: []corev1.Toleration{
    {
        Key:      "browser-worker-node",
        Operator: corev1.TolerationOpExists,
        Effect:   corev1.TaintEffectNoSchedule,
    },
}
```

语义：

- 这不是把 spot 节点排除掉。
- 它只是降低 spot 节点的优先级。
- 当正常节点不可调度时，spot 节点仍是合法目标。

## Manifests 改动

仓库：`lexmount-k8s-manifests`

PR：<https://github.com/lexmount/lexmount-k8s-manifests/pull/514>

主要改动：

- 更新 `browser-operator-image` 到包含调度策略的版本。
- 同步环境：
  - `office-nanjing`
  - `office-beijing`
  - `qcloud-nanjing`
  - `qcloud-beijing`
  - `qcloud-hk`

相关镜像：

- `browser-operator:3a1d408-20260521-150003`

## 相关监控联动

这次调度策略改动后，后续又补了两类观测，用来判断 spot 节点承载情况。

### Watermark 节点范围拆分

文档：`watermark-node-scope-metrics-2026-05-21.md`

watermark-monitor 上报两套节点范围：

- `browser_ready_all`：所有 `browser-ready=true` 节点。
- `browser_ready_non_spot`：排除 `lexmount/spot=true` 的正常 browser-ready 节点。

容量和水位判断优先看 `browser_ready_non_spot`，避免 spot 容量掩盖正常节点池压力。

### Spot 节点浏览器实例数

仓库：`qcloud-k8s-tool`

PR：<https://github.com/lexmount/qcloud-k8s-tool/pull/28>

新增指标：

```promql
qcloud_spot_capacity_controller_spot_node_browser_instance_running_total{region="..."}
```

含义：当前调度在 `lexmount/spot=true` 节点上的 Running 浏览器实例 Pod 数。

统计方式：

1. 列出 `lexmount/spot=true` 的节点。
2. 列出 `app=browser-instance` 的 Pod。
3. 只统计 `Running` 且 `spec.nodeName` 在 spot 节点集合里的 Pod。

Manifests PR：<https://github.com/lexmount/lexmount-k8s-manifests/pull/518>

- 新增 Prometheus scrape job：`qcloud-spot-capacity-controller`
- 更新 `qcloud-nanjing` / `qcloud-beijing` 的 controller 镜像。

Grafana PR：

- <https://github.com/lexmount/demo-nodejs-backend/pull/192>
- <https://github.com/lexmount/lexmount-k8s-manifests/pull/521>

新增 dashboard：`qcloud spot capacity 指标`

核心面板：

- `Spot 节点上的浏览器实例数`

## 验证

已完成验证：

- `k8s-chrome-daemon` 代码校验通过。
- manifests 多环境 `kubectl kustomize` 通过。
- 相关 PR 已合入。
- Grafana dashboard-init 镜像已生成并同步到对应环境。

## 设计结论

调度策略应分为两层：

1. 硬约束：必须是浏览器工作节点。
2. 软偏好：正常节点优先，spot 节点兜底。

不要用硬性排除 `lexmount/spot=true`，否则正常节点资源耗尽时无法 fallback 到 spot。

对于后续新增节点池，建议继续遵循这个模式：

- 用 label 表示节点池类型。
- 用 required affinity / selector 表示必须满足的业务条件。
- 用 preferred affinity 表示成本、稳定性、优先级偏好。
- 用 Prometheus 指标验证实际调度是否符合预期。

## 后续注意事项

- 如果 spot 节点增加专用 taint，需要同步补 BrowserInstance Pod toleration。
- 如果 browser-ready 标签被自动维护逻辑修改，需要确认 spot 节点仍保留 fallback 能力。
- 如果看到 spot 上实例数持续大于 0，应结合 `browser_ready_non_spot` 水位判断是否正常节点池容量不足。
