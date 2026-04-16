# 发布总览

## 目的

这篇文档用于说明后端平台当前最核心的发布链路，以及发布时应先看哪个仓库、改哪个位置、怎么把变更落到环境里。

## 当前核心发布对象

当前最常见的发布对象主要有两类：

1. `browser-manager`
2. `browser-operator`（来自 `k8s-chrome-daemon`）

对应涉及的核心仓库是：

1. `demo-nodejs-backend/browser-manager`
2. `k8s-chrome-daemon`
3. `lexmount-k8s-manifests`

可以把它们理解成：

1. 代码仓库负责产出镜像
2. `lexmount-k8s-manifests` 负责把镜像和配置真正发布到环境

## 发布基本原则

### 1. 永远不要直接在本地 `main` 上开发

开发前必须先同步最新 `origin/main`，再从干净的本地 `main` 拉开发分支。

### 2. 不要直接推送 `main`

所有需求改动都应先在开发分支完成，再通过 PR 合并到主线。

### 3. 多仓库联动时使用统一分支名

如果同一个需求涉及多个仓库，所有仓库都应使用同一套开发分支名，便于联调和追溯。

当前约定格式：

```text
wangli_dev_<日期>
```

例如：

```text
wangli_dev_20260413
wangli_dev_20260413_1
```

## 发布链路总图

可以用下面这个顺序理解当前发布流程：

1. 在代码仓库完成开发
2. 构建并推送镜像
3. 把新镜像地址回填到 `lexmount-k8s-manifests`
4. 在目标环境目录执行 `kubectl apply -k`
5. 检查 rollout 和运行状态
6. 做最小可用验证

## 发布时的仓库分工

### `browser-manager`

需要关注：

1. 代码修改是否已经完成
2. 镜像是否构建并推送成功
3. `lexmount-k8s-manifests` 中对应环境的镜像配置是否已更新

### `k8s-chrome-daemon`

需要关注：

1. controller 代码是否已经完成
2. `browser-operator` 镜像是否已构建并推送
3. manifest 中的 `browser-operator-image` 是否已同步
4. 如有配置项新增，是否已同步到 configmap / deployment / overlay

### `lexmount-k8s-manifests`

需要关注：

1. office / qcloud / qcloud-hk 的镜像 tag 是否同步
2. configmap 是否同步了新增开关
3. kustomization 是否已完成映射
4. 目标环境是否已执行 `kubectl apply -k`

## 常见发布路径

### 路径 1：只更新镜像

适用于：

1. 代码改动不涉及新配置项
2. 只需要回填新的镜像 tag

最小步骤：

1. 在代码仓库执行 `make push`
2. 复制目标镜像地址
3. 更新目标环境 `images-configmap.yaml`
4. 执行 `kubectl apply -k`
5. 检查 rollout

### 路径 2：镜像和配置同时更新

适用于：

1. 新增环境变量
2. 新增 configmap 字段
3. 新增 overlay 映射

最小步骤：

1. 在代码仓库执行 `make push`
2. 更新 `images-configmap.yaml`
3. 更新 `config.yaml` / `manager.yaml`
4. 更新各集群 `app-config.yaml`
5. 更新各集群 `kustomization.yaml`
6. 执行 `kubectl apply -k`
7. 验证 Deployment 实际环境变量和镜像

## office 环境为什么最重要

当前知识和流程里，`office` 是默认测试环境。

原因：

1. 文档最完整
2. 本机可直接通过 `kubectl` 访问
3. 大多数发布验证流程都以 `office` 为第一落点

因此，大多数变更建议都先在 `office` 验证成功，再考虑其他环境同步。

## 发布后的最小验证

无论发布哪个服务，至少应做 3 类检查：

### 1. rollout 检查

确认 Deployment 或相关 Pod 已成功滚动完成。

### 2. 镜像与配置检查

确认实际运行中的镜像 tag 和环境变量是预期值。

### 3. 业务最小验证

确认服务不是“只启动成功但功能失效”。

例如：

1. `browser-manager` 的接口可调用
2. `browser-operator` 能成功创建实例
3. 相关 session / context / 下载链路可用

## 推荐阅读顺序

如果要继续深入发布细节，建议按下面顺序读：

1. `00-overview/environments.md`
2. `02-operations/release-browser-operator.md`
3. 后续补充的 `02-operations/release-browser-manager.md`

## 当前结论

当前后端平台发布流程的核心认知是：

1. 代码仓库产出镜像
2. `lexmount-k8s-manifests` 负责最终落地
3. `office` 是默认验证环境
4. 发布完成不等于结束，必须跟上 rollout、配置和业务验证
