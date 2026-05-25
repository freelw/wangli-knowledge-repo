# K8s Node 预留 5% 资源操作手册

日期：2026-05-25

## 目标

让线上 K8s node 至少保留约 5% CPU / 内存给系统组件、kubelet 和突发缓冲，避免业务 pod `request` 很低但 `limit` 很高时，运行期资源上涨把节点打死。

核心手段有两层：

1. 调度层：通过 kubelet `systemReserved` / `kubeReserved` 降低 `Allocatable`，让 scheduler 不把节点排满。
2. 运行层：通过 kubelet eviction 阈值，在内存逼近危险水位前驱逐 pod，而不是等 node OOM。

## 推荐配置

按节点容量预留约 5%。如果节点规格差异较大，优先用每类节点池独立配置，不要全局套一个固定值。

示例，适合约 16C / 32Gi 级别节点作为起点：

```yaml
systemReserved:
  cpu: "500m"
  memory: "1Gi"
kubeReserved:
  cpu: "300m"
  memory: "512Mi"
evictionHard:
  memory.available: "5%"
  nodefs.available: "10%"
  imagefs.available: "10%"
evictionSoft:
  memory.available: "8%"
evictionSoftGracePeriod:
  memory.available: "1m"
```

如果 kubelet 不接受百分比内存阈值，使用绝对值，例如：

```yaml
evictionHard:
  memory.available: "2Gi"
  nodefs.available: "10%"
  imagefs.available: "10%"
evictionSoft:
  memory.available: "3Gi"
evictionSoftGracePeriod:
  memory.available: "1m"
```

## 线上变更步骤

以下步骤适用于自管 kubelet 配置的节点。托管节点池如果不能直接改节点文件，应通过节点池 kubelet 参数或新节点池灰度迁移实现。

1. 选一个非关键 worker 节点。

```bash
kubectl get nodes -o wide
kubectl describe node <node-name> | sed -n '/Capacity:/,/Allocatable:/p'
kubectl describe node <node-name> | sed -n '/Allocatable:/,/System Info:/p'
```

2. 暂停调度并迁移业务 pod。

```bash
kubectl cordon <node-name>
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data --grace-period=60 --timeout=10m
```

3. 在节点上备份 kubelet 配置。

```bash
sudo cp /var/lib/kubelet/config.yaml /var/lib/kubelet/config.yaml.bak.$(date +%Y%m%d%H%M%S)
```

4. 修改 `/var/lib/kubelet/config.yaml`，加入或调整 `systemReserved`、`kubeReserved`、`evictionHard`、`evictionSoft`、`evictionSoftGracePeriod`。

5. 重启 kubelet。

```bash
sudo systemctl daemon-reload
sudo systemctl restart kubelet
sudo systemctl status kubelet --no-pager
```

6. 等节点恢复 Ready。

```bash
kubectl get node <node-name> -w
```

7. 验证 `Allocatable` 下降。

```bash
kubectl describe node <node-name> | sed -n '/Capacity:/,/System Info:/p'
kubectl top node <node-name>
```

8. 恢复调度。

```bash
kubectl uncordon <node-name>
```

9. 分批推广到同一节点池其它节点，每次只处理少量节点，观察 10 到 30 分钟。

## 验证点

变更前后重点对比：

```bash
kubectl describe node <node-name> | grep -A8 '^Allocatable:'
kubectl describe node <node-name> | grep -A8 '^Allocated resources:'
kubectl top node <node-name>
```

预期结果：

- `Allocatable.cpu` 小于 `Capacity.cpu`，差值接近 `systemReserved.cpu + kubeReserved.cpu`。
- `Allocatable.memory` 小于 `Capacity.memory`，差值接近 `systemReserved.memory + kubeReserved.memory + evictionHard` 隐含预留。
- 节点内存压力升高时，优先出现 pod eviction / 重调度，而不是 node NotReady 或系统 OOM。

## 回滚步骤

如果节点异常，执行：

```bash
kubectl cordon <node-name>
sudo cp /var/lib/kubelet/config.yaml.bak.<timestamp> /var/lib/kubelet/config.yaml
sudo systemctl restart kubelet
kubectl get node <node-name> -w
kubectl uncordon <node-name>
```

如果 kubelet 起不来：

```bash
sudo journalctl -u kubelet -n 200 --no-pager
```

优先检查 YAML 缩进、字段名和单位格式。

## 风险和边界

- CPU 预留只能降低调度密度，不能阻止已经运行的 pod 抢 CPU；CPU 满通常表现为延迟升高，不太会直接打死节点。
- 内存保护更关键。`limit` 很高的 pod 同时上涨时，必须依赖 eviction 或合理的 pod limit/request 比例。
- 如果业务 pod 没有设置内存 limit，kubelet 只能按 QoS / 使用量驱逐，行为可能不符合业务预期。
- 如果节点池由 TKE 等云厂商托管，直接修改节点上的 kubelet 文件可能会被重置；应优先使用节点池 kubelet 参数或新建节点池灰度。

## 配套治理

Node 预留只能兜底，还需要配套控制业务 pod：

1. 给关键服务设置合理 `requests`，避免 request 严重低估。
2. 控制 `limit/request` 比例，例如内存不超过 2 到 4 倍。
3. 对高风险 workload 使用 dedicated node pool、taint/toleration 或 nodeAffinity 隔离。
4. 通过 Prometheus 监控 `node_memory_MemAvailable_bytes`、`kube_node_status_allocatable`、`container_memory_working_set_bytes` 和 pod eviction 事件。
