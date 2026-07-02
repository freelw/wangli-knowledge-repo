# QCloud 当前资源使用情况梳理（2026-07-02）

- 生成时间：2026-07-02 08:21 CST
- 查询工具：`tccli 3.1.69.1`
- 查询区域：`ap-nanjing`、`ap-beijing`、`ap-hongkong`
- 查询范围：CVM、TencentDB PostgreSQL、CLB、VPC/子网/安全组/NAT/EIP/路由表、Redis
- 安全说明：本文只记录资源清单和使用统计，不包含密钥、密码或访问凭证。

## 总览

| Region | CVM | CPU | 内存 GiB | PostgreSQL | CLB | VPC | 子网 | 安全组 | NAT | EIP | 路由表 | Redis |
| --- | ---: | ---: | ---: | --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | --- |
| `ap-nanjing` | 14 | 160 | 318 | 权限不足 | 2 | 4 | 6 | 13 | 2 | 6 | 4 | 权限不足 |
| `ap-beijing` | 8 | 84 | 232 | 权限不足 | 2 | 2 | 3 | 6 | 2 | 3 | 2 | 权限不足 |
| `ap-hongkong` | 12 | 114 | 226 | 权限不足 | 1 | 2 | 5 | 8 | 2 | 8 | 2 | 权限不足 |

- CVM 合计：34 台，358 核，776 GiB 内存。
- PostgreSQL 查询失败：当前 `tccli` 凭证缺少 `postgres:DescribeDBInstances` 权限。
- Redis 查询失败：当前 `tccli` 凭证缺少 `redis:DescribeInstances` 权限。

## CVM

### ap-nanjing

- 数量：14 台
- CPU / 内存：160 核 / 318 GiB
- 机型分布：`S9.2XLARGE16`=3，`S9.4XLARGE32`=3，`S9.LARGE8`=3，`S9.MEDIUM4`=1，`SA3.2XLARGE16`=1，`SA9.8XLARGE64`=2，`SA9.MEDIUM2`=1
- 计费分布：`PREPAID`=12，`SPOTPAID`=2
- 可用区分布：`ap-nanjing-1`=6，`ap-nanjing-3`=8

| ID | 名称 | 可用区 | 机型 | CPU | 内存 GiB | 计费 | 内网 IP | 公网 IP | VPC | 子网 | 角色 | 标签 |
| --- | --- | --- | --- | ---: | ---: | --- | --- | --- | --- | --- | --- | --- |
| `ins-662anudo` | 未命名2 | ap-nanjing-1 | SA9.8XLARGE64 | 32 | 64 | SPOTPAID | 10.0.2.15 |  | vpc-oi69rofn | subnet-acxlzj0o | cos-operator | level=auto |
| `ins-kts1f406` | 未命名1 | ap-nanjing-1 | SA9.8XLARGE64 | 32 | 64 | SPOTPAID | 10.0.2.3 |  | vpc-oi69rofn | subnet-acxlzj0o | cos-operator | level=auto |
| `ins-jumbul7k` | load-balance-kong1 | ap-nanjing-3 | S9.2XLARGE16 | 8 | 16 | PREPAID | 10.206.0.63 |  | vpc-j14fqvjv | subnet-bkkl5mzq |  | k8s=worker,level=base |
| `ins-fxjohi2y` | net-speed-up | ap-nanjing-1 | SA9.MEDIUM2 | 2 | 2 | PREPAID | 10.0.0.14 | 119.45.94.33 | vpc-i1rlk89r | subnet-ewhsgoo4 |  | level=base |
| `ins-dlg3lr3c` | k8s-worker-0 | ap-nanjing-1 | S9.4XLARGE32 | 16 | 32 | PREPAID | 10.0.2.6 |  | vpc-oi69rofn | subnet-acxlzj0o | cos-operator | k8s=worker |
| `ins-bzoj5uyu` | k8s-worker-1 | ap-nanjing-1 | S9.4XLARGE32 | 16 | 32 | PREPAID | 10.0.2.7 |  | vpc-oi69rofn | subnet-acxlzj0o | cos-operator | k8s=worker |
| `ins-io3uuv9i` | k8s-worker-2 | ap-nanjing-1 | S9.4XLARGE32 | 16 | 32 | PREPAID | 10.0.2.5 |  | vpc-oi69rofn | subnet-acxlzj0o | cos-operator | k8s=worker |
| `ins-kmzj0yrk` | wireguard-gateway | ap-nanjing-3 | S9.MEDIUM4 | 2 | 4 | PREPAID | 10.206.0.79 |  | vpc-j14fqvjv | subnet-bkkl5mzq |  | level=base |
| `ins-ph1kz9ym` | agent-0 | ap-nanjing-3 | S9.2XLARGE16 | 8 | 16 | PREPAID | 10.206.0.2 |  | vpc-j14fqvjv | subnet-bkkl5mzq | cos-operator | level=base |
| `ins-i3guuqem` | k8s-control-plane-1 | ap-nanjing-3 | S9.LARGE8 | 4 | 8 | PREPAID | 10.206.0.45 |  | vpc-j14fqvjv | subnet-bkkl5mzq | scf-emitter | level=base,k8s=control |
| `ins-d1vs0v0i` | k8s-control-plane-2 | ap-nanjing-3 | S9.LARGE8 | 4 | 8 | PREPAID | 10.206.0.6 |  | vpc-j14fqvjv | subnet-bkkl5mzq | scf-emitter | level=base,k8s=control |
| `ins-g4chfp48` | k8s-control-plane-3 | ap-nanjing-3 | S9.LARGE8 | 4 | 8 | PREPAID | 10.206.0.30 |  | vpc-j14fqvjv | subnet-bkkl5mzq | scf-emitter | level=base,k8s=control |
| `ins-01l52yfe` | load-balance-kong0 | ap-nanjing-3 | S9.2XLARGE16 | 8 | 16 | PREPAID | 10.206.0.23 |  | vpc-j14fqvjv | subnet-bkkl5mzq |  | level=base,k8s=worker |
| `ins-ky2g91es` | livekit-demo | ap-nanjing-3 | SA3.2XLARGE16 | 8 | 16 | PREPAID | 10.10.0.2 | 119.45.14.38 | vpc-ai2bq69d | subnet-kvsisr5u |  | level=base |

### ap-beijing

- 数量：8 台
- CPU / 内存：84 核 / 232 GiB
- 机型分布：`SA9.4XLARGE32`=2，`SA9.8XLARGE128`=1，`SA9.LARGE8`=5
- 计费分布：`POSTPAID_BY_HOUR`=1，`PREPAID`=6，`SPOTPAID`=1
- 可用区分布：`ap-beijing-6`=8

| ID | 名称 | 可用区 | 机型 | CPU | 内存 GiB | 计费 | 内网 IP | VPC | 子网 | 角色 | 标签 |
| --- | --- | --- | --- | ---: | ---: | --- | --- | --- | --- | --- | --- |
| `ins-bmbz5vgj` | auto-k8s-worker-20260702-080032 | ap-beijing-6 | SA9.4XLARGE32 | 16 | 32 | POSTPAID_BY_HOUR | 10.0.0.2 | vpc-ctnsn4oc | subnet-geb40ml5 | cos-operator | level=auto |
| `ins-exx5i3v3` | 未命名 | ap-beijing-6 | SA9.8XLARGE128 | 32 | 128 | SPOTPAID | 10.0.0.4 | vpc-ctnsn4oc | subnet-geb40ml5 | cos-operator | level=auto |
| `ins-6kvbk91l` | load-balance-kong1 | ap-beijing-6 | SA9.LARGE8 | 4 | 8 | PREPAID | 172.21.0.6 | vpc-adabp2ha | subnet-nzjxgchl |  | level=base |
| `ins-cyys62az` | k8s-worker-0 | ap-beijing-6 | SA9.4XLARGE32 | 16 | 32 | PREPAID | 10.0.0.9 | vpc-ctnsn4oc | subnet-geb40ml5 |  | level=base |
| `ins-inw6dk3t` | load-balance-kong0 | ap-beijing-6 | SA9.LARGE8 | 4 | 8 | PREPAID | 172.21.0.14 | vpc-adabp2ha | subnet-nzjxgchl |  | level=base |
| `ins-oops9xu3` | k8s-master-1-bj | ap-beijing-6 | SA9.LARGE8 | 4 | 8 | PREPAID | 172.21.0.5 | vpc-adabp2ha | subnet-nzjxgchl | scf-emitter | level=base |
| `ins-lmnm5bwv` | k8s-master-0-bj | ap-beijing-6 | SA9.LARGE8 | 4 | 8 | PREPAID | 172.21.0.10 | vpc-adabp2ha | subnet-nzjxgchl | scf-emitter | level=base |
| `ins-5e6yvybr` | k8s-master-2-bj | ap-beijing-6 | SA9.LARGE8 | 4 | 8 | PREPAID | 172.21.0.11 | vpc-adabp2ha | subnet-nzjxgchl | scf-emitter | level=base |

### ap-hongkong

- 数量：12 台
- CPU / 内存：114 核 / 226 GiB
- 机型分布：`S8.2XLARGE16`=4，`S8.4XLARGE32`=2，`S8.LARGE8`=4，`S8.MEDIUM2`=1，`SA4.8XLARGE64`=1
- 计费分布：`PREPAID`=11，`SPOTPAID`=1
- 可用区分布：`ap-hongkong-2`=9，`ap-hongkong-3`=3

| ID | 名称 | 可用区 | 机型 | CPU | 内存 GiB | 计费 | 内网 IP | 公网 IP | VPC | 子网 | 角色 | 标签 |
| --- | --- | --- | --- | ---: | ---: | --- | --- | --- | --- | --- | --- | --- |
| `ins-9fw1uhsw` | 未命名 | ap-hongkong-2 | SA4.8XLARGE64 | 32 | 64 | SPOTPAID | 172.16.0.11 |  | vpc-aib9lae0 | subnet-grl1hwx1 | cos-operator | level=auto |
| `ins-47iop1zi` | load-balance-2 | ap-hongkong-2 | S8.LARGE8 | 4 | 8 | PREPAID | 172.19.0.7 |  | vpc-g6ihi84c | subnet-0uexzra9 |  | level=base |
| `ins-ircesq1w` | wireguard | ap-hongkong-2 | S8.MEDIUM2 | 2 | 2 | PREPAID | 172.19.0.6 |  | vpc-g6ihi84c | subnet-0uexzra9 |  | level=base |
| `ins-ib4l871m` | k8s-control-plane-hk-1 | ap-hongkong-3 | S8.LARGE8 | 4 | 8 | PREPAID | 172.19.16.2 |  | vpc-g6ihi84c | subnet-6nce03xn | scf-emitter | level=base |
| `ins-hjfu1lb8` | k8s-control-plane-hk-2 | ap-hongkong-3 | S8.LARGE8 | 4 | 8 | PREPAID | 172.19.16.13 |  | vpc-g6ihi84c | subnet-6nce03xn | scf-emitter | level=base |
| `ins-lfac58zm` | k8s-control-plane-hk-3 | ap-hongkong-3 | S8.LARGE8 | 4 | 8 | PREPAID | 172.19.16.17 |  | vpc-g6ihi84c | subnet-6nce03xn | scf-emitter | level=base |
| `ins-fgpngx2k` | load-balance | ap-hongkong-2 | S8.2XLARGE16 | 8 | 16 | PREPAID | 172.19.0.9 |  | vpc-g6ihi84c | subnet-0uexzra9 |  | level=base |
| `ins-3isr3p2m` | agent-0 | ap-hongkong-2 | S8.2XLARGE16 | 8 | 16 | PREPAID | 172.19.0.10 | 119.28.12.108 | vpc-g6ihi84c | subnet-0uexzra9 | cos-operator | level=base |
| `ins-i0of03s8` | pocketbase-hk | ap-hongkong-2 | S8.2XLARGE16 | 8 | 16 | PREPAID | 172.19.0.15 | 119.28.212.234 | vpc-g6ihi84c | subnet-0uexzra9 |  | level=base |
| `ins-8goggfts` | bubench | ap-hongkong-2 | S8.2XLARGE16 | 8 | 16 | PREPAID | 172.19.0.3 | 43.154.206.213 | vpc-g6ihi84c | subnet-0uexzra9 |  | level=base |
| `ins-qborcesq` | k8s-worker-0 | ap-hongkong-2 | S8.4XLARGE32 | 16 | 32 | PREPAID | 172.16.0.7 |  | vpc-aib9lae0 | subnet-grl1hwx1 | cos-operator | level=base |
| `ins-jrv2uj2k` | k8s-worker-1 | ap-hongkong-2 | S8.4XLARGE32 | 16 | 32 | PREPAID | 172.16.0.3 |  | vpc-aib9lae0 | subnet-grl1hwx1 | cos-operator | level=base |

## PostgreSQL

三地均未查询成功。当前 `tccli` 凭证缺少：

- `postgres:DescribeDBInstances`
- 资源范围示例：`qcs::postgres:<region>:uin/100044136021:DBInstanceId/*`

## 负载均衡 CLB

| Region | ID | 名称 | 类型 | VPC | VIP | 计费 | 创建时间 |
| --- | --- | --- | --- | --- | --- | --- | --- |
| ap-nanjing | `lb-achyyuq6` | lexmount-inner-lb | INTERNAL | vpc-j14fqvjv | 10.206.0.89 | POSTPAID_BY_HOUR | 2026-05-12 20:29:39 |
| ap-nanjing | `lb-bin957k8` | lexmount-lb | INTERNAL | vpc-j14fqvjv | 10.206.0.26 | POSTPAID_BY_HOUR | 2026-02-02 11:34:47 |
| ap-beijing | `lb-f8ioisu9` | lexmount-lb | INTERNAL | vpc-adabp2ha | 172.21.0.3 | POSTPAID_BY_HOUR | 2026-05-12 20:45:35 |
| ap-beijing | `lb-izahtbhf` | lexmount-inner-lb | INTERNAL | vpc-adabp2ha | 172.21.0.2 | POSTPAID_BY_HOUR | 2026-05-12 19:35:52 |
| ap-hongkong | `lb-i0i3drvy` | lb-69c3a1a6-fHkO | INTERNAL | vpc-g6ihi84c | 172.19.240.7 | POSTPAID_BY_HOUR | 2026-03-25 16:49:45 |

## VPC 相关

### VPC

| Region | VPC | 名称 | CIDR | 默认 VPC |
| --- | --- | --- | --- | --- |
| ap-nanjing | `vpc-ai2bq69d` | for-agent | 10.10.0.0/16 | False |
| ap-nanjing | `vpc-i1rlk89r` | for-net-speed-up | 10.0.0.0/16 | False |
| ap-nanjing | `vpc-oi69rofn` | browser-pool | 10.0.0.0/16 | False |
| ap-nanjing | `vpc-j14fqvjv` | Default-VPC | 10.206.0.0/16 | True |
| ap-beijing | `vpc-ctnsn4oc` | browser-pool | 10.0.0.0/16 | False |
| ap-beijing | `vpc-adabp2ha` | Default-VPC | 172.21.0.0/16 | True |
| ap-hongkong | `vpc-aib9lae0` | browser-pool | 172.16.0.0/16 | False |
| ap-hongkong | `vpc-g6ihi84c` | Default-VPC | 172.19.0.0/16 | True |

### 子网数量

| Region | 子网数量 | 主要子网 |
| --- | ---: | --- |
| ap-nanjing | 6 | `browser-pool-subnet-2`、`browser-pool-subnet-1`、`kong-subnet`、`pg-scf-subnet`、`live`、`subnet0` |
| ap-beijing | 3 | `browser-pool-subnet-0`、`Default-Subnet`、`pg-scf-subnet` |
| ap-hongkong | 5 | `browser-pool-0`、`browser-pool-1`、`kong-subnet-0`、`kong-subnet-1`、`pg-scf-subnet` |

### NAT / EIP

| Region | NAT | EIP | 说明 |
| --- | ---: | ---: | --- |
| ap-nanjing | 2 | 6 | `default-vpc-nat`，`browser-pool外网访问打通` |
| ap-beijing | 2 | 3 | `default-vpc-nat`，browser-pool NAT |
| ap-hongkong | 2 | 8 | `default-vpc-nat`，`browser-pool外网访问打通` |

### 安全组

| Region | 安全组数量 | 关键安全组 |
| --- | ---: | --- |
| ap-nanjing | 13 | `browser-pool`、`kong-pool`、`lexmount-pg`、`lexmount-redis`、`wireguard-gateway` |
| ap-beijing | 6 | `browser-pool`、`kong-pool`、`lexmount-pg`、`lexmount-redis` |
| ap-hongkong | 8 | `browser-pool`、`lexmount-kong`、`lexmount-pg`、`lexmount-redis`、`public-visit` |

## Redis

三地均未查询成功。当前 `tccli` 凭证缺少：

- `redis:DescribeInstances`
- 资源范围示例：`qcs:id/0:redis:<region>:uin/100044136021:instance/*`

## 后续建议

1. 给当前查询账号补充只读权限：`postgres:DescribeDBInstances`、`redis:DescribeInstances`。
2. 如果要长期做资源审计，建议将本次查询脚本固化为只读巡检任务，输出到 knowledge-repo 或对象存储。
3. CVM 侧已经能看到 spot/base、K8s control/worker、browser-pool/default-vpc 的资源结构；后续可以增加费用维度和过期时间维度。
