# 内部知识与对外文档映射

## 目的

这篇文档用于把三类东西分清楚：

1. 哪些内容属于内部知识库判断
2. 哪些内容已经有正式对外文档
3. 哪些内容当前必须明确写成“禁止对外承诺”

## 先看结论

当前最重要的规则只有三条：

1. `knowledge-repo` 用于内部判断、评估和收口
2. 对外正式说明以 `lex-home/content/docs/*` 为准
3. 任何仍处于评估阶段的接入能力，都不能被转述成“已经支持”

## 当前映射关系

### Codex

内部知识：

1. `04-sdk/agent-access-landscape.md`
2. `00-overview/repository-map.md` 里关于 `browser-skill` 的说明
3. 相关任务归档和安装器经验

对外正式文档：

1. `lex-home/content/docs/codex.zh.mdx`

当前可对外口径：

1. 可以通过 Lexmount Codex skill / browser-skill 接入平台浏览器能力

### OpenClaw

内部知识：

1. `04-sdk/agent-access-landscape.md`
2. 相关架构和仓库地图说明

对外正式文档：

1. `lex-home/content/docs/openclaw.zh.mdx`

当前可对外口径：

1. 可以按现有文档，通过远程 CDP / browser profile 接入

需要保留的边界：

1. 对外文档字段如果和上游版本漂移，应先修正文档，再对外引用

### OpenCLI

内部知识：

1. `04-sdk/opencli-integration-assessment.md`
2. `04-sdk/agent-access-landscape.md`

对外正式文档：

1. 当前没有

当前口径：

1. 只允许作为内部评估结论存在
2. 不允许被转述成已完成支持

## 禁止对外承诺的通用规则

下面这些情况，当前都不应直接写进官网、销售材料或外部接入说明：

1. 仍需要兼容性验证的接入能力
2. 只有内部评估文档、还没有正式产品化入口的能力
3. 任何带有“可能可行”“存在接入空间”“待验证”的结论

## 可直接复用的“禁止对外承诺”句子

### OpenCLI

1. 本文仅代表内部接入评估，不代表 Lexmount 已完成对 OpenCLI 的产品级支持。
2. 当前不应对外表述为“Lexmount 已支持 OpenCLI”。
3. 在远程 CDP 兼容性验证完成前，OpenCLI 相关结论不得作为官网或销售承诺使用。

### OpenClaw

1. 当前对外能力以现有正式文档为准，不应擅自扩大到未验证配置字段或未校正文档版本。
2. 如内部结论与现有对外文档不一致，应先修正文档，再对外引用。

### Codex / browser-skill

1. 对外说明应以现有安装器和正式文档能力范围为准，不应扩大到尚未验证的 agent 场景。
2. 不应把内部实验性 workflow 直接表述为正式产品能力。

### 评估类文档通用句

1. 本文用于内部判断与方案评估，不直接构成对外能力承诺。
2. 如需对外说明，请以 `lex-home` 中的正式文档为准。

## 当前结论

如果只记住一句话，可以记住：

1. 内部判断看 `knowledge-repo`
2. 对外说明看 `lex-home/content/docs`
3. 评估结论在正式验证前，一律不能写成“已经支持”
