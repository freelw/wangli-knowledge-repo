# 2026-06-01 API Key Hash Annotation 与 Session Timeout 加固

## 背景

5 月 27 日完成第一阶段 API Key 脱敏后，session timeout 链路继续暴露出一个边界问题：

- timeout 配置表 `session_timeout_configs` 中保存的是安全 hash。
- browser session 资源原来通过 `hashed_api_key` label 传递 key 标识。
- 历史 label 使用的是短 hash，不适合继续作为业务鉴权或 timeout 查询依据。
- 评审指出如果 daemon 直接信任客户端传入的 hash，可能让客户端伪造 hash 继承其他租户的 timeout，或省略 hash 跳过 timeout enforcement。

本轮目标是把 timeout 所需的 api key hash 传递收敛到安全、可控、不可伪造的内部链路，同时彻底移除 session pod 上的 `hashed_api_key` label。

## 核心原则

- 明文 `api_key` 不下传到 browser-operator / daemon。
- daemon 不接受外部客户端直接控制 timeout hash。
- browser-manager 在已有明文 api key 的边界内计算 SHA-256 hash。
- browser-manager 调 browser-operator create API 必须携带内部 token。
- hash 存到 Kubernetes annotation，而不是 label。
- session pod / BrowserInstance 不再写 `hashed_api_key` label。

## 设计决策

### 不按 review 建议把明文 key 传给 daemon

review 曾建议 daemon 读取 `X-API-Key` 后调用 `sessiontimeout.HashAPIKey(apiKey)`。

这个方向能解决“客户端控制 hash”的问题，但会重新扩大明文 api key 的传输范围，违背本轮脱敏目标。因此最终没有采用。

最终方案：

1. browser-manager 仍然是处理明文 api key 的边界。
2. browser-manager 计算 SHA-256 `api_key_hash`。
3. browser-manager 调 browser-operator create API 时带：
   - `X-API-Key-Hash`
   - `X-Internal-Token`
4. browser-operator / daemon 只信任带内部 token 的调用。
5. daemon 将 hash 写入 annotation `lexmount.net/api-key-hash`。

### fail-closed

browser-operator `/chromium` create 现在改为 fail-closed：

- `INTERNAL_BROWSER_OPERATOR_TOKEN` 未配置：返回 503。
- `X-Internal-Token` 缺失或不匹配：返回 401。
- 真实 session 创建请求存在 `project_id` 和 `session_id`，但缺少 `X-API-Key-Hash`：返回 400。

这样即使有人直接调用 daemon 并伪造 `X-API-Key-Hash`，也无法绕过内部信任边界。

### 使用 annotation 替代 label

Kubernetes label 更适合短值、筛选和分组，不适合放完整 SHA-256。

本轮改为：

- annotation：`lexmount.net/api-key-hash=<sha256>`
- label：不再写 `hashed_api_key`

timeout controller 从 annotation 读取完整 hash，再结合 `project_id` 查询 timeout 配置。

## 实现内容

### k8s-chrome-daemon

PR：

- https://github.com/lexmount/k8s-chrome-daemon/pull/75

主要改动：

- 移除 BrowserInstance / pod `hashed_api_key` label。
- 新增 annotation `lexmount.net/api-key-hash`。
- `/chromium` create 增加内部 token 校验。
- session create 缺少 hash 时 fail-closed。
- timeout 相关日志优化为简短、清晰、可定位。

最终镜像：

```text
browser-operator:18d9cf6-20260601-154201
```

### demo-nodejs-backend / browser-manager

PR：

- https://github.com/lexmount/demo-nodejs-backend/pull/204

主要改动：

- browser-manager 在内部计算 api key SHA-256 hash。
- 调 browser-operator create API 时发送 `X-API-Key-Hash`。
- 新增 `BROWSER_OPERATOR_INTERNAL_TOKEN` 配置。
- 调 daemon create API 时发送 `X-Internal-Token`。
- 不再把明文 api key 传给 `startChromeContainer`。

最终镜像：

```text
browser-manager:cbc1d5c-20260601-153431
```

### lexmount-k8s-manifests

PR：

- https://github.com/lexmount/lexmount-k8s-manifests/pull/558

主要改动：

- browser-manager 注入 `BROWSER_OPERATOR_INTERNAL_TOKEN`。
- browser-operator 注入 `INTERNAL_BROWSER_OPERATOR_TOKEN`。
- office-nanjing / office-beijing 更新 browser-manager 和 browser-operator 镜像。
- qcloud-nanjing / qcloud-beijing 同步镜像 tag。
- qcloud-hk 按要求不推送、不更新。

## 发布范围

已发布并验证：

- office-nanjing
- office-beijing

同步 manifests tag：

- qcloud-nanjing
- qcloud-beijing

未变更：

- qcloud-hk

## 验证结果

### office-nanjing

创建 session 成功：

```text
session_1780300040465_6cd0033id
```

验证 BrowserInstance：

- `metadata.annotations["lexmount.net/api-key-hash"]` 存在。
- `spec.labels.hashed_api_key` 为空。
- `spec.labels.session_id` 正常。

### office-beijing

创建 session 成功：

```text
session_1780300044182_fs6pua1r0
```

验证 BrowserInstance：

- `metadata.annotations["lexmount.net/api-key-hash"]` 存在。
- `spec.labels.hashed_api_key` 为空。
- `spec.labels.session_id` 正常。

### daemon 直连拒绝

对 office-nanjing browser-operator API 直连 `/chromium`，即使伪造 `X-API-Key-Hash`，不带 `X-Internal-Token` 也返回：

```text
HTTP/1.1 401 Unauthorized
```

这验证了 hash 不再是客户端可直接控制的信任输入。

## 运维注意事项

- `hashed_api_key` label 已退出历史舞台，后续监控、排障、timeout 逻辑都不应继续依赖它。
- 需要查询 session 对应 key hash 时，使用 `lexmount.net/api-key-hash` annotation。
- daemon create API 必须配置 `INTERNAL_BROWSER_OPERATOR_TOKEN`，否则会 fail-closed。
- browser-manager 必须配置相同的 `BROWSER_OPERATOR_INTERNAL_TOKEN`。
- 如果 session 创建返回 400 `X-API-Key-Hash is required`，优先检查 browser-manager 是否正确计算并传递 hash。
- 如果 session 创建返回 401，优先检查 browser-manager / browser-operator 内部 token 是否一致。

## 后续工作

1. 确认 qcloud-nanjing / qcloud-beijing 发布后 timeout controller 读取 annotation 正常。
2. 复扫所有依赖 `hashed_api_key` label 的脚本、监控、日志和排障命令。
3. 数据完全稳定后，可以把旧文档中“保留 `hashed_api_key` label”的描述更新为历史说明。
4. 持续抽查 session pod / BrowserInstance，确认没有明文 api key 和短 hash label 回流。
