# spot controller kubeadm join kubelet 配置修正

## 背景

`qcloud-spot-capacity-controller` 通过腾讯云 TAT 在新购买的 spot CVM 上执行 `kubeadm join`，把实例加入 Kubernetes 集群。

这次需要在 join 时定制 kubelet 配置，为 spot worker node 预留系统资源并设置 eviction 阈值：

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

涉及仓库：

- `qcloud-k8s-tool`
- `lexmount-k8s-manifests`

## PR

最终有效 PR：

- `qcloud-k8s-tool` PR #35：`fix: use kubeadm patches for kubelet config`
- `lexmount-k8s-manifests` PR #548：`chore: update spot controller kubeadm patches image`

前置 PR #34 / #547 曾合入第一版实现，但第一版经真实节点验证后发现路径不正确，随后用 #35 / #548 修正。

## 第一版问题

第一版尝试把 `KubeletConfiguration` 作为第二个 YAML document 放进 `kubeadm join --config` 使用的配置文件：

```yaml
apiVersion: kubeadm.k8s.io/v1beta3
kind: JoinConfiguration
...
---
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
systemReserved:
  cpu: "500m"
  memory: "1Gi"
...
```

实际新建 spot 节点后，查看：

```bash
cat /var/lib/kubelet/config.yaml
```

发现目标字段没有落盘。说明在当前 join 路径下，单独追加 `KubeletConfiguration` document 不会被可靠持久化到 kubelet 最终配置。

需要注意：Go 代码里如果在 `fmt.Sprintf` 模板中输出 `%`，源码必须写成 `%%`。例如源码 `"5%%"` 实际生成 YAML 时是 `"5%"`。这不是 bug，但第一版路径本身不可靠。

## 最终方案

最终改为 kubeadm 原生 patches 机制。

生成的 `JoinConfiguration` 中增加：

```yaml
patches:
  directory: /root/kubeadm-patches
```

执行 `kubeadm join` 前，在实例上生成 patch 文件：

```bash
mkdir -p /root/kubeadm-patches
tee /root/kubeadm-patches/kubeletconfiguration0+merge.yaml <<'EOF'
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
EOF
```

然后执行：

```bash
kubeadm join --config kubeadm-config.yaml
```

patch 文件命名 `kubeletconfiguration0+merge.yaml` 的含义：

- target：`kubeletconfiguration`
- patch index：`0`
- patch type：`merge`

这样 kubeadm 会在 join 过程中把 patch merge 到它生成并写入 `/var/lib/kubelet/config.yaml` 的 `KubeletConfiguration`。

## 实际验证

新建 spot 节点后，查看 `/var/lib/kubelet/config.yaml`，目标字段已经存在：

```yaml
evictionHard:
  imagefs.available: 10%
  memory.available: 5%
  nodefs.available: 10%
evictionSoft:
  memory.available: 8%
evictionSoftGracePeriod:
  memory.available: 1m
kubeReserved:
  cpu: 300m
  memory: 512Mi
systemReserved:
  cpu: 500m
  memory: 1Gi
```

YAML 中引号被 kubeadm/kubelet 规范化去掉是正常现象，语义不变。

## 镜像和 manifests

最终 controller 镜像 tag：

```text
e3ba012-20260525-213944
```

三地 manifests 已同步：

- `qcloud-nanjing`: `lexmount.tencentcloudcr.com/cloud/qcloud-spot-capacity-controller:e3ba012-20260525-213944`
- `qcloud-beijing`: `lexmount-bj.tencentcloudcr.com/cloud/qcloud-spot-capacity-controller:e3ba012-20260525-213944`
- `qcloud-hk`: `lexmoun-tcr-hk.tencentcloudcr.com/cloud/qcloud-spot-capacity-controller:e3ba012-20260525-213944`

发布前验证：

- `go test ./...`
- `kubectl kustomize apps/clusters/qcloud-nanjing`
- `kubectl kustomize apps/clusters/qcloud-beijing`
- `kubectl kustomize apps/clusters/qcloud-hk`
- `skopeo inspect` 确认三地 registry tag 均存在

## 同期配置

同一轮 manifests 中，`qcloud-hk` 的 `watermark-monitor` 自动扩容已关闭，只保留 Prometheus metrics 上报：

```yaml
watermark-monitor-enable-scf-trigger: "false"
```

并通过 kustomize replacement 注入到：

```text
ENABLE_SCF_TRIGGER=false
```

## 经验记录

1. 不要假设 `kubeadm join --config` 中的多 document `KubeletConfiguration` 一定会落到最终 kubelet 配置；真实节点要以 `/var/lib/kubelet/config.yaml` 验证。
2. 修改 kubeadm 生成的 kubelet config，优先使用 kubeadm patches 机制，而不是 join 后手工 append 配置文件。
3. 对 `fmt.Sprintf` 模板中的 `%`，源码必须写 `%%`，测试要验证生成结果没有残留 `%%`。
4. manifests 引用多地域 registry tag 前，要确认对应 registry 中 tag 可用，避免部署时 image pull 失败。
