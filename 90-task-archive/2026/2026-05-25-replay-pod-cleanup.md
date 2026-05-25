# 2026-05-25 replay pod 异常清理

## 背景

`browser-replay-processor` 被调度到 spot 节点后，spot 节点被回收时会残留异常 Pod 记录，例如：

- `ContainerStatusUnknown`
- `Evicted`
- `Error`

这些 Pod 不再承载业务，但会长期留在列表里，影响排查和运维判断。因此需要定期清理异常 replay pod。

## 目标

- 由 `qcloud-spot-capacity-controller` 周期性清理异常 replay pod。
- 清理范围只限定在 replay pod 所在 namespace 和 label selector。
- 避免授予 cluster-wide pod delete 权限。
- 清理逻辑不影响 spot 扩容主流程。

## 实现

代码仓库：`qcloud-k8s-tool`

PR：

- https://github.com/lexmount/qcloud-k8s-tool/pull/33

核心变更：

- 增加 replay pod cleanup 配置：
  - `REPLAY_POD_CLEANUP_ENABLED`
  - `REPLAY_POD_CLEANUP_NAMESPACE`
  - `REPLAY_POD_CLEANUP_LABEL_SELECTOR`
  - `REPLAY_POD_CLEANUP_MIN_AGE_SECONDS`
- 增加 `cleanupAbnormalReplayPods`：
  - 分页 list 指定 namespace 下的 replay pod。
  - 只清理 `PodFailed` / `PodUnknown`。
  - 跳过带 `DeletionTimestamp` 的对象。
  - 通过最小 age 避免刚进入异常状态的对象被立即处理。
- cleanup 放在 controller 主循环顶层执行，一轮 reconcile 只执行一次，不按 region 重复执行。
- `Delete` 调用只等待 apiserver 接受删除请求，不等待 Pod 最终从集群消失。

最终镜像：

```text
qcloud-spot-capacity-controller:65cb254-20260525-165533
```

## manifests

仓库：`lexmount-k8s-manifests`

PR：

- https://github.com/lexmount/lexmount-k8s-manifests/pull/542
- https://github.com/lexmount/lexmount-k8s-manifests/pull/543

变更点：

- qcloud 三个集群更新 `qcloud-spot-capacity-controller-image` 到最终镜像。
- 增加 replay pod cleanup env 配置。
- `pods list` 保留在 `ClusterRole` 中。
- `pods delete` 单独放入 namespace-scoped `Role`。
- replay pod 实际位于 `system` namespace，因此最终修正为：
  - `REPLAY_POD_CLEANUP_NAMESPACE=system`
  - cleanup `Role` / `RoleBinding` 也位于 `system` namespace。

注意：最初 namespace 误配置为 `default`，导致 controller 启动正常但清理数量一直为 0；修正后清理范围与实际 replay pod namespace 对齐。

## spot 询价失败明细

同一轮工作中，`qcloud-spot-capacity-controller` 还补充了 spot 询价失败明细记录：

- 新增 `spot_capacity_quote_failures` 表。
- 记录询价失败、价格超过上限、dry-run 失败等候选规格失败原因。
- Web 页面在 plan 详情底部展示失败明细。
- 诊断数据写入失败只记录 warning，不阻断后续 spot 购买流程。

## 校验

代码侧：

```bash
cd qcloud-spot-capacity-controller
go test ./...
git diff --check
```

manifests：

```bash
kubectl kustomize apps/clusters/qcloud-nanjing
kubectl kustomize apps/clusters/qcloud-beijing
kubectl kustomize apps/clusters/qcloud-hk
git diff --check
```

## 后续观察

发布后重点看：

```bash
kubectl -n system get pod -l app=browser-replay-processor
```

如果仍未清理，优先确认：

1. Pod 是否在 `system` namespace。
2. label 是否为 `app=browser-replay-processor`。
3. phase 是否为 `Failed` 或 `Unknown`。
4. age 是否超过 `REPLAY_POD_CLEANUP_MIN_AGE_SECONDS`。
5. controller 日志是否出现 `已删除异常 replay pod` 或 `清理异常 replay pod 失败`。
