# browser-operator 发布手册

## 适用范围

这篇文档适用于以下场景：

1. 发布 `k8s-chrome-daemon`
2. 更新 `browser-operator` 镜像
3. 同步 `browser-operator` 配置到目标环境
4. 验证 `browser-operator-controller-manager` 的运行状态

相关仓库：

1. `k8s-chrome-daemon`
2. `lexmount-k8s-manifests`

## 组件职责

`browser-operator` 的核心职责包括：

1. 接收 `browser-manager` 或其他入口发出的浏览器实例创建请求
2. 创建 `BrowserInstance` CR
3. 生成浏览器 Pod
4. 管理浏览器实例生命周期
5. 在删除时做对应清理

当前关键服务通常包括：

1. Deployment：`browser-operator-controller-manager`
2. Service：`browser-operator-api`

## 发布前检查

发布前至少确认以下几点：

### 1. 开发分支是否正确

应基于最新 `origin/main` 拉开发分支，不要直接在 `main` 上提交需求改动。

### 2. 本地代码是否包含预期改动

尤其要确认：

1. controller 逻辑修改是否完成
2. 新增环境变量是否已接入
3. 挂载、PVC、`emptyDir`、context、downloads 等行为是否已覆盖

### 3. manifest 是否需要同步配置

如果只是镜像更新，通常只需要改 `images-configmap.yaml`。

如果引入了新配置，还要同步：

1. `apps/chrome-daemon/manager/config.yaml`
2. `apps/chrome-daemon/manager/manager.yaml`
3. 各集群 `app-config.yaml`
4. 各集群 `kustomization.yaml`

## 第一步：构建并推送镜像

在 `k8s-chrome-daemon` 根目录执行：

```bash
cd /home/lexmount/project/backend/k8s-chrome-daemon
make push
```

输出中重点关注两类镜像地址：

1. `code.lexmount.net/wangli/browser-operator:<tag>`
2. `lexmount.tencentcloudcr.com/cloud/browser-operator:<tag>`

如涉及香港环境，只需要在 manifest 中补齐 HK registry 对应 tag。HK registry 的实际镜像推送由 `@freelw` 执行，agent 不要自行推送。

## 第二步：更新 manifest 中的镜像

至少需要检查或更新：

1. `lexmount-k8s-manifests/apps/clusters/office/images-configmap.yaml`
2. `lexmount-k8s-manifests/apps/clusters/qcloud/images-configmap.yaml`
3. `lexmount-k8s-manifests/apps/clusters/qcloud-hk/images-configmap.yaml`

注意：更新 `qcloud-hk` 时必须保留 `lexmoun-tcr-hk.tencentcloudcr.com/...` 这类 HK registry 前缀，只替换 tag。不要把它改成 `lexmount.tencentcloudcr.com/...` 或 `code.lexmount.net/...`。

核心字段：

```text
browser-operator-image
```

如果当前只验证 `office`，可以先只回填 `office`。

## 第三步：同步新增配置

如果本次改动引入了新的环境变量或开关，还要同步以下位置。

### 基础配置

1. `apps/chrome-daemon/manager/config.yaml`
2. `apps/chrome-daemon/manager/manager.yaml`

### 各环境 overlay

1. 各集群 `app-config.yaml`
2. 各集群 `kustomization.yaml`

常见目标是把 overlay 里的配置值映射到：

```text
browser-operator-config.data.<KEY>
```

## 第四步：发布到 office

当前推荐先在 `office` 环境做验证。

执行：

```bash
cd /home/lexmount/project/backend/lexmount-k8s-manifests/apps/clusters/office
kubectl apply -k .
```

## 第五步：检查 rollout

执行：

```bash
kubectl rollout status deployment/browser-operator-controller-manager -n system
```

目标：

1. rollout 成功
2. 没有旧 Pod 长时间残留

## 第六步：检查镜像与开关

### 检查 Deployment

```bash
kubectl get deployment browser-operator-controller-manager -n system
```

关注：

1. `READY`
2. `AVAILABLE`

### 检查当前环境变量和镜像

```bash
kubectl get deployment browser-operator-controller-manager -n system -o jsonpath='{.spec.template.spec.containers[0].env[?(@.name=="ENABLE_CFS_MOUNTS")].value} {.spec.template.spec.containers[0].image}{"\n"}'
```

确认：

1. 环境变量值符合预期
2. 镜像 tag 已切到本次版本

### 检查 Pod

```bash
kubectl get pods -n system -l control-plane=controller-manager -o wide
```

确认：

1. Pod 状态为 `Running`
2. 副本数正常
3. 没有异常重启

### 检查 Service

```bash
kubectl get svc browser-operator-api -n system
```

## 第七步：做最小功能验证

发布成功后，至少要确认实例创建链路仍可用。

推荐验证方式：

### 1. 端口转发

优先转发 service，避免 Pod 滚动时中断：

```bash
kubectl port-forward -n system svc/browser-operator-api 19220:9090
```

### 2. 创建测试实例

通过 `browser-operator-api` 创建临时实例，验证：

1. 实例能成功创建
2. 状态能进入 `Running`
3. 关键环境变量已注入
4. volume / mount 行为符合预期

### 3. 验证完成后删除测试实例

不要把临时验证实例长期留在环境里。

## 典型发布场景：`ENABLE_CFS_MOUNTS`

这是当前文档里一个典型开关。

### 开启时

当 `ENABLE_CFS_MOUNTS=true` 时：

1. 挂载 `browser-user-data-pvc`
2. download 使用 PVC subPath
3. replay 使用 PVC subPath
4. context 使用持久化路径
5. extension 走挂载路径

### 关闭时

当 `ENABLE_CFS_MOUNTS=false` 时：

1. 不挂 `browser-user-data-pvc`
2. 不挂 context 路径
3. 不挂 extension 路径
4. `/config/Downloads` 改为本地 `emptyDir`
5. `/tmp/operation-replay` 改为本地 `emptyDir`

### 适用场景

适合在这些情况下作为降级开关使用：

1. CFS 故障
2. PVC 挂载异常
3. 需要优先恢复浏览器可用性

### 注意

关闭后，持久化能力会下降，Pod 删除后本地数据会丢失。

## 常见问题

### 问题 1：镜像已经推送，但环境没生效

优先检查：

1. `images-configmap.yaml` 是否已更新
2. 是否已执行 `kubectl apply -k`
3. rollout 是否真的完成

### 问题 2：新开关代码里有，但环境里没有

优先检查：

1. `config.yaml`
2. `manager.yaml`
3. 各集群 `app-config.yaml`
4. 各集群 `kustomization.yaml`

### 问题 3：Deployment 正常，但创建实例失败

优先检查：

1. `browser-operator-api` 是否可访问
2. 测试实例是否进入 `Running`
3. Pod 挂载和环境变量是否符合预期

## 当前结论

`browser-operator` 的发布重点不只是“镜像推上去了”，而是四件事都要闭环：

1. 镜像构建完成
2. manifest 已同步
3. rollout 正常
4. 实例创建链路验证通过
