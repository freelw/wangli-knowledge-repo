# 区域与路由

## 目的

这篇文档用于回答 3 个问题：

1. 当前主要环境和区域怎么分
2. 不同环境的流量和发布路径有什么差异
3. 遇到跨环境访问、入口、Grafana 路由问题时应该先看哪里

## 当前主要环境

目前这套平台里，基础设施层最常出现的环境是：

1. `office`
2. `qcloud`
3. `qcloud-hk`

其中协作上的默认规则已经明确：

1. 日常开发和验证默认在 `office`
2. 正式发布由负责人统一执行

## 基础分工

### `office`

定位：

1. 默认开发 / 验证环境
2. 当前机器可直接 `kubectl` 连接
3. 大多数链路先在这里做最小闭环验证

特点：

1. 发布路径最清晰
2. 常用镜像多使用 `code.lexmount.net/wangli/...`
3. 很多“先验证再推广”的动作都先落这里

### `qcloud`

定位：

1. 腾讯云主环境
2. 更接近正式运行环境
3. 常用于验证腾讯云侧配置和外部访问链路

特点：

1. 常用镜像仓库是 `lexmount.tencentcloudcr.com/cloud/...`
2. 会涉及腾讯云网络、Kong、入口机、白名单等问题
3. 不是默认本机直连验证环境

### `qcloud-hk`

定位：

1. 香港区域环境
2. 用于区域部署、网络差异和 HK 镜像同步

特点：

1. 镜像仓库通常是 HK TCR
2. 很多问题不在业务代码，而在镜像同步、区域网络和安全组
3. 常和 `qcloud` 成对出现，但不能简单视为完全相同

## 基础流量理解

从基础设施视角，可以先把流量分成三类：

1. 用户 / SDK 请求先进入入口层，例如 Kong
2. 控制面请求进入 `browser-manager`
3. websocket 长连接进入 `browser-ws-gateway`

如果是运维类入口，还会出现：

1. VPN / 入口机
2. Kong NodePort
3. `ops-grafana` 这类内部运维路由

## 跨环境路由的常见差异

### 1. 入口差异

`office` 更适合直接做开发验证；`qcloud` / `qcloud-hk` 更容易遇到：

1. 内外网入口差异
2. Kong 路由差异
3. 白名单 / `ip-restriction` 差异

### 2. 镜像差异

同一个版本在不同环境不一定是同一个 registry 地址：

1. `office` 常用 `code.lexmount.net`
2. `qcloud` 常用腾讯云主区域 TCR
3. `qcloud-hk` 常用香港 TCR

所以跨环境问题里，先确认“是不是同一份镜像”比先怀疑代码更重要。

### 3. 网络差异

`qcloud` 和 `qcloud-hk` 经常需要额外确认：

1. VPC / 子网
2. Peering
3. NAT
4. 安全组
5. 路由表

这些信息在 `office` 场景下一般不是第一排查点，但在腾讯云区域问题里经常是根因。

## 遇到问题先看哪里

### 场景 1：开发验证为什么默认先走 `office`

优先看：

1. `00-overview/environments.md`
2. `02-operations/office-runbook.md`
3. `02-operations/release-overview.md`

### 场景 2：为什么 `qcloud` 正常、`qcloud-hk` 不正常

优先看：

1. `03-infra/image-sync.md`
2. `03-infra/tencent-network-topology.md`
3. 对应环境的 `lexmount-k8s-manifests/apps/clusters/*`

### 场景 3：Grafana / Kong / 入口机访问链路异常

优先看：

1. `demo-nodejs-backend/kong-init/config-*.js`
2. `lexmount-k8s-manifests/apps/kong/*`
3. `03-infra/tencent-network-topology.md`

## 当前结论

如果只记住一句话，可以记住：

1. `office` 是默认开发验证环境
2. `qcloud` / `qcloud-hk` 是更偏区域和基础设施差异的环境
3. 基础设施排查里，入口、镜像仓库和网络边界通常比业务代码更先出问题
