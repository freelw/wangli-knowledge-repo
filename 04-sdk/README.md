# SDK 与集成

这一层不只放 SDK 本身，还放“外部如何接入平台”的相关知识。

当前建议阅读顺序：

1. `agent-access-landscape.md`
2. `opencli-integration-assessment.md`

## 当前已有文档

### `agent-access-landscape.md`

回答：

1. 当前有哪些 agent / 集成接入方式
2. 哪些已经有稳定入口
3. 哪些只是内部评估
4. 哪些内容可以对外说，哪些还不能对外承诺

### `opencli-integration-assessment.md`

回答：

1. `jackwener/opencli` 和 Lexmount 的接点在哪里
2. 为什么当前只能写成“接入评估”
3. 下一步应优先验证什么

## 建议后续补充

后续如果继续补这一层，优先顺序建议是：

1. `codex-browser-skill-access.md`
2. `sdk-landscape.md`
3. `quickstart-alignment.md`

其中：

1. `codex-browser-skill-access.md` 更偏“Codex / browser-skill 当前稳定入口”
2. `sdk-landscape.md` 更偏 Python / JS SDK 的能力对齐
3. `quickstart-alignment.md` 更偏示例仓库覆盖情况

## 当前结论

这层文档里最重要的区分不是“是不是都算 SDK”，而是：

1. 哪些已经是正式接入入口
2. 哪些只是内部评估
3. 哪些已有对外文档
4. 哪些还不应该对外承诺
