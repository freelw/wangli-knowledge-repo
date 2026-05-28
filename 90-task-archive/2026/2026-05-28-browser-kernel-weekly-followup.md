# Browser 内核专项本周进展 - 2026-05-28

## TL;DR

1. 本周内核专项没有大规模新增交付，主要由云平台侧围绕 BrowserInstance 调度、spot 兜底和 API 安全改造间接推动内核交付链路稳定性。
2. `Chrome 交付 for 云平台` 仍处于 `custom_image_id` 全链路收口后的验证与观测阶段，API key 脱敏、E2E 和 gateway 修复降低了云端镜像交付链路的风险。
3. 下周重点应继续补齐 lightmount / WebFetch / Semantic Browser State 的可见开发进展来源，避免专项信息页只有状态、缺少周粒度事实。

## 专项 1：WebFetch

**负责人**：Allen / 内核团队  
**当前阶段**：M2 收口中（国内节点 + 外部客户体验对齐）  
**结构来源**：[Browser 内核专项信息](https://www.notion.so/36ce7e998fb2801d8131cfec03e1f7e6) / [1.1 WebFetch](https://www.notion.so/36de7e998fb2811eb952f15a3e093f94)

### 里程碑进度

- `✅ M1 基础能力与内部验证`：已结束。
- `🟡 M2 国内节点体验与外部客户体验对齐`：进行中。
  - `🟡 国内节点体验`：本周无明确新增交付记录；专项仍处于收口中。来源：[Browser 内核专项信息](https://www.notion.so/36ce7e998fb2801d8131cfec03e1f7e6)
  - `⚪ 失败归因 SLA`：本周无新增。来源：[1.1 WebFetch](https://www.notion.so/36de7e998fb2811eb952f15a3e093f94)
  - `⚪ 服务正规化`：本周无新增。来源：[1.1 WebFetch](https://www.notion.so/36de7e998fb2811eb952f15a3e093f94)
- `⚪ M3 SLA / 成本 / 容灾正规化`：未开始。

## 专项 2：Moli 内核（lightmount）

**负责人**：Corel  
**当前阶段**：M2 进行中（云平台轻量镜像替换）  
**结构来源**：[Browser 内核专项信息](https://www.notion.so/36ce7e998fb2801d8131cfec03e1f7e6) / [1.2 Moli 内核（lightmount）](https://www.notion.so/36de7e998fb281faa8ebca9a38f725d6)

### 里程碑进度

- `✅ M1 基础可运行版本`：已结束。
- `🟡 M2 云平台轻量镜像替换`：进行中。
  - `🟡 轻量镜像替换`：本周有 custom image 场景问题排查，未指定 region 时 session create 卡住，指定 `office-beijing` 不会卡住；后续定位到 office-nanjing CRD/PVC 环境问题并修 manifests。来源：#custumid-debug 线程、[Browser 文档导读](https://www.notion.so/eef9ae77e87d4d479dd1c488471a30e9)
  - `⚪ 成本优势量化`：本周无新增。来源：[1.2 Moli 内核（lightmount）](https://www.notion.so/36de7e998fb281faa8ebca9a38f725d6)
  - `⚪ 走向决策`：本周无新增。来源：[1.2 Moli 内核（lightmount）](https://www.notion.so/36de7e998fb281faa8ebca9a38f725d6)
  - `⚪ 轻量场景替代`：本周无新增。来源：[1.2 Moli 内核（lightmount）](https://www.notion.so/36de7e998fb281faa8ebca9a38f725d6)
- `⚪ M3 成本 / 质量对比决策`：未开始。

## 专项 3：协议 for Agent（Semantic Browser State）

**负责人**：王枫（协议） + 嘉伟 / Corel（内核实现）  
**当前阶段**：M2 进行中（Strict Semantic JSON-only 验证）  
**结构来源**：[Browser 内核专项信息](https://www.notion.so/36ce7e998fb2801d8131cfec03e1f7e6) / [1.3 协议 for Agent（Semantic Browser State）](https://www.notion.so/36de7e998fb2813ab654fec7017d5662)

### 里程碑进度

- `✅ M1 协议草案与方向确认`：已结束。
- `🟡 M2 Strict Semantic JSON-only 验证`：进行中。
  - `🟡 Strict JSON 验证`：本周无明确新增交付记录；仍需每周至少 2 条可见开发进展来支撑 M2。来源：[1.3 协议 for Agent（Semantic Browser State）](https://www.notion.so/36de7e998fb2813ab654fec7017d5662)
  - `⚪ 内核接口落地`：本周无新增。来源：[1.3 协议 for Agent（Semantic Browser State）](https://www.notion.so/36de7e998fb2813ab654fec7017d5662)
  - `⚪ 多源感知合并`：本周无新增。来源：[1.3 协议 for Agent（Semantic Browser State）](https://www.notion.so/36de7e998fb2813ab654fec7017d5662)
  - `⚪ 标准化外部 PoC`：本周无新增。来源：[1.3 协议 for Agent（Semantic Browser State）](https://www.notion.so/36de7e998fb2813ab654fec7017d5662)
- `⚪ M3 外部 Agent PoC`：未开始。

## 专项 4：内核日常升级

**负责人**：嘉伟 / Corel  
**当前阶段**：M2 进行中（Chromium 149 升级）  
**结构来源**：[Browser 内核专项信息](https://www.notion.so/36ce7e998fb2801d8131cfec03e1f7e6) / [1.4 内核日常升级](https://www.notion.so/36de7e998fb28113bca0e20592f4e599)

### 里程碑进度

- `✅ M1 常规升级机制`：已结束。
- `🟡 M2 Chromium 149 升级`：进行中。
  - `🟡 Chromium 149 升级`：本周无明确新增交付记录。来源：[1.4 内核日常升级](https://www.notion.so/36de7e998fb28113bca0e20592f4e599)
  - `⚪ 升核自动化回归`：本周无新增。来源：[1.4 内核日常升级](https://www.notion.so/36de7e998fb28113bca0e20592f4e599)
- `⚪ M3 升级自动化与质量基线`：未开始。

## 专项 5：Chrome 交付 for 云平台

**负责人**：李望（平台侧） + 嘉伟（内核侧）  
**当前阶段**：M2 收口中（custom_image_id 全链路）  
**结构来源**：[Browser 内核专项信息](https://www.notion.so/36ce7e998fb2801d8131cfec03e1f7e6) / [1.5 Chrome 交付 for 云平台](https://www.notion.so/36de7e998fb2811bbe94c797824612a3)

### 里程碑进度

- `✅ M1 基础镜像交付链路`：已结束。
- `🟡 M2 custom_image_id 全链路`：收口中。
  - `🟡 custom_image_id 全链路`：本周继续验证 custom image 创建链路；发现未指定 region 时可能卡在 `sessions.create`，并排查到 office-nanjing CRD/PVC 环境问题，已推动 manifests 修复。来源：#custumid-debug 线程
  - `🟡 镜像灰度回滚`：本周 API key 脱敏和 E2E 更新覆盖 session 创建、下载、gateway API，降低镜像交付链路回归风险。来源：[2026-05-27 API Key 脱敏改造](./2026-05-27-api-key-redaction.md)
  - `⚪ 内核版本可观测`：本周无新增。来源：[1.5 Chrome 交付 for 云平台](https://www.notion.so/36de7e998fb2811bbe94c797824612a3)
  - `⚪ 云端网页登录`：本周无新增。来源：[1.5 Chrome 交付 for 云平台](https://www.notion.so/36de7e998fb2811bbe94c797824612a3)
- `⚪ M3 云平台镜像交付正规化`：未开始。

## 专项 6：Agent 沙箱

**负责人**：春池  
**当前阶段**：M1 进行中（行业洞察 + 方案选型）  
**结构来源**：[Browser 内核专项信息](https://www.notion.so/36ce7e998fb2801d8131cfec03e1f7e6) / [1.6 Agent 沙箱](https://www.notion.so/36de7e998fb281caaa0ae6e2c2234aa2)

### 里程碑进度

- `🟡 M1 行业洞察 + 方案选型`：进行中。
  - `🟡 行业洞察选型`：本周无明确新增交付记录。来源：[1.6 Agent 沙箱](https://www.notion.so/36de7e998fb281caaa0ae6e2c2234aa2)
  - `⚪ Sandbox Runtime MVP`：本周无新增。来源：[1.6 Agent 沙箱](https://www.notion.so/36de7e998fb281caaa0ae6e2c2234aa2)
  - `⚪ 主动防御 MDM`：本周无新增。来源：[1.6 Agent 沙箱](https://www.notion.so/36de7e998fb281caaa0ae6e2c2234aa2)
  - `⚪ 跨端适配`：本周无新增。来源：[1.6 Agent 沙箱](https://www.notion.so/36de7e998fb281caaa0ae6e2c2234aa2)
  - `⚪ 商业化接入`：本周无新增。来源：[1.6 Agent 沙箱](https://www.notion.so/36de7e998fb281caaa0ae6e2c2234aa2)
- `⚪ M2 Runtime MVP`：未开始。

## 风险与下周动作

- `🔴 进展来源不足`：内核专项多项仍只有专项信息页状态，缺少本周可引用的 PR / 晨报 / 会议纪要。下周需要按专项补周粒度来源。
- `🟡 custom_image_id 继续收口`：未指定 region 的创建链路已经暴露环境依赖问题，下周应继续用 E2E 固化 office-nanjing / office-beijing 双地域验证。
- `🟡 内核与云平台交付边界`：Chrome 交付链路依赖云平台 region、PVC、CRD、gateway 等环境正确性；后续周报应把内核镜像问题与平台环境问题分开标注。
