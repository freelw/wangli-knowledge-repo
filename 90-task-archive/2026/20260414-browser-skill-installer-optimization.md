# 2026-04-14 browser-skill 安装脚本优化

## 背景

browser-skill 安装流程需要同时适配 China region endpoint 与 Global region endpoint，并且应尽量复用用户已有环境变量，降低配置成本和误填概率。

## 任务目标

1. 在安装开始时让用户明确选择环境
2. 自动识别已有环境变量并提示导入
3. 对 Global region endpoint 补充 `LEXMOUNT_BASE_URL`
4. 明确告知不同环境获取 `project_id` / `api_key` 的入口

## 涉及仓库

1. `browser-skill`

## 关键改动

这次任务的核心是安装脚本交互优化方向，关键点包括：

1. 让用户在最开始选择 region preset：
   - `China region` -> `browser.lexmount.cn`
   - `Global region` -> `browser.lexmount.com`
2. 检测本地是否已有：
   - `LEXMOUNT_PROJECT_ID`
   - `LEXMOUNT_API_KEY`
   - `LEXMOUNT_BASE_URL`
3. 如果已存在配置，则提示是否导入
4. 对 Global region endpoint 显式补：
   - `LEXMOUNT_BASE_URL=https://api.lexmount.com`
5. 给出不同环境的 API key 页面入口

## 验证方式

这次任务材料里没有记录最终代码验证结果。

后续合理的验证方式应包括：

1. China region 空配置首次安装
2. Global region 空配置首次安装
3. 已存在 China region 配置的导入流程
4. 已存在 Global region 配置的导入流程
5. 用户中途切换环境时的行为

## 产出的长期知识

这次任务后续可沉淀进：

1. `04-sdk/sdk-landscape.md`
2. browser-skill 的正式安装文档

## 分支与 PR

当前材料未记录对应的开发分支与 PR。

## 结论

1. 这次任务明确了 browser-skill 安装器需要具备环境识别能力
2. China region endpoint / Global region endpoint 的差异不应再靠用户手动理解
3. 安装器应主动复用已有配置，减少重复填写
4. 这为后续跨环境 agent 接入体验统一打下了基础
