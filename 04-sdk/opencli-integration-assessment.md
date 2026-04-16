# OpenCLI 接入评估

## 文档定位

这篇文档记录 `jackwener/opencli` 与 Lexmount 的接入研判结果。

本文不是“已完成接入手册”，而是“当前可确认的接入空间、限制和建议验证路径”。

仓库：

- <https://github.com/jackwener/opencli>

## 一句话结论

OpenCLI 不是一个像浏览器 profile 那样把 Lexmount 直接挂进去就能稳定跑通的 agent 产品。

它更像一个把网站、浏览器、Electron 应用和本地工具统一包装成 CLI / agent 能力的平台。

对 Lexmount 来说，最关键的接点在浏览器控制层，尤其是 CDP。

当前更准确的判断是：

1. 有接入空间
2. 但不是零适配
3. 需要先做远程 CDP 兼容性验证

## OpenCLI 是什么

从当前仓库和说明来看，OpenCLI 主要有三类能力面：

### 1. 网站适配器

把具体网站包装成可复用的命令，例如：

1. `opencli hackernews top`
2. `opencli bilibili hot`
3. `opencli reddit hot`

这类能力强调“把网站操作沉淀成稳定 CLI”。

### 2. `opencli browser`

这是更接近 agent/browser 场景的底层浏览器能力。

当前支持的动作包括：

1. `open`
2. `state`
3. `click`
4. `type`
5. `wait`
6. `get`
7. `network`
8. `screenshot`

### 3. 桌面应用和本地工具

它还可以控制 Electron 应用，或者把本地 CLI 纳入统一体系。

## 和 Lexmount 的接点

Lexmount 和 OpenCLI 的主要交点，不在网站适配器层，而在浏览器控制层。

### 相关点 1：OpenCLI 的关键能力建立在真实浏览器操作上

它的很多价值都依赖：

1. 打开真实页面
2. 复用登录态
3. 点击、输入、抓请求
4. 从浏览器行为中沉淀出可复用命令

这与 Lexmount 的远程浏览器能力属于同一问题域。

### 相关点 2：仓库里确实存在 CDP 接入面

当前调研里已确认这些点：

1. README 中存在 `OPENCLI_CDP_ENDPOINT`
2. 支持 `OPENCLI_CDP_TARGET`
3. 代码中存在 `CDPBridge`

这说明它在协议层已经预留了 CDP 能力入口。

从接入可能性上看，这意味着：

1. Lexmount 提供的远程 CDP 地址有机会接入
2. OpenCLI 不一定只接受本地浏览器

### 相关点 3：但当前主路径仍明显偏本地桥接模式

这是当前最关键的边界。

按调研结果，OpenCLI 当前更成熟的浏览器工作流仍依赖：

1. 本地 Chrome / Chromium
2. Browser Bridge extension
3. local daemon

也就是说，当前最常用的 `opencli browser` 工作流，并不是把远程浏览器当默认后端。

## 当前可确认的结论

### 可以确认的部分

以下内容可以作为当前稳定结论写入知识库：

1. OpenCLI 不是 OpenClaw 这类 profile 型接入产品
2. 它和 Lexmount 的接点主要在浏览器控制层，尤其是 CDP
3. 仓库里确实存在 `OPENCLI_CDP_ENDPOINT` / `OPENCLI_CDP_TARGET` 和 `CDPBridge`
4. 它当前主浏览器工作流仍偏本地 Chrome + extension + daemon
5. Lexmount 若要接入，下一步应先做远程 CDP 兼容性验证

### 不能写太满的部分

以下内容当前还只是调研推断，不能直接写成“已成立事实”：

1. OpenCLI 已经完整支持 Lexmount 这类远程浏览器
2. 只要提供一个 `ws` 地址就能无缝跑通全部 browser workflow
3. 不需要任何额外适配就能直接作为稳定后端

这些结论目前都缺实测支撑。

## 建议的接入分层

### 第一阶段：协议兼容性验证

先不要追求“正式接入”，先验证几个基础问题：

1. OpenCLI 是否能连上 Lexmount 提供的远程 CDP
2. target 选择是否稳定
3. 页面状态读取是否正常
4. click / type / wait / screenshot 是否可用
5. network 抓包能力是否完整

这一步的目标是确认“协议层可行性”。

### 第二阶段：产品级适配评估

如果第一阶段通过，再评估以下产品级问题：

1. 登录态如何持久化和复用
2. session / context 生命周期是否能匹配 OpenCLI 习惯
3. 长连接与超时模型是否兼容
4. 并发与多页面 target 选择是否稳定
5. 认证方式是否足够安全

## Lexmount 侧需要关注的能力边界

如果未来要让 Lexmount 成为 OpenCLI 的远程浏览器后端，至少要明确这些边界：

### 1. 远程 CDP 提供方式

需要明确：

1. 是否直接提供 `ws` / `wss` 地址
2. 是否还需要 `http(s)://.../json` 这种 target 枚举入口

### 2. 登录态模型

OpenCLI 对“复用登录态”的需求很强。

Lexmount 侧需要说明：

1. session / context 如何复用
2. 持久态如何保存
3. 不同会话之间是否隔离

### 3. target 模型

需要明确：

1. 一个远程浏览器实例里 target 是否稳定
2. target 数量和筛选逻辑是否可控
3. 是否支持按 target 做稳定选择

### 4. network 能力

OpenCLI 的一些高价值场景依赖 network 抓包。

需要确认：

1. 远程 CDP 下能否完整拿到请求
2. 响应预览和请求元数据是否可用

### 5. 认证暴露方式

如果只是把 `api_key` 长期拼到查询参数里，短期能用，但长期不优雅也不安全。

更合理的长期方向是：

1. 短时效 token
2. 一次性签名连接地址
3. 控制台或 API 生成临时连接入口

## 推荐验证路径

建议按下面顺序推进，而不是直接写“已支持接入”。

### 步骤 1：做最小 CDP 验证

验证：

1. 连接远程 CDP
2. 枚举 target
3. 打开页面
4. 读取页面状态

### 步骤 2：做基础交互验证

验证：

1. 点击
2. 输入
3. 等待
4. 截图

### 步骤 3：做 network 验证

验证：

1. 请求事件是否完整
2. 响应预览是否完整
3. 弱网或长会话下是否稳定

### 步骤 4：再讨论正式产品接入

只有前面三步通过后，才适合讨论：

1. 是否把 Lexmount 正式作为 OpenCLI 的远程后端
2. 是否需要补专门接入文档
3. 是否需要产品侧增加临时连接地址、认证升级等能力

## 当前结论

如果现在要给团队一个可以复述的说法，可以直接用下面这段：

> OpenCLI 和 Lexmount 的接点主要在浏览器控制层，尤其是 CDP。仓库里已经有 `OPENCLI_CDP_ENDPOINT`、`OPENCLI_CDP_TARGET` 和 `CDPBridge`，说明协议层存在接入空间；但从当前 README 和代码看，它的主浏览器工作流仍明显依赖本地 Chrome、扩展和本地 daemon，而不是直接把远程浏览器当默认后端。因此，这件事当前更适合写成“接入评估”，下一步应先做远程 CDP 兼容性验证，而不是直接写成已经完成接入。
