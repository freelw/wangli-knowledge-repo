# 2026-04-22 setaria 数据采集

## 背景

这次任务是为 `setaria` deployment 增加 Prometheus 数据抓取与 Grafana 可视化，覆盖 CPU、内存和网络流量，并把对应镜像/tag 同步到多环境。

## 任务目标

- 在 `lexmount-k8s-manifests` 中新增 `setaria` scrape job
- 补 CPU / 内存 / 网络 recording rules
- 在 `grafana-dashboard-init` 中新增 `setaria` dashboard
- 完成 `office`、`qcloud`、`qcloud-hk` 三边镜像/tag 同步
- 处理 dashboard UID 过长导致的 Grafana 注入失败

## 涉及仓库

- `lexmount-k8s-manifests`
- `demo-nodejs-backend`

## 关键改动

- 新增 `setaria` Prometheus 抓取配置
- 新增 `setaria-resource-metrics` 规则组
- 新增 `setaria Deployment 资源指标` dashboard
- 修复 dashboard UID 过长问题
- 新镜像 tag 同步到 `office`、`qcloud`、`qcloud-hk`

## 验证方式

- `python3 -m py_compile demo-nodejs-backend/grafana-dashboard-init/import_dashboards.py`
- `kubectl kustomize apps/clusters/office`
- `kubectl kustomize apps/clusters/qcloud`
- `kubectl kustomize apps/clusters/qcloud-hk`

## 产出的长期知识

- [setaria 监控接入](../../03-infra/setaria-monitoring.md)

## 分支与 PR

- `lexmount-k8s-manifests`: <https://github.com/lexmount/lexmount-k8s-manifests/pull/334>
- `demo-nodejs-backend`: <https://github.com/lexmount/demo-nodejs-backend/pull/118>

## 结论

- `setaria` 已接入平台 Prometheus + Grafana 监控链路。
- 资源指标中 CPU / 内存按容器维度，网络按 Pod 维度，是当前实现口径。
- dashboard UID 长度限制已被明确纳入后续新增 dashboard 的检查项。
