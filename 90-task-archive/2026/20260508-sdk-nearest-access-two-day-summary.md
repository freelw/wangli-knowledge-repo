# SDK 就近接入 2026-05-07 ~ 2026-05-08 工作总结

## 背景

本轮继续推进 SDK 就近接入和多 region 控制面落地。当前 office 环境用于模拟双 region：

- `office-nanjing`：模拟南京主 region，承载官网、主控数据、catalog、跨 region 同步服务。
- `office-beijing`：模拟北京从 region，承载独立 K8s 集群、北京 Kong、browser-manager、daemon、浏览器实例。

核心目标是：用户只维护一套 `project_id` / `api_key`，SDK 可以按 region 创建实例，官网后续可以管理多 region session。

## 2026-05-07：Region 数据面和控制面能力收口

### Kong Key 同步

已补齐 Kong key 跨 region 同步链路：

- 南京作为主 region，维护用户的 `project_id` / `api_key`。
- 北京 Kong 通过同步获得同一套 key 数据。
- `kong-key-sync` 以 `project_id` 为粒度做 reconciliation。
- lexhome 创建 / 删除 key 后，可以触发对应 project 的同步。
- 上线前 backfill 任务已补：支持分批同步所有历史 project，避免一次性处理过多 project 超时。

当前约束：

- `qcloud` / `qcloud-hk` 未配置 `KONG_KEY_SYNC_BASE_URL` 时，不触发同步逻辑。
- Kong credential id 可以不同，但 key 值必须一致。

### Region Data Plane Gateway

`region-data-plane-gateway` 用作跨 region 内部数据面代理，主要承担南京访问北京 region 内部能力的入口。

已补能力包括：

- 北京 Kong Admin 内部代理。
- Kong consumer / key-auth credential 查询、创建、删除。
- 内部 token 鉴权。
- region 数据面归属校验和审计能力。

### Session Quota

已实现全局 session quota 租约服务，用于保证多 region active session 总数不超过项目并发上限。

关键点：

- 南京维护全局 quota 真值。
- 各 region 创建 session 前先申请租约。
- session 创建失败或关闭后释放租约。
- 未配置 `SESSION_QUOTA_BASE_URL` / `SESSION_QUOTA_TOKEN` 的环境不会启用 quota 服务。
- `qcloud` / `qcloud-hk` 当前保持无感。

### Timeout 跨 Region 链路

已收口 daemon timeout 配置跨 region 方案：

- timeout 真值在南京。
- 北京 daemon 不直接读取北京 PG 的 timeout 真值。
- 北京 daemon 通过南京代理数据面读取 timeout 配置。
- 官网修改 timeout 后，通知目标 region daemon refresh cache。
- 对多个目标 region 采用 fan-out 通知，不再保留单 target region 限制。

### Public Endpoint Catalog 阶段启动

开始实现 SDK 自动选路依赖的 public endpoint catalog。

最终收口为 host-only 模型：

- catalog 返回 `region_id`、`display_name`、`default`、`host`、`probe_path`。
- 不再返回 `endpoint_ips`、`base_url`、`wss_prefix`。
- SDK 通过 `host` 推导：
  - API：`https://<host>`
  - WebSocket：`wss://<host>`
  - Probe：`https://<host>/v1/region/probe`

office 规则：

- `office-nanjing` 是主 region，响应 catalog。
- `office-beijing` 不响应 catalog，只作为 catalog 中的接入点。
- `office-nanjing` catalog 返回南京和北京两个接入点。

qcloud 规则：

- `qcloud` 和 `qcloud-hk` 当前仍是两个隔离主环境。
- 暂不改成 qcloud-nanjing / qcloud-beijing。

## 2026-05-08：SDK 选路、Quickstart 和 Office 验证

### Python SDK 0.5.1

Python SDK 已升级到 `0.5.1`，支持 host-only region selection。

主要能力：

- 支持 `Lexmount(region="...")` 手动选择 region。
- 未传 region 时使用 catalog 中唯一 default region。
- 支持 `region_info()` 查看当前 region 选择结果。
- 支持 `catalog_info()` 查看 catalog 原始信息。
- 服务端没有 catalog / probe 时，fallback 到原始 base URL 逻辑。
- 选中 region 后，普通 API 请求会重写到 region host。

### JS SDK 0.5.1

JS SDK 已升级到 `0.5.1`，并完成 review 修复。

关键修复：

- catalog host 增加 lexmount 域名 allowlist，避免 SSRF。
- region resolution 加 Promise mutex，避免并发请求跳过尚未完成的选路。
- 显式指定 region 但 catalog 不存在时输出 warning。
- `catalog_info()` 和 probe 不带业务鉴权 header。

### Office Catalog / Probe 验证

office 环境验证结果：

- `office-nanjing` catalog 返回：
  - `office-nanjing`：`apitest.local.lexmount.net`
  - `office-beijing`：`apitest-beijing.local.lexmount.net`
- `office-beijing` 不响应 catalog。
- 两个 region 的 probe 均可通过各自 host 访问。
- SDK 选择 `office-beijing` 后，请求会进入北京 Kong / browser-manager。

### Browser Operator 创建失败修复

在 office-beijing 验证中发现一个 daemon 创建链路问题。

现象：

- SDK 已正确选到 `office-beijing`。
- 北京 K8s 中 Pod 已进入 `Running`。
- daemon creating reconcile 访问 `http://<podIP>:9222/json/version` 时出现 `connection refused`。
- 旧逻辑把该错误误判成终态失败，session 被标成 `create_failed`。

根因：

- K8s Pod `Running` 只表示容器进程已启动。
- Chrome DevTools 9222 端口可能还没 ready。
- `/json/version` 在启动窗口期 `connection refused` 应该视为 transient，而不是终态失败。

修复：

- `k8s-chrome-daemon` 将 `dial tcp`、`connection refused`、`i/o timeout` 等 `/json/version` 探测错误归类为 `creating`。
- 下一轮 reconcile 继续重试，直到 Chrome 9222 ready 或整体创建超时。

发布镜像：

- office：`code.lexmount.net/wangli/browser-operator:65529cf-20260508-160959`
- qcloud tag 对齐：`lexmount.tencentcloudcr.com/cloud/browser-operator:65529cf-20260508-160959`
- qcloud-hk tag 对齐：`lexmoun-tcr-hk.tencentcloudcr.com/cloud/browser-operator:65529cf-20260508-160959`

注意：只推送了 office 使用的 `code.lexmount.net/wangli` 镜像，qcloud / qcloud-hk 镜像由人工按流程推送。

### Office 发布验证

已部署到：

- `office-nanjing`
- `office-beijing`

验证结果：

- `browser-operator-controller-manager` rollout 成功。
- `Lexmount(region="office-beijing")` 创建 session 成功。
- 返回 WebSocket：`wss://apitest-beijing.local.lexmount.net/...`
- 返回 Inspect URL 中 `api_host=apitest-beijing.local.lexmount.net`。
- 验证 session 已关闭清理。

### Python Quickstart 更新

`lexmount-python-sdk-quickstart` 已更新：

#### `catalog_info.py`

用于展示 SDK 0.5.1 的 catalog 信息。

#### `session_list.py`

改为只 list session，不创建、不删除。

当前逻辑：

- 读取 catalog。
- 遍历 catalog 中所有 region。
- 分别用 `Lexmount(region=region_id)` 查询各 region active sessions。
- 输出每个 session 的 region、status、WebSocket URL、Inspect URL。

这个改动解决了“北京 session 在 K8s 中存在，但默认 list 查不到”的问题。原因是默认 client 只查 default region（南京），北京 session 只在北京 browser-manager 中。

#### `demo.py`

新增可选命令行参数：

```bash
python demo.py
python demo.py --region office-beijing
python demo.py --region office-nanjing
```

行为：

- 不传 `--region`：调用 `Lexmount()`，使用 default region。
- 传 `--region`：调用 `Lexmount(region=args.region)`。

## 已合入仓库

以下 PR 均已合入，并且本地仓库已切回最新 `main`：

- `demo-nodejs-backend`
- `k8s-chrome-daemon`
- `lexmount-k8s-manifests`
- `lexmount-python-sdk`
- `lexmount-js-sdk`
- `lexmount-python-sdk-quickstart`

## 当前风险和后续事项

### 1. qcloud / qcloud-hk 镜像推送

manifests 已对齐 qcloud / qcloud-hk 的 `browser-operator` tag，但镜像本身需要人工按既定流程推送。

### 2. Lexhome 管理动作跨 Region

lexhome 侧 inspect / reconnect / close / 其他管理动作的跨 region 改造还未正式开始。

当前结论：

- SDK 直接连某个 region 的 Kong 创建 session 已经可行。
- 官网管理多 region session 仍需要后续在 lexhome / session gateway 层补齐。

### 3. Session 原生 Region 字段

目前 quickstart 可以从 region 请求上下文或 WebSocket / Inspect URL host 反推出 region。

长期建议：

- browser-manager session 数据中原生记录并返回 `region_id`。
- SDK `SessionInfo` 模型中增加 `region_id` 字段。
- quickstart 不长期依赖 host 反推 region。

### 4. 正式 qcloud 多 Region 拆分

当前只是 office 环境模拟南京 / 北京。正式 qcloud 拆成 qcloud-nanjing / qcloud-beijing 仍未开始。
