# 2026-05-19 custom_image_id 与 browser-spider-bench 总结

## 背景

本轮工作围绕 office 环境更换 light 镜像展开。最初需求是把 office-nanjing、office-beijing 的 `chrome-light-image` 切到新的 lightmount 镜像，并发布 office。

后续需求扩展为：

- SDK、browser-manager、k8s-chrome-daemon 支持 `custom_image_id`。
- `/instance` 老接口和 `/instance/v2` 都要支持。
- quickstart 增加命令行设置 `custom_image_id` 的示例。
- browser-spider-bench 前端支持填写 `custom_image_id`，用于 light 对照测试。
- manifests 更新镜像、CRD 和 office/qcloud 相关配置。
- office 发布并处理 PR comments。

## 涉及仓库

- `demo-nodejs-backend`
- `k8s-chrome-daemon`
- `lexmount-js-sdk`
- `lexmount-python-sdk`
- `lexmount-js-sdk-quickstart`
- `lexmount-python-sdk-quickstart`
- `browser-spider-bench`
- `lexmount-k8s-manifests`

## 核心实现

### SDK

JS SDK 增加 `customImageId` 参数，Python SDK 增加 `custom_image_id` 参数。

该字段会进入 session create payload：

- JS SDK: `customImageId` -> API body `custom_image_id`
- Python SDK: `custom_image_id` -> API body `custom_image_id`

字段同时覆盖：

- `/instance/v2` async create
- `/instance` legacy create

### browser-manager

`browser-manager` 在 `/instance` 和 `/instance/v2` 中读取 `custom_image_id`。

处理逻辑：

- 非字符串返回 400。
- 空字符串按未设置处理。
- 非空时做授权检查。
- 通过检查后传给创建 session / instance 的链路。
- 最终传给 k8s-chrome-daemon。

安全策略从项目 allowlist 调整为镜像前缀 allowlist：

- 配置项：`CUSTOM_IMAGE_ID_ALLOWED_PREFIXES`
- 默认值：空字符串，表示默认拒绝所有 custom image。
- office-nanjing 显式配置：`code.lexmount.net/neng`
- 其他环境默认空，不允许 custom image。

最后一次 comments 处理后，代码默认值也改成了空字符串，避免不配置时意外放开。

### k8s-chrome-daemon

k8s-chrome-daemon 的 HTTP create request 支持 `custom_image_id`。

实现方式：

- HTTP 请求字段是 `custom_image_id`。
- 内部作为 image name 传入。
- 写入现有 `BrowserInstance.spec.image`。

因此 CRD 不需要新增字段；manifests 中的 CRD 与 daemon 仓库保持一致即可。

### quickstart

JS quickstart 和 Python quickstart 都增加了 custom image 示例。

能力：

- 通过命令行传 `custom_image_id`。
- 使用新版 SDK 创建 session。
- quickstart 依赖升级到支持该字段的版本。

版本：

- JS SDK 发布为 `lexmount@0.5.7`。
- Python SDK 对应 quickstart 依赖提升到新版。

### browser-spider-bench

browser-spider-bench 前端新增 `Custom image ID` 输入框。

传递链路：

`UI` -> `/pipeline/run` -> `pipelineService` -> `pipelineExecutor` -> `spiderRunner` -> `lexmountProvider`

行为规则：

- normal 服务保持不变，不使用 custom image。
- `lexmount_light` 未填写 `customImageId` 时，仍传 `browserMode: 'light'`。
- `lexmount_light` 填写 `customImageId` 时，只传 `customImageId`，不传 `browserMode`。

处理 comments 后增加了输入清洗：

- 去除控制字符，避免日志注入。
- 最大长度限制为 256 字符。

## manifests 改动

### browser-manager

base config 增加：

- `CUSTOM_IMAGE_ID_ALLOWED_PREFIXES`

deployment 中把该配置注入 browser-manager env。

环境配置：

- office-nanjing: `CUSTOM_IMAGE_ID_ALLOWED_PREFIXES=code.lexmount.net/neng`
- office-beijing: 空
- qcloud-nanjing: 空
- qcloud-hk: 空
- qcloud-beijing: 空
- guoge: 空

### browser request 资源配置

office 环境默认 browser request 调整为和 qcloud-nanjing 一致：

- `browser-cpu-request: 500m`
- `browser-memory-request: 1G`

limit 保持：

- `browser-cpu-limit: 4`
- `browser-memory-limit: 8G`

影响环境：

- office-nanjing
- office-beijing

### browser-spider-bench office region

要求 office 环境的 bench 一定打到 office-nanjing。

配置结果：

- office-nanjing: `LEXMOUNT_SESSION_REGION=office-nanjing`
- office-beijing: `LEXMOUNT_SESSION_REGION=office-nanjing`

这样即使 bench 服务部署在 office-beijing，也会指定创建到 office-nanjing。

## 镜像

最终相关镜像：

- `browser-manager:5c560af-20260518-234127`
- `browser-operator:ce78622-20260518-205400`
- `browser-spider-bench:b8eae00-20260518-234127`

browser-manager 推送范围：

- `code.lexmount.net/wangli/browser-manager:5c560af-20260518-234127`
- `lexmount.tencentcloudcr.com/cloud/browser-manager:5c560af-20260518-234127`
- `lexmoun-tcr-hk.tencentcloudcr.com/cloud/browser-manager:5c560af-20260518-234127`
- `lexmount-bj.tencentcloudcr.com/cloud/browser-manager:5c560af-20260518-234127`

browser-spider-bench 推送范围：

- `code.lexmount.net/wangli/browser-spider-bench:b8eae00-20260518-234127`
- `lexmount.tencentcloudcr.com/cloud/browser-spider-bench:b8eae00-20260518-234127`

注意：曾经在用户要求“不要推香港”前启动过一次 HK bench image push，命令已经完成。但 manifests 没有更新 qcloud-hk 的 browser-spider-bench tag，也没有发布 HK bench。

## 发布状态

office-nanjing 和 office-beijing 已发布。

执行内容：

- `kubectl apply -k apps/clusters/office-nanjing`
- `kubectl apply -k apps/clusters/office-beijing`
- rollout browser-manager
- rollout browser-spider-bench

最终确认：

- office-nanjing browser-manager 镜像为 `5c560af-20260518-234127`
- office-beijing browser-manager 镜像为 `5c560af-20260518-234127`
- office-nanjing browser-spider-bench 镜像为 `b8eae00-20260518-234127`
- office-beijing browser-spider-bench 镜像为 `b8eae00-20260518-234127`
- office 两边 `BROWSER_CPU_REQUEST=500m`
- office 两边 `BROWSER_MEMORY_REQUEST=1G`
- office 两边 browser-spider-bench `LEXMOUNT_SESSION_REGION=office-nanjing`

qcloud 状态：

- manifests 已包含 qcloud-nanjing 的 browser-manager 和 browser-spider-bench tag。
- manifests 已包含 qcloud-hk / qcloud-beijing 的 browser-manager tag。
- qcloud 环境本轮没有发布。
- qcloud-hk 的 browser-spider-bench tag 没有更新。

## 验证

代码验证：

- browser-manager: `node --check browser-manager/config.js`
- browser-spider-bench:
  - `node --check app/server/routes.js`
  - `node --check app/spider/runner.js`
  - `node --check app/providers/lexmount-provider.js`

镜像验证：

- `make push` browser-manager
- `make push` browser-spider-bench

manifests 验证：

- `kubectl kustomize apps/clusters/office-nanjing`
- `kubectl kustomize apps/clusters/office-beijing`
- `kubectl kustomize apps/clusters/qcloud-nanjing`
- `kubectl kustomize apps/clusters/qcloud-hk`
- `kubectl kustomize apps/clusters/qcloud-beijing`
- `kubectl kustomize apps/clusters/guoge`

office 发布验证：

- office-nanjing / office-beijing apply 成功。
- browser-manager rollout 成功。
- browser-spider-bench rollout 成功。
- live deployment 镜像和 ConfigMap 值确认正确。

## PR

已合入：

- demo-nodejs-backend PR: https://github.com/lexmount/demo-nodejs-backend/pull/179
- k8s-chrome-daemon PR: https://github.com/lexmount/k8s-chrome-daemon/pull/72
- lexmount-js-sdk PR: https://github.com/lexmount/lexmount-js-sdk/pull/15
- lexmount-python-sdk PR: https://github.com/lexmount/lexmount-python-sdk/pull/98
- lexmount-js-sdk-quickstart PR: https://github.com/lexmount/lexmount-js-sdk-quickstart/pull/10
- lexmount-python-sdk-quickstart PR: https://github.com/lexmount/lexmount-python-sdk-quickstart/pull/23
- browser-spider-bench PR: https://github.com/lexmount/browser-spider-bench/pull/15
- lexmount-k8s-manifests PR: https://github.com/lexmount/lexmount-k8s-manifests/pull/486

## 发布注意事项

- qcloud 发布前确认 `CUSTOM_IMAGE_ID_ALLOWED_PREFIXES` 仍为空，避免 qcloud 意外允许 custom image。
- qcloud-nanjing 如果要发布 bench，需要使用 qcloud registry tag `b8eae00-20260518-234127`。
- qcloud-hk 本轮没有更新 bench manifests tag；不要误以为 HK bench 已切到新版本。
- browser-manager 已同步到 HK/BJ registry，但是否发布到 qcloud 仍需要按生产发布流程单独执行。
- custom image 当前只在 office-nanjing 开放 `code.lexmount.net/neng` 前缀。

## 结论

本轮完成了从 SDK 到 browser-manager、k8s-chrome-daemon、quickstart、browser-spider-bench、manifests 的 `custom_image_id` 全链路支持。

office 环境已经发布并验证。qcloud 相关 manifests 和镜像准备完成，但本轮未发布 qcloud；qcloud-hk 的 bench tag 保持旧值。
