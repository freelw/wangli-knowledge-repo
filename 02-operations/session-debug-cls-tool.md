# Session Debug CLS Tool Runbook

## 文档定位

这篇文档记录 `lexmount-k8s-manifests` 中 `apps/tools/session_debug_cls.py` 的使用方式和排障口径。

该工具用于按 `session_id` 从腾讯云 CLS 中快速导出浏览器云平台关键日志，适合排查：

- session 创建失败或创建慢。
- display / downloads / devtools / websocket 路径异常。
- browser-manager、browser-ws-gateway、dosk-chromium 三方日志需要对齐时间线的问题。

相关实现：

- 仓库：`lexmount-k8s-manifests`
- 路径：`apps/tools/session_debug_cls.py`
- PR：<https://github.com/lexmount/lexmount-k8s-manifests/pull/607>

## 前提

本机需要能访问腾讯云 CLS，并已配置 `tccli` 凭证。

验证：

```bash
tccli cls help
tccli configure list
```

如果还未配置：

```bash
tccli configure
```

不要把 `SecretId` / `SecretKey` 写入群聊、文档或仓库。

## 基本用法

在 `lexmount-k8s-manifests` 仓库根目录执行：

```bash
python3 apps/tools/session_debug_cls.py session_1780650411191_cf8bh10bz
```

默认 region 是 `nanjing`，等价于：

```bash
python3 apps/tools/session_debug_cls.py session_1780650411191_cf8bh10bz --region nanjing
```

指定其他 region：

```bash
python3 apps/tools/session_debug_cls.py session_1780650411191_cf8bh10bz --region beijing
python3 apps/tools/session_debug_cls.py session_1780650411191_cf8bh10bz --region hk
```

也可以直接传腾讯云 region 名：

```bash
python3 apps/tools/session_debug_cls.py session_1780650411191_cf8bh10bz --region ap-nanjing
python3 apps/tools/session_debug_cls.py session_1780650411191_cf8bh10bz --region ap-beijing
python3 apps/tools/session_debug_cls.py session_1780650411191_cf8bh10bz --region ap-hongkong
```

查看参数：

```bash
python3 apps/tools/session_debug_cls.py --help
```

## 查询范围

默认查询最近 1 小时。

工具会动态发现 `k8s-browser-worker` 日志集和目标 topic，不依赖写死的 topic id。

目标 topic：

- `browser-manager`
- `browser-ws-gateway`
- `dosk-chromium`

查询关键字：

- `browser-manager`：原始 `session_id`
- `browser-ws-gateway`：原始 `session_id`
- `dosk-chromium`：将 `session_id` 中的 `_` 替换成 `-`

示例：

```text
session_1780650411191_cf8bh10bz
```

对应 dosk-chromium 查询关键字：

```text
session-1780650411191-cf8bh10bz
```

## 输出文件

工具会在当前目录生成输出目录：

```text
session-debug-output/<region>_<session_id>_<timestamp>/
```

目录中包含三类文件：

```text
browser-manager.jsonl
browser-ws-gateway.jsonl
dosk-chromium.jsonl
```

每个文件是一行一条 CLS 日志记录，便于后续用 `jq`、`rg`、脚本或编辑器继续分析。

快速查看：

```bash
ls -lh session-debug-output/nanjing_session_1780650411191_cf8bh10bz_*/
```

查看命中数量：

```bash
wc -l session-debug-output/nanjing_session_1780650411191_cf8bh10bz_*/*.jsonl
```

按关键字过滤：

```bash
rg 'error|failed|404|500|close|timeout' session-debug-output/nanjing_session_1780650411191_cf8bh10bz_*/
```

## 推荐排障流程

1. 先确认用户提供的 `session_id` 和 region。
2. 运行工具导出最近 1 小时日志。
3. 先看 `browser-manager.jsonl`，确认 session 创建、激活、关闭、lease 释放是否完整。
4. 再看 `browser-ws-gateway.jsonl`，确认 websocket / connection / devtools 相关错误。
5. 最后看 `dosk-chromium.jsonl`，确认容器内浏览器侧日志和底层退出信息。
6. 如果三份文件均为空，优先确认时间窗口、region、CLS 权限和 topic 索引。

## 常见判断

### browser-manager 有日志，dosk-chromium 没日志

可能原因：

- BrowserInstance 尚未真正创建到 Pod。
- 查询时间窗口不包含 chromium 运行时间。
- dosk-chromium 日志中 session id 使用了短横线格式，但传入查询的转换不匹配。

处理：

1. 确认 `browser-manager` 中是否有 daemon start / activate 成功日志。
2. 用转换后的短横线 session id 手工补查 dosk-chromium。
3. 必要时扩大时间窗口或使用 tccli runbook 手工查。

### 三份日志都为空

优先检查：

1. region 是否正确。
2. session 是否超过默认最近 1 小时窗口。
3. 当前账号是否具备 `cls:DescribeLogsets`、`cls:DescribeTopics`、`cls:SearchLog` 权限。
4. 对应 topic 是否开启索引。

### browser-ws-gateway 有断联日志

结合以下信息判断：

- 是否是 Kong timeout。
- 是否是 browser-ws-gateway 进程侧错误。
- 是否是上游 BrowserInstance / pod 已关闭。
- 是否有对应 `dosk-chromium` 退出或 close 日志。

如果怀疑是 Kong timeout，可对照 DevTools WS 断联修复记录和 Kong `BrowserWsGateway` service timeout 配置。

## 和 tccli Runbook 的关系

已有文档 `02-operations/tencent-cloud-cls-tccli-runbook.md` 记录手工 tccli 查询方法。

使用建议：

- 日常按 session 排障：优先用 `session_debug_cls.py`。
- 需要跨更多 topic、扩大时间范围、查上下文、查 topic 权限：使用 tccli runbook。
- 工具输出异常或为空：回退到 tccli 手工查询，确认是工具问题还是 CLS 数据 / 权限问题。

## 注意事项

- 不要把腾讯云密钥写入命令历史之外的共享位置。
- 不要把包含用户敏感信息的完整日志直接贴到群聊，先裁剪关键行。
- 只查必要时间窗口，避免大范围扫 CLS。
- 输出目录可以作为临时排障产物，不建议长期提交到代码仓库。
