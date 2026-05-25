# 2026-05-25 ACE_WINDOW_SIZE 注入

## 任务

SDK 增加窗口尺寸参数，使创建 browser instance 时可以把 `ACE_WINDOW_SIZE=<width,height>` 注入到最终浏览器 Pod 环境变量中。

要求：

- JS SDK 支持 `windowSize`。
- Python SDK 支持 `window_size`。
- `/instance` 老接口和 `/instance/v2` 新接口都支持。
- 未传参数时，Pod 不应出现 `ACE_WINDOW_SIZE`。
- quickstart 需要提供示例。
- office 和 qcloud 相关环境 browser-manager 镜像需要对齐。

## 完成内容

- `browser-manager` 在 `/instance` 和 `/instance/v2` 接收 `window_size`。
- 服务端校验格式为 `width,height`，且宽高必须为正整数。
- 创建容器时通过 k8s-chrome-daemon generic `env` 字段传入 `ACE_WINDOW_SIZE=<width,height>`。
- 修复 office 验证发现的透传断点：`browser-manager/chrome/docker.js` 之前没有继续把 `windowSize` 传给真正调用 daemon create 的 `utils/docker.startContainer()`，导致日志中 `has_window_size=true` 但 `BrowserInstance.spec.env` 为空。
- JS SDK 新增 `windowSize` 参数，并在 SDK 侧做格式校验。
- Python SDK 新增 `window_size` 参数。
- JS quickstart 增加 `window-size-demo.ts`。
- Python quickstart 增加 `window_size_demo.py`。
- manifests 更新 office-nanjing、office-beijing、qcloud-nanjing、qcloud-beijing、qcloud-hk 的 browser-manager 镜像 tag。

## 关键结论

k8s-chrome-daemon 不需要新增专门的 `windowSize` 字段。当前 daemon 已支持通用 env 透传：

- create request 支持 `env`。
- `BrowserInstance.spec.env` 会写入 CR。
- controller 创建 Pod 时会把 `spec.env` 追加到容器环境变量。

因此本次实现保持在 browser-manager 层把业务参数转换为通用 env。

## PR

- `demo-nodejs-backend` #199
- `lexmount-js-sdk` #16
- `lexmount-python-sdk` #101
- `lexmount-js-sdk-quickstart` #11
- `lexmount-python-sdk-quickstart` #27
- `lexmount-k8s-manifests` #546

## 镜像

- `code.lexmount.net/wangli/browser-manager:c3fa86c-20260525-190736`
- `lexmount.tencentcloudcr.com/cloud/browser-manager:c3fa86c-20260525-190736`
- `lexmount-bj.tencentcloudcr.com/cloud/browser-manager:c3fa86c-20260525-190736`
- `lexmoun-tcr-hk.tencentcloudcr.com/cloud/browser-manager:c3fa86c-20260525-190736`
- digest: `sha256:e11be416bbd7b849ac8b02fd3cfb5d9b68fb05c71d3c18b49e9076de1cb8926b`

注意：@freelw 后续强调不要主动推 HK 镜像；本次 HK tag 是在看到提醒前已同步完成，后续发布应遵守该规则。

## 发布和验证

- office-nanjing 已 apply，并 rollout `browser-manager` / `browser-ws-gateway`。
- office-beijing 已 apply，并 rollout `browser-manager` / `browser-ws-gateway`。
- 验证两个 office 环境的 browser-manager deployment 均指向 `c3fa86c-20260525-190736`。
- manifests 通过以下 kustomize：
  - `apps/clusters/office-nanjing`
  - `apps/clusters/office-beijing`
  - `apps/clusters/qcloud-nanjing`
  - `apps/clusters/qcloud-beijing`
  - `apps/clusters/qcloud-hk`
- SDK / quickstart 验证：
  - JS SDK `npm test -- tests/sessions.test.ts`
  - JS SDK `npm run typecheck`
  - Python SDK `pytest tests/test_sessions.py`
  - quickstart syntax checks

## 使用示例

Python quickstart：

```bash
python3 window_size_demo.py --window_size "2560,1440"
```

JS quickstart：

```bash
npm run window-size-demo -- --window_size 2560,1440
```

参数必须是单行 `宽,高`，不要带换行。
