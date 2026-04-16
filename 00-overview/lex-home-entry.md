# lex-home 入口

## 目的

这篇文档用于给前端、联调和控制台相关工作一个最短入口，回答：

1. `lex-home` 在整套系统里是什么
2. 代码、部署、镜像、环境变量分别看哪里
3. 前端联调出问题时先排查哪几层

## `lex-home` 是什么

`lex-home` 当前同时承担：

1. 官网入口
2. 文档站入口
3. 控制台前端入口

它不是浏览器平台后端本身，但它是用户和控制台操作最常见的第一入口。

## 代码与部署入口

### 代码仓库

```text
/home/lexmount/project/backend/lex-home
```

### K8s 配置

```text
/home/lexmount/project/backend/lexmount-k8s-manifests/apps/lex-home
```

### 关键前端动作入口

当前最值得先看的文件：

1. `lex-home/src/actions/kong.ts`

它会直接使用这些环境变量：

1. `KONG_ADMIN_BASE_URL`
2. `BROWSER_BASE_URL`
3. `POCKETBASE_URL`

同时需要区分两类变量：

1. `KONG_ADMIN_BASE_URL`、`BROWSER_BASE_URL`、`POCKETBASE_URL` 这类服务端运行时依赖
2. `NEXT_PUBLIC_*` 这类前端可见配置

对前端来说，这两类变量不能混为一谈：

1. 服务端运行时依赖更偏联调与后端链路
2. `NEXT_PUBLIC_*` 更直接影响页面行为、开关和展示
3. 如果页面行为没变，也要考虑是不是前端镜像或前端配置没有真正更新

## 部署与镜像

### 命名空间与对象

当前 `lex-home` 在 K8s 里最关键的对象是：

1. namespace：`frontend`
2. Deployment：`lex-home`
3. Service：`lex-home`

### 镜像字段

各环境统一通过 `lex-home-image` 回填镜像。

常见位置：

1. `apps/clusters/office/images-configmap.yaml`
2. `apps/clusters/qcloud/images-configmap.yaml`
3. `apps/clusters/qcloud-hk/images-configmap.yaml`

### 镜像仓库差异

1. `office`：`code.lexmount.net/feixiang/lex-home`
2. `qcloud`：`lexmount.tencentcloudcr.com/cloud/lex-home`
3. `qcloud-hk`：`lexmoun-tcr-hk.tencentcloudcr.com/cloud/lex-home`

### 常用脚本

当前已有两个直接相关脚本：

1. `apps/tools/set_lex_home_image_tags.sh`
2. `apps/tools/sync_lex_home.sh`

## 前端最小联调路径

从前端视角理解最小链路，可以先按下面顺序：

1. 用户在 `lex-home` 页面触发操作
2. `lex-home` 的 server actions 调用内部依赖
3. 控制面请求进入 `browser-manager`
4. 如涉及 websocket / CDP，再进入 `browser-ws-gateway`
5. 如涉及实例创建，再进入 `k8s-chrome-daemon`

所以前端联调异常，不要只盯页面本身，要先区分是哪一层出问题：

1. `lex-home` 页面 / server actions
2. `browser-manager` HTTP 控制面
3. `browser-ws-gateway` websocket 链路
4. `lexmount-k8s-manifests` 环境配置

## 页面入口到后端动作的常见映射

下面这张表覆盖当前最常用的四条页面链路，目的是让新人第一次接活时不用先把整份 `src/actions/kong.ts` 倒推一遍。

| 页面路由 | 关键页面 / 组件 | 主要 server action / API route | 最终后端接口落点 | 说明 |
| --- | --- | --- | --- | --- |
| `/settings/api-keys` | `src/components/settings/api-keys/api-keys-card.tsx` | `upsertConsumer` / `listApiKeys` / `createApiKey` / `deleteApiKey` | Kong Admin API：`/consumers/{id}`、`/consumers/{id}/key-auth` | 这是前端拿到可用 API key 的起点；后面的 sessions / contexts / extensions 都依赖这里先有 key |
| `/settings/sessions` | `src/components/settings/sessions/sessions-page.tsx`、`sessions-card.tsx` | `getSessions` / `createSession` / `stopSession` / `deleteSession` / `getSessionTimeout` / `setSessionTimeout` / `listContexts` / `listExtensions` / `createContext` | `BROWSER_BASE_URL` 下的 `POST /instance/v2/sessions`、`POST /instance`、`POST /instance/stop`、`DELETE /instance`，以及 contexts / extensions 相关接口 | 这是控制台联调最复杂的一页，既会查 sessions，也会在创建 session 前联动 contexts 和 extensions |
| `/settings/contexts` | `src/components/settings/contexts/contexts-page.tsx`、`contexts-card.tsx` | `upsertConsumer` / `listApiKeys` / `listContexts` / `createContext` / `deleteContext` / `forceReleaseContext` | `BROWSER_BASE_URL` 下的 contexts 相关接口：`/instance/v1/contexts/create-context`、`/list-contexts`、`/{contextId}`、`/{contextId}/force-release` | 页面先拿默认 API key，再对 context 做增删和强制释放 |
| `/settings/extensions` | `src/components/settings/extensions/extensions-page.tsx`、`extensions-card.tsx` | `upsertConsumer` / `listApiKeys` / `listExtensions` / `getExtension` / `deleteExtension` / `/api/extensions/upload` | `BROWSER_BASE_URL` 下的 extension 相关接口：`/instance/v1/extension/list`、`/info`、`/{extensionId}`、`/upload` | 上传不是组件直接打 `browser-manager`，而是先走 Next.js route `/api/extensions/upload`，再由 route 调 `uploadExtensionOctetStream` |

补充理解：

1. 这些页面共同的起点是先通过 `upsertConsumer` + `listApiKeys` 拿到 consumer 与 API key
2. 真正的平台控制面调用主要集中在 `lex-home/src/actions/kong.ts`
3. 如果只是页面状态异常，先看对应 page / card 组件；如果是联调异常，再往 `src/actions/kong.ts`、`BROWSER_BASE_URL` 和 `browser-manager` 继续排

## 当前最常见的后端调用路径

如果从后端联调视角只记最常用的几条，可以先记这些：

1. `POST ${BROWSER_BASE_URL}/instance/v2/sessions`
2. `POST ${BROWSER_BASE_URL}/instance`
3. `POST ${BROWSER_BASE_URL}/instance/stop`
4. `DELETE ${BROWSER_BASE_URL}/instance`
5. Kong Admin 的 `consumers` / `key-auth`
6. PocketBase 相关调用

这里需要特别注意：

1. `BROWSER_BASE_URL` 当前指向的是 `browser-manager` 控制面 Service
2. 虽然端口是 `9222`，但这里不是前端直连浏览器实例或直接连 Chrome DevTools
3. 真正 websocket / DevTools 链路仍由 `browser-ws-gateway` 承载

## 前端第一次排查建议阅读顺序

如果是第一次接 `lex-home` 相关问题，建议先按下面顺序：

1. 页面组件，例如：
   - `src/components/settings/api-keys/api-keys-card.tsx`
   - `src/components/settings/sessions/sessions-page.tsx`
   - `src/components/settings/contexts/contexts-page.tsx`
   - `src/components/settings/extensions/extensions-page.tsx`
2. `lex-home/src/actions/kong.ts`
3. `lexmount-k8s-manifests/apps/lex-home/lex-home-configmap.yaml`
4. `browser-manager` 对应控制面接口

## office 环境最小验证

如果当前只做 `office` 联调，建议最少确认：

1. `frontend` namespace 下 `lex-home` Deployment 正常
2. `lex-home-image` 已切到目标镜像
3. `lex-home` 页面能打开
4. 页面触发的关键联调动作能命中后端
5. `/settings/api-keys` 可创建 / 列出 API key
6. `/settings/sessions` 会话列表可加载，创建 session 后能看到新记录
7. `/settings/contexts` 列表和创建动作可用
8. `/settings/extensions` 列表可加载，上传走 `/api/extensions/upload`

## 当前结论

如果只记住一句话，可以记住：

1. `lex-home` 是前端入口层
2. 代码看 `lex-home`
3. 部署看 `apps/lex-home`
4. 环境看 `images-configmap.yaml` 和 `lex-home-configmap.yaml`
5. 联调异常要沿“页面 -> server actions -> browser-manager -> ws / instance”往下拆
