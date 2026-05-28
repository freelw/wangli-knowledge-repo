# Browser 内核专项本周进展 - 2026-05-28

## 数据来源约束

本版只使用已读取到的 Notion 页面作为事实来源：

- [Browser 内核专项信息](https://www.notion.so/36ce7e998fb2801d8131cfec03e1f7e6)
- [Browser 文档导读（先看我）](https://www.notion.so/eef9ae77e87d4d479dd1c488471a30e9)
- [1.1 WebFetch](https://www.notion.so/36de7e998fb2811eb952f15a3e093f94)
- [1.2 Moli 内核（lightmount）](https://www.notion.so/36de7e998fb281faa8ebca9a38f725d6)
- [1.3 协议 for Agent（Semantic Browser State）](https://www.notion.so/36de7e998fb2813ab654fec7017d5662)
- [1.4 内核日常升级](https://www.notion.so/36de7e998fb28113bca0e20592f4e599)
- [1.5 Chrome 交付 for 云平台](https://www.notion.so/36de7e998fb2811bbe94c797824612a3)
- [1.6 Agent 沙箱](https://www.notion.so/36de7e998fb281caaa0ae6e2c2234aa2)

未在上述 Notion 来源中确认的本地记录、Slock 线程、PR 和 knowledge-repo 内容，均不作为本周事实来源写入。

## TL;DR

1. `Browser 内核专项信息` 当前列出 6 条专项：WebFetch、Moli 内核、协议 for Agent、内核日常升级、Chrome 交付 for 云平台、Agent 沙箱。
2. 已读 Notion 来源显示：WebFetch 和 Chrome 交付处于 M2 收口中；Moli、协议、内核升级处于 M2 进行中；Agent 沙箱处于 M1 进行中。
3. 本次未从已读 Notion 来源中确认到内核专项的新增完成项；因此本周进展按专项当前状态保守记录，不引入非 Notion 事实。

## 专项 1：WebFetch

**负责人**：Allen / 内核团队  
**当前阶段**：M2 收口中（国内节点 + 外部客户体验对齐）  
**来源**：[Browser 内核专项信息](https://www.notion.so/36ce7e998fb2801d8131cfec03e1f7e6) / [1.1 WebFetch](https://www.notion.so/36de7e998fb2811eb952f15a3e093f94)

### 里程碑进度

- `🟡 M2 国内节点体验与外部客户体验对齐`：收口中。
  - `🟡 国内节点体验`：当前专项信息页标记为收口中；本周未在已读 Notion 来源中确认新增完成项。来源：[Browser 内核专项信息](https://www.notion.so/36ce7e998fb2801d8131cfec03e1f7e6)
  - `⚪ 失败归因 SLA`：待开始。来源：[1.1 WebFetch](https://www.notion.so/36de7e998fb2811eb952f15a3e093f94)
  - `⚪ 服务正规化`：待开始。来源：[1.1 WebFetch](https://www.notion.so/36de7e998fb2811eb952f15a3e093f94)

## 专项 2：Moli 内核（lightmount）

**负责人**：Corel  
**当前阶段**：M2 进行中（云平台轻量镜像替换）  
**来源**：[Browser 内核专项信息](https://www.notion.so/36ce7e998fb2801d8131cfec03e1f7e6) / [1.2 Moli 内核（lightmount）](https://www.notion.so/36de7e998fb281faa8ebca9a38f725d6)

### 里程碑进度

- `🟡 M2 云平台轻量镜像替换`：进行中。
  - `🟡 轻量镜像替换`：当前专项信息页标记为开发中；本周未在已读 Notion 来源中确认新增完成项。来源：[1.2 Moli 内核（lightmount）](https://www.notion.so/36de7e998fb281faa8ebca9a38f725d6)
  - `⚪ 成本优势量化`：待开始。来源：[1.2 Moli 内核（lightmount）](https://www.notion.so/36de7e998fb281faa8ebca9a38f725d6)
  - `⚪ 走向决策`：待开始。来源：[1.2 Moli 内核（lightmount）](https://www.notion.so/36de7e998fb281faa8ebca9a38f725d6)
  - `⚪ 轻量场景替代`：待开始。来源：[1.2 Moli 内核（lightmount）](https://www.notion.so/36de7e998fb281faa8ebca9a38f725d6)

## 专项 3：协议 for Agent（Semantic Browser State）

**负责人**：王枫（协议） + 嘉伟 / Corel（内核实现）  
**当前阶段**：M2 进行中（Strict Semantic JSON-only 验证）  
**来源**：[Browser 内核专项信息](https://www.notion.so/36ce7e998fb2801d8131cfec03e1f7e6) / [1.3 协议 for Agent（Semantic Browser State）](https://www.notion.so/36de7e998fb2813ab654fec7017d5662)

### 里程碑进度

- `🟡 M2 Strict Semantic JSON-only 验证`：进行中。
  - `🟡 Strict JSON 验证`：当前专项信息页标记为开发中；本周未在已读 Notion 来源中确认新增完成项。来源：[1.3 协议 for Agent（Semantic Browser State）](https://www.notion.so/36de7e998fb2813ab654fec7017d5662)
  - `⚪ 内核接口落地`：待开始。来源：[1.3 协议 for Agent（Semantic Browser State）](https://www.notion.so/36de7e998fb2813ab654fec7017d5662)
  - `⚪ 多源感知合并`：待开始。来源：[1.3 协议 for Agent（Semantic Browser State）](https://www.notion.so/36de7e998fb2813ab654fec7017d5662)
  - `⚪ 标准化外部 PoC`：待开始。来源：[1.3 协议 for Agent（Semantic Browser State）](https://www.notion.so/36de7e998fb2813ab654fec7017d5662)

## 专项 4：内核日常升级

**负责人**：嘉伟 / Corel  
**当前阶段**：M2 进行中（Chromium 149 升级）  
**来源**：[Browser 内核专项信息](https://www.notion.so/36ce7e998fb2801d8131cfec03e1f7e6) / [1.4 内核日常升级](https://www.notion.so/36de7e998fb28113bca0e20592f4e599)

### 里程碑进度

- `🟡 M2 Chromium 149 升级`：进行中。
  - `🟡 Chromium 149 升级`：当前专项信息页标记为开发中；本周未在已读 Notion 来源中确认新增完成项。来源：[1.4 内核日常升级](https://www.notion.so/36de7e998fb28113bca0e20592f4e599)
  - `⚪ 升核自动化回归`：待开始。来源：[1.4 内核日常升级](https://www.notion.so/36de7e998fb28113bca0e20592f4e599)

## 专项 5：Chrome 交付 for 云平台

**负责人**：李望（平台侧） + 嘉伟（内核侧）  
**当前阶段**：M2 收口中（custom_image_id 全链路）  
**来源**：[Browser 内核专项信息](https://www.notion.so/36ce7e998fb2801d8131cfec03e1f7e6) / [1.5 Chrome 交付 for 云平台](https://www.notion.so/36de7e998fb2811bbe94c797824612a3)

### 里程碑进度

- `🟡 M2 custom_image_id 全链路`：收口中。
  - `🟡 custom_image_id 全链路`：当前专项信息页标记为收口中；本周未在已读 Notion 来源中确认新增完成项。来源：[1.5 Chrome 交付 for 云平台](https://www.notion.so/36de7e998fb2811bbe94c797824612a3)
  - `⚪ 镜像灰度回滚`：待开始。来源：[1.5 Chrome 交付 for 云平台](https://www.notion.so/36de7e998fb2811bbe94c797824612a3)
  - `⚪ 内核版本可观测`：待开始。来源：[1.5 Chrome 交付 for 云平台](https://www.notion.so/36de7e998fb2811bbe94c797824612a3)
  - `⚪ 云端网页登录`：待开始。来源：[1.5 Chrome 交付 for 云平台](https://www.notion.so/36de7e998fb2811bbe94c797824612a3)

## 专项 6：Agent 沙箱

**负责人**：春池  
**当前阶段**：M1 进行中（行业洞察 + 方案选型）  
**来源**：[Browser 内核专项信息](https://www.notion.so/36ce7e998fb2801d8131cfec03e1f7e6) / [1.6 Agent 沙箱](https://www.notion.so/36de7e998fb281caaa0ae6e2c2234aa2)

### 里程碑进度

- `🟡 M1 行业洞察 + 方案选型`：进行中。
  - `🟡 行业洞察选型`：当前专项信息页标记为开发中；本周未在已读 Notion 来源中确认新增完成项。来源：[1.6 Agent 沙箱](https://www.notion.so/36de7e998fb281caaa0ae6e2c2234aa2)
  - `⚪ Sandbox Runtime MVP`：待开始。来源：[1.6 Agent 沙箱](https://www.notion.so/36de7e998fb281caaa0ae6e2c2234aa2)
  - `⚪ 主动防御 MDM`：待开始。来源：[1.6 Agent 沙箱](https://www.notion.so/36de7e998fb281caaa0ae6e2c2234aa2)
  - `⚪ 跨端适配`：待开始。来源：[1.6 Agent 沙箱](https://www.notion.so/36de7e998fb281caaa0ae6e2c2234aa2)
  - `⚪ 商业化接入`：待开始。来源：[1.6 Agent 沙箱](https://www.notion.so/36de7e998fb281caaa0ae6e2c2234aa2)

## 风险与下周动作

- `🔴 Notion 周粒度来源不足`：本次已读 Notion 来源主要是专项结构页，缺少每个内核专项本周 PR / 会议纪要 / 晨报级事实；下周需要按专项补充可引用的 Notion 进展页。
- `🟡 状态不应从非 Notion 推断`：后续周跟进如果要写“完成 / 延误 / 风险”，必须先在 Notion 找到对应来源，不能从 Slock 线程或本地归档直接写入。
