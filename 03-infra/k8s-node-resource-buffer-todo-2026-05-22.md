# K8s Node 预留 5% 资源 TODO

日期：2026-05-22

## 背景

线上节点可能出现 pod `requests` 不高但 `limits` 很高的情况。调度时 scheduler 只看 `requests`，运行时多个 pod 同时上涨时可能把 node 内存打满，导致 node 不稳定或 OOM。

目标是在 node 层面预留约 5% CPU / 内存，保证系统组件和 kubelet 有缓冲空间。

## TODO

1. 调研各线上集群是否能直接修改 kubelet 配置。
2. 对自建或可控节点池，配置 `systemReserved` / `kubeReserved`，让 Node `Allocatable` 比真实容量少约 5%。
3. 配置 kubelet eviction 策略，优先保护内存，例如 `memory.available<5%` 或按机器规格使用绝对值，如 `memory.available<2Gi`。
4. 分批在非关键节点验证：`cordon -> drain -> 修改 kubelet config -> restart kubelet -> uncordon`。
5. 验证 `kubectl describe node` 中 `Allocatable` 是否下降，压测确认内存上涨时 kubelet 会驱逐 pod 而不是 node OOM。
6. 补充 pod 侧策略，限制 `limit/request` 比例，避免低 request 高 limit 的 pod 绕过调度层保护。

## 建议配置方向

示例仅用于说明，实际值应按节点规格折算：

```yaml
systemReserved:
  cpu: "500m"
  memory: "1Gi"
kubeReserved:
  cpu: "500m"
  memory: "1Gi"
evictionHard:
  memory.available: "5%"
  nodefs.available: "10%"
evictionSoft:
  memory.available: "8%"
evictionSoftGracePeriod:
  memory.available: "1m"
```

## 注意事项

- CPU 主要是争抢和限流问题，通常不会直接导致 node 死机；内存是更关键的保护对象。
- `kubeReserved` / `systemReserved` 解决调度层预留，`evictionHard` / `evictionSoft` 解决运行时突增。
- 如果是托管 K8s / TKE，可能不能直接改 `/var/lib/kubelet/config.yaml`，需要通过节点池 kubelet 参数、启动参数或新建节点池迁移来落地。
- 仅修改业务 pod manifests 不能保证 node 留 5%，必须落到 kubelet reserved/eviction 或节点池配置。
