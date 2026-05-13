# Dotcom IP Verification and Webfetch Routing

## 背景

2026-05-09，`qcloud-hk` 需要基于真实客户端 IP 对 `browser-manager` 业务接口做地域拦截：

- Kong 在转发请求时向 upstream 注入 `X-Lexmount-Client-IP`
- `browser-manager` 读取该 header
- `qcloud-hk` 开启 IP 地域检查
- 中国大陆 IP 直接拒绝服务
- 查询 IP 地理位置使用本地库，避免运行时依赖外网

同一轮还补了 `webfetch` 在 HK Kong / internal Kong 的路由配置。

## 关键结论

1. Kong 侧应在 Nginx `location` 层注入 header，而不是只在 `server` 层注入。
2. 当前云环境不一定有 `X-Forwarded-For`，但 Kong access log 能拿到 `realip_remote_addr` / `$remote_addr`。
3. `geoip-lite` 对部分阿里云中国 IP 判断不可靠，例如 `8.152.100.174` 被判成 `SG`。
4. `ip-geo-service` 已改用 `ip2region` 本地库，`8.152.100.174` 返回 `country=CN`。
5. `browser-manager` 的 IP 检查中间件已覆盖所有业务接口，但跳过健康检查和监控接口。

## Kong 客户端 IP Header

### 原始问题

Kong access log 中存在：

```text
x_forwarded_for="-"
realip_remote_addr=221.218.159.215
```

说明在云环境中不能依赖 `X-Forwarded-For`，需要使用 Kong 当前已解析出的 remote address。

### 最终配置

在 `lexmount-k8s-manifests/apps/kong/deployment.yaml` 中使用：

```yaml
- name: KONG_NGINX_LOCATION_PROXY_SET_HEADER
  value: "X-Lexmount-Client-IP $remote_addr"
```

### 为什么不是 `KONG_NGINX_PROXY_PROXY_SET_HEADER`

最初尝试过 server 级别注入，但 Kong 生成的 proxy `location` 内已有自己的 `proxy_set_header`，Nginx 的 `proxy_set_header` 不会从 server 级别继续继承到已有 `proxy_set_header` 的 location。

所以必须使用 `KONG_NGINX_LOCATION_PROXY_SET_HEADER`，让 header 进入实际代理 location。

## browser-manager 中间件

### 配置项

`browser-manager` 中新增：

```env
CLIENT_IP_GEO_BLOCK_ENABLED=false
CLIENT_IP_HEADER_NAME=X-Lexmount-Client-IP
CLIENT_IP_GEO_SERVICE_URL=
CLIENT_IP_GEO_BLOCK_COUNTRIES=CN
CLIENT_IP_GEO_LOOKUP_TIMEOUT_MS=1000
```

`qcloud-hk` 开启：

```env
CLIENT_IP_GEO_BLOCK_ENABLED=true
CLIENT_IP_GEO_SERVICE_URL=http://ip-geo-service.system.svc.cluster.local:9236
```

`office` / `qcloud` 默认不开启拦截。

### 覆盖范围

中间件现在挂在 `browser-manager` 所有业务 route 前。

跳过以下路径：

- `/health`
- `/healthz`
- `/readyz`
- `/metrics`

原因是这些路径会被 kube probe / Prometheus 调用，通常不会携带 `X-Lexmount-Client-IP`，不应产生误拦截或大量 `missing_header` 日志。

### 日志

创建 instance 时打印：

```text
instance-create-client-ip|route=v2|phase=reserved|session_id=...|header=X-Lexmount-Client-IP|client_ip=...|has_client_ip_header=true
```

IP 地域检查打印：

- `result=missing_header`
- `result=allowed`
- `result=blocked`
- `result=lookup_error`

这能区分：

1. Kong 没传 header
2. header 已传但 IP 不在拦截国家
3. IP 命中拦截国家
4. geo service 查询失败

## ip-geo-service

### 服务接口

新增独立服务 `ip-geo-service`：

- `GET /healthz`
- `GET /lookup?ip=8.152.100.174`
- `POST /lookup`，body: `{ "ip": "8.152.100.174" }`

### 本地库选择

第一版使用 `geoip-lite`，但发现：

```text
8.152.100.174 -> country=SG
```

外部数据源和业务判断均显示该 IP 是北京阿里云：

```text
country=CN
city=Beijing
org=AS37963 Hangzhou Alibaba Advertising Co.,Ltd.
```

因此改用 `ip2region` 本地库。

当前结果：

```json
{
  "ip": "8.152.100.174",
  "valid": true,
  "country": "CN",
  "country_name": "中国",
  "isp": "阿里巴巴",
  "source": "ip2region"
}
```

### 运行时依赖

`ip2region` 数据随 npm 包进入镜像，运行时不依赖访问外网。

## Webfetch Kong 路由

### qcloud-hk 外部 Kong

`kong-init/config-qcloud-hk.js` 中：

- route: `webfetch-setaria`
- host: `webfetch.lexmount.com`
- service: `Setaria`
- paths: `['/v1']`
- `strip_path: false`

目标：外部域名只允许 `/v1` 前缀通过。

### qcloud-hk internal Kong

`kong-init/config-qcloud-hk-internal.js` 中新增：

- service: `SetariaInternal`
- route: `webfetch-setaria-internal`
- host: `webfetch.local.lexmount.net`
- 不限制 `paths`
- `strip_path: false`

证书：

```js
{
  cert: './cert/office/fullchain.pem',
  key: './cert/office/privkey.pem',
  target_tag: 'lexmount_net',
}
```

SNI：

```js
{
  target_tag: 'lexmount_net',
  name: 'webfetch.local.lexmount.net'
}
```

## PR 和镜像

### demo-nodejs-backend

- PR #146：IP geo blocking 初版
- PR #147：创建 instance 打印 session_id 和 client IP
- PR #148：`ip-geo-service` 切换到 `ip2region`
- PR #149：`browser-manager` 所有业务接口挂 IP 中间件
- PR #150：HK webfetch Kong 配置

### lexmount-k8s-manifests

- PR #415 - #421：对应镜像 tag / Kong 配置发布链路

### 最终关键镜像

- `browser-manager:e88b02b-20260509-212339`
- `ip-geo-service:446ce29-20260509-210024`
- `kong-init:d5e0b63-20260509-213818`

## HK 镜像规则

`qcloud-hk` manifests tag 已对齐，但不要由开发 agent 直接推 HK 镜像。

约定：

- office 镜像：`code.lexmount.net/wangli/...`
- qcloud 镜像：`lexmount.tencentcloudcr.com/cloud/...`
- qcloud-hk 镜像：`lexmoun-tcr-hk.tencentcloudcr.com/cloud/...`

HK tag 可以在 manifests 中对齐，但 HK 镜像推送应走正式同步流程。

## 排查手册

### 1. browser-manager 仍然显示 `missing_header`

先确认请求是否经过 Kong。

如果是内部服务直连 `browser-manager.system.svc.cluster.local:9222`，不会天然带 `X-Lexmount-Client-IP`。

再确认 Kong 生成配置中是否有：

```text
proxy_set_header X-Lexmount-Client-IP $remote_addr;
```

### 2. 能拿到 header 但没有拦截

看 `client-ip-geo-check` 日志中的：

- `client_ip`
- `country`
- `source`
- `result`

如果 `result=allowed` 且 `country` 不是 `CN`，说明是地理库判定问题或 IP 实际不属于拦截范围。

### 3. lookup 失败

`browser-manager` 当前对 lookup error 采用 fail-open，避免 geo service 异常影响整体可用性。

需要看：

- `CLIENT_IP_GEO_SERVICE_URL`
- `ip-geo-service` pod 是否存在
- `/lookup?ip=...` 返回结果

### 4. webfetch 路由不符合预期

检查：

- 外部 Kong：`config-qcloud-hk.js`
- internal Kong：`config-qcloud-hk-internal.js`
- `kong-init-image` 是否已经同步到目标环境
- SNI 是否绑定到 `lexmount_net`
