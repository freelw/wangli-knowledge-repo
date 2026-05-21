# QCloud Spot Capacity Controller 页面调整

日期：2026-05-21

## 背景

`qcloud-spot-capacity-controller` 已经提供 Web 页面用于查看竞价实例容量计划，包括 plan、报价明细、事件流水和实例记录。

后续使用中发现页面信息层级需要调整：用户首先关心当前还活跃的实例，其次才是某个 plan 的报价与执行细节。原页面的“实例记录”更偏 plan 详情，容易误解成全局当前实例列表。

## 目标

页面需要明确区分两类实例信息：

1. 当前实例列表：全局当前活跃实例，放在页面最上方，紧跟总体状态。
2. Plan 实例记录：某个 plan 下产生的实例记录，仍然留在 plan 详情区域。

同时修复页面刷新与用户点击 plan 之间的竞态，避免用户刚选中的 plan 被刷新逻辑清空。

## 代码改动

仓库：`qcloud-k8s-tool`

PR：<https://github.com/lexmount/qcloud-k8s-tool/pull/27>

主要改动：

- 新增页面顶部“当前实例列表”。
- 当前实例列表查询所有活跃实例：
  - `status = created`
  - `status = k8s_joined`
- 保留 plan 详情中的“Plan 实例记录”，用于查看选中 plan 关联的实例。
- 新增 API：`/api/instances?region=...`。
- 调整页面模块顺序，当前实例列表放在 plan 相关信息前面。
- 修复刷新竞态：加载最新 plan 列表时，不再无条件清空用户已经选中的 plan 详情。

相关镜像：

- `qcloud-spot-capacity-controller:15c68d2-20260521-144039`

## Region Data Plane Gateway 改动

仓库：`demo-nodejs-backend`

PR：<https://github.com/lexmount/demo-nodejs-backend/pull/187>

主要改动：

- `region-data-plane-gateway` 增加 `/v1/spot/instances` 转发。
- 下游代理到 capacity-controller 的 `/api/instances`。
- 让外部入口可以读取“当前实例列表”。

相关镜像：

- `region-data-plane-gateway:79c90d5-20260521-143605`

## Manifests 改动

仓库：`lexmount-k8s-manifests`

PR：<https://github.com/lexmount/lexmount-k8s-manifests/pull/512>

主要改动：

- 更新 `qcloud-nanjing` / `qcloud-beijing` 的 `qcloud-spot-capacity-controller-image`。
- 更新 `qcloud-nanjing` / `qcloud-beijing` 的 `region-data-plane-gateway-image`。

## 页面语义

页面上现在存在两类实例列表：

- 当前实例列表：
  - 全局视角。
  - 展示当前所有活跃实例。
  - 与当前选中的 plan 无关。

- Plan 实例记录：
  - Plan 详情视角。
  - 只展示当前 plan 创建或关联的实例记录。
  - 用于追溯某次容量计划的执行结果。

这个区分需要保留，避免后续把全局实例列表误绑定到 plan 上。

## 验证

已验证项：

- qcloud-k8s-tool 构建通过。
- demo-nodejs-backend 相关路由改动通过基础校验。
- manifests kustomize 校验通过。
- PR 合入后用户确认页面调整完成。

## 后续注意事项

- 如果实例状态枚举扩展，需要同步检查“当前实例列表”筛选条件。
- 如果新增地域，`/api/instances?region=...` 和 gateway 转发需要保持 region 参数透传。
- 页面顶部“当前实例列表”是全局活跃实例视图，不应随 plan 选择变化。
