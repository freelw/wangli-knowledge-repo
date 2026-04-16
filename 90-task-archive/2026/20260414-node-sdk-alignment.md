# 2026-04-14 Node SDK 对齐

## 背景

Python SDK 已增加新能力，Node SDK 与 Node quickstart 落后于 Python 侧，需要补齐能力并修复已知返回值问题。

## 任务目标

1. 对齐 `lexmount-js-sdk` 与 Python SDK 的能力
2. 对齐 `lexmount-js-sdk-quickstart` 与 Python quickstart 的示例集
3. 修复 Node SDK 中 `inspectUrl` 返回错误

## 涉及仓库

1. `lexmount-js-sdk`
2. `lexmount-js-sdk-quickstart`
3. `lexmount-python-sdk`
4. `lexmount-python-sdk-quickstart`

## 关键改动

### `lexmount-js-sdk`

1. 删除过时的 `inspectUrlDbg`
2. 修正 `sessions.create()` 返回值中的 `inspectUrl`
3. 新增 `get(sessionId)` 能力
4. 抽出 `mapSessionInfo()` 统一 session 映射
5. 版本从 `0.2.4` 升到 `0.2.5`

### `lexmount-js-sdk-quickstart`

1. 补齐 `extension-basic.ts`
2. 补齐 `proxy-demo.ts`
3. 补齐 `inspect-url-demo.ts`
4. 补齐 `session-downloads.ts`
5. 更新 README 与脚本
6. 依赖提升到 `^0.2.5`

## 验证方式

### `lexmount-js-sdk`

已执行：

1. `npm test`
2. `npm run typecheck`

结果：

1. 45 个测试通过
2. TypeScript 类型检查通过

### `lexmount-js-sdk-quickstart`

已执行：

1. `./node_modules/.bin/tsc --noEmit`

结果：

1. TypeScript 编译检查通过

## 产出的长期知识

当前知识库里尚未专门为这次任务补出 SDK 对齐专题文档，但后续可归入：

1. `04-sdk/js-sdk.md`
2. `04-sdk/quickstart-alignment.md`

## 分支与 PR

### 分支

1. `lexmount-js-sdk`: `wangli_dev_20260414_1`
2. `lexmount-js-sdk-quickstart`: `wangli_dev_20260414_1`

### PR

1. <https://github.com/lexmount/lexmount-js-sdk/pull/new/wangli_dev_20260414_1>
2. <https://github.com/lexmount/lexmount-js-sdk-quickstart/pull/new/wangli_dev_20260414_1>

## 结论

1. Node SDK 已修复 `inspectUrl` 返回错误
2. Node quickstart 已补齐主要示例能力
3. SDK 版本已提升到 `0.2.5`
4. 这次任务后，Node 侧与 Python 侧的差距明显缩小
