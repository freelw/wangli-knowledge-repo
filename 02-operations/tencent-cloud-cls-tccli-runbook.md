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

## 案例：按 session_id 查询 Beijing session 生命线

问题：

```text
session_1779868595819_9k1mtmy9k
```

目标：

1. 确认 session 是否创建成功。
2. 查是否有 downloads/list 404。
3. 查是否被 stopall/delete。
4. 查 lease 是否释放。

查询范围：

```text
region: ap-beijing
time: 最近 2 小时
query: session_1779868595819_9k1mtmy9k
```

先查 `browser-manager`：

```bash
SESSION_ID='session_1779868595819_9k1mtmy9k'
FROM=$(date -d '2 hours ago' +%s%3N)
TO=$(date +%s%3N)

tccli cls SearchLog \
  --region ap-beijing \
  --TopicId 57bdb34c-f1ad-4bc5-b362-250c7e5f4d48 \
  --From "$FROM" \
  --To "$TO" \
  --QueryString "$SESSION_ID" \
  --QuerySyntax 1 \
  --Sort asc \
  --Limit 100 \
  --UseNewAnalysis true
```

如果需要排查完整链路，再按相关 topic 查询：

```bash
SESSION_ID='session_1779868595819_9k1mtmy9k'
FROM=$(date -d '2 hours ago' +%s%3N)
TO=$(date +%s%3N)

for item in \
  '57bdb34c-f1ad-4bc5-b362-250c7e5f4d48:browser-manager' \
  'aa34388a-2038-430b-9273-b13e56a3e4c0:region-data-plane-gateway' \
  'b191479a-6bf3-456c-9636-33600f71f54e:k8s-chrome-daemon' \
  '454913da-70db-4072-b5b3-7ed8bb378cbd:kong-access-log' \
  '0160b4bb-2a34-41d1-a65d-f424ed1da010:browser-ws-gateway'
do
  topic_id=${item%%:*}
  topic_name=${item#*:}
  echo "==== ${topic_name}"

  tccli cls SearchLog \
    --region ap-beijing \
    --TopicId "$topic_id" \
    --From "$FROM" \
    --To "$TO" \
    --QueryString "$SESSION_ID" \
    --QuerySyntax 1 \
    --Sort asc \
    --Limit 100 \
    --UseNewAnalysis true \
    | jq -r '.Results[]?.LogJson'
done
```

这次实际命中的关键日志：

```text
15:56:35 browser-manager session-quota acquire
15:56:35 browser-manager session-create reserved
15:56:35 browser-manager finalize-start
15:56:37 browser-operator creating-session-reconcile-runner pending
15:56:38 browser-manager daemon-start success，elapsed_ms=2760
15:56:38 browser-manager assert-instance success
15:56:38 browser-manager activate success
15:56:38 browser-manager done success
15:56:38 browser-manager instance-create-client-ip，route=v1，has_client_ip_header=false
15:56:41 region-data-plane-gateway downloads/list 返回 404
15:56:47 browser-manager deleteAllActiveSessions-killBrowserInstance
15:56:47 browser-manager docker-deleteContainer
15:56:47 browser-operator server-ModifyHandler
15:56:47-15:56:48 browser-operator Recorded pod exit code，exit_code=0
15:56:48 browser-manager db-closeSession，reason=deleteAllActiveSessions
15:56:48 browser-manager session-quota release，released=true，reason=deleteAllActiveSessions
```

结论：

1. session 创建成功，daemon 启动耗时约 2760ms。
2. `downloads/list` 的 404 发生在 session 生命周期早期，且早于 `deleteAllActiveSessions`。
3. 这次没有看到创建失败，也没有看到 pod 非 0 退出。
4. 后续 stopall/delete 路径正常执行，lease 已释放。
5. 如果业务期望是“没有下载记录时返回空列表”，则该 404 应继续按 downloads API 语义问题排查。

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
