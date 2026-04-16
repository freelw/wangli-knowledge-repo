# Agent 接入全景

## 目的

这篇文档用于统一回答：

1. 当前有哪些 agent / 集成接入方式
2. 哪些已经有稳定入口
3. 哪些只是内部评估
4. 哪些内容可以对外说，哪些还不能对外承诺

## 先看结论

当前最重要的边界可以先记成三类：

1. Codex：当前已有相对稳定的接入入口，走 `browser-skill` / installer
2. OpenClaw：当前已有对外文档，走远程 CDP / browser profile
3. OpenCLI：当前只处于内部接入评估，不代表已完成产品级支持

## 当前已知接入类型

### 1. Codex

当前定位：

1. 已有较稳定入口
2. 主要通过 `browser-skill` 和安装器接入
3. 已有对外文档

当前相关材料：

1. `browser-skill`
2. `lex-home/content/docs/codex.zh.mdx`
3. 知识库中的 `00-overview/repository-map.md`

当前适合对外的说法：

1. 可以通过 Lexmount 的 Codex skill / browser-skill 接入平台浏览器能力

### 2. OpenClaw

当前定位：

1. 已有对外文档
2. 接入模型偏 remote CDP / browser profile
3. 对外文档存在，但版本字段仍应持续校正

当前相关材料：

1. `lex-home/content/docs/openclaw.zh.mdx`
2. 知识库中的仓库地图 / 系统地图相关上下文

当前适合对外的说法：

1. 可以按现有对外文档，通过远程 CDP 方式接入

需要保留的边界：

1. 对外文档字段如果和官方版本有漂移，应先修文档，不要直接扩大承诺

### 3. OpenCLI

当前定位：

1. 仅为内部接入评估
2. 还没有正式产品级支持
3. 当前不应对外承诺“已经支持接入”

当前相关材料：

1. `04-sdk/opencli-integration-assessment.md`

当前适合内部的说法：

1. OpenCLI 和 Lexmount 在 CDP 层存在接入空间
2. 下一步应先做远程 CDP 兼容性验证

当前不适合对外的说法：

1. “Lexmount 已支持 OpenCLI”
2. “只要提供远程浏览器地址就能稳定跑通”

## 按接入方式理解

如果按技术模型来分，当前可以先分成三类：

### 1. SDK 型

典型对象：

1. Python SDK
2. Node.js SDK

特点：

1. 直接调用平台 API
2. 更适合程序化接入

### 2. Skill / Installer 型

典型对象：

1. Codex
2. `browser-skill`

特点：

1. 通过 skill、安装器和环境变量引导接入
2. 更适合 agent 工作流

### 3. Remote CDP / Browser Profile 型

典型对象：

1. OpenClaw
2. OpenCLI（当前仅评估，不算已落地）

特点：

1. 主要接点在远程浏览器和 CDP
2. 更依赖浏览器控制层而不是普通 SDK 调用

## 内部知识与对外文档的边界

### 已有对外文档

当前已明确的对外材料有：

1. `lex-home/content/docs/codex.zh.mdx`
2. `lex-home/content/docs/openclaw.zh.mdx`

### 当前仅限内部知识

当前应保留在知识库内部的内容有：

1. `opencli-integration-assessment.md`
2. 各类尚未实测闭环的接入研判
3. 仅用于内部架构判断的接入边界说明

### 当前禁止对外承诺的内容

当前不应拿去做官网、销售、外部接入承诺的内容包括：

1. OpenCLI 已完成接入
2. 所有 agent 都已经有正式支持入口
3. 任何还没经过正式验证的 remote CDP 兼容性结论

## 当前结论

如果只记住一句话，可以记住：

1. Codex：已有稳定入口
2. OpenClaw：已有对外文档，走 remote CDP
3. OpenCLI：当前只是内部评估，不能对外说成已支持
