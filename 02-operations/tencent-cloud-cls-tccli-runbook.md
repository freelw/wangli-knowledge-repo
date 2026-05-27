# 腾讯云 CLS tccli 查询 Runbook

## 文档定位

这篇文档记录通过 `tccli` 查询腾讯云日志服务 CLS 和 CVM 信息的最小操作步骤。

CLS 日志是云平台最重要的故障分析入口之一。排查线上问题时，应优先通过日志集、日志主题、时间范围和关键字缩小范围，再进入具体服务或实例排查。

## 前提

本机需要安装并配置 `tccli`。

验证：

```bash
tccli cls help
tccli cvm help
tccli configure list
```

配置密钥：

```bash
tccli configure
```

按提示输入：

```text
TencentCloud API secretId: <SecretId>
TencentCloud API secretKey: <SecretKey>
Default region name: ap-nanjing
Default output format: json
```

也可以使用 profile，避免覆盖默认配置：

```bash
tccli configure --profile qcloud-nanjing
tccli cvm DescribeInstances --profile qcloud-nanjing --region ap-nanjing --Limit 100
```

临时验证也可以使用环境变量：

```bash
export TENCENTCLOUD_SECRET_ID='<SecretId>'
export TENCENTCLOUD_SECRET_KEY='<SecretKey>'
export TENCENTCLOUD_REGION='ap-nanjing'
```

不要把 `SecretId` / `SecretKey` 发送到群聊或提交到仓库。

## CLS 最小权限

只查询日志时，核心权限是：

```text
cls:SearchLog
```

常用排障权限集合：

```json
{
  "version": "2.0",
  "statement": [
    {
      "effect": "allow",
      "action": [
        "cls:SearchLog",
        "cls:DescribeLogsets",
        "cls:DescribeTopics",
        "cls:DescribeIndex",
        "cls:DescribeLogContext"
      ],
      "resource": "*"
    }
  ]
}
```

用途拆分：

```text
cls:SearchLog          查询日志
cls:DescribeLogsets    列日志集
cls:DescribeTopics     列日志主题
cls:DescribeIndex      查看索引配置
cls:DescribeLogContext 查看某条日志上下文
```

如果已经知道 `TopicId`，只查日志最小只需要 `cls:SearchLog`。

资源可以进一步收紧到指定 topic/logset：

```text
qcs::cls:<region>:uin/<uin>:topic/<topicId>
qcs::cls:<region>:uin/<uin>:logset/<logsetId>
```

落地建议：

1. 先用 `resource: "*"` 验证能查。
2. 确认目标 topic/logset 后再收紧资源范围。
3. 多 topic 查询需要覆盖所有目标 topic 的 `cls:SearchLog` 权限。

## CVM 查询权限

查看 CVM 实例列表需要：

```text
cvm:DescribeInstances
```

查询南京 CVM：

```bash
tccli cvm DescribeInstances --region ap-nanjing --Limit 100
```

只输出实例关键信息：

```bash
tccli cvm DescribeInstances --region ap-nanjing --Limit 100 \
  | jq -r '.InstanceSet[] | [
      .InstanceId,
      .InstanceName,
      .Placement.Zone,
      .InstanceState,
      .InstanceChargeType,
      (.CPU|tostring),
      (.Memory|tostring),
      (.PrivateIpAddresses|join(","))
    ] | @tsv'
```

## 查询 CLS 日志集

南京地域：

```bash
tccli cls DescribeLogsets --region ap-nanjing --Limit 100
```

2026-05-27 查询到的南京日志集：

```text
0d5c6aca-f836-4abb-905c-5d5fad1871e2  k8s-control-plane      topics=4
3f5223da-51a4-443f-8c15-9e5ce6a64eca  k8s-browser-worker     topics=11
af432fae-23e5-449e-a9d4-1a89868bb4d9  cls_service_logging    topics=3
6692d2a8-b0dc-401d-a71b-8d5ef77c6fdd  qcloud-k8s             topics=0
00596969-fe2a-49be-98ff-fe23e0d6ef6b  SCF_logset_pmLvIkyt    topics=1
```

## 查询 k8s-browser-worker 日志主题

`k8s-browser-worker` 的 `LogsetId`：

```text
3f5223da-51a4-443f-8c15-9e5ce6a64eca
```

查询命令：

```bash
tccli cls DescribeTopics \
  --region ap-nanjing \
  --Filters '[{"Key":"logsetId","Values":["3f5223da-51a4-443f-8c15-9e5ce6a64eca"]}]' \
  --Limit 100
```

2026-05-27 查询到的 topic 清单：

```text
session-quota                 9ec7bb8e-a3d1-47ec-bb82-e8724722133d  partitions=1
region-data-plane-gateway     4de72b41-c049-4e49-9fcf-7cd2bba3e691  partitions=1
browser-ws-gateway            392cb5b1-d58d-4287-94b5-f91c588cfeb8  partitions=1
session-gateway               49e7d879-0008-4888-950c-1101290edcbd  partitions=1
browser-replay-processor      01d6c949-14f3-4dc2-a20c-bf054314e84f  partitions=1
browser-bench                 de75e74f-b25e-4e30-95df-5b398dbc8044  partitions=1
kong-access-log               15a393b7-91bf-4720-a88c-65d4310fae64  partitions=1
lex-home                      57adc8e0-8eb3-4608-98c2-4fbe7e828c13  partitions=1
browser-manager               0be82bac-3f44-4d05-8264-c0fe555ffae5  partitions=1
dosk-chromium                 1f8c2f1b-6ca6-41ce-8b34-85bc4f136343  partitions=11
k8s-chrome-daemon             f16098fa-268b-4b2c-9d88-27b2ff0aa288  partitions=1
```

这些 topic 均开启了索引，可用于 `SearchLog` 查询。

## 查询日志内容

`SearchLog` 必须指定毫秒时间戳。

示例：查询最近 15 分钟 `session-gateway` 日志。

```bash
FROM=$(date -d '15 minutes ago' +%s%3N)
TO=$(date +%s%3N)

tccli cls SearchLog \
  --region ap-nanjing \
  --TopicId 49e7d879-0008-4888-950c-1101290edcbd \
  --From "$FROM" \
  --To "$TO" \
  --QueryString '*' \
  --QuerySyntax 1 \
  --Sort desc \
  --Limit 100 \
  --UseNewAnalysis true
```

按关键字查询：

```bash
tccli cls SearchLog \
  --region ap-nanjing \
  --TopicId 49e7d879-0008-4888-950c-1101290edcbd \
  --From "$FROM" \
  --To "$TO" \
  --QueryString 'error OR status:500' \
  --QuerySyntax 1 \
  --Sort desc \
  --Limit 100 \
  --UseNewAnalysis true
```

## 注意事项

1. `--region` 必须和 CLS topic 所在地域一致。
2. `SearchLog` 的 `From` / `To` 是毫秒时间戳，不是秒。
3. topic 必须开启索引，否则无法检索。
4. `TopicId` 和 `Topics` 二选一。
5. 单 topic 查询并发不要超过腾讯云限制。
6. 原始日志单次 `Limit` 最大 1000。
7. 大范围排障时，先用 topic、时间范围和关键字缩小查询，不要直接扫长时间窗口。

## 常见故障

### SecretId 不存在

现象：

```text
AuthFailure.SecretIdNotFound: SecretId不存在，请输入正确的密钥
```

处理：

1. 检查 `tccli configure list`。
2. 重新执行 `tccli configure`。
3. 确认 SecretId / SecretKey 来自有效 CAM 用户。

### 能列 topic 但查不到日志

优先检查：

1. topic 是否开启索引。
2. 查询时间范围是否正确，尤其是毫秒时间戳。
3. `--region` 是否和 topic 地域一致。
4. `QueryString` 是否过窄。

### 权限不足

优先确认 CAM policy 是否包含：

```text
cls:DescribeLogsets
cls:DescribeTopics
cls:SearchLog
```

如果需要查上下文或索引，再补：

```text
cls:DescribeLogContext
cls:DescribeIndex
```
