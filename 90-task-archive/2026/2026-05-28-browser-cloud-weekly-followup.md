# browser 云平台专项本周进展 - 2026-05-28

## TL;DR

1. 本周云平台主线集中在 qcloud 多地域容量治理：spot/normal 扩容逻辑从 `watermark-monitor + SCF` 迁移到 `qcloud-spot-capacity-controller`，并补齐水位线指标、节点调度、spot 兜底和 Grafana 观测。
2. API 安全和 gateway 稳定性继续收口：完成 API key 脱敏、downloads 404 修复线索确认、CLS/tccli 排障 runbook、跨地容灾方案梳理。
3. 最大风险是多地域容灾尚未落地，南京 / 北京目前仍是两套独立 region；下一步应优先做“未指定 region 的新 session 自动 failover”。

## 专项 1：云平台官网

**负责人**：李望  
**当前阶段**：M1 启动  
**结构来源**：[browser 云平台专项信息](https://www.notion.so/36ce7e998fb28003b1c1fde6ac2d399e) / [1.1 云平台官网](https://www.notion.so/36de7e998fb281ff9420de3aee04b2f1)

### 里程碑进度

- `🟡 M1 Playground 最短路径`：进行中。
  - `🟡 Playground 最短路径`：本周 API key 脱敏改造覆盖官网、SDK、gateway、Kong 和 E2E，降低免登录 / 一键进入 Playground 后凭据泄露风险。来源：[2026-05-27 API Key 脱敏改造](./2026-05-27-api-key-redaction.md)
  - `⚪ 注册认证体系`：本周无新增。来源：[1.1 云平台官网](https://www.notion.so/36de7e998fb281ff9420de3aee04b2f1)
  - `⚪ 订阅计费审计`：本周无新增。来源：[1.1 云平台官网](https://www.notion.so/36de7e998fb281ff9420de3aee04b2f1)
- `⚪ M2 商业化闭环`：未开始。

## 专项 2：私有化

**负责人**：王正  
**当前阶段**：M1 进行中（宁德 5/29 Demo 收口窗口）  
**结构来源**：[browser 云平台专项信息](https://www.notion.so/36ce7e998fb28003b1c1fde6ac2d399e) / [1.2 私有化](https://www.notion.so/36de7e998fb281f78cd9da010bf141fc)

### 里程碑进度

- `🟡 M1 宁德 Demo 收口`：进行中。
  - `🟡 宁德 AD 演示`：本周无明确新增交付记录；仍处于 5/29 Demo 收口窗口。来源：[1.2 私有化](https://www.notion.so/36de7e998fb281f78cd9da010bf141fc)
  - `⚪ 主路径对齐验收`：本周无新增。来源：[1.2 私有化](https://www.notion.so/36de7e998fb281f78cd9da010bf141fc)
  - `⚪ 标准交付物`：本周无新增。来源：[1.2 私有化](https://www.notion.so/36de7e998fb281f78cd9da010bf141fc)
- `⚪ M2 标准私有化交付`：未开始。

## 专项 3：服务端开发

**负责人**：李望  
**当前阶段**：M1 收口中，M2 启动准备  
**结构来源**：[browser 云平台专项信息](https://www.notion.so/36ce7e998fb28003b1c1fde6ac2d399e) / [1.3 服务端开发](https://www.notion.so/36de7e998fb28140b3d4f25f2f8b77c7)

### 里程碑进度

- `🟡 M1 单云多 region`：收口中。
  - `🟡 单云多 region`：本周持续推进 qcloud-nanjing / qcloud-beijing / qcloud-hk 多环境 manifests、镜像 tag、gateway 和监控对齐；session-gateway downloads 404 修复线已确认 PR 组。来源：#all 线程、[腾讯云 CLS tccli 查询能力接入总结](./2026-05-27-tencent-cloud-cls-tccli-access.md)
  - `🟡 日志 Trace 审计`：本周沉淀腾讯云 CLS + tccli 查询 runbook，覆盖南京 / 北京 / 香港日志集、topic、按 session_id 排障路径。来源：[2026-05-27 腾讯云 CLS tccli 查询能力接入总结](./2026-05-27-tencent-cloud-cls-tccli-access.md)
- `🟡 M2 多云多地容灾`：启动准备。
  - `🔴 多云多地容灾`：本周已梳理南京 / 北京跨地容灾缺口；当前两地仍是独立 region，缺少 `RegionHealth + RegionPicker + session create fallback`。来源：#跨地容灾:2a359924
  - `🟡 扩缩容与 spot`：本周把水位线计算和原 SCF 扩缩容逻辑迁移进 `qcloud-spot-capacity-controller`，后续将替代 `watermark-monitor`。来源：#扩容逻辑迁移:3d3cf18d
- `⚪ M3 多云容灾与统一调度`：未开始。

## 专项 4：成本监控

**负责人**：李望  
**当前阶段**：M1 进行中（dry_run → 真实购买切换前夜）  
**结构来源**：[browser 云平台专项信息](https://www.notion.so/36ce7e998fb28003b1c1fde6ac2d399e) / [1.4 成本监控](https://www.notion.so/36de7e998fb28113933ec1bd84204afb)

### 里程碑进度

- `🟡 M1 spot controller 基础`：进行中。
  - `🟡 spot controller 基础`：本周继续完善 qcloud spot controller：plan 页面、实例列表、spot 节点浏览器实例数指标、Grafana 面板、kubelet reserved/eviction 配置。来源：[qcloud spot capacity controller](../../03-infra/qcloud-spot-capacity-controller.md)、[spot kubelet 配置修正](./2026-05-25-spot-kubelet-config.md)
  - `🟡 真实购买与补偿`：本周已有 kubeadm join、NotReady spot node 补偿、异常 replay pod 清理和 kubelet 配置验证；真实购买能力仍受 region 启动模板与开关控制。来源：[qcloud spot capacity controller 二期总结](./2026-05-21-qcloud-spot-capacity-controller-phase2.md)、[2026-05-25 replay pod 异常清理](./2026-05-25-replay-pod-cleanup.md)
  - `🟡 实时成本计算`：本周 controller 继续使用 `InquiryPriceRunInstances` / plan / quote 明细记录；页面支持按 region 查看 plan、报价明细、事件流水、实例记录。来源：[qcloud 竞价实例询价控制器](../../03-infra/qcloud-spot-capacity-controller.md)
  - `🟡 成本调度闭环`：本周完成 BrowserInstance 调度策略：正常 `browser-ready=true` 节点优先，`lexmount/spot=true` 节点仅作为兜底；同时新增 spot 节点浏览器实例数指标。来源：[浏览器实例调度策略](../../03-infra/browser-scheduling-spot-fallback-2026-05-21.md)、[Spot 节点浏览器实例上报](../../03-infra/qcloud-spot-controller-spot-node-browser-metrics-2026-05-21.md)
- `🟡 M2 正常实例扩缩容归并`：进行中。
  - `🟡 水位线统一`：本周把机器水位线、node 数量、CPU/内存总量、请求量、使用率等指标从 `watermark-monitor` 迁移到 `qcloud-spot-capacity-controller` 暴露。来源：#扩容逻辑迁移:3d3cf18d
  - `🟡 正常实例扩缩容`：本周参考原 SCF Python 逻辑迁移正常实例扩容/缩容路径，普通实例 `CamRoleName` 最终按 `cos-operator` 配置。来源：#扩容逻辑迁移:3d3cf18d
  - `🔴 执行开关`：`capacity_scaling_enabled` 等执行开关仍需谨慎开启，真实扩缩容前必须确认各 region 启动模板 ID / 版本和运行成本上限。来源：#扩容逻辑迁移:3d3cf18d
- `⚪ M3 全局容量调度`：未开始。

## 本周横向基础设施进展

- `🟡 网络质量探针`：已在 browser-ready 节点通过 DaemonSet 部署 blackbox-exporter，Prometheus/Grafana 展示外网访问成功率和耗时，覆盖 office 与 qcloud 多环境。来源：[网络质量探针接入总结](../../03-infra/network-quality-probe-2026-05.md)
- `🟡 API key 脱敏`：完成 SDK / gateway / Kong / E2E 链路改造，API key 改走 header，服务端长期只保留 hash。来源：[2026-05-27 API Key 脱敏改造](./2026-05-27-api-key-redaction.md)
- `🟡 排障能力`：CLS/tccli runbook 已覆盖三地日志集、topic 和按 session_id 查询方法。来源：[2026-05-27 腾讯云 CLS tccli 查询能力接入总结](./2026-05-27-tencent-cloud-cls-tccli-access.md)

## 风险与下周动作

- `🔴 跨地容灾未闭环`：南京 / 北京还没有自动 failover，建议下周优先做 `RegionHealth cache + RegionPicker + 未指定 region 创建 fallback`。
- `🟡 扩缩容开关仍需保护`：正常实例扩缩容代码路径已迁移，但真实开关开启前必须补齐启动模板、成本上限、回滚预案。
- `🟡 watermark-monitor 去留`：扩容逻辑已经迁移方向明确，后续应继续收敛 Prometheus 指标和 manifests，避免两个组件重复上报或重复决策。
- `🟡 多地域 E2E`：API key 脱敏、downloads、custom_image_id、gateway API 需要在 qcloud-nanjing / qcloud-beijing / qcloud-hk 继续固化自动化验证。
