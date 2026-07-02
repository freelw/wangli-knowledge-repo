# QCloud 当前资源使用情况梳理（脱敏版，2026-07-02）

- 生成时间：2026-07-02 08:21 CST
- 查询工具：`tccli 3.1.69.1`
- 查询区域：`ap-nanjing`、`ap-beijing`、`ap-hongkong`
- 查询范围：CVM、CLB、VPC/子网/安全组/NAT/EIP/路由表
- 脱敏规则：不记录完整资源 ID、公网 IP、内网 IP、VPC CIDR、子网 CIDR、资源名称或业务标签；只保留资源数量、规格分布、计费分布、地域/可用区分布。

## 总览

| Region | CVM | CPU | 内存 GiB | CLB | VPC | 子网 | 安全组 | NAT | EIP | 路由表 |
| --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| `ap-nanjing` | 14 | 160 | 318 | 2 | 4 | 6 | 13 | 2 | 6 | 4 |
| `ap-beijing` | 8 | 84 | 232 | 2 | 2 | 3 | 6 | 2 | 3 | 2 |
| `ap-hongkong` | 12 | 114 | 226 | 1 | 2 | 5 | 8 | 2 | 8 | 2 |

- CVM 合计：34 台，358 核，776 GiB 内存。
- 计费类型覆盖包年包月、按量计费和竞价计费。

## CVM 分布

### ap-nanjing

- 数量：14 台
- CPU / 内存：160 核 / 318 GiB
- 机型分布：`S9.2XLARGE16`=3，`S9.4XLARGE32`=3，`S9.LARGE8`=3，`S9.MEDIUM4`=1，`SA3.2XLARGE16`=1，`SA9.8XLARGE64`=2，`SA9.MEDIUM2`=1
- 计费分布：`PREPAID`=12，`SPOTPAID`=2
- 可用区分布：`ap-nanjing-1`=6，`ap-nanjing-3`=8

### ap-beijing

- 数量：8 台
- CPU / 内存：84 核 / 232 GiB
- 机型分布：`SA9.4XLARGE32`=2，`SA9.8XLARGE128`=1，`SA9.LARGE8`=5
- 计费分布：`POSTPAID_BY_HOUR`=1，`PREPAID`=6，`SPOTPAID`=1
- 可用区分布：`ap-beijing-6`=8

### ap-hongkong

- 数量：12 台
- CPU / 内存：114 核 / 226 GiB
- 机型分布：`S8.2XLARGE16`=4，`S8.4XLARGE32`=2，`S8.LARGE8`=4，`S8.MEDIUM2`=1，`SA4.8XLARGE64`=1
- 计费分布：`PREPAID`=11，`SPOTPAID`=1
- 可用区分布：`ap-hongkong-2`=9，`ap-hongkong-3`=3

## 负载均衡 CLB

| Region | CLB 数量 | 类型分布 |
| --- | ---: | --- |
| `ap-nanjing` | 2 | `INTERNAL`=2 |
| `ap-beijing` | 2 | `INTERNAL`=2 |
| `ap-hongkong` | 1 | `INTERNAL`=1 |

## VPC 相关

### VPC / 子网

| Region | VPC 数量 | 子网数量 |
| --- | ---: | ---: |
| `ap-nanjing` | 4 | 6 |
| `ap-beijing` | 2 | 3 |
| `ap-hongkong` | 2 | 5 |

### NAT / EIP / 路由表

| Region | NAT | EIP | 路由表 |
| --- | ---: | ---: | ---: |
| `ap-nanjing` | 2 | 6 | 4 |
| `ap-beijing` | 2 | 3 | 2 |
| `ap-hongkong` | 2 | 8 | 2 |

### 安全组

| Region | 安全组数量 |
| --- | ---: |
| `ap-nanjing` | 13 |
| `ap-beijing` | 6 |
| `ap-hongkong` | 8 |

## 观察

1. CVM 数量：南京 14 台，香港 12 台，北京 8 台。
2. CPU 总量：南京 160 核，香港 114 核，北京 84 核。
3. 内存总量：南京 318 GiB，北京 232 GiB，香港 226 GiB。
4. EIP 数量：香港 8 个，南京 6 个，北京 3 个。
5. 可用区分布：北京集中在 1 个可用区，南京和香港各分布在 2 个可用区。

