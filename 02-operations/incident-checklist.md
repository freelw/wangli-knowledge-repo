# 故障检查清单

## 文档定位

这篇文档用于在后端平台出现异常时，给出一份“先看什么、按什么顺序排”的最小检查清单。

目标不是替代详细 runbook，而是让排障时先把方向拉直。

## 使用原则

遇到问题时，先区分它更像哪一类：

1. 控制面问题
2. websocket 问题
3. 浏览器实例问题
4. 发布 / 配置问题
5. 文档与现状不一致问题

不同问题优先看不同层，不要一上来全仓库乱翻。

## 类别 1：控制面问题

表现通常包括：

1. `browser-manager` 接口报错
2. 创建 session 失败
3. context / extension / downloads 操作失败
4. `/json`、`/json/version` 返回异常

优先检查：

1. `browser-manager` Deployment 是否正常
2. `browser-manager` 当前镜像是否正确
3. `/health`、`/readyz` 是否正常
4. 相关接口日志是否有直接报错
5. 最近是否刚发过 `browser-manager`

优先阅读：

1. `01-architecture/browser-manager-overview.md`
2. `02-operations/release-browser-manager.md`

## 类别 2：websocket 问题

表现通常包括：

1. websocket 无法建立
2. 已有连接掉线
3. `/connection` 连接异常
4. `/devtools/*` relay 不通

优先检查：

1. `browser-ws-gateway` Deployment 是否正常
2. `browser-ws-gateway` 是否刚经历 rollout / draining
3. ws gateway 当前镜像是否正确
4. 相关连接路径是否在日志里报错

特别注意：

当前普通 `browser-manager` 发布不应再被直接等同于 websocket 一起掉线。

现在更应优先看：

1. `browser-ws-gateway` 自身发布
2. `/connection` 握手逻辑
3. draining 期间表现

优先阅读：

1. `01-architecture/websocket-relay.md`
2. `01-architecture/browser-manager-split-plan.md`

## 类别 3：浏览器实例问题

表现通常包括：

1. 实例创建失败
2. Pod 没起来
3. 实例起了但挂载或环境变量不对
4. browser operator 行为异常

优先检查：

1. `browser-operator-controller-manager` 是否正常
2. `browser-operator-api` 是否正常
3. 目标实例是否进入 `Running`
4. 关键挂载、环境变量、卷是否符合预期
5. 如果涉及开关，当前开关值是否正确

特别关注：

1. `ENABLE_CFS_MOUNTS`

优先阅读：

1. `02-operations/release-browser-operator.md`
2. `02-operations/office-runbook.md`

## 类别 4：发布 / 配置问题

表现通常包括：

1. `kubectl apply -k .` 后行为没变化
2. rollout 成功但镜像没切
3. 新配置没生效
4. 只有某个环境异常

优先检查：

1. `images-configmap.yaml` 是否真的更新
2. `kustomization.yaml` 是否已映射该镜像 / 配置字段
3. Deployment 实际镜像是否切换
4. 当前环境是否真的是目标环境
5. 是否只在 `office` 发布了，但误以为其他环境也已经发布

优先阅读：

1. `00-overview/environments.md`
2. `02-operations/release-overview.md`
3. `02-operations/office-runbook.md`

## 类别 5：文档与现状不一致

表现通常包括：

1. 文档里写的架构和代码不一致
2. 文档里的发布口径和真实流程不一致
3. 测试按文档执行，但和集群现状对不上

优先检查：

1. 当前代码是否已经实现了文档里还写成“规划”的内容
2. manifests 是否已经落地了新服务 / 新字段
3. 当前文档是否引用了旧的 `codex-plans` 结论

处理原则：

1. 先以代码和 manifests 事实为准
2. 再回头修知识库
3. 不要继续把过时文档当现状

这个问题当前是高频风险，因为知识库仍处于持续收口阶段。

## office 环境最小排障顺序

如果问题发生在 `office`，建议先按下面顺序看：

1. `kubectl get deployment -n system`
2. `kubectl get pods -n system -o wide`
3. `kubectl rollout status <目标 deployment> -n system`
4. 实际镜像是否符合预期
5. 最小业务链路是否还能跑通

最小业务链路可按问题类型选：

1. HTTP：接口可调
2. websocket：连接能建
3. 实例：能进入 `Running`
4. 监控：target / metric / dashboard 是否正常

## 当前结论

当前平台排障时最重要的不是“记住所有命令”，而是先判断问题属于哪一层：

1. 控制面
2. websocket 承载面
3. 实例编排层
4. 发布配置层
5. 文档漂移层

只要先分层，后面的检查路径就会清楚很多。
