# Browser 晨间早报生成示例

## 背景

2026-05-27 晚上，按 `Browser 晨报中心` 下的 `指令说明` 试运行生成了一份临时晨报：

```text
【临时】晨间简报——5/27
```

输出页面：

```text
https://www.notion.so/36de7e998fb2813288b9e41211d58069
```

本页记录这次试运行的实际流程、踩坑和后续可复用步骤。

## 目标位置

Browser 晨报输出到 Notion 数据库：

```text
🌻 Browser 晨报中心
https://www.notion.so/1f4e7e998fb282d683ec016b6b9bf759
```

数据库路径：

```text
Browser as a Service -> Browser 周报 -> 🌻 Browser 晨报中心
```

数据源：

```text
collection://da7e7e99-8fb2-8338-b4a4-87a4c6c926dd
```

主要字段：

- `名称`
- `TL;DR 摘要`

## Notion 访问方式

应使用 Codex 配置的 Notion MCP OAuth，不要使用项目 `.env` 里的 `NOTION_KEY`。

参考：

```text
02-operations/notion-codex-access.md
```

本次开始时遇到过 Notion MCP `Auth required` / `invalid_token`，重新完成 OAuth 后恢复。

可先验证：

```bash
codex mcp list
```

期望看到：

```text
notion  https://mcp.notion.com/mcp  enabled  OAuth
```

如果 token 失效，重新登录：

```bash
codex mcp login notion
```

注意：OAuth callback 是 `127.0.0.1`，需要在运行 Codex 的机器上完成授权，否则 token 不会写回本机。

## 指令说明位置

可通过 Notion 搜索：

```text
Browser 晨间早报 指令说明
```

实际命中的页面：

```text
指令说明
https://www.notion.so/7e1e7e998fb283f8850e011679b57ba4
```

核心规则：

- 先创建晨报页面骨架，再边查边写。
- 信息权重：会议纪要 > 文档变更 > 今日待办。
- 用 Notion search 代替逐页遍历。
- 不硬编码人员名单或项目列表。
- 责任人不明确时标注 `待确认`，不要编造。
- 如果上下文或调用次数接近限制，按优先级降级，保证可交付。

## 本次生成流程

### 1. 创建临时页面骨架

在 `Browser 晨报中心` 数据源下创建页面：

```text
名称: 【临时】晨间简报——5/27
TL;DR 摘要: 生成中
```

第一版只写骨架：

- TL;DR
- 昨日要闻与会议
- 重大进展与关键卡点
- 今日行动清单
- 文档更新概览

### 2. 搜索会议纪要

使用 Notion search，日期范围聚焦 2026-05-26：

```text
Browser 5月26日 会议 纪要
```

实际命中的关键会议：

```text
2026-05-26 | 王枫的视频会议 | Browser agent对接IAM实现OA自动登录de
https://www.notion.so/36ce7e998fb2810cbf86e2d5a85fe525
```

提取内容：

- Browser agent 对接 IAM / OA 自动登录路径。
- 复用袖璋 IAM/OAuth demo。
- Agent 平台完成 IAM 登录后换取 token，并注入浏览器。
- 待办中责任人多数不明确，保留 `待确认`。

### 3. 搜索文档变更

搜索关键词：

```text
Browser 2026-05-26 项目 文档 更新 云平台 内核
```

关键页面：

- `0526 Browser 项目梳理-prompt`
- `browser 云平台专项信息`
- `Browser 内核专项信息`
- `2.3 服务端开发`
- `1.2 Moli 内核（lightmount）`
- `Moli 匿名 Binary-only 发布：问题分析与 TODO`
- `1.5 Chrome 交付 for 云平台`

提炼出的主线：

- Browser 项目从散点信息收敛为专项结构。
- 云平台专项包括 dotcom、私有化、服务端开发、成本监控。
- 内核专项包括 WebFetch、Moli、协议 for Agent、内核升级、Chrome 交付、Agent 沙箱。
- 服务端开发处于 M2 收口 / M3 准备阶段。
- Moli binary-only 技术预览发布需要补齐信任体系。

### 4. 追加 5月27日当日增量

用户要求把 5 月 27 日工作也带入后，继续搜索当天内容：

```text
Browser 2026-05-27 云平台 工作 会议 文档
2026-05-27 CLS tccli 腾讯云 日志 Browser 云平台
2026-05-27 计费 审计 手机登录 lexhome 支付
```

补入的当日增量：

- `2026-05-27 API Key 脱敏改造 李望`
- `2026-05-27 腾讯云 CLS tccli 查询能力接入总结 李望`
- Slock `#计费-审计-手机登录` 中的计费、审计、手机登录、支付宝支付系统缺口分析
- 本次 Browser 晨报 Notion MCP 生成链路试运行

## 格式踩坑

第一版页面“完全没法看”，主要问题：

- 一次性 `replace_content` 塞入过长引用块。
- 混用复杂表格、长链接、长 callout。
- Notion 渲染后可读性差。

修复方式：

- 去掉复杂表格。
- 去掉长引用块。
- 改用短 TL;DR。
- 使用 H1 / H2 / H3 分级标题。
- 文档概览改成多个小节，不用大表格。
- 待办使用真实 checkbox。
- 链接只保留必要来源链接。

后续默认使用这种朴素格式，避免优先追求复杂排版。

## 推荐页面结构

```text
# TL;DR

- 3-6 条关键结论

# 一、昨日要闻与会议

## 会议：...

来源：
时间：
背景：

### 关键结论
1. ...

### 待办事项
- [ ] ...

### 风险与依赖
- ...

# 二、重大进展与关键卡点

## 主题 A

来源：
内容：

# 三、当日增量

## 主题 B

来源：
内容：

# 四、今日行动清单

- [ ] ...

# 五、更新文档概览

## 1. 文档名

链接：
更新内容：
```

## 质量规则

- 不编造责任人。
- 不用 `last_edited_time` 直接判断实质更新。
- 不追求穷尽所有页面，优先保证会议、关键文档和行动项可交付。
- 对降级或未完整覆盖的地方在页面尾部说明。
- 生成后要打开页面检查格式，如果可读性差，立即重排。

## 本次产出

临时晨报：

```text
https://www.notion.so/36de7e998fb2813288b9e41211d58069
```

覆盖内容：

- 2026-05-26 Browser agent + IAM/OA 自动登录会议。
- 2026-05-26 Browser 项目专项化梳理。
- 2026-05-26/27 云平台 M2/M3 风险。
- 2026-05-27 API Key 脱敏改造。
- 2026-05-27 腾讯云 CLS + tccli 查询能力。
- 2026-05-27 计费 / 审计 / 手机登录 / 支付系统缺口分析。
- 本次晨报生成链路试运行经验。
