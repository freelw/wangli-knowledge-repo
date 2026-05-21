# qcloud spot capacity controller 二期总结

## 背景

这期目标是把 `qcloud-spot-capacity-controller` 从南京单点、多地域集中配置，调整为更贴近真实 qcloud 多地域部署的形态，并补齐竞价实例购买、加入 Kubernetes、回收补偿、跨地域页面查看和发布注意事项。

涉及仓库：

- `qcloud-k8s-tool`
- `demo-nodejs-backend`
- `lexmount-k8s-manifests`

核心环境：

- `qcloud-nanjing`
- `qcloud-beijing`

## 主要 PR

代码侧：

- `qcloud-k8s-tool` PR #20：拆分 qcloud spot controller 地域配置，支持北京真实购买、TAT 执行 kubeadm join、spot termination monitor。
- `qcloud-k8s-tool` PR #22：恢复南京统一页面的多地域查看能力，并通过 region-data-plane-gateway 读取远端地域 plan。
- `qcloud-k8s-tool` PR #23：补 `max_price`，同时在询价和购买请求里带 `InstanceMarketOptions.SpotOptions.MaxPrice`。
- `qcloud-k8s-tool` PR #24：补充 spot 询价进度日志。
- `qcloud-k8s-tool` PR #25：补齐被回收实例和 NotReady spot node 的补偿后处理。
- `demo-nodejs-backend` PR #184：region-data-plane-gateway 暴露 spot plan list/detail 代理接口。
- `demo-nodejs-backend` PR #185：修正 region-data-plane-gateway 访问 qcloud-spot-capacity-controller 的默认 Service URL。

manifests 侧：

- `lexmount-k8s-manifests` PR #491：部署 qcloud-beijing spot controller / termination monitor 资源。
- `lexmount-k8s-manifests` PR #503：南京统一页面配置多地域查看，并更新 controller / data-plane 镜像。
- `lexmount-k8s-manifests` PR #504：更新 region-data-plane-gateway 镜像。
- `lexmount-k8s-manifests` PR #505：给北京配置 `max_price`。
- `lexmount-k8s-manifests` PR #506：给南京配置 `max_price` 并更新询价进度日志镜像。
- `lexmount-k8s-manifests` PR #508：更新 NotReady 补偿逻辑 controller 镜像。
- `lexmount-k8s-manifests` PR #511：给 qcloud-spot-capacity-controller 补 `nodes list` RBAC。

## 当前架构

### controller 部署

`qcloud-spot-capacity-controller` 在每个 qcloud 地域以本地域控制器形态运行。购买和 Kubernetes join 都由本地域 controller 负责，避免南京 controller 直接操作北京集群节点。

当前页面访问仍从南京入口走：

```text
spot-nanjing.local.lexmount.net
```

南京页面可以切换查看多个 region：

- `ap-nanjing` / `nanjing-1`
- `ap-beijing` / `beijing-1`

南京本地 plan 直接读本地 DB。北京 plan 通过 `region-data-plane-gateway` 转发到北京的 `qcloud-spot-capacity-controller`。

### region-data-plane-gateway

新增接口：

```text
GET /v1/spot/plans?region=...
GET /v1/spot/plans/:id?region=...
```

北京 region-data-plane-gateway 代理到本地：

```text
http://qcloud-spot-capacity-controller.system.svc.cluster.local
```

注意 Service 是 `80 -> 3000`，因此默认 base URL 应使用 Service 80 端口，不应写容器端口 `3000`。

## 购买和选型逻辑

每轮 reconcile：

1. 查询已购买实例状态，处理被回收实例。
2. 扫描 `lexmount/spot=true` 且 NotReady 的 Kubernetes node，确认 CVM 是否被回收。
3. 清理过期 plan。
4. 计算当前 active capacity。
5. 如果容量不足，查询腾讯云竞价实例配置。
6. 按配置过滤和询价。
7. 选择最便宜方案。
8. 如果 `dry_run=false`，调用 `RunInstances` 购买。
9. 等待实例 RUNNING。
10. 查询实例实际价格并写入 DB。
11. 通过 TAT 在新实例上执行 kubeadm join。
12. 给 Kubernetes node 设置标签和 taint。

## 关键配置

### `max_hourly_price_rmb` 和 `max_price`

这两个字段不是同一层面的限制。

`max_hourly_price_rmb` 是我们自己的总价预算过滤。代码会用询价返回的小时单价乘以实例数量，得到这一组实例的总小时价，超过这个值就丢掉候选方案。

`max_price` 是传给腾讯云 CVM spot API 的 `InstanceMarketOptions.SpotOptions.MaxPrice`，属于腾讯云竞价请求的最高出价参数。此前真实调用 `InquiryPriceRunInstances` 时遇到过：

```text
MissingParameter: The request is missing a required parameter `MaxPrice`.
```

因此不能只保留 `max_hourly_price_rmb`。当前两者都配置成 `1` 的含义是：腾讯云出价上限为 1，同时我们自己的总小时价预算也不超过 1 RMB。

### 购买上限

北京开启真实购买时的约束：

- 总 CPU 目标控制在 10 核以内。
- 总消耗每小时不超过 1 RMB。
- 默认 zone 是 `ap-beijing-6`。

南京默认 zone 是 `ap-nanjing-1`。

## Kubernetes join

购买成功后，通过腾讯云 TAT 在实例上执行 kubeadm join。join 后需要确认 node 具备：

```text
label: lexmount/spot=true
label: cvm-instance-id=<instance_id>
taint: browser-worker-node=true:NoSchedule
```

`cvm-instance-id` 用于后续从 Kubernetes node 反查 CVM 实例。

节点名称在实际集群里是小写 `vm-ins-...`。补偿逻辑不能固定拼 `VM-<instance_id>`。当前逻辑为：

1. 优先按 label 查 node：`lexmount/spot=true,cvm-instance-id=<instance_id>`。
2. 找不到再尝试 `vm-<instance_id>`、`VM-<instance_id>`、`<instance_id>`。

## 被回收实例补偿

### managed instance 补偿

定期扫描 DB 中状态为：

- `created`
- `k8s_joined`

的已购买实例，调用腾讯云接口检查 CVM 是否还存在。如果实例 missing 或处于回收状态，则走补偿后处理。

补偿后处理包括：

1. 标记 `spot_capacity_instances.status = terminated`。
2. 写 `managed_instance_terminated_compensation` 事件。
3. drain 并删除 Kubernetes node。
4. 写成功或失败事件。

为了避免跨周期重复插入 `managed_instance_terminated_compensation`，写事件前会检查同一 `event_type + instance_id` 是否已经存在。事件已存在时，只重试 node drain/delete，不重复写 terminated compensation 事件。

### NotReady spot node 补偿

定期列出 Kubernetes 中：

```text
lexmount/spot=true
```

且 `Ready != True` 的 node。对每个 NotReady node：

1. 从 `cvm-instance-id` label 获取实例 ID。
2. 如果 label 缺失，兼容从 `vm-` / `VM-` node name 反推。
3. 查询 CVM 状态。
4. 如果 CVM 仍存在且不是回收态，不处理。
5. 如果 CVM missing 或已回收，复用 managed instance 的补偿后处理。

为避免同一轮 reconcile 中 managed instance 和 NotReady node 重复处理同一个实例，`reconcileManagedInstances` 会返回本轮已处理的 instance id set，NotReady 扫描会跳过这些实例。

### 补偿周期

补偿周期复用 controller 的 reconcile 周期：

```text
CHECK_INTERVAL_SECONDS=300
```

也就是 5 分钟一次。

进程启动时会立即跑一次，之后每 300 秒跑一次。

## drain/delete 日志

`drainAndDeleteSpotNode` 已补充详细日志：

- 开始 drain/delete。
- 获取 node 失败。
- 非 `lexmount/spot=true` node 拒绝处理。
- cordon 成功/失败。
- node 上 pod 总数。
- 每个 pod 是跳过 DaemonSet、跳过 deleting、开始 evict、evict 成功提交、evict 失败继续。
- pod 驱逐阶段汇总。
- 开始删除 node。
- delete node 失败。
- drain/delete 完成。

这能帮助判断补偿流程卡在 Kubernetes 的哪一步。

## RBAC

qcloud-spot-capacity-controller 需要集群级别权限：

- `nodes`: `get`, `list`, `update`, `delete`
- `pods`: `list`
- `pods/eviction`: `create`

曾经遇到：

```text
nodes is forbidden: User "system:serviceaccount:system:qcloud-spot-capacity-controller" cannot list resource "nodes" in API group "" at the cluster scope
```

原因是 NotReady node 扫描需要 `nodes list`，但 RBAC 只给了 `get/update/delete`。PR #511 已补齐。

## 数据库

主要表：

- `spot_capacity_plans`
- `spot_capacity_quotes`
- `spot_capacity_instances`
- `spot_capacity_events`

重要状态：

- plan `quoted`：已选型，只询价。
- plan `purchasing`：正在购买。
- plan `purchased`：购买并 join 成功。
- plan `purchase_failed`：购买失败。
- plan `k8s_join_failed`：实例购买后 join 失败。
- instance `created`：RunInstances 成功，已写入实例记录。
- instance `k8s_joined`：已加入 Kubernetes。
- instance `k8s_join_failed`：join 失败。
- instance `terminated`：确认实例被回收或终止。

实例实际单价在购买后查询并写入：

- `spot_capacity_instances.hourly_price`
- `spot_capacity_instances.raw_price_json`
- `spot_capacity_instances.price_queried_at`

## 发布注意事项

### qcloud-beijing 数据库

qcloud-beijing 需要有 `spotcontroller` 数据库，否则 controller 启动后会在 DB 连接或 migration 阶段失败。

### 镜像和 manifests

controller 镜像需要同步到对应地域 registry：

- qcloud-nanjing：`lexmount.tencentcloudcr.com/cloud/qcloud-spot-capacity-controller:<tag>`
- qcloud-beijing：`lexmount-bj.tencentcloudcr.com/cloud/qcloud-spot-capacity-controller:<tag>`

manifests 需要同步更新：

- `apps/clusters/qcloud-nanjing/images-configmap.yaml`
- `apps/clusters/qcloud-beijing/images-configmap.yaml`

### apply 后检查

发布后检查：

```bash
kubectl -n system rollout status deployment/qcloud-spot-capacity-controller
kubectl -n system logs -f deployment/qcloud-spot-capacity-controller
```

关键日志：

```text
qcloud spot capacity controller started
qcloud spot capacity region configured
start reconcile
spot quote selection started
spot quote inquiry progress
spot quote selection finished
```

如果启用购买，还要观察：

```text
RunInstances 成功
等待竞价实例进入 RUNNING
生成 kubeadm join command
执行 kubeadm join
```

补偿相关日志：

```text
spot 实例疑似被回收，执行补偿后处理
开始 spot node drain/delete
spot node drain/delete 完成
完成 NotReady spot node 回收补偿扫描
```

## 已知注意点

1. `TagResources` 可能因为 CAM 权限不足失败；当前逻辑会记录 warning 并继续 Kubernetes join。
2. 如果实例已回收但 Kubernetes node 还在，需要依赖 managed instance 补偿或 NotReady node 补偿清理。
3. NotReady node 补偿必须有 `nodes list` RBAC。
4. 目前选型策略仍是相对简单的单机型组合；复杂混合机型组合还未实现。
5. 询价速度受腾讯云 API 调用耗时影响，已补每 20 次 Inquiry 的进度日志。
6. 页面查询北京数据依赖 region-data-plane-gateway 到北京 controller 的网络可达性。
