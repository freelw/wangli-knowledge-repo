# 2026-04-13 browser-operator CFS 开关与挂载优化

## 背景

浏览器实例当前对 CFS / PVC 挂载依赖较重，一旦挂载链路异常，实例可用性会受影响。同时，日志落宿主机和 session 信息缺失也影响定位和运行稳定性。

## 任务目标

1. 调整日志卷，避免继续落宿主机 `hostPath`
2. 创建 Chrome 时注入 `ACE_SESSION_ID`
3. 增加 `ENABLE_CFS_MOUNTS` 开关
4. 在 `office` 环境完成发布和验证

## 涉及仓库

1. `k8s-chrome-daemon`
2. `lexmount-k8s-manifests`

## 关键改动

### `k8s-chrome-daemon`

1. 主容器新增 `ACE_SESSION_ID`
2. `/log` 从 `hostPath` 改为 `emptyDir`
3. 引入 `ENABLE_CFS_MOUNTS`
4. 开关关闭时，download / replay 改走本地卷
5. 开关关闭时，不再挂 context / extension / user-data PVC

### `lexmount-k8s-manifests`

1. 把 `ENABLE_CFS_MOUNTS` 接入 manager 配置
2. 在多个环境 overlay 中增加 `enable-cfs-mounts`
3. 同步 browser-operator 镜像 tag 到多个环境

## 验证方式

已在 `office` 环境验证：

1. rollout 成功
2. `ENABLE_CFS_MOUNTS=true` 时默认路径正常
3. `ENABLE_CFS_MOUNTS=false` 时降级路径正常
4. 验证完成后已恢复 `office` 为 `ENABLE_CFS_MOUNTS=true`

## 产出的长期知识

这次任务已沉淀进当前知识库：

1. `02-operations/release-browser-operator.md`
2. `02-operations/office-runbook.md`

## 分支与 PR

### 分支

1. `k8s-chrome-daemon`: `wangli_dev_20260409_1`
2. `lexmount-k8s-manifests`: `wangli_dev_20260413_1`

### PR

1. <https://github.com/lexmount/k8s-chrome-daemon/pull/new/wangli_dev_20260409_1>
2. <https://github.com/lexmount/lexmount-k8s-manifests/pull/new/wangli_dev_20260413_1>

## 结论

1. browser-operator 已支持 CFS 挂载开关
2. 日志卷和 session 环境变量问题已修正
3. `office` 默认路径与降级路径都已验证通过
4. 这次任务为后续浏览器实例运行稳定性提供了更强的降级能力
