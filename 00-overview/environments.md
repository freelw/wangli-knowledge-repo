# 环境说明

## 目的

这篇文档用于统一记录当前后端平台的环境信息、访问方式和常用环境变量。

## 当前已知环境

`lexmount-k8s-manifests` 中当前已知的主要环境有：

1. `office`
2. `qcloud`

文档和历史任务中还出现过：

1. `qcloud-hk`
2. `aws`
3. `guoge`

其中当前最常用、最明确的两个环境是 `office` 和 `qcloud`。

## `office` 环境

### 角色

`office` 是当前主要测试环境。

### API 基址

```text
https://apitest.local.lexmount.net
```

### 示例接口

```text
https://apitest.local.lexmount.net/instance
```

### 本地调用环境变量

```bash
LEXMOUNT_API_KEY=SABG5D4NStyRFeFLLRNSEeyUIRABeLm5
LEXMOUNT_PROJECT_ID=11872275-f529-4b63-bbce-eba62c4257e1
LEXMOUNT_BASE_URL=https://apitest.local.lexmount.net
```

### 本机访问能力

当前机器上可直接使用 `kubectl` 连接 `office` 测试环境。

这意味着以下工作通常可以直接在本机完成：

1. `kubectl get`
2. `kubectl apply -k`
3. `kubectl rollout status`
4. `kubectl port-forward`

## `qcloud` 环境

### 角色

`qcloud` 是腾讯云环境。

### 本地调用环境变量

```bash
LEXMOUNT_API_KEY=WSUNLQVYICXDUWjXcHxa7vZLMb20R0dd
LEXMOUNT_PROJECT_ID=d5c3237b-8f94-40b2-89d1-8752a1997294
```

### 说明

在 `qcloud` 场景下，不需要设置 `LEXMOUNT_BASE_URL`。

## 其他环境

### `qcloud-hk`

从历史任务来看，`qcloud-hk` 主要用于香港区域相关部署与镜像同步。

常见场景：

1. HK registry 镜像同步
2. 香港区域网络差异验证

### `aws`

从历史配置变更记录看，`aws` 曾出现在环境配置同步范围内，但当前没有更多稳定文档说明，后续需要在实际使用时补充。

### `guoge`

从历史配置变更记录看，`guoge` 曾出现在集群配置同步范围内，但当前不是主要对外说明环境。

## 常用访问与检查方式

### 1. 发布 office 环境

常见执行路径：

```bash
cd /home/lexmount/project/backend/lexmount-k8s-manifests/apps/clusters/office
kubectl apply -k .
```

### 2. 检查 controller rollout

```bash
kubectl rollout status deployment/browser-operator-controller-manager -n system
```

### 3. 检查 Deployment 当前镜像和开关

```bash
kubectl get deployment browser-operator-controller-manager -n system -o jsonpath='{.spec.template.spec.containers[0].env[?(@.name=="ENABLE_CFS_MOUNTS")].value} {.spec.template.spec.containers[0].image}{"\n"}'
```

### 4. 只读查看腾讯云资源

本机已安装 `tccli`，并且当前配置为只读权限。

适合用于：

1. 查看网络资源
2. 查看地域差异
3. 查询只读基础设施信息

不适合用于直接修改云资源。

## 环境使用建议

### 开发验证

优先使用 `office`。

原因：

1. 文档最完整
2. 本机 `kubectl` 可直接访问
3. 发布验证流程最清晰

### 腾讯云基础设施问题

优先结合 `qcloud` 与 `tccli` 文档排查。

### 香港相关镜像或网络问题

优先关注 `qcloud-hk` 相关配置和同步链路。

## 当前结论

当前知识层面最重要的环境认知是：

1. `office` 是主测试环境
2. `qcloud` 是主腾讯云环境
3. `office` 的访问与发布路径最清晰，应作为多数开发验证的默认环境
