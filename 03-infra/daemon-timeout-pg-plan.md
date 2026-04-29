# Daemon Session Timeout 配置迁移到 PG 计划

## 背景

当前 daemon 侧后续不应再依赖 PocketBase。

session 最大超时时间这类配置需要从 PocketBase 迁移到 PostgreSQL，由统一管理面维护，再通知 daemon 更新本地 timeout controller 数据。

## 目标

- daemon 不再依赖 PocketBase
- session 最大超时时间存储在 PostgreSQL
- `lexhome` 修改 session 最大超时时间时直接写 PG
- 修改成功后通知 daemon 更新 timeout controller
- daemon 本地 timeout controller 以 PG 数据为准

## 第一阶段：PG 表设计

新增一张 session timeout 配置表。

建议字段：

- `project_id`
- `api_key_id` 或 `api_key_hash`
- `timeout_minutes`
- `region_id`
- `updated_at`
- `updated_by`
- `version`

约束建议：

- 第一版可以按 `project_id + region_id` 唯一
- 如果 timeout 仍然跟 API key 绑定，再扩展为 `project_id + api_key_id + region_id`
- `timeout_minutes` 必须有上限，不能由前端任意写

## 第二阶段：lexhome 写 PG

`lexhome` 修改 session 最大超时时间时：

1. 完成用户权限校验
2. 校验 project / region 权限
3. 写入或更新 PG 中的 timeout 配置表
4. 增加 `version`
5. 提交事务
6. 通知 daemon 配置变更

这里不再写 PocketBase。

## 第三阶段：通知 daemon

修改成功后需要通知 daemon 刷新 timeout controller。

可选方式：

1. `lexhome` / 管理面直接调用 daemon 或 region data-plane gateway 的刷新接口
2. 写 PG 后发布事件，由 daemon 订阅
3. daemon 定时 polling PG，按 `version / updated_at` 增量刷新

第一版建议用显式刷新接口：

```text
lexhome
  -> management API
  -> region data-plane gateway
  -> daemon timeout controller refresh
```

接口语义可以是：

```text
POST /internal/session-timeout/refresh
```

请求至少包含：

- `project_id`
- `region_id`
- `version`

## 第四阶段：daemon 读取 PG

Daemon 启动时：

1. 从 PG 加载本 region 的 timeout 配置
2. 初始化 timeout controller 本地缓存
3. 开始按本地缓存执行 timeout 控制

Daemon 收到刷新通知时：

1. 按 `project_id / region_id` 查询 PG 最新配置
2. 对比 `version`
3. 更新 timeout controller 本地缓存
4. 记录刷新结果和错误日志

## 故障处理

如果通知 daemon 失败：

- PG 已经是配置真值
- `lexhome` 应展示“已保存，daemon 刷新待重试”或内部记录 pending 状态
- 后台任务或 daemon polling 需要最终补齐

如果 daemon 重启：

- daemon 启动时从 PG 全量加载本 region timeout 配置
- 不依赖历史通知事件恢复状态

如果 PG 临时不可用：

- daemon 继续使用本地已有 timeout cache
- 新配置刷新失败并告警
- 不应退回 PocketBase

## 多 Region 边界

- timeout 配置可以按 `region_id` 生效
- 南京 `lexhome` 是统一管理入口
- 北京 daemon 只加载和执行北京 region 的 timeout 配置
- 南京修改北京 region timeout 时，通过北京 region data-plane gateway 通知北京 daemon

## 明确不做

第一版不做：

- 不再通过 PocketBase 存取 timeout 配置
- 不让 daemon 同时读 PocketBase 和 PG 两套真值
- 不做跨 region daemon 直接互调
- 不把 timeout controller 状态反向同步到南京

## 推荐落地顺序

1. 设计并创建 PG timeout 配置表
2. `lexhome` timeout 写入路径从 PocketBase 改为 PG
3. 增加 region data-plane gateway 到 daemon 的刷新接口
4. daemon 启动时从 PG 加载 timeout 配置
5. daemon 收到刷新通知后更新 timeout controller
6. 增加失败重试和启动全量恢复
7. 删除 PocketBase timeout 读写路径
