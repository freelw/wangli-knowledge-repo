# 网络质量探针接入总结

日期：2026-05-20

## 背景

这轮工作的目标是在所有带有 `browser-ready=true` 标签的浏览器工作节点上部署网络质量探针，用来持续探测节点访问外网的质量，并把指标上报 Prometheus、展示到 Grafana。

核心诉求：

- 每个浏览器节点都要有探针。
- 探针只运行在 `browser-ready=true` 节点。
- 支持被 `browser-worker-node=true:NoSchedule` taint 的节点调度。
- 上报外网访问成功率、总耗时、DNS、TCP、TLS、HTTP 状态码等指标。
- Grafana 曲线名称要简短，能看清探测目标和节点名，例如 `baidu_lexmout-64`。

## 相关 PR

### demo-nodejs-backend

- https://github.com/lexmount/demo-nodejs-backend/pull/182

主要内容：

- `grafana-dashboard-init` 增加 `网络质量探针` dashboard。
- `抓取状态` dashboard 增加 `network-quality-probe` 抓取状态面板。
- 网络质量探针相关曲线 legend 统一改为 `{{probe_target}}_{{node}}`，避免 Grafana 曲线名过长导致 node name 看不到。
- 新 dashboard-init 镜像 tag：`35bb04b-20260520-115006`。

### lexmount-k8s-manifests

- https://github.com/lexmount/lexmount-k8s-manifests/pull/493
- https://github.com/lexmount/lexmount-k8s-manifests/pull/495

主要内容：

- 新增 `apps/network-quality-probe`。
- 将 `network-quality-probe` 接入 office / qcloud / guoge 各环境 overlay。
- Prometheus 增加 blackbox probe scrape jobs。
- 补齐 qcloud-nanjing / qcloud-beijing / qcloud-hk 镜像 tag 和镜像同步。
- 收敛 office-nanjing / office-beijing 旧的完整 Prometheus config patch，统一使用 base `monitoring/prometheus/prometheus-configmap.yaml`。

## 组件选择

本次使用 Prometheus 官方生态的 `blackbox-exporter`。

它是黑盒探测器，不依赖被测服务暴露内部指标，而是从探针 Pod 主动发起 HTTP 请求，并把结果转换成 Prometheus 指标。

这类探测适合回答：

- 某个节点能不能访问外网。
- 某个节点访问指定目标是否成功。
- DNS / TCP / TLS / HTTP 响应耗时分别是多少。
- 不同节点访问同一目标的质量是否有差异。

## Kubernetes 部署形态

### DaemonSet

资源目录：

- `apps/network-quality-probe/`

核心资源：

- `configmap.yaml`
- `daemonset.yaml`
- `service.yaml`
- `kustomization.yaml`

DaemonSet 关键配置：

```yaml
nodeSelector:
  browser-ready: "true"

tolerations:
  - key: browser-worker-node
    operator: Equal
    value: "true"
    effect: NoSchedule
```

含义：

- 只部署到 `browser-ready=true` 的节点。
- 如果浏览器节点带有 `browser-worker-node=true:NoSchedule` taint，探针仍然可以调度上去。

### 安全上下文

review comments 指出容器不应该默认 root 运行，因此补了容器级 `securityContext`：

```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 65534
  readOnlyRootFilesystem: true
  allowPrivilegeEscalation: false
  capabilities:
    drop:
      - ALL
```

原因：`blackbox-exporter` 只需要发起 HTTP 探测，没有提权、写根文件系统或 Linux capabilities 的需求。

### 镜像

当前 blackbox-exporter tag：

- `v0.27.0`

各环境镜像：

- office / guoge：`code.lexmount.net/wangli/blackbox-exporter:v0.27.0`
- qcloud-nanjing：`lexmount.tencentcloudcr.com/cloud/blackbox-exporter:v0.27.0`
- qcloud-hk：`lexmoun-tcr-hk.tencentcloudcr.com/cloud/blackbox-exporter:v0.27.0`
- qcloud-beijing：`lexmount-bj.tencentcloudcr.com/cloud/blackbox-exporter:v0.27.0`

本次已实际同步：

- office registry
- qcloud registry
- qcloud-hk registry
- qcloud-beijing registry

## Prometheus 接入

统一配置位置：

- `monitoring/prometheus/prometheus-configmap.yaml`

本次增加三个 scrape job：

- `network-quality-probe-lexmount-browser`
- `network-quality-probe-tencent-cloud`
- `network-quality-probe-baidu`

探测目标：

- `https://browser.lexmount.cn/`
- `https://cloud.tencent.com/`
- `https://www.baidu.com/`

Prometheus scrape 方式：

- 通过 Kubernetes pod discovery 找到 `system` namespace 下 `app=network-quality-probe` 的 Pod。
- 只抓容器端口 `9115`。
- 请求路径为 `/probe`。
- 使用 blackbox module `http_2xx`。

关键 relabel：

```yaml
- source_labels: [__param_target]
  target_label: instance
```

这个 relabel 是 review comments 中指出的必要修正。没有它时，`instance` 默认是 `pod-ip:9115`；加上以后，`instance` 会变成真实探测 URL，例如：

- `https://browser.lexmount.cn/`
- `https://cloud.tencent.com/`
- `https://www.baidu.com/`

同时保留自定义短标签：

- `probe_target=lexmount-browser`
- `probe_target=tencent-cloud`
- `probe_target=baidu`

## Grafana 展示

Dashboard：

- UID：`browser-platform-network-quality-probe`
- 标题：`网络质量探针`

展示指标：

- 外网探测成功率
- 外网探测总耗时 P95
- DNS 解析耗时 P95
- TCP 连接耗时 P95
- TLS 握手耗时 P95
- HTTP 状态码

同时在 `抓取状态` dashboard 中补充：

- `network-quality-probe` 存活状态
- 按节点查看抓取状态
- 抓取耗时
- 单次抓取样本数

### Legend 规则

用户反馈原始曲线名称太长，node name 看不到。因此网络探针相关面板统一使用：

```text
{{probe_target}}_{{node}}
```

示例：

- `baidu_lexmout-64`
- `tencent-cloud_lexmout-64`
- `lexmount-browser_lexmout-64`

## 常用查询

### 探测是否成功

```promql
probe_success{job="network-quality-probe"}
```

### 总耗时

```promql
probe_duration_seconds{job="network-quality-probe"}
```

### 总耗时 P95

```promql
quantile_over_time(0.95, probe_duration_seconds{job="network-quality-probe"}[5m])
```

### DNS 耗时

```promql
probe_dns_lookup_time_seconds{job="network-quality-probe"}
```

### TCP connect 耗时

```promql
probe_http_duration_seconds{job="network-quality-probe", phase="connect"}
```

### TLS 握手耗时

```promql
probe_http_duration_seconds{job="network-quality-probe", phase="tls"}
```

### 服务端响应等待 / processing 阶段

```promql
probe_http_duration_seconds{job="network-quality-probe", phase="processing"}
```

这个指标可以用来观察 HTTP 请求发出后到响应开始之间的等待时间。

### 内容传输阶段

```promql
probe_http_duration_seconds{job="network-quality-probe", phase="transfer"}
```

## 已验证环境

### office-nanjing

发布命令：

```bash
kubectl --context office-nanjing apply -k apps/clusters/office-nanjing
```

验证结果：

- `network-quality-probe` DaemonSet rollout 成功。
- Pod `1/1 Running`。
- 实际运行节点：`lexmout-64`。
- `securityContext` 已生效。
- Prometheus 已抓到 `probe_success=1`。
- 三个探测目标均成功：
  - `https://browser.lexmount.cn/`
  - `https://cloud.tencent.com/`
  - `https://www.baidu.com/`
- `instance` 标签已是 URL，不再是 Pod IP。
- `grafana-dashboard-init` 已同步 `网络质量探针` dashboard。

### office-beijing

发布命令：

```bash
kubectl --context office-beijing apply -k apps/clusters/office-beijing
```

验证结果：

- `network-quality-probe` DaemonSet rollout 成功。
- Pod `1/1 Running`。
- 实际运行节点：`lexmount-59-15`。
- `securityContext` 已生效。
- Prometheus 已抓到 `probe_success=1`。
- 三个探测目标均成功：
  - `https://browser.lexmount.cn/`
  - `https://cloud.tencent.com/`
  - `https://www.baidu.com/`

## qcloud 三环境发布准备

用户确认 PR 合入后指出 qcloud 三个环境需要都放进去。检查最新 main 后发现：

- `qcloud-nanjing` 已包含 `network-quality-probe`。
- `qcloud-beijing` 已包含 `network-quality-probe`。
- `qcloud-hk` 已包含 `network-quality-probe`。

补充动作：

- 同步 `blackbox-exporter:v0.27.0` 到 qcloud / hk / beijing registry。
- 同步 `grafana-dashboard-init:35bb04b-20260520-115006` 到 qcloud / hk / beijing registry。
- 对齐 qcloud-beijing 的 `grafana-dashboard-init-image`。
- 在 DaemonSet 中补充 `browser-worker-node=true:NoSchedule` toleration。

PR：

- https://github.com/lexmount/lexmount-k8s-manifests/pull/495

## 发布注意事项

### 必须显式指定 context

这轮 office-beijing 发布过程中出现过一次发布失误：当前 kube context 是 `office-nanjing`，但执行了：

```bash
kubectl apply -k apps/clusters/office-beijing
```

结果导致 office-nanjing 集群短暂被应用了 office-beijing overlay 中的部分 Service 配置。

后续跨环境发布必须显式指定 context：

```bash
kubectl --context office-nanjing apply -k apps/clusters/office-nanjing
kubectl --context office-beijing apply -k apps/clusters/office-beijing
kubectl --context qcloud-nanjing apply -k apps/clusters/qcloud-nanjing
kubectl --context qcloud-beijing apply -k apps/clusters/qcloud-beijing
kubectl --context qcloud-hk apply -k apps/clusters/qcloud-hk
```

不要依赖当前默认 context。

### office-nanjing kong-proxy 修复

误 apply 后，office-nanjing 的 `kong-proxy.externalIPs` 曾被临时改成北京 IP。

修复方式：

```bash
kubectl --context office-nanjing apply -k apps/clusters/office-nanjing
```

修复后确认：

- `kong-proxy.spec.externalIPs=10.3.3.176`
- `80 -> 8000`
- `443 -> 8443`
- `nodePort 30443`
- endpoints 正常包含 `:8443` 和 `:8000`

### office-nanjing kong-service-patch

office-nanjing 本身已有正确 patch：

- `apps/clusters/office-nanjing/kong-service-patch.yaml`

内容：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: kong-proxy
  namespace: system
spec:
  externalIPs:
  - 10.3.3.176
```

这个 patch 没问题；之前的问题是 overlay 和 kube context 不匹配。

## 后续建议

1. qcloud 三环境合入后按显式 context 逐个发布并验证 `probe_success`。
2. 如果需要更细分响应耗时，可以在 Grafana 增加 `phase="processing"` 和 `phase="transfer"` 面板。
3. 如后续新增探测目标，优先新增 `probe_target` 短标签，避免 legend 过长。
4. 发布脚本或 runbook 应加入 overlay/context 一致性检查，避免再次把 A 环境 overlay apply 到 B 集群。
