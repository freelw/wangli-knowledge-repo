# 2026-05-09 Dotcom IP Verification

## 任务

在 `qcloud-hk` 实现基于真实客户端 IP 的地域拦截，并补齐相关 Kong / webfetch 路由配置。

## 完成内容

- Kong 转发到 upstream 时注入 `X-Lexmount-Client-IP`
- `browser-manager` 新增 IP geo 检查中间件
- `qcloud-hk` 开启中国大陆 IP 拦截
- `ip-geo-service` 使用 `ip2region` 本地库提供 `/lookup`
- `browser-manager` 所有业务接口接入中间件，健康检查/metrics 跳过
- HK 外部 `webfetch.lexmount.com` 限制 `/v1` 前缀
- HK internal 新增 `webfetch.local.lexmount.net` route / cert / SNI

## 关键 PR

- `demo-nodejs-backend` #146 - #150
- `lexmount-k8s-manifests` #415 - #421

## 关键镜像

- `browser-manager:e88b02b-20260509-212339`
- `ip-geo-service:446ce29-20260509-210024`
- `kong-init:d5e0b63-20260509-213818`

## 正式文档

见：`03-infra/dotcom-ip-verification-and-webfetch-routing.md`
