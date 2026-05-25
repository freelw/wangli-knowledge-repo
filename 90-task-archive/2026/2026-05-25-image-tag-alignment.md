# 2026-05-25 qcloud-hk / office 镜像 tag 对齐

## 背景

replay pod cleanup 发布后，继续检查各环境关键服务镜像 tag 是否落后于 `qcloud-nanjing`。重点关注：

- `browser-manager`
- `browser-operator` / daemon
- `watermark-monitor`
- `region-data-plane-gateway`
- `browser-replay-processor`
- `browser-spider-bench`
- `chrome`

## qcloud-hk 对齐

仓库：`lexmount-k8s-manifests`

PR：

- https://github.com/lexmount/lexmount-k8s-manifests/pull/544

检查结果：

- `browser-manager`：已与 qcloud-nanjing 对齐。
- `browser-operator`：已与 qcloud-nanjing 对齐。
- `watermark-monitor`：落后。
- `region-data-plane-gateway`：落后。
- `browser-spider-bench`：落后。

对齐后的 tag：

```text
region-data-plane-gateway:79c90d5-20260521-143605
k8s-watermark-monitor:c525983-20260521-135658
browser-spider-bench:b8eae00-20260518-234127
```

同时将对应镜像同步到 HK 仓库：

```text
lexmoun-tcr-hk.tencentcloudcr.com/cloud/region-data-plane-gateway:79c90d5-20260521-143605
lexmoun-tcr-hk.tencentcloudcr.com/cloud/k8s-watermark-monitor:c525983-20260521-135658
lexmoun-tcr-hk.tencentcloudcr.com/cloud/browser-spider-bench:b8eae00-20260518-234127
```

## office-nanjing / office-beijing 对齐

仓库：`lexmount-k8s-manifests`

PR：

- https://github.com/lexmount/lexmount-k8s-manifests/pull/545

检查结果：

- `browser-manager`：已与 qcloud-nanjing 对齐。
- `browser-operator`：已与 qcloud-nanjing 对齐。
- `watermark-monitor`：已与 qcloud-nanjing 对齐。
- `region-data-plane-gateway`：落后。
- `browser-replay-processor`：落后。
- `chrome`：落后。
- `kong-key-sync`：office-nanjing 落后。

对齐后的 tag：

```text
region-data-plane-gateway:79c90d5-20260521-143605
browser-replay-processor:debug-20260522-01
chrome:20260522-174137-0bc5586c86-experimental
kong-key-sync:1488190-20260512-213227
```

同步状态：

- `code.lexmount.net/wangli/region-data-plane-gateway:79c90d5-20260521-143605`：已同步。
- `code.lexmount.net/wangli/browser-replay-processor:debug-20260522-01`：已同步。
- `code.lexmount.net/neng/chrome:20260522-174137-0bc5586c86-experimental`：已存在。
- `code.lexmount.net/wangli/kong-key-sync:1488190-20260512-213227`：已存在。

## 未按“落后”处理的差异

以下差异不是简单落后，不在本次对齐范围内：

- `lex-home-image`：`cn-*` / `com-*` / `dev-*` 属于不同构建目标。
- `kong-key-sync-image`、`session-quota-image` 在不同集群是否存在，取决于该环境是否承担主 region / quota / key sync 职责。
- qcloud 与 office 使用不同 registry 域名，比较时只对比 image tag。

## 校验

qcloud-hk 对齐：

```bash
kubectl kustomize apps/clusters/qcloud-hk
kubectl kustomize apps/clusters/qcloud-nanjing
git diff --check
```

office 对齐：

```bash
kubectl kustomize apps/clusters/office-nanjing
kubectl kustomize apps/clusters/office-beijing
git diff --check
```

## 经验

镜像对齐不能只看服务是否存在，需要区分三类情况：

1. 同服务同语义：应对齐 tag。
2. 同服务但不同构建目标：不能直接按 tag 对齐，例如 `lex-home` 的 cn/com/dev 包。
3. 服务缺失：先判断是否为架构差异，再决定是否补齐。
