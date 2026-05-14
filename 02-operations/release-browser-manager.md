# browser-manager 发布手册

## 适用范围

这篇文档适用于以下场景：

1. 发布 `browser-manager`
2. 发布 `browser-manager-reconciler`
3. 发布 `browser-ws-gateway`
4. 将这些服务的新镜像同步到 `office` 或其他环境

相关仓库：

1. `demo-nodejs-backend/browser-manager`
2. `demo-nodejs-backend/browser-manager-reconciler`
3. `demo-nodejs-backend/browser-ws-gateway`
4. `lexmount-k8s-manifests`

## 当前发布对象

当前和 `browser-manager` 主链路直接相关的三个服务分别是：

1. `browser-manager`
   - HTTP 控制面
2. `browser-manager-reconciler`
   - 后台补偿和清理
3. `browser-ws-gateway`
   - websocket 接入与 relay

这三者在 `office` 环境里都有独立镜像字段，不应再只把“发布 browser-manager”理解成只发一个 Deployment。

## 第一步：构建并推送镜像

### 1.1 `browser-manager`

在项目根目录执行：

```bash
cd /home/lexmount/project/backend/demo-nodejs-backend/browser-manager
make push
```

当前 `Makefile` 会推送至少两类镜像：

1. `code.lexmount.net/wangli/browser-manager:<tag>`
2. `lexmount.tencentcloudcr.com/cloud/browser-manager:<tag>`

其中 `office` 测试环境使用的是以 `code.lexmount.net/wangli` 开头的镜像。

### 1.2 `browser-manager-reconciler`

当前 reconciler 在 manifests 里是独立 Deployment，因此如果修改了它的代码，也需要单独构建并同步镜像。

构建入口：

```bash
cd /home/lexmount/project/backend/demo-nodejs-backend/browser-manager-reconciler
make push
```

镜像字段见后文 `browser-manager-reconciler-image`。

### 1.3 `browser-ws-gateway`

当前 websocket relay 已经是独立服务，因此如果修改了 `browser-ws-gateway` 代码，也需要单独构建并同步镜像。

构建入口：

```bash
cd /home/lexmount/project/backend/demo-nodejs-backend/browser-ws-gateway
make push
```

镜像字段见后文 `browser-ws-gateway-image`。

### 1.4 服务入口速查表

| 服务 | 代码目录 | 镜像键 | Deployment |
| --- | --- | --- | --- |
| `browser-manager` | `demo-nodejs-backend/browser-manager` | `browser-manager-image` | `browser-manager` |
| `browser-manager-reconciler` | `demo-nodejs-backend/browser-manager-reconciler` | `browser-manager-reconciler-image` | `browser-manager-reconciler` |
| `browser-ws-gateway` | `demo-nodejs-backend/browser-ws-gateway` | `browser-ws-gateway-image` | `browser-ws-gateway` |

## 第二步：更新 office 环境镜像

当前 `office` 环境镜像统一记录在：

`/home/lexmount/project/backend/lexmount-k8s-manifests/apps/clusters/office/images-configmap.yaml`

和 `browser-manager` 主链路直接相关的字段包括：

1. `browser-manager-image`
2. `browser-manager-reconciler-image`
3. `browser-ws-gateway-image`

更新原则：

1. 只改本次涉及服务的镜像字段
2. 使用 `code.lexmount.net/wangli/...` 的镜像地址
3. 回填后再通过 kustomization 发布

## 第三步：确认 office kustomization 已接入目标服务

当前 `office` 集群配置：

`/home/lexmount/project/backend/lexmount-k8s-manifests/apps/clusters/office/kustomization.yaml`

已经包含：

1. `../../browser-manager`
2. `../../browser-ws-gateway`

并且已配置以下镜像替换：

1. `browser-manager-image`
2. `browser-manager-reconciler-image`
3. `browser-ws-gateway-image`

因此，对这三类镜像的回填会在 `kubectl apply -k` 时生效。


## Chrome 镜像配置变更注意事项

`CHROME_IMAGE` 不只被 `browser-manager` 使用。

当前 `browser-ws-gateway` 通过 `envFrom` 也会读取 `browser-manager-config`，并且 `/connection` 路径在握手阶段会直接调用 `createSession(project_id, api_key, 'normal', contextId)` 创建浏览器实例。

因此，如果只更新了 `browser-manager-config.CHROME_IMAGE`，但没有重启 `browser-ws-gateway`，会出现：

1. 普通 SDK session 已经使用新 Chrome 镜像
2. e2e-tool 的 `connection` case 仍创建旧 Chrome 镜像
3. Pod label / annotation 里看到旧的 `docker_image`

典型现象：

```text
label docker_image=chrome-20260424-161313-a0934b9362-experimental
container image=.../chrome:20260424-161313-a0934b9362-experimental
```

但 `browser-manager` 中看到的环境变量已经是新值。原因是 `/connection` 实际走的是 `browser-ws-gateway`。

排查时必须同时检查：

```bash
kubectl -n system exec deploy/browser-manager -- printenv CHROME_IMAGE
kubectl -n system exec deploy/browser-ws-gateway -- printenv CHROME_IMAGE
```

如果 `browser-ws-gateway` 仍是旧值，执行：

```bash
kubectl -n system rollout restart deploy/browser-ws-gateway
kubectl -n system rollout status deploy/browser-ws-gateway
```

发布 Chrome 镜像配置时，建议至少重启：

1. `browser-manager`
2. `browser-ws-gateway`

因为两者都会通过环境变量消费 `browser-manager-config`，Pod 不重启不会刷新环境变量。

## 第四步：发布到 office

执行：

```bash
cd /home/lexmount/project/backend/lexmount-k8s-manifests/apps/clusters/office
kubectl apply -k .
```

## 第五步：检查 rollout

根据本次发布对象分别检查。

### 5.1 发布 `browser-manager`

```bash
kubectl rollout status deployment/browser-manager -n system
```

### 5.2 发布 `browser-manager-reconciler`

```bash
kubectl rollout status deployment/browser-manager-reconciler -n system
```

### 5.3 发布 `browser-ws-gateway`

```bash
kubectl rollout status deployment/browser-ws-gateway -n system
```

## 第六步：检查实际镜像

### 6.1 `browser-manager`

```bash
kubectl get deployment browser-manager -n system -o jsonpath='{.spec.template.spec.containers[?(@.name=="browser-manager")].image}{"\n"}'
```

### 6.2 `browser-manager-reconciler`

```bash
kubectl get deployment browser-manager-reconciler -n system -o jsonpath='{.spec.template.spec.containers[?(@.name=="browser-manager-reconciler")].image}{"\n"}'
```

### 6.3 `browser-ws-gateway`

```bash
kubectl get deployment browser-ws-gateway -n system -o jsonpath='{.spec.template.spec.containers[?(@.name=="browser-ws-gateway")].image}{"\n"}'
```

## 第七步：做最小业务验证

### 7.1 发布 `browser-manager` 后

至少验证：

1. `/health`、`/readyz` 是否正常
2. `instance` 相关接口是否可调用
3. `/json`、`/json/version` 返回是否正常

### 7.2 发布 `browser-ws-gateway` 后

至少验证：

1. websocket 新连接是否正常建立
2. `/devtools/*` 路径是否可用
3. `/connection` 连接是否正常工作
4. draining 期间旧连接行为是否符合预期

### 7.3 发布 `browser-manager-reconciler` 后

至少验证：

1. 进程正常启动
2. creating session reconcile 正常运行
3. active session cleanup / downloads cleanup / closed history cleanup 没有明显异常

## 当前最容易忽略的点

### 1. 不要只更新 `browser-manager-image`

当前主链路已经拆成多个服务，改动落在哪个服务，就应更新对应镜像字段。

### 2. 不要把 ws 风险继续归因到 `browser-manager` 未拆分

当前 websocket 承载已经独立到 `browser-ws-gateway`。

因此：

1. 普通 `browser-manager` 发布不应直接等同于 ws 一起重启
2. 当前更需要重点关注的是 `browser-ws-gateway` 自身发布时的连接表现

### 3. 不要只看 rollout 成功

还要补：

1. 实际镜像确认
2. 最小接口验证
3. websocket 或实例链路验证

## 当前结论

当前 `browser-manager` 相关发布更准确的理解是：

1. 不是单一服务发布
2. 而是一组相关服务的独立发布与验证
3. `office` 环境至少要关注 `browser-manager`、`browser-manager-reconciler`、`browser-ws-gateway` 三类镜像和对应 Deployment
