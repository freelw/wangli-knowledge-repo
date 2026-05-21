# Kustomize patch 兼容性注意事项

## 背景

在 `lexmount-k8s-manifests` 中给 qcloud 的 `browser-replay-processor` / `browser-replay-cleaner` 增加 spot node 调度约束时，最初把两个 Deployment 的 strategic merge patch 放在同一个 YAML 文件里，用 `---` 分隔。

本地 `kubectl kustomize` 可以通过，但线上执行：

```bash
kubectl apply -k apps/clusters/qcloud-nanjing
```

报错：

```text
error: trouble configuring builtin PatchTransformer with config:
path: browser-replay-spot-node-patch.yaml
: unable to parse SM or JSON patch from [...]
```

## 结论

为了兼容不同环境中的 kustomize / kubectl 版本，在 `kustomization.yaml` 的 `patches` 里引用 strategic merge patch 时，不要把多个资源写在同一个 patch 文件里。

推荐规则：

- 一个 patch 文件只 patch 一个资源。
- 多个资源分别建多个 patch 文件。
- 在 `patches` 中逐个引用这些文件。

## 推荐写法

例如不要写一个包含两个 Deployment 的文件：

```yaml
patches:
  - path: browser-replay-spot-node-patch.yaml
```

其中 `browser-replay-spot-node-patch.yaml` 内部再用 `---` 分隔多个 Deployment。

应拆成：

```yaml
patches:
  - path: browser-replay-processor-spot-node-patch.yaml
  - path: browser-replay-cleaner-spot-node-patch.yaml
```

其中每个文件只包含一个 Deployment patch。

## 示例

`browser-replay-processor-spot-node-patch.yaml`：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: browser-replay-processor
  namespace: system
spec:
  template:
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: lexmount/spot
                    operator: In
                    values:
                      - "true"
      tolerations:
        - key: browser-worker-node
          operator: Equal
          value: "true"
          effect: NoSchedule
```

`browser-replay-cleaner-spot-node-patch.yaml`：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: browser-replay-cleaner
  namespace: system
spec:
  template:
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: lexmount/spot
                    operator: In
                    values:
                      - "true"
      tolerations:
        - key: browser-worker-node
          operator: Equal
          value: "true"
          effect: NoSchedule
```

## 验证建议

提交前至少跑：

```bash
kubectl kustomize apps/clusters/qcloud-nanjing
kubectl kustomize apps/clusters/qcloud-beijing
kubectl kustomize apps/clusters/qcloud-hk
git diff --check
```

如果变更会由远端流水线或不同机器执行 `kubectl apply -k`，优先采用“一文件一资源”的写法，避免本地 kustomize 版本较新但远端较旧导致 apply 失败。
