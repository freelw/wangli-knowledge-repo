# Codex 中读取 Notion 的正确方式

## 背景

2026-05-27 在测试 Notion 读取能力时，先误用了本地项目里的 `NOTION_KEY`。该 key 来自 `/home/lexmount/project/backend/fetch-issues-to-notion/.env`，对应 Notion integration：

```text
openclaw-git-issue
```

这个 integration 只能看到它被授权的 `issues` database，不能代表用户在 Codex 中通过网页 OAuth 授权的 Notion 访问范围。

## 结论

在 Codex 中，如果用户说“我已经授权了 Notion”或“用 Codex 授权的 Notion”，应使用 Codex 配置的 Notion MCP，而不是项目里的 `NOTION_KEY` / integration token。

不要把下面两种授权面混在一起：

1. **Notion integration token**
   - 形态：`NOTION_KEY`
   - 来源：项目 `.env`
   - 权限：只取决于该 integration 被 share 到哪些页面/数据库
   - 本次例子：只能访问 `issues` database

2. **Codex Notion MCP OAuth**
   - 形态：Codex MCP server `notion`
   - 配置：`~/.codex/config.toml` 中的 `[mcp_servers.notion]`
   - 登录：`codex mcp login notion`
   - 权限：用户在 Codex/Notion OAuth 中授权的范围
   - 本次例子：能读到 `Browser as a Service` 页面树

## 当前机器状态

当前机器的 Codex MCP 已配置 Notion：

```bash
codex mcp list
```

可看到：

```text
Name    Url                         Status   Auth
notion  https://mcp.notion.com/mcp  enabled  OAuth
```

`~/.codex/config.toml` 中也有：

```toml
[mcp_servers.notion]
url = "https://mcp.notion.com/mcp"
```

## 正确读取方式

在当前 Slock/Codex agent 工具面如果没有直接注入 `Notion:notion-search` / `Notion:notion-fetch` 函数，可以通过一次子 Codex 调用让 Codex 使用自己的 Notion MCP：

```bash
codex exec \
  -s read-only \
  --skip-git-repo-check \
  -C /home/lexmount/.slock/agents/12bbe8ab-f66a-4974-a3d7-52b59cfe8098 \
  -o /tmp/notion-codex-out.txt \
  "Use the configured Notion MCP only. Do not use shell or environment variables. Search Notion for 'Browser as a Service'. If found, fetch it or list a few child/page titles under it. Output only concise titles and whether you used Notion MCP."
```

关键约束：

- 明确要求 `Use the configured Notion MCP only`
- 明确禁止 `shell or environment variables`
- 不读取项目 `.env`
- 不使用 `NOTION_KEY`

## 本次验证结果

使用 Codex Notion MCP 后，能够看到 `Browser as a Service`，并读到下面的 title：

- `Browser as a Service`
- `Browser 文档导读（先看我）`
- `2025~2026 Browser as a Service 工作总结`
- `Notion AI Prompts`
- `Browser 项目`
- `Browser 周报`
- `沟通纪要`
- `归档`

## 经验规则

1. 用户明确说“Codex 中授权了 Notion”时，默认走 Codex Notion MCP。
2. 不要因为本地某个项目存在 `.env` / `NOTION_KEY` 就直接使用 integration token。
3. 如果只能看到很小的数据库范围，例如只看到 `issues`，应先怀疑自己用了 integration token，而不是用户没有授权页面。
4. 回复用户时要明确说明使用的是哪一种授权面：`Codex Notion MCP OAuth` 或 `NOTION_KEY integration token`。
5. 读取用户 Notion 内容默认只读；需要写入或创建页面时先明确告知目标页面/数据库。
