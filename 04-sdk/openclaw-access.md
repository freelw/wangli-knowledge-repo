# OpenClaw 接入说明

## 文档定位

这篇文档说明 OpenClaw 如何接入 Lexmount，以及这条接入路线和 Codex Skill 的区别。

本文优先沉淀“当前可稳定复用的接入知识”，对于仍需复核的部分会明确标注。

## 名称说明

当前业务口头里提到的 `opencli`，根据现有上下文和调研结果，更大概率实际指的是 **OpenClaw**。

因此，当前知识库统一使用 **OpenClaw** 作为正式名称。

如果后续确认用户说的是另一个独立产品，再单独拆分文档，不在这里混写。

## 核心结论

OpenClaw 接入 Lexmount 的最短路径是：

1. 不需要先做专门 SDK 适配
2. 直接走 **Remote CDP**
3. Lexmount 提供可访问的 WebSocket CDP 地址
4. OpenClaw 把这个地址配置成 browser profile
5. 后续通过 OpenClaw 自己的 browser CLI / browser tool 控制远程浏览器

也就是说，这条路线本质上不是“做一个 OpenClaw 专用集成”，而是“让 OpenClaw 复用 Lexmount 已提供的远程浏览器接入面”。

## 为什么能这样接

当前 Lexmount 已具备以下能力：

1. 提供远程浏览器会话
2. 提供 WebSocket CDP 连接地址
3. 具备 `project_id` / `api_key` 鉴权体系
4. 已经在服务 Codex、Claude Code、browser skill 等 agent/browser 场景

OpenClaw 的 browser profile 模型和这套能力是对齐的。

因此，对 OpenClaw 来说，Lexmount 更像“远程浏览器提供方”。

## 接入模型

可以把这条链路理解成：

1. OpenClaw 启动本地或网关进程
2. OpenClaw 的 browser profile 指向 Lexmount 的 `wss://...` 地址
3. Lexmount 通过 `project_id` 与 `api_key` 识别项目和权限
4. OpenClaw 通过自己的 browser 命令完成打开页面、点击、输入、截图等操作

## 最小配置思路

当前最短可行接入模型可以理解成下面这种结构：

```json
{
  "plugins": {
    "allow": ["browser"]
  },
  "browser": {
    "enabled": true,
    "defaultProfile": "lexmount",
    "profiles": {
      "lexmount": {
        "cdpUrl": "wss://<base>/connection?project_id=<project_id>&api_key=<api_key>"
      }
    }
  }
}
```

这里的重点不是字段样式本身，而是接入模型：

1. 开启 browser 能力
2. 定义一个 Lexmount profile
3. `cdpUrl` 指向 Lexmount 提供的远程连接地址

## 最小验证命令

如果要做一条最短 smoke test，可以优先验证“能否通过 profile 打开页面”。

可参考：

```bash
openclaw browser profiles
openclaw browser open "https://www.bilibili.com" --browser-profile lexmount
openclaw browser snapshot --target-id <id>
```

如果只保留一条验证命令，第二条通常就够用。

## Lexmount 侧至少要提供什么

如果要把 OpenClaw 接入做成可交付能力，Lexmount 至少要清楚提供以下信息：

1. 稳定可用的 `wss` 连接地址
2. `project_id` 与 `api_key` 的获取入口
3. `.cn` / `.com` 环境差异
4. session 生命周期说明
5. 并发或连接限制说明
6. 连接失败时的基本排查入口

## 环境差异

当前知识库应至少区分两类环境：

### `.cn`

适用于国内环境。

接入文档中需要明确：

1. 控制台入口
2. API 基址
3. WebSocket 地址前缀

### `.com`

适用于国际环境。

接入文档中需要明确：

1. 控制台入口
2. API 基址
3. 与 `.cn` 不同的环境变量或接入路径

这里建议和 `00-overview/environments.md` 保持互相引用，不要各写一套环境说明。

## 当前认证方式与后续演进

### 当前可用方案

当前最短可行方案是把 `project_id` 和 `api_key` 直接拼进 `cdpUrl` 查询参数。

优点：

1. 接入门槛低
2. 很容易做演示和快速验证

缺点：

1. 密钥容易长期固化在本地配置里
2. 截图、录屏、日志里更容易泄露
3. 长期治理成本高

### 中期建议

中期更合理的方向是：

1. 保留 query 参数方案作为最低门槛接入
2. 增加短时效 token 或签名 URL
3. 把“生成临时连接地址”做成控制台动作或 API

## 和 Codex Skill 的区别

这两条接入路线不要混写。

### Codex Skill

特点：

1. 更偏 Lexmount 原生接入
2. 通常通过 skill 安装器和环境变量完成配置
3. 重点在 agent 使用体验和平台内能力集成

### OpenClaw

特点：

1. 更偏通用远程浏览器接入
2. 通过 OpenClaw 自己的 browser profile 接入
3. 重点是复用 Remote CDP，而不是做专用 SDK

## 当前相关材料

当前本地已有材料包括：

1. `lex-home/content/docs/openclaw.zh.mdx`
2. `browser-skill/README.md`
3. `codex-plans` 中的相关归档材料

这些材料可以作为上下文参考，但不能不复核就直接视为最终版知识。

## 待复核项

以下内容目前应视为“需要复核后再对外定稿”的部分：

### 1. `opencli` 这个名称

当前不要把它当正式产品名写入对外知识，除非后续确认它确实是另一个独立产品。

### 2. 旧版配置字段可能漂移

当前本地 `lex-home/content/docs/openclaw.zh.mdx` 中的配置字段，与最近调研到的官方写法可能存在轻微版本漂移。

因此在更新官网文档前，应先再次核对：

1. browser 配置结构
2. plugin / tool 的允许项字段
3. profile 配置格式

### 3. 查询参数直传密钥的方案

当前可用，但不建议直接把它当长期最佳实践写死。

## 推荐知识库落点

这部分知识建议分两层维护：

### 一层：knowledge repo

放在：

1. `04-sdk/openclaw-access.md`

这里主要沉淀：

1. 接入模型
2. 与平台能力的关系
3. 环境差异
4. 认证演进方向
5. 与 Codex Skill 的差异

### 二层：对外产品文档

例如：

1. `lex-home/content/docs/openclaw.zh.mdx`

这里主要负责：

1. 可复制的配置示例
2. 参数说明
3. 最小验证命令
4. 常见报错排查

## 当前结论

如果现在要给团队一个简洁、可执行的说法，可以直接用下面这段：

> OpenClaw 接入 Lexmount 的最短路径是 Remote CDP。Lexmount 提供带鉴权参数的 WebSocket CDP 地址，OpenClaw 把它配置成 browser profile 后，就可以直接通过自己的 browser CLI 和 browser tool 控制远程浏览器。当前知识库可以先沉淀这套接入模型，但在更新正式对外文档前，需要先复核产品名与配置字段，避免把旧版配置结构固化下来。
