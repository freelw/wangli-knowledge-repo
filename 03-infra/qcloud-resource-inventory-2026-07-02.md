# QCloud 当前资源使用情况梳理（脱敏版，2026-07-02）

- 生成时间：2026-07-02 08:21 CST
- 查询工具：`tccli 3.1.69.1`
- 查询区域：`ap-nanjing`、`ap-beijing`、`ap-hongkong`
- 查询范围：CVM、CLB、VPC/子网/安全组/NAT/EIP/路由表
- 脱敏规则：不记录完整资源 ID、公网 IP、内网 IP、VPC CIDR、子网 CIDR；只保留资源数量、规格分布、计费分布和用途归类。

## 总览

| Region | CVM | CPU | 内存 GiB | CLB | VPC | 子网 | 安全组 | NAT | EIP | 路由表 |
| --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| `ap-nanjing` | 14 | 160 | 318 | 2 | 4 | 6 | 13 | 2 | 6 | 4 |
| `ap-beijing` | 8 | 84 | 232 | 2 | 2 | 3 | 6 | 2 | 3 | 2 |
| `ap-hongkong` | 12 | 114 | 226 | 1 | 2 | 5 | 8 | 2 | 8 | 2 |

- CVM 合计：34 台，358 核，776 GiB 内存。
- 当前资源以 K8s control-plane、K8s worker、browser-pool、Kong/load-balance、WireGuard/网络出口、agent/工具机几类为主。
- 南京和香港各有少量 spot/auto 机器；北京同时存在一台按量 auto 机器和一台 spot 机器。

## CVM 分布

### ap-nanjing

- 数量：14 台
- CPU / 内存：160 核 / 318 GiB
- 机型分布：`S9.2XLARGE16`=3，`S9.4XLARGE32`=3，`S9.LARGE8`=3，`S9.MEDIUM4`=1，`SA3.2XLARGE16`=1，`SA9.8XLARGE64`=2，`SA9.MEDIUM2`=1
- 计费分布：`PREPAID`=12，`SPOTPAID`=2
- 可用区分布：`ap-nanjing-1`=6，`ap-nanjing-3`=8
- 用途归类：
  - K8s control-plane：3 台，小规格 `S9.LARGE8`
  - K8s worker：3 台，`S9.4XLARGE32`
  - Kong/load-balance：2 台，`S9.2XLARGE16`
  - browser-pool spot/auto：2 台，`SA9.8XLARGE64`
  - agent / WireGuard / 网络加速 / demo 机器：4 台

### ap-beijing

- 数量：8 台
- CPU / 内存：84 核 / 232 GiB
- 机型分布：`SA9.4XLARGE32`=2，`SA9.8XLARGE128`=1，`SA9.LARGE8`=5
- 计费分布：`POSTPAID_BY_HOUR`=1，`PREPAID`=6，`SPOTPAID`=1
- 可用区分布：`ap-beijing-6`=8
- 用途归类：
  - K8s control-plane：3 台，`SA9.LARGE8`
  - K8s worker/base：1 台，`SA9.4XLARGE32`
  - Kong/load-balance：2 台，`SA9.LARGE8`
  - browser-pool auto/spot：2 台，`SA9.4XLARGE32` / `SA9.8XLARGE128`

### ap-hongkong

- 数量：12 台
- CPU / 内存：114 核 / 226 GiB
- 机型分布：`S8.2XLARGE16`=4，`S8.4XLARGE32`=2，`S8.LARGE8`=4，`S8.MEDIUM2`=1，`SA4.8XLARGE64`=1
- 计费分布：`PREPAID`=11，`SPOTPAID`=1
- 可用区分布：`ap-hongkong-2`=9，`ap-hongkong-3`=3
- 用途归类：
  - K8s control-plane：3 台，`S8.LARGE8`
  - K8s worker/base：2 台，`S8.4XLARGE32`
  - browser-pool spot/auto：1 台，`SA4.8XLARGE64`
  - Kong/load-balance：2 台
  - agent / WireGuard / PocketBase / bench 工具机：4 台

## 负载均衡 CLB

| Region | CLB 数量 | 类型 | 用途归类 |
| --- | ---: | --- | --- |
| `ap-nanjing` | 2 | INTERNAL | 内部入口 / 内部控制面入口 |
| `ap-beijing` | 2 | INTERNAL | 内部入口 / 内部控制面入口 |
| `ap-hongkong` | 1 | INTERNAL | 内部入口 |

## VPC 相关

### VPC / 子网

| Region | VPC 数量 | 子网数量 | 主要网络用途 |
| --- | ---: | ---: | --- |
| `ap-nanjing` | 4 | 6 | default-vpc、browser-pool、agent/demo、网络加速 |
| `ap-beijing` | 2 | 3 | default-vpc、browser-pool |
| `ap-hongkong` | 2 | 5 | default-vpc、browser-pool |

### NAT / EIP / 路由表

| Region | NAT | EIP | 路由表 | 说明 |
| --- | ---: | ---: | ---: | --- |
| `ap-nanjing` | 2 | 6 | 4 | default-vpc 和 browser-pool 都有外网出口 |
| `ap-beijing` | 2 | 3 | 2 | default-vpc 和 browser-pool 都有外网出口 |
| `ap-hongkong` | 2 | 8 | 2 | default-vpc 和 browser-pool 都有外网出口，EIP 数量相对更多 |

### 安全组

| Region | 安全组数量 | 关键用途归类 |
| --- | ---: | --- |
| `ap-nanjing` | 13 | browser-pool、Kong、数据服务、WireGuard、调试访问 |
| `ap-beijing` | 6 | browser-pool、Kong、数据服务、基础模板 |
| `ap-hongkong` | 8 | browser-pool、Kong、数据服务、公网访问、办公访问 |

## 观察

1. 三地基础结构基本一致：default-vpc 承载 Kong/control-plane 相关资源，browser-pool VPC 承载浏览器 worker/spot/auto 资源。
2. 南京资源规模最大：CVM、VPC、子网、安全组数量均高于北京和香港。
3. 香港 EIP 数量最高，说明公网入口和出口资源更多，需要重点关注 EIP 绑定关系和公网暴露面。
4. 北京资源集中在单可用区 `ap-beijing-6`；南京和香港有跨可用区分布。
5. Spot/auto 机器主要出现在 browser-pool 相关资源中，和 qcloud-spot-capacity-controller 的定位一致。

## 后续建议

1. 将资源审计分为“脱敏版”和“运维原始版”：脱敏版进入 knowledge-repo，原始版只保留在受控位置。
2. 后续巡检建议增加费用、到期时间、EIP 绑定状态、CLB 监听器数量、安全组入站规则数量等维度。
3. 对香港公网资源单独做一次暴露面梳理，重点看 EIP、CLB、Kong、安全组规则。
