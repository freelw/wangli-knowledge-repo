# 2026-04-14 browser-manager 架构拆分讨论

## 背景

当时 `browser-manager` 发布会直接打断 websocket 连接，主要原因是 HTTP 控制面和 websocket relay 承载在同一个发布单元里。

## 任务目标

1. 分析 websocket 断连的结构性根因
2. 规划控制面和 relay 的拆分边界
3. 明确长期应演进到什么目标结构

## 涉及仓库

1. `demo-nodejs-backend/browser-manager`
2. `demo-nodejs-backend/browser-manager-reconciler`
3. `lexmount-k8s-manifests`
4. `lexmount-js-sdk`
5. `lexmount-python-sdk`

## 关键改动

这次任务的核心产出不是代码，而是架构判断：

1. 识别出问题根因是“发布单元划分错误”，不是简单副本不足
2. 提出前台拆成控制面 HTTP 和 websocket relay 的方向
3. 明确 `/connection` 是否保留控制面逻辑，是拆分过程中的关键边界

## 验证方式

这次讨论本身不属于代码落地验证任务，验证方式主要体现为：

1. 后续架构是否按该方向演进
2. 后续代码和 manifests 是否出现独立 `browser-ws-gateway`

从当前代码回看，这次讨论已经部分转化为实际架构落地。

## 产出的长期知识

这次任务已经沉淀进当前知识库：

1. `01-architecture/websocket-relay.md`
2. `01-architecture/browser-manager-split-plan.md`
3. `01-architecture/browser-manager-overview.md`

## 分支与 PR

这次材料更偏架构讨论文档，没有直接记录统一的代码分支与 PR。

## 结论

1. 这次讨论为 `browser-ws-gateway` 的独立化提供了方向依据
2. 当前系统已经不再停留在“还没拆”的阶段
3. 但 `/connection` 和共享模块边界纯化仍然是后续未完成项
