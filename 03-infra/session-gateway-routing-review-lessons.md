# Session Gateway 路由改动 Review 经验

日期：2026-05-11
背景：修复 lexhome 通过 `session-gateway` 访问 context / extension 接口失败时，review 暴露出多处 gateway 设计失误。本文记录后续修改 `session-gateway` / `region-data-plane-gateway` 必须遵守的规则。

## 核心结论

`session-gateway` 是跨 region 聚合与路由入口，不是 local-only gateway。新增 browser-manager 能力到 `session-gateway` 时，必须先判断该接口是：

- 单 region 定向接口
- 跨 region fan-out 聚合接口
- 创建类接口，需要默认选择 local/default region
- 内部控制面接口，需要 local 写入再通知其他 region

不要因为当前 office-nanjing 只有一个常用入口，就把新路由写成只访问 local region。

## 这次暴露的问题

### 1. 不应绕开现有 route 匹配模型

错误倾向：

```js
if (isSomePath(url.pathname)) {
  await specialProxy(req, res);
  return;
}
```

问题：

- `handleRequest()` 会逐渐堆特殊分支
- 鉴权、body 读取、handler 分发路径不统一
- 后续新增 path 时容易继续复制特殊逻辑

推荐做法：

- 把新路由放进 `routes`
- 让 `findRoute()` 同时支持固定 `path` 与 `pattern`
- route handler 只处理该路由的业务分发

### 2. 不要引入重复 proxy 实现

这次一开始新增了独立 proxy 函数，后来发现不用 raw body 后，它和现有 `callRegion()` 重复。

规则：

- 如果是 JSON 请求转发，优先复用 `callRegion()`
- 不要重复实现 timeout、fetch、JSON parse、错误返回
- 只有确实需要透明转发非 JSON / multipart / binary 时，才单独引入 raw proxy

### 3. 不要默认使用 raw body

错误倾向：把 passthrough route 默认做成 raw body。

这次用户明确要求不要 raw。后续默认规则：

- 当前 browser-manager context / extension API 都是 JSON，就使用 `bodySource: 'json'`
- 只有接口明确需要上传文件、multipart、binary 或必须保持原始 body 时，才使用 raw body
- 引入 raw body 前必须说明理由，并确认是否会影响鉴权、日志和测试

### 4. 跨 region gateway 不应只处理 local region

错误倾向：context / extension 路由直接 `pickLocalRegion()`。

问题：

- `session-gateway` 的职责是根据 region registry 路由 / 聚合各 region
- lexhome 通过 `session-gateway` 查询管理面数据时，不能默认只看本地 region
- qcloud / office 多 region 场景下会漏数据

推荐语义：

- body 中带 `region_id` / `region_ids`：按指定 region 路由
- 不带 region：需要根据接口类型判断
  - list 类接口：fan-out 到所有 region 并合并结果
  - get/info 类接口：按 region 顺序查，返回第一个成功结果
  - create 类接口：明确默认 region 策略，不要隐式 fan-out

### 5. 路由字段不要传给下游业务服务

`region_id` / `region_ids` 是 gateway 的路由控制字段，不一定是 browser-manager 的业务字段。

规则：

- 转发给 browser-manager 前，应清理 gateway routing 字段
- 复用已有 `withoutRoutingFields()`，不要新增重复函数
- 例外：历史 session API 当前会把 `region_id` 传给下游，不能在无专项评估时顺手改语义

## Review 前检查清单

修改 `session-gateway` / `region-data-plane-gateway` 前，逐项确认：

1. 新接口是 local-only、指定 region、还是 all-region fan-out？
2. 是否已有 `callRegion()` / `proxyJsonTo()` / `withoutRoutingFields()` 可复用？
3. 是否真的需要 raw body？如果只是 JSON API，不要用 raw。
4. `region_id` / `region_ids` 是否只是路由字段？是否应从 upstream body 中移除？
5. `findRoute()` 是否统一覆盖新增 path，而不是在 `handleRequest()` 增加特殊分支？
6. e2e 是否从 `session-gateway` 入口覆盖，而不是只测 browser-manager / SDK？
7. manifests 是否覆盖所有实际部署该组件的环境，不要漏 office-beijing 这类 data-plane-only 环境。

## 与本次修复的关系

本次最终收敛后的方向：

- browser-manager：保留已有 context / extension JSON API，不新增 `force-release`
- region-data-plane-gateway：把 context / extension route 纳入 route 表，JSON proxy 到 browser-manager
- session-gateway：把 context / extension route 纳入 route 表，按 region 路由 / fan-out 到 data-plane
- e2e：新增从 `session-gateway` 入口验证 session、context、extension 的用例

