# 环境说明

## 目的

这篇文档用于统一记录当前平台环境信息、访问方式和常用环境变量。

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

### 平台调用环境变量口径

```bash
LEXMOUNT_API_KEY=<office-api-key>
LEXMOUNT_PROJECT_ID=<office-project-id>
LEXMOUNT_BASE_URL=https://apitest.local.lexmount.net
```

说明：

1. 这里不再记录真实 key / project_id
2. 真实值应从对应环境的安全配置或负责人处获取
3. 知识库只保留变量名、用途和获取方式

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

### 平台调用环境变量口径

```bash
LEXMOUNT_API_KEY=<qcloud-api-key>
LEXMOUNT_PROJECT_ID=<qcloud-project-id>
```

### 说明

在 `qcloud` 场景下，不需要设置 `LEXMOUNT_BASE_URL`。

当前知识库还需要明确一个操作边界：

1. 本机默认不要把 `qcloud` 当成“可直接 `kubectl apply -k`”的环境
2. 当前机器上已经确认可直接 `kubectl` 的是 `office`
3. `qcloud` / `qcloud-hk` 的实际发布应通过对应负责人或正确环境入口执行

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

### `lex-home` 前端联调相关环境变量

`lex-home` 当前至少依赖这些关键变量：

```bash
KONG_ADMIN_BASE_URL=<kong-admin-base-url>
BROWSER_BASE_URL=<browser-manager-base-url>
POCKETBASE_URL=<pocketbase-base-url>
```

当前已知 `office` 集群内默认值可以在下面位置查看：

1. `lexmount-k8s-manifests/apps/lex-home/lex-home-configmap.yaml`

其中当前配置是：

1. `KONG_ADMIN_BASE_URL="http://kong-admin.system.svc.cluster.local:8001/"`
2. `BROWSER_BASE_URL="http://browser-manager.system.svc.cluster.local:9222/"`
3. `POCKETBASE_URL="http://pocketbase-service.default.svc.cluster.local:8090"`

对于前端联调，更重要的是记住：

1. 值从 manifests 来
2. 不要把它们当成前端仓库里的固定常量
3. 如果页面联调异常，要先确认环境里实际注入的值
4. `BROWSER_BASE_URL` 当前指向的是 `browser-manager` 控制面 Service，而不是前端直连 Chrome 实例

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

## `lex-home` 环境补充

### 仓库与部署入口

前端控制台 / 官网相关的主要入口是：

1. 代码仓库：`/home/lexmount/project/backend/lex-home`
2. K8s 配置：`/home/lexmount/project/backend/lexmount-k8s-manifests/apps/lex-home`

### 镜像回填字段

各环境都通过 `images-configmap.yaml` 里的 `lex-home-image` 回填镜像。

常见位置：

1. `apps/clusters/office/images-configmap.yaml`
2. `apps/clusters/qcloud/images-configmap.yaml`
3. `apps/clusters/qcloud-hk/images-configmap.yaml`

### 当前已知镜像仓库差异

1. `office`：`code.lexmount.net/feixiang/lex-home`
2. `qcloud`：`lexmount.tencentcloudcr.com/cloud/lex-home`
3. `qcloud-hk`：`lexmoun-tcr-hk.tencentcloudcr.com/cloud/lex-home`

### 常用脚本

`lex-home` 相关镜像同步和回填脚本当前在：

1. `apps/tools/set_lex_home_image_tags.sh`
2. `apps/tools/sync_lex_home.sh`

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
3. 当前机器默认可直接 `kubectl` 的是 `office`
4. `office` 的访问与发布路径最清晰，应作为多数开发验证的默认环境
