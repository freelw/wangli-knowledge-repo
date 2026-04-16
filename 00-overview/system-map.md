# 系统地图

## 目的

这篇文档从整体视角说明当前浏览器云平台后端的主链路结构，帮助新加入的人先回答这几个问题：

1. 整个系统由哪些核心服务组成
2. 请求是怎么从控制面走到浏览器实例的
3. 哪些服务负责 HTTP，哪些服务负责 websocket，哪些服务负责后台补偿
4. 环境配置和发布落在哪一层

## 当前主链路

当前系统主链路可以简化成下面 5 层：

1. 客户端 / SDK / agent
2. `browser-manager`
3. `browser-ws-gateway`
4. `k8s-chrome-daemon`
5. `lexmount-k8s-manifests`

其中：

1. `browser-manager` 负责 HTTP 控制面
2. `browser-ws-gateway` 负责 websocket 接入与 relay
3. `k8s-chrome-daemon` 负责浏览器实例编排
4. `lexmount-k8s-manifests` 负责环境部署配置

后台还有一个关键服务：

1. `browser-manager-reconciler`

它负责 creating session 补偿、active session cleanup、downloads cleanup、closed history cleanup 等后台循环任务。

## 一张文字版结构图

```text
客户端 / SDK / Agent
        |
        | HTTP 控制面请求
        v
browser-manager
        |
        | 创建 / 查询 / 删除 session, context, extension, downloads
        | 生成对外 inspect / ws 地址
        v
k8s-chrome-daemon
        |
        | 创建 BrowserInstance / 浏览器 Pod
        v
浏览器实例

客户端 / SDK / Agent
        |
        | WebSocket / CDP
        v
browser-ws-gateway
        |
        | relay 到真实 Chrome DevTools websocket
        v
浏览器实例

browser-manager-reconciler
        |
        | 后台补偿 / 清理 / 收尾
        v
session_history / contexts / downloads 等状态

lexmount-k8s-manifests
        |
        | 配置上述服务在各环境中的镜像、配置、部署方式
        v
office / qcloud / qcloud-hk / aws / guoge
```

## 各层职责

## 1. 客户端 / SDK / agent

这一层包括：

1. Python SDK
2. Node SDK
3. quickstart 示例
4. browser-skill
5. 其他 agent / 工具接入

它们负责消费平台能力，而不是管理浏览器生命周期本身。

## 2. `browser-manager`

这是当前 HTTP 控制面入口。

当前主要负责：

1. session 创建 / 查询 / 删除
2. context 创建 / 锁定 / 删除
3. extension 管理
4. downloads 管理
5. `/json`、`/json/version` 等 inspect 代理
6. 对外 websocket 地址改写

这是“控制面”。

## 3. `browser-ws-gateway`

这是当前独立的 websocket 接入服务。

当前主要负责：

1. `/devtools/*`
2. `/connection`
3. client <-> chrome 双向 relay
4. CDP activity 上报
5. CDP tracking
6. draining 和优雅下线

这是“长连接承载面”。

## 4. `k8s-chrome-daemon`

这是浏览器实例编排层。

当前主要负责：

1. 接收创建请求
2. 创建 `BrowserInstance`
3. 生成浏览器 Pod
4. 管理实例生命周期
5. 管理运行时挂载、环境变量和实例行为

这是“实例执行面”。

## 5. `browser-manager-reconciler`

这是后台补偿与清理层。

当前主要负责：

1. creating session 补偿
2. active session 清理
3. downloads 清理
4. closed history 清理

它让 session 生命周期不只依赖前台请求，而有后台自愈和收尾能力。

## 6. `lexmount-k8s-manifests`

这是部署配置层。

当前主要负责：

1. 管理各环境镜像 tag
2. 管理 configmap / secret / deployment / service / overlay
3. 通过 `kubectl apply -k` 把服务真正发布到目标环境

这是“环境落地层”。

## 当前最重要的架构认知

## 1. HTTP 控制面和 websocket 承载面已经拆开

当前系统不能再简单描述成“`browser-manager` 同时承载 HTTP + websocket”。

更准确的现状是：

1. `browser-manager` 负责 HTTP
2. `browser-ws-gateway` 负责 websocket relay

## 2. 但边界还没有完全纯化

虽然服务形态已经拆开，但 `/connection` 仍然带有一部分控制面逻辑。

因此当前仍处于：

1. 服务已拆
2. 边界继续纯化

这个阶段。

## 3. session 生命周期是跨服务联合完成的

session 的创建、激活、使用、关闭和清理，不是在单个服务里独立完成的，而是跨：

1. `browser-manager`
2. `browser-ws-gateway`
3. `browser-manager-reconciler`
4. `k8s-chrome-daemon`

联合完成。

## 环境视角

当前最重要的环境是：

1. `office`
2. `qcloud`

其中：

1. `office` 是默认测试 / 验证环境
2. 本机可以直接 `kubectl` 连接 `office`
3. 大多数开发、测试、发布验证都先落 `office`

## 推荐阅读顺序

如果看完这篇还要继续补上下文，建议按下面顺序读：

1. `00-overview/repository-map.md`
2. `00-overview/environments.md`
3. `01-architecture/browser-manager-overview.md`
4. `01-architecture/websocket-relay.md`
5. `01-architecture/browser-manager-split-plan.md`
6. `01-architecture/session-lifecycle.md`

## 当前结论

如果只记住一句话，可以记成：

> 当前浏览器云平台后端已经形成“HTTP 控制面、websocket 承载面、实例编排层、后台补偿层、环境配置层”这 5 层主结构，而知识库的目标就是把这 5 层的职责和边界稳定沉淀下来。
