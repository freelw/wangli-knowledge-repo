# 镜像同步

## 目的

这篇文档用于说明：

1. 为什么不同环境需要不同镜像 tag / registry
2. 跨环境镜像通常怎么同步
3. 遇到“代码一样但环境行为不同”时该先查什么

## 当前基本事实

平台当前至少有这几类环境镜像路径：

1. `office`：通常使用 `code.lexmount.net/wangli/...`
2. `qcloud`：通常使用 `lexmount.tencentcloudcr.com/cloud/...`
3. `qcloud-hk`：通常使用 HK 区域 registry

这意味着同一个版本在不同环境里，经常不是同一个完整镜像地址。

## 常见同步模式

### 模式 1：直接在目标环境使用对应 registry 的 tag

这是最常见的方式：

1. 为 `office` 构建并推送 office 用镜像
2. 为 `qcloud` 推送腾讯云主区域镜像
3. 为 `qcloud-hk` 推送香港镜像
4. 再分别回填到各环境 `images-configmap.yaml`

### 模式 2：按环境做 tag 约定

历史材料里已有一类典型约定：

1. `office` 用 `dev-{hash}`
2. `qcloud` 用 `cn-{hash}`
3. `qcloud-hk` 用 `com-{hash}`

这类约定的重点不是名字本身，而是：

1. 一眼看出镜像属于哪个环境
2. 降低把错误 tag 回填到错误环境的概率

### 模式 3：跨 registry 拉取后再推送

当同一产物需要在不同 registry 复用时，常见流程是：

1. `docker pull` 源镜像
2. `docker tag` 为目标 registry 地址
3. `docker push` 到目标 registry

历史材料里已经明确提过两段常见同步方向：

1. 从 `office` 同步到 `qcloud`
2. 从 `qcloud` 再同步到 `qcloud-hk`

## 当前最关键的配置落点

不同环境镜像是否生效，最终还是看 `lexmount-k8s-manifests`。

最常见的回填位置：

1. `apps/clusters/office/images-configmap.yaml`
2. `apps/clusters/qcloud/images-configmap.yaml`
3. `apps/clusters/qcloud-hk/images-configmap.yaml`

如果只改了代码仓库或只推了镜像，但没有回填这里，环境通常不会真的切到新版本。

## 常见问题

### 1. office 已更新，但 qcloud-hk 还没更新

这通常不是“代码没发”，而更像是：

1. HK registry 没同步
2. HK 环境没有回填对应 tag
3. HK overlay 没有重新发布

### 2. 三个环境 tag 看起来相似，但行为不一致

优先确认：

1. 完整镜像地址是否真的对应
2. 各环境回填字段是否一致
3. 目标 overlay 是否已 `apply`

### 3. 发布后仍像旧版本

优先确认：

1. 镜像有没有真正推到目标 registry
2. `images-configmap.yaml` 是否已更新
3. `kustomization.yaml` 是否引用到对应字段
4. Deployment 实际运行的镜像是不是新地址

## 推荐排查顺序

1. 先确认代码仓库产出的镜像 tag
2. 再确认目标 registry 是否已有该镜像
3. 再确认目标环境 `images-configmap.yaml` 是否回填
4. 再确认环境是否已重新发布
5. 最后再确认 Pod 实际镜像

## 当前结论

镜像同步问题里，最常见的根因不是“代码没改对”，而是这三类：

1. 推错 registry
2. 回填错环境
3. 发布后实际运行镜像没有切过去
