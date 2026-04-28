# SDK 与集成

这一层不只放 SDK 本身，还放“外部如何接入平台”的相关知识。

当前建议阅读顺序：

1. `agent-access-landscape.md`
2. `external-doc-mapping.md`
3. `opencli-integration-assessment.md`
4. `session-targets-and-inspect-url.md`
5. `sdk-nearest-access-design.md`

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

### `external-doc-mapping.md`

回答：

1. 哪些内容属于内部知识库
2. 哪些内容已有 `lex-home` 对外文档
3. 哪些说法必须写成“禁止对外承诺”

### `session-targets-and-inspect-url.md`

回答：

1. session 本体与 target 级数据的边界怎么划分
2. `/json` 与 `/json/version` 在这次任务里分别承担什么职责
3. JS / Python SDK 如何暴露 session targets
4. browser-manager 变更后为什么还要同步 `office` / `qcloud` / `qcloud-hk` 三边 tag

### `sdk-nearest-access-design.md`

回答：

1. SDK 就近接入到底在选什么，哪些层不应该暴露给 SDK
2. 为什么正式默认路径只能消费 `scope=public`
3. `region / endpoint / catalog` 三层模型应该怎么划分
4. 为什么对外 `region` 不应该直接用 `qcloud / office / qcloud-hk`
5. 第一版自动选路为什么固定为 `catalog -> filter -> probe -> select`

## 建议后续补充

后续如果继续补这一层，优先顺序建议是：

1. `codex-browser-skill-access.md`
2. `sdk-landscape.md`
3. `quickstart-alignment.md`
4. `sdk-nearest-access-implementation.md`

其中：

1. `codex-browser-skill-access.md` 更偏“Codex / browser-skill 当前稳定入口”
2. `sdk-landscape.md` 更偏 Python / JS SDK 的能力对齐
3. `quickstart-alignment.md` 更偏示例仓库覆盖情况
4. `sdk-nearest-access-implementation.md` 更偏正式落地时的 SDK 参数、缓存、fallback 与日志实现细节

## 当前结论

这层文档里最重要的区分不是“是不是都算 SDK”，而是：

1. 哪些已经是正式接入入口
2. 哪些只是内部评估
3. 哪些已有对外文档
4. 哪些还不应该对外承诺
