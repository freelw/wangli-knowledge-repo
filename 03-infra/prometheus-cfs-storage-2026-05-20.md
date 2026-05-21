# Prometheus CFS 持久化改造记录（2026-05-20）

## 背景

`lexmount-k8s-manifests` 中 Prometheus 原来使用 `emptyDir` 作为 TSDB 数据目录：

```yaml
- name: prometheus-storage
  emptyDir: {}
```

这意味着 Prometheus Pod 重建后历史指标会丢失。Grafana 已经使用 CFS/NFS 静态 PV/PVC 做持久化，因此本次将 Prometheus 按同一模式切换到 CFS，并限制最多保留约 2 个月数据。

对应 PR：

- lexmount-k8s-manifests: https://github.com/lexmount/lexmount-k8s-manifests/pull/499
- 合入提交分支：`flash1_prometheus_cfs_20260520`
- 最终处理后 head：`57d5d38`

## 最终方案

### 1. Prometheus 使用静态 PV/PVC

新增：

- `monitoring/prometheus/prometheus-pv.yaml`
- `monitoring/prometheus/prometheus-pvc.yaml`

PV/PVC 命名：

- PV：`prometheus-pv-v2`
- PVC：`prometheus-pvc-v2`
- Namespace：`monitoring`

PV 使用和 Grafana 一致的 NFS/CFS mount options：

```yaml
mountOptions:
  - vers=3
  - nolock
  - proto=tcp
  - noresvport
  - hard
  - timeo=600
  - retrans=2
  - nosuid
  - nodev
  - noatime
```

PV 通过 `claimRef` 预绑定到 `prometheus-pvc-v2`，PVC 通过 `volumeName: prometheus-pv-v2` 显式绑定，避免被其他 PVC 误绑定。

### 2. 复用集群 CFS root，用 subPath 隔离数据

Prometheus 和 Grafana 复用同一套集群 `nfs-path` / `nfs-server` replacement，但通过不同 `subPath` 隔离目录：

- Grafana：`subPath: grafana`
- Prometheus：`subPath: prometheus`

Prometheus Deployment 挂载方式：

```yaml
- name: prometheus-storage
  mountPath: /prometheus
  subPath: prometheus
```

这样不会新增一套 CFS，只是在现有 CFS root 下单独使用 `prometheus` 子目录保存 TSDB。

### 3. 保留期改为 60 天

Prometheus 参数从 30 天改为 60 天：

```yaml
--storage.tsdb.retention.time=60d
```

这个值对应“最多保存 2 个月数据”的要求。实际磁盘占用仍取决于采集量、label cardinality 和 compaction 情况。

### 4. 权限初始化

新增 `fix-permissions` initContainer，对 `/prometheus` 做权限初始化：

```sh
chown -R 65534:65534 /prometheus
chmod -R 750 /prometheus
```

注意：review 阶段曾指出 `chmod -R 755` 会让 Prometheus TSDB 数据 world-readable。最终已改为 `750`，避免其他非授权进程读取指标块数据。

initContainer 镜像通过各集群 `app-images-config.data.busybox-image` 替换，不写死镜像。

## 覆盖的集群

所有包含 `monitoring/prometheus` 的 overlay 都补齐了 replacement：

- `apps/clusters/office-nanjing`
- `apps/clusters/office-beijing`
- `apps/clusters/qcloud-nanjing`
- `apps/clusters/qcloud-beijing`
- `apps/clusters/qcloud-hk`
- `apps/clusters/guoge`

每个 overlay 都会将 `app-config` 中的：

- `nfs-path`
- `nfs-server`

替换到 `prometheus-pv-v2.spec.nfs.path/server`。

每个 overlay 也会将：

- `app-images-config.data.busybox-image`

替换到 Prometheus `fix-permissions` initContainer 镜像。

## 验证记录

PR 合入前验证过以下 overlay 渲染：

```bash
kubectl kustomize apps/clusters/office-nanjing
kubectl kustomize apps/clusters/office-beijing
kubectl kustomize apps/clusters/qcloud-nanjing
kubectl kustomize apps/clusters/qcloud-beijing
kubectl kustomize apps/clusters/qcloud-hk
kubectl kustomize apps/clusters/guoge
```

检查点：

- 渲染结果包含 `prometheus-pv-v2`
- 渲染结果包含 `prometheus-pvc-v2`
- Prometheus Deployment 使用 `claimName: prometheus-pvc-v2`
- Prometheus initContainer 和主容器都使用 `subPath: prometheus`
- Prometheus 参数包含 `--storage.tsdb.retention.time=60d`
- 渲染结果中没有 `placeholder-nfs-path` / `placeholder-nfs-server`
- Prometheus 权限初始化为 `chmod -R 750 /prometheus`

## 发布注意事项

1. **首次发布会创建 PV/PVC**
   - PV reclaim policy 是 `Retain`。
   - 如果集群里已经存在同名 PV/PVC，需要先确认是否来自同一次改造，避免绑定冲突。

2. **Prometheus Pod 会重建**
   - 从 `emptyDir` 切换到 PVC 后，Prometheus Pod 需要重新调度/重建才能挂载新卷。
   - 原 emptyDir 中的历史数据不会自动迁移。

3. **CFS 子目录权限**
   - initContainer 会创建/修正 `/prometheus` 挂载子路径权限。
   - 如果 CFS 开启特殊 root squash 行为，需确认 `chown 65534:65534` 是否符合当前 CFS 权限策略。

4. **保留期不是容量上限**
   - `60d` 限制的是时间维度，不是磁盘容量上限。
   - 如果采集量或 label cardinality 上升，仍需关注 CFS 使用量。

5. **Grafana 与 Prometheus 共享 CFS root，但目录隔离**
   - Grafana 数据在 `grafana` subPath。
   - Prometheus 数据在 `prometheus` subPath。
   - 不应把 Prometheus 数据目录改到 Grafana subPath 下。

## 后续建议

- 在发布后确认：

```bash
kubectl -n monitoring get pv,pvc | grep prometheus
kubectl -n monitoring describe pvc prometheus-pvc-v2
kubectl -n monitoring get pod -l app=prometheus -o wide
kubectl -n monitoring logs deploy/prometheus-deployment -c prometheus --tail=100
```

- 在 Prometheus UI 或日志中确认 TSDB 启动路径仍为 `/prometheus`。
- 观察 CFS 使用量，确认 60 天保留期下容量增长可接受。
