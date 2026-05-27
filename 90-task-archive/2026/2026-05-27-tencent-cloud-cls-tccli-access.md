# 2026-05-27 腾讯云 CLS tccli 查询能力接入总结

## 任务

确认 `tccli` 是否可以查询腾讯云日志服务 CLS，梳理所需 CAM 权限，并把南京、北京、香港三地的关键云平台日志集和按 `session_id` 排障方法记录为可复用知识。

## 背景

云平台故障分析高度依赖 CLS 中的服务日志。尤其是 `browser-manager`、`session-gateway`、`region-data-plane-gateway`、`browser-ws-gateway`、`k8s-chrome-daemon`、`kong-access-log` 等 topic，是定位 session 创建、跨 region 路由、下载接口、websocket 和实例生命周期问题的核心入口。

## 完成内容

新增正式 runbook：

```text
02-operations/tencent-cloud-cls-tccli-runbook.md
```

已覆盖：

1. `tccli` SecretId / SecretKey 配置方式。
2. CLS 查询需要的最小 CAM 权限。
3. CVM 实例查询所需权限和南京实例查询命令。
4. 南京 CLS logset 清单。
5. 南京 `k8s-browser-worker` 下 11 个 topic 的 TopicId。
6. 北京和香港 `k8s-browser-worker` topic 情况。
7. 按 `session_id` 跨 topic 查询 session 生命周期的实际案例。
8. 常见故障和排查注意事项。

## 权限结论

CLS 常用排障权限：

```text
cls:DescribeLogsets
cls:DescribeTopics
cls:SearchLog
cls:DescribeIndex
cls:DescribeLogContext
```

如果只知道 `TopicId` 并只需要查日志，最小权限是：

```text
cls:SearchLog
```

如果需要枚举日志集和日志主题，还需要：

```text
cls:DescribeLogsets
cls:DescribeTopics
```

CVM 实例列表查询需要：

```text
cvm:DescribeInstances
```

## 关键命令

查询日志集：

```bash
tccli cls DescribeLogsets --region ap-nanjing --Limit 100
```

查询某个 logset 下的 topic：

```bash
tccli cls DescribeTopics \
  --region ap-nanjing \
  --Filters '[{"Key":"logsetId","Values":["3f5223da-51a4-443f-8c15-9e5ce6a64eca"]}]' \
  --Limit 100
```

按 `session_id` 查日志：

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

## 三地日志现状

南京 `ap-nanjing`：

```text
k8s-browser-worker  logset_id=3f5223da-51a4-443f-8c15-9e5ce6a64eca  topics=11
```

北京 `ap-beijing`：

```text
k8s-browser-worker  logset_id=0d87c537-501e-4004-938c-f1b8ac74b275  topics=7
```

香港 `ap-hongkong`：

```text
k8s-browser-worker  logset_id=36c43b21-64fe-4d26-9fb7-3d78c136c210  topics=7
```

北京 `region-data-plane-gateway` topic 起初 `index=false`，后续已开启并确认 `index=true`。

## 实际排障案例

查询 session：

```text
session_1779868595819_9k1mtmy9k
```

关键结论：

1. session 创建成功，`daemon-start` 成功，耗时约 2760ms。
2. `region-data-plane-gateway` 在 15:56:41 对 `downloads/list` 返回 404。
3. 该 404 发生在 `deleteAllActiveSessions` 之前，因此不是 stopall 后访问导致。
4. 后续 `deleteAllActiveSessions`、容器删除、DB close 和 `session-quota release` 均正常。
5. 如果业务期望“没有下载记录时返回空列表”，该问题应继续按 downloads API 语义排查。

## 正式文档

见：

```text
02-operations/tencent-cloud-cls-tccli-runbook.md
```

## 提交记录

```text
a0aa650 docs: add tencent cls tccli runbook
492ad0c docs: add cls session query example
```
