# 2026-05-25 K8s Node 5% 资源预留操作手册

## 任务

整理线上 K8s node 如何预留约 5% CPU / 内存的操作手册，目标是避免 pod `request` 不高但 `limit` 很高时，运行期资源上涨导致 node 被打满或死机。

## 完成内容

- 新增正式操作手册：`03-infra/k8s-node-5-percent-resource-buffer-runbook-2026-05-25.md`。
- 更新 `03-infra/README.md` 索引。
- 提交到 knowledge-repo `main`。

## 关键方案

节点保护分两层：

- 调度层：配置 kubelet `systemReserved` / `kubeReserved`，降低 Node `Allocatable`，避免 scheduler 把节点排满。
- 运行层：配置 kubelet eviction 阈值，例如 `memory.available`，让 kubelet 在危险水位前驱逐 pod，而不是等 node OOM。

## 操作手册覆盖内容

- 推荐 kubelet reserved / eviction 示例配置。
- 线上单节点灰度流程：
  - `cordon`
  - `drain`
  - 备份 kubelet config
  - 修改 kubelet config
  - restart kubelet
  - 验证 node Ready / Allocatable
  - `uncordon`
- 验证命令。
- 回滚步骤。
- 托管 K8s / TKE 节点池注意事项。
- pod 侧配套治理，例如控制 `limit/request` 比例。

## 正式文档

见：`03-infra/k8s-node-5-percent-resource-buffer-runbook-2026-05-25.md`

## 提交

- `63c16a9 docs: add k8s node resource buffer runbook`
