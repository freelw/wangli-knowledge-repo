# office 环境运行手册

## 目的

这篇文档用于记录当前 `office` 环境的最小可执行操作方式，包括：

1. 发布入口
2. 常用检查命令
3. 关键服务验证路径

## 环境定位

`office` 是当前默认测试 / 验证环境。

当前机器上可直接使用 `kubectl` 连接该环境，因此大多数开发、发布和测试验证都应优先在这里完成。

## 常用目录

### 发布目录

```bash
/home/lexmount/project/backend/lexmount-k8s-manifests/apps/clusters/office
```

### 镜像配置

```bash
/home/lexmount/project/backend/lexmount-k8s-manifests/apps/clusters/office/images-configmap.yaml
```

### Kustomization

```bash
/home/lexmount/project/backend/lexmount-k8s-manifests/apps/clusters/office/kustomization.yaml
```

## 常用服务

在当前主链路里，`office` 至少需要关注这些服务：

1. `browser-manager`
2. `browser-manager-reconciler`
3. `browser-ws-gateway`
4. `browser-operator-controller-manager`
5. `browser-operator-api`

如果涉及前端 / 控制台联调，还需要补充关注：

6. `frontend` namespace 下的 `lex-home`

## 最常用操作

### 1. 发布整个 office overlay

```bash
cd /home/lexmount/project/backend/lexmount-k8s-manifests/apps/clusters/office
kubectl apply -k .
```

### 2. 查看核心 Deployment

```bash
kubectl get deployment -n system
```

### 3. 查看核心 Pod

```bash
kubectl get pods -n system -o wide
```

### 4. 查看服务

```bash
kubectl get svc -n system
```

## 镜像检查

### `browser-manager`

```bash
kubectl get deployment browser-manager -n system -o jsonpath='{.spec.template.spec.containers[?(@.name=="browser-manager")].image}{"\n"}'
```

### `browser-manager-reconciler`

```bash
kubectl get deployment browser-manager-reconciler -n system -o jsonpath='{.spec.template.spec.containers[?(@.name=="browser-manager-reconciler")].image}{"\n"}'
```

### `browser-ws-gateway`

```bash
kubectl get deployment browser-ws-gateway -n system -o jsonpath='{.spec.template.spec.containers[?(@.name=="browser-ws-gateway")].image}{"\n"}'
```

### `browser-operator-controller-manager`

```bash
kubectl get deployment browser-operator-controller-manager -n system -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
```

### `lex-home`

```bash
kubectl get deployment lex-home -n frontend -o jsonpath='{.spec.template.spec.containers[?(@.name=="lex-home")].image}{"\n"}'
```

## rollout 检查

### `browser-manager`

```bash
kubectl rollout status deployment/browser-manager -n system
```

### `browser-manager-reconciler`

```bash
kubectl rollout status deployment/browser-manager-reconciler -n system
```

### `browser-ws-gateway`

```bash
kubectl rollout status deployment/browser-ws-gateway -n system
```

### `browser-operator-controller-manager`

```bash
kubectl rollout status deployment/browser-operator-controller-manager -n system
```

### `lex-home`

```bash
kubectl rollout status deployment/lex-home -n frontend
```

## 最小验证路径

### 路径 1：控制面验证

目标：

确认 `browser-manager` 本身能正常工作。

最小检查项：

1. `browser-manager` Pod 正常运行
2. 对应 Deployment rollout 成功
3. `/health`、`/readyz` 可用
4. `instance` 相关接口可以调用

### 路径 2：websocket 验证

目标：

确认 `browser-ws-gateway` 接入和 relay 可用。

最小检查项：

1. `browser-ws-gateway` Pod 正常运行
2. 新 websocket 连接可以建立
3. `/devtools/*` 和 `/connection` 仍可正常工作
4. 若刚发布过，draining 行为符合预期

### 路径 3：浏览器实例验证

目标：

确认实例编排链路可用。

最小检查项：

1. 可以创建实例
2. 实例能进入 `Running`
3. 关键环境变量和挂载符合预期

### 路径 4：发布闭环验证

目标：

避免“发布成功但链路没通”。

最小检查项：

1. 镜像是否真的切到新版本
2. rollout 是否完成
3. 最小接口 / websocket / 实例链路是否都可用

### 路径 5：前端 / 控制台联调验证

目标：

确认 `lex-home` 自己的发布和前端联调链路正常。

最小检查项：

1. `frontend` namespace 下 `lex-home` Pod 正常运行
2. `lex-home` Deployment rollout 成功
3. `lex-home-image` 已切到目标镜像
4. 页面可访问
5. 页面触发的关键动作能正确命中后端控制面
6. `/settings/api-keys` 可创建 / 列出 API key
7. `/settings/sessions` 列表能加载，创建 session 后可见新记录
8. `/settings/contexts` 列表和创建动作可用
9. `/settings/extensions` 列表可加载，上传入口可用

## 常见问题

### 1. `kubectl apply -k .` 成功，但服务没变化

优先检查：

1. `images-configmap.yaml` 是否真的更新了
2. kustomization 是否已映射该字段
3. Deployment 实际镜像是否切换

### 2. `browser-manager` 正常，但 ws 连接异常

优先检查：

1. `browser-ws-gateway` 是否 rollout 成功
2. ws gateway 是否处于 draining
3. 发布的是不是 `browser-ws-gateway` 而不是 `browser-manager`

### 3. 发布看起来成功，但实例创建失败

优先检查：

1. `browser-manager` 接口是否可调用
2. `browser-operator-api` 是否正常
3. `browser-operator-controller-manager` 是否正常
4. office 环境的 operator 镜像和配置是否匹配

### 4. 前端页面正常打开，但联调动作异常

优先检查：

1. `lex-home` 的环境变量是否已正确注入
2. `lex-home` 当前镜像是不是预期版本
3. `lex-home/src/actions/kong.ts` 依赖的 `KONG_ADMIN_BASE_URL`、`BROWSER_BASE_URL`、`POCKETBASE_URL` 是否和环境一致
4. 后端对应接口是否真的可用

## 当前结论

`office` 环境当前最重要的使用原则是：

1. 把它当默认验证环境
2. 不只看发布命令是否成功
3. 而是要把镜像、rollout、控制面、websocket、实例链路一起验证闭环
