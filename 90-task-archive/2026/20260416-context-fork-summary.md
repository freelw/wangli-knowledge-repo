# 2026-04-16 context fork 完整交付总结

## 背景

这轮工作的目标，是在 Lexmount 现有 `context` 体系上补出正式的 `fork context` 语义，并把这条能力从“方案讨论”推进到可真实使用、可测试、可发布、可对外接入的完整交付面。

最开始的缺口主要有 3 类：

1. 后端还没有正式的 `fork` 接口
2. JS / Python SDK 还没有对外暴露 `contexts.fork(...)`
3. quickstart、文档、环境验证和发布承载面都还没有跟上

## 任务目标

1. 在 `browser-manager` 落 `context fork` 后端核心链路
2. 在 JS / Python SDK 补 `contexts.fork(...)`
3. 把 manifests、文档、示例、quickstart 一起补齐
4. 在 `office` 环境完成真实验证
5. 把整个交付面收成可 review、可继续接活、可长期引用的状态

## 涉及仓库

这条需求最终涉及 6 个承载仓库：

1. `demo-nodejs-backend`
2. `lexmount-js-sdk`
3. `lexmount-python-sdk`
4. `lexmount-k8s-manifests`
5. `lexmount-js-sdk-quickstart`
6. `lexmount-python-sdk-quickstart`

## 关键改动

### 1. 后端：`demo-nodejs-backend`

在 `browser-manager` 内正式落了 `fork context` 后端能力：

1. 新增接口：
   - `POST /instance/v1/contexts/:context_id/fork`
2. `contexts-manager` 新增 `forkContext(...)`
3. `contexts-fs` 新增 profile 目录复制能力
4. `contexts-db` 补 `forked_from` 持久化字段
5. 明确错误码：
   - `CONTEXT_FORK_SOURCE_LOCKED`
   - `CONTEXT_FORK_FAILED`
6. rollback 语义落地：
   - copy 失败时清理 target 目录
   - 删除 target DB 记录

同时这轮 review 修正里，又进一步收了这些关键点：

1. `forked_from` 不再只在响应里临时返回，而是正式持久化到 DB
2. source context 在 fork copy 窗口内通过 `weak lock` 收住竞态
3. rollback cleanup 逻辑统一归到 manager 层
4. 绝对 symlink 改成 fail-fast，不再静默复制
5. fork 出来的 target 不再默认继承 source 的 `api_key`

### 2. JS SDK：`lexmount-js-sdk`

JS SDK 正式补了：

1. `client.contexts.fork(contextId)`
2. `CONTEXT_FORK_SOURCE_LOCKED -> ContextLockedError`
3. 对应 README 与 examples
4. 版本提升：
   - `0.2.5 -> 0.2.6`

review 收口后，JS SDK 还补了：

1. `fork()` 返回的 `ContextInfo` 现在会保留 `metadata`
2. `examples/context-fork.ts` 不再用 `process.exit(1)` 破坏 `finally`
3. 示例成功后会清理 forked context
4. README 运行命令里补了 `npm run context-fork`

### 3. Python SDK：`lexmount-python-sdk`

Python SDK 正式补了：

1. `client.contexts.fork(context_id)`
2. `CONTEXT_FORK_SOURCE_LOCKED -> ContextLockedError`
3. 对应 README 与 examples
4. 版本提升：
   - `0.4.7 -> 0.4.8`

review 收口后，Python SDK 又补了：

1. `examples/context_fork.py` 成功后会清理 forked context
2. `fork()` docstring 补齐 `Raises`
3. `_handle_error_response` 的错误口径更新为同时包含 `CONTEXT_FORK_SOURCE_LOCKED`
4. 去掉示例中冗余的 `load_dotenv`

### 4. manifests：`lexmount-k8s-manifests`

这条需求的环境承载最终落在：

1. `office`
2. `qcloud`
3. `qcloud-hk`

关键动作有两轮：

1. 第一轮为了 `office` 验证，先把 `browser-manager-image` 切到：
   - `ee7c32e-20260416-155257`
2. review 修正后，又重出了新镜像 tag：
   - `36ea8b9-20260416-162658`

最终 manifests 口径统一成：

1. `apps/clusters/office/images-configmap.yaml`
2. `apps/clusters/qcloud/images-configmap.yaml`
3. `apps/clusters/qcloud-hk/images-configmap.yaml`

都同步回填到同一个新 tag：`36ea8b9-20260416-162658`

## 5. quickstart：两个 quickstart 仓库

### 第一轮 quickstart 收口

这轮先把 quickstart 和已发布 SDK 版本对齐，并补上 `context fork` 入口：

1. JS quickstart
   - 依赖切到 `lexmount ^0.2.6`
   - 新增 `context-fork.ts`
   - 新增 `npm run context-fork`
   - README / README.zh.md 补 fork quickstart 说明
2. Python quickstart
   - 依赖切到 `lexmount>=0.4.8`
   - 新增 `context_fork.py`
   - README / README.zh.md 补 fork quickstart 说明

### 第二轮 quickstart 语义调整

随后又按新增要求，把两个 quickstart 示例统一改成更简洁的对外接入口径：

1. 输入一个已有 `context_id`
2. 输出一个 fork 后的新 `id`
3. 不再做删除动作

也就是说，quickstart 最终语义不是“自己创建 source 再 cleanup”，而是：

1. 传入 source context id
2. 打印 forked context id

## 验证方式

这条线最终不是只做了静态检查，而是按 5 层做了完整验证：

### 1. 后端本地真实链路

已验证：

1. 真 Postgres
2. 真目录复制
3. 真 HTTP 接口
4. `normal / locked / rollback` 三条路径都跑过

### 2. `office` 环境真实验证

已基于新 tag `36ea8b9-20260416-162658` 在 `office` 重跑验证：

1. `normal`：通过
2. `locked`：`409 + CONTEXT_FORK_SOURCE_LOCKED`
3. `rollback`：`500 + CONTEXT_FORK_FAILED`
4. target 目录不残留

### 3. JS SDK 接入验证

已对真实后端验证：

1. normal：通过
2. locked：抛 `ContextLockedError`
3. not found：抛 `ContextNotFoundError`

### 4. Python SDK 接入验证

已对真实后端验证：

1. normal：通过
2. locked：抛 `ContextLockedError`
3. not found：抛 `ContextNotFoundError`

### 5. quickstart 实跑

这条后来又补到了“已实跑”口径，而不只是基础校验：

1. JS quickstart 新示例已实跑
2. Python quickstart 新示例已实跑
3. 两边都能在 live `office` `browser-manager` 上跑通
4. 两边都能打印出 fork 后的新 id

另外，quickstart 输入/输出语义调整后的版本，也已再次实跑验证：

1. 传入已有 `context_id`
2. 输出新的 `forked context id`

## 产出的长期知识

这轮任务直接沉淀出的长期知识包括：

1. `knowledge-repo/00-overview/engineering-conventions.md`
   - 新增“PR 提出后必须主动看 review comments”的工程规范
   - 新增“不要推 HK 镜像，HK 由 @freelw 自己推送”的工程规范
   - 新增“所有镜像推送必须走 make push”的工程规范
2. 本归档文档：
   - `90-task-archive/2026/20260416-context-fork-summary.md`

## 分支与 PR

### 主交付 PR

1. `demo-nodejs-backend`
   - PR：<https://github.com/lexmount/demo-nodejs-backend/pull/115>
2. `lexmount-js-sdk`
   - PR：<https://github.com/lexmount/lexmount-js-sdk/pull/6>
3. `lexmount-python-sdk`
   - PR：<https://github.com/lexmount/lexmount-python-sdk/pull/90>
4. `lexmount-k8s-manifests`
   - PR：<https://github.com/lexmount/lexmount-k8s-manifests/pull/322>
5. `lexmount-js-sdk-quickstart`
   - PR：<https://github.com/lexmount/lexmount-js-sdk-quickstart/pull/3>
6. `lexmount-python-sdk-quickstart`
   - PR：<https://github.com/lexmount/lexmount-python-sdk-quickstart/pull/13>

### quickstart 输入/输出语义后续调整 PR

在主 quickstart 对齐之后，又新增两条 follow-up PR，把示例语义改成“输入 `context_id` -> 输出 forked id”：

1. `lexmount-js-sdk-quickstart`
   - 分支：`wangli_dev_20260416_context_fork_example_input`
   - PR：<https://github.com/lexmount/lexmount-js-sdk-quickstart/pull/4>
2. `lexmount-python-sdk-quickstart`
   - 分支：`wangli_dev_20260416_context_fork_example_input`
   - PR：<https://github.com/lexmount/lexmount-python-sdk-quickstart/pull/14>

## 结论

1. `context fork` 这条需求已经从语义设计推进成完整交付面，不再只是讨论稿
2. 后端、SDK、manifests、文档、示例、quickstart 都已经补齐
3. `office` 环境已基于 review 修正后的新 tag 完成真实复验
4. 当前这条可以按“完整链路已测试”来表述：
   - 后端测过
   - `office` 测过
   - JS / Python SDK 测过
   - JS / Python quickstart 也测过
5. 后续如果要引用这条需求的完整承载面，应同时看主交付 6 个 PR，以及 quickstart 输入/输出语义调整的 2 个 follow-up PR
