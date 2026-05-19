# qcloud 竞价实例询价控制器

## 背景

这轮工作的目标是为腾讯云 CVM 竞价实例做一个独立控制器：给定目标 CPU / 内存、最小机型限制和购买上限，自动枚举候选竞价机型、询价、选择最便宜的容量组合，并把 plan、报价明细、事件流水和实例记录落到 PostgreSQL。

最初考虑过集成到 `watermark-monitor`，后续明确单独放到 `qcloud-k8s-tool`，避免和水印监控职责混在一起。

## 代码仓库

代码仓库：`qcloud-k8s-tool`

目录：`qcloud-spot-capacity-controller/`

已合入 PR：

- https://github.com/lexmount/qcloud-k8s-tool/pull/14 - 新增 qcloud spot capacity controller
- https://github.com/lexmount/qcloud-k8s-tool/pull/16 - 修正 PostgreSQL 连接默认 SSL 行为
- https://github.com/lexmount/qcloud-k8s-tool/pull/17 - 增强询价过滤诊断日志
- https://github.com/lexmount/qcloud-k8s-tool/pull/18 - 增加 plan Web 查看页
- https://github.com/lexmount/qcloud-k8s-tool/pull/19 - 支持多地域、多地域独立 plan id

## manifests 仓库

manifests 仓库：`lexmount-k8s-manifests`

基础资源目录：`apps/qcloud-spot-capacity-controller/`

当前只在 `qcloud-nanjing` 集群部署这个负载；但控制器进程内部同时处理：

- `ap-nanjing`
- `ap-beijing`

已合入 PR：

- https://github.com/lexmount/lexmount-k8s-manifests/pull/454 - 新增 qcloud spot capacity controller manifests
- https://github.com/lexmount/lexmount-k8s-manifests/pull/485 - 配置多地域 spot controller
- https://github.com/lexmount/lexmount-k8s-manifests/pull/487 - 修正 Beijing `image_id`
- https://github.com/lexmount/lexmount-k8s-manifests/pull/488 - 更新 qcloud-nanjing 相关 secret patch

## 当前部署形态

### 只部署在 qcloud-nanjing

用户明确要求：这个负载只在 `qcloud-nanjing` 部署。

虽然它会查询南京和北京两个地域，但 Kubernetes Deployment 只放在 qcloud-nanjing 集群里。

### Service / Web 页面

控制器暴露 HTTP 服务：

- 容器监听：`:3000`
- K8s Service：`qcloud-spot-capacity-controller.system.svc.cluster.local`
- Service port：`80`

页面能力：

- 按地域选择 `nanjing` / `beijing`
- 倒序展示最新 plan id
- 输入 plan id 查看详情
- 展示 plan 详情、报价明细、事件流水、实例记录

注意：页面里的 plan id 是地域内独立递增的 `region_plan_id`，不是数据库全局 `id`。

## 配置模型

### 使用 `SPOT_REGION_CONFIGS`

早期设计使用大量 env，例如每新增一个 region 就要增加一批环境变量。用户指出这个模式扩展性差，因此改为单个 JSON 配置：`SPOT_REGION_CONFIGS`。

当前 manifests 中的核心配置位于：

- `apps/qcloud-spot-capacity-controller/configmap.yaml`

当前结构：

```json
{
  "regions": [
    {
      "region": "ap-nanjing",
      "dry_run": true,
      "target_cpu": 64,
      "target_memory_gb": 128,
      "min_instance_cpu": 8,
      "min_instance_memory_gb": 16,
      "max_instance_count": 20,
      "system_disk_type": "CLOUD_PREMIUM",
      "system_disk_size_gb": 50,
      "image_id": "img-hxgbrryk"
    },
    {
      "region": "ap-beijing",
      "dry_run": true,
      "target_cpu": 64,
      "target_memory_gb": 128,
      "min_instance_cpu": 8,
      "min_instance_memory_gb": 16,
      "max_instance_count": 20,
      "system_disk_type": "CLOUD_PREMIUM",
      "system_disk_size_gb": 50,
      "image_id": "img-6ev2slv1"
    }
  ]
}
```

### 重要字段

- `region`：腾讯云地域，例如 `ap-nanjing`、`ap-beijing`。
- `dry_run`：是否只询价和落库，不真正调用 `RunInstances` 购买。
- `target_cpu`：目标总 CPU 核数。
- `target_memory_gb`：目标总内存 GB。
- `min_instance_cpu`：候选机型单台最小 CPU。
- `min_instance_memory_gb`：候选机型单台最小内存 GB。
- `max_instance_count`：单个方案最多购买多少台。
- `allowed_instance_families`：可选机型族白名单；为空表示不限制。
- `excluded_instance_types`：排除的具体机型列表。
- `image_id`：腾讯云 `InquiryPriceRunInstances` / `RunInstances` 都需要合法镜像 ID。南京当前是 `img-hxgbrryk`，北京当前是 `img-6ev2slv1`。
- `system_disk_type`：系统盘类型，当前为 `CLOUD_PREMIUM`。
- `system_disk_size_gb`：系统盘大小，当前为 `50`。

### POSTGRES_DSN

`POSTGRES_DSN` 从 Secret 读取。

当前 PostgreSQL 是同一个 PG 实例，不同业务用不同 database。这里曾经遇到：

```text
pq: SSL is not enabled on the server
```

修复方式不是要求 manifests 的 DSN 额外加 `sslmode=disable`，而是在代码侧改用 `pgx` 驱动，跟其他服务连接同一个 PG 的行为对齐，避免 lib/pq 默认 sslmode 带来的差异。

## 腾讯云鉴权

鉴权逻辑要求对齐 `k8s-watermark-monitor`：

- 使用 `META_CAM_URL`
- 通过机器 role 从腾讯云 metadata 获取临时 key
- 使用 TC3-HMAC-SHA256 签名
- 签名逻辑参考 `k8s-watermark-monitor` 的 `generateTC3Signature`
- 获取临时凭证逻辑参考 `getCAMAuthRequestInfo`

不要在配置里写固定 SecretId / SecretKey。

## 选型流程

每轮 reconcile 的核心流程：

1. 调用 `DescribeSpotInstanceConfigs` 获取当前地域竞价实例配置。
2. 根据约束过滤候选机型：
   - `Status == SELL`
   - 单台 CPU 不低于 `min_instance_cpu`
   - 单台内存不低于 `min_instance_memory_gb`
   - 不在 `excluded_instance_types`
   - 如果配置了 `allowed_instance_families`，机型族必须命中白名单
3. 根据目标 CPU / 内存计算需要购买的台数：
   - `ceil(target_cpu / instance_cpu)`
   - `ceil(target_memory_gb / instance_memory_gb)`
   - 取两者较大值
4. 如果需要台数超过 `max_instance_count`，跳过。
5. 对候选方案调用 `InquiryPriceRunInstances` 精确询价。
6. 根据价格排序，选择最低总小时价方案。
7. 同价时依次按 `over_cpu`、`over_memory_gb`、`instance_type` 排序。
8. 落库 plan / quotes / events。
9. 如果 `dry_run=true`，跳过真正购买。
10. 如果 `dry_run=false`，调用 `RunInstances`，然后写入实例记录。

## 价格字段解释

腾讯云 `InquiryPriceRunInstances` 的返回示例：

```json
{
  "InstancePrice": {
    "Discount": 7,
    "UnitPrice": 9.11,
    "ChargeUnit": "HOUR",
    "UnitPriceDiscount": 0.68
  }
}
```

这里的 `Discount: 7` 不是 7 折，而更接近 7%。从数值上看：

```text
0.68 / 9.11 ~= 0.0746
```

所以实际用于比较的价格应该优先使用 `UnitPriceDiscount`。当前代码的小时单价选择顺序是：

1. `UnitPriceDiscount`
2. `UnitPrice`
3. `UnitPriceSecondStep`
4. `DiscountPrice`

总小时价是：

```text
hourly_price = hourly_unit_price * instance_count
```

## over_cpu / over_memory_gb

`over_cpu` 表示选中方案总 CPU 超出目标 CPU 的数量。

`over_memory_gb` 表示选中方案总内存超出目标内存的 GB 数。

例如目标是 64C / 128GB，某机型 16C / 32GB，需要 4 台：

- 总 CPU：64，`over_cpu=0`
- 总内存：128，`over_memory_gb=0`

如果某方案总内存是 160GB，则 `over_memory_gb=32`。

这些字段用于同价情况下优先选择资源浪费更少的方案。

## 失败诊断

曾经出现过：

```text
no eligible spot instance quote found
```

后续补了诊断信息，错误里会包含：

- `total_configs`：腾讯云返回的配置总数
- `ineligible`：过滤原因计数，例如 `cpu_below_min`、`memory_below_min`、`not_sell`
- `count_over_limit`：需要购买台数超过上限的候选数
- `quote_attempts`：实际发起询价次数
- `quote_errors`：询价失败数量
- `first_quote_errors`：前 5 个询价错误样本

`first_quote_errors` 只保留前 5 个，是为了避免日志过长；它只用于诊断，不影响完整选型逻辑。

## ImageId 经验

腾讯云 `InquiryPriceRunInstances` 询价接口也会校验 `ImageId`。

当缺少 `ImageId` 时，曾经报错：

```text
MissingParameter: The request is missing the required parameter `ImageId`.
```

虽然询价阶段不真正创建机器，但接口参数仍要求提供合法镜像 ID。真正购买后可以再重装系统，但询价请求本身仍需要填一个合法 `ImageId`。

当前值：

- 南京：`img-hxgbrryk`
- 北京：`img-6ev2slv1`

## 多地域与 plan id

控制器支持一个进程同时处理多个地域。

当前每轮 ticker 会并发执行多个 region reconciler，避免地域之间串行阻塞。

数据库里有两个 id：

- `id`：数据库全局主键。
- `region_plan_id`：按 region 独立递增的 plan id。

Web 页面和 API 对用户展示 / 查询时使用 `region_plan_id`，并要求带 region：

- `/api/plans?region=ap-nanjing`
- `/api/plans/6?region=ap-nanjing`
- `/api/plans?region=ap-beijing`
- `/api/plans/1?region=ap-beijing`

这样南京和北京可以各自从 plan id 1 开始，不互相影响。

## 数据库表

主要表：

- `spot_capacity_plans`
- `spot_capacity_quotes`
- `spot_capacity_instances`
- `spot_capacity_events`

关系：

- quotes 通过 `plan_id` 关联 plans，plan 删除时级联删除。
- instances 通过 `plan_id` 关联 plans，plan 删除时级联删除。
- events 通过 `plan_id` 关联 plans，plan 删除时置空。

## plan 保留策略

用户要求 plan 记录最多保留 14 天。

当前配置：

```yaml
PLAN_RETENTION_DAYS: "14"
```

控制器在两个时机做清理：

- 进程启动后首次 reconcile 前
- 每轮 ticker reconcile 前

清理逻辑删除 `spot_capacity_plans.created_at` 早于保留期的 plan。quotes / instances 会随 plan 级联删除，events 保留但 `plan_id` 置空。

## active plan 防重复购买

每个 region 在购买前会检查是否存在近期 active plan。

active 状态包括：

- `quoted`
- `purchasing`
- `purchased`

时间窗口：30 分钟。

如果存在近期 active plan，则跳过本轮购买，避免频繁重复购买。

## 调度策略

部署策略参考 `watermark-monitor`，要求调度到 control-plane/master 节点。

关键配置：

- `nodeSelector`：优先匹配 `node-role.kubernetes.io/control-plane`
- `nodeAffinity`：同时兼容 `control-plane` 和旧版 `master` 标签
- `tolerations`：容忍 control-plane/master 的 `NoSchedule` 和 `NoExecute` taint

含义：

- `NoSchedule` toleration：允许 Pod 调度到有对应 taint 的 control-plane/master 节点。
- `NoExecute` toleration：如果节点有对应 taint，Pod 不会因为这个 taint 被驱逐。
- affinity 的 `requiredDuringSchedulingIgnoredDuringExecution`：调度时必须匹配 control-plane 或 master 标签；运行后标签变化不会自动驱逐。

## qcloud-nanjing Kong 暴露

用户要求 qcloud-nanjing Kong 指向这个服务，域名：

```text
spot-nanjing.local.lexmount.net
```

用途是通过内网域名访问 plan Web 页面和 API。

## 当前镜像

当前 qcloud-nanjing manifests 中的镜像：

```text
lexmount.tencentcloudcr.com/cloud/qcloud-spot-capacity-controller:de0b45d-20260518-212503
```

这版包含：

- 多地域配置
- 并发 region reconcile
- region-local plan id
- plan Web 页面
- 14 天保留策略
- 询价诊断
- pgx PostgreSQL 连接

## 发布和运行观察

已在 qcloud-nanjing 发布过。

典型启动日志：

```text
qcloud spot capacity controller started regions=[ap-nanjing ap-beijing] check_interval=5m0s plan_retention_days=14
qcloud spot capacity region configured region=ap-nanjing ...
qcloud spot capacity region configured region=ap-beijing ...
start reconcile region=ap-nanjing
start reconcile region=ap-beijing
```

如果 `dry_run=true` 且选型成功，会出现：

```text
dry-run enabled, skip RunInstances plan_id=...
```

## 注意事项

1. 新增地域时，优先改 `SPOT_REGION_CONFIGS`，不要再新增一批平铺 env。
2. 每个地域都需要填合法 `image_id`，否则询价接口可能直接失败。
3. `dry_run` 上线初期应保持 `true`，确认 plan / quotes / events 正常后再考虑打开购买。
4. 询价接口调用量会随候选机型数增长；当前每轮周期是 300 秒，并有 active plan 防重复购买。
5. 腾讯云询价接口是否有明确 QPS 限制，之前没有在文档里找到直接结论；如果后续扩大地域和候选范围，需要加限速或分批询价。
6. 不要把固定腾讯云 Secret 写进配置，继续使用机器 role + metadata 临时凭证。
7. 如果要真正购买，需要补齐对应 region 的 VPC、Subnet、安全组、UserData 等运行参数，并确认 `RunInstances` 权限。
