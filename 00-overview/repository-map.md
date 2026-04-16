# 仓库地图

## 目的

这篇文档用于回答两个问题：

1. 后端平台当前有哪些核心仓库
2. 遇到问题时应该优先去哪个仓库看

## 核心主链路仓库

后端平台当前最核心的主链路，不应再只看三段，而应看四个核心服务层：

1. `demo-nodejs-backend/browser-manager`
2. `demo-nodejs-backend/browser-ws-gateway`
3. `k8s-chrome-daemon`
4. `lexmount-k8s-manifests`

这四者共同组成：

1. HTTP 控制面
2. websocket relay 承载面
3. 浏览器实例编排层
4. 环境部署配置层

### `demo-nodejs-backend/browser-manager`

职责：

1. 对外提供 HTTP 接口入口
2. 提供 session、context、extension、downloads 等控制面能力
3. 提供 `/json`、`/json/version` 等 inspect 相关代理
4. 生成对外 inspect / websocket 地址

适合优先查看的场景：

1. API 行为异常
2. session / context 控制逻辑问题
3. inspect URL 返回问题
4. `/json`、`/json/version` 等地址改写问题

### `demo-nodejs-backend/browser-ws-gateway`

职责：

1. 独立承载 websocket 接入
2. 处理 `/devtools/*`
3. 处理 `/connection`
4. relay client <-> chrome 的 CDP 消息
5. 承载 CDP activity / tracking 和 draining 逻辑

适合优先查看的场景：

1. websocket 无法建立
2. `/devtools/*` 路径异常
3. `/connection` 握手异常
4. 已建立连接在发布 / draining 期间的表现
5. relay 相关日志、追踪、消息转发问题

### `k8s-chrome-daemon`

职责：

1. 负责 Chrome 实例创建
2. 负责 `BrowserInstance` 生命周期维护
3. 根据请求生成和销毁浏览器 Pod
4. 承接浏览器运行时相关的挂载、环境变量、实例行为控制

适合优先查看的场景：

1. 浏览器实例创建失败
2. Pod 生命周期异常
3. context / download / extension 挂载问题
4. 浏览器容器环境变量注入问题

### `lexmount-k8s-manifests`

职责：

1. 管理浏览器云平台相关服务的 Kubernetes 配置
2. 按环境维护镜像、配置和 overlay
3. 负责发布时的镜像回填和配置落地

适合优先查看的场景：

1. 某环境发布后行为不一致
2. office / qcloud / qcloud-hk 配置差异
3. 镜像 tag 没有同步
4. 环境变量、configmap、kustomization 配置问题

## relay 链路在仓库里的位置

如果要专门理解 relay 链路，当前最相关的仓库位置是：

1. `demo-nodejs-backend/browser-ws-gateway/server.js`
2. `demo-nodejs-backend/browser-ws-gateway/websocket.js`
3. `demo-nodejs-backend/browser-ws-gateway/session-activity.js`
4. `demo-nodejs-backend/browser-ws-gateway/cdp-tracking.js`

同时还需要结合：

1. `demo-nodejs-backend/browser-manager/chrome`
2. `demo-nodejs-backend/browser-manager/routes/proxy.js`
3. `demo-nodejs-backend/browser-manager/config.js`

原因是：

1. `browser-manager` 仍负责对外 websocket 地址生成和 inspect 代理
2. `browser-ws-gateway` 负责真正的 websocket relay 承载

所以 relay 链路不是“只看一个目录”就能完全理解，而是：

1. `browser-manager` 负责入口地址改写
2. `browser-ws-gateway` 负责长连接承载与 relay

## SDK 与示例仓库

这层仓库主要负责“客户端接入”和“快速验证”。

### Python 侧

1. `lexmount-python-sdk`
   - 官方 Python SDK
2. `lexmount-python-sdk-quickstart`
   - Python 快速上手示例

适合优先查看的场景：

1. Python 侧 API 用法
2. context / session 在 Python 中的使用方式
3. quickstart 示例是否已覆盖某项能力

### Node.js 侧

1. `lexmount-js-sdk`
   - 官方 Node.js SDK
2. `lexmount-js-sdk-quickstart`
   - Node.js 快速上手示例

适合优先查看的场景：

1. Node.js 侧 API 用法
2. SDK 是否已与 Python 对齐
3. 某能力在 quickstart 中是否已有示例

## 其他相关仓库与工具

### `browser-skill`

职责：

1. 为 Codex、Claude Code、OpenClaw 等提供 browser skill 能力
2. 承担平台在 agent 侧的接入与安装逻辑

适合优先查看的场景：

1. agent 如何使用平台能力
2. 安装脚本、环境变量引导、skill 使用体验

## 推荐排查顺序

### 场景 1：接口层或 SDK 层异常

优先顺序：

1. `browser-manager`
2. 对应 SDK 仓库
3. 对应 quickstart 仓库

### 场景 2：websocket / relay 异常

优先顺序：

1. `browser-ws-gateway`
2. `browser-manager`
3. `lexmount-k8s-manifests`

### 场景 3：浏览器实例启动或运行异常

优先顺序：

1. `k8s-chrome-daemon`
2. `lexmount-k8s-manifests`
3. `browser-manager`

### 场景 4：环境发布或配置异常

优先顺序：

1. `lexmount-k8s-manifests`
2. 镜像来源仓库，例如 `browser-manager`、`browser-ws-gateway` 或 `k8s-chrome-daemon`

### 场景 5：agent 接入异常

优先顺序：

1. `browser-skill`
2. `browser-manager`
3. 对应 SDK / quickstart

## 当前结论

如果只记住一件事，可以记住这个分工：

1. `browser-manager` 负责 HTTP 控制面
2. `browser-ws-gateway` 负责 websocket relay 承载
3. `k8s-chrome-daemon` 负责浏览器实例编排
4. `lexmount-k8s-manifests` 负责环境配置和发布落地

其他仓库大多围绕这条主链路提供 SDK、示例、agent 接入或专项能力。
