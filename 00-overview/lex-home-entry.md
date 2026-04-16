# lex-home 入口

## 目的

这篇文档用于给前端、联调和控制台相关工作一个最短入口，回答：

1. `lex-home` 在整套系统里是什么
2. 代码、部署、镜像、环境变量分别看哪里
3. 前端联调出问题时先排查哪几层

## `lex-home` 是什么

`lex-home` 当前同时承担：

1. 官网入口
2. 文档站入口
3. 控制台前端入口

它不是浏览器平台后端本身，但它是用户和控制台操作最常见的第一入口。

## 代码与部署入口

### 代码仓库

```text
/home/lexmount/project/backend/lex-home
```

### K8s 配置

```text
/home/lexmount/project/backend/lexmount-k8s-manifests/apps/lex-home
```

### 关键前端动作入口

当前最值得先看的文件：

1. `lex-home/src/actions/kong.ts`

它会直接使用这些环境变量：

1. `KONG_ADMIN_BASE_URL`
2. `BROWSER_BASE_URL`
3. `POCKETBASE_URL`

## 部署与镜像

### 命名空间与对象

当前 `lex-home` 在 K8s 里最关键的对象是：

1. namespace：`frontend`
2. Deployment：`lex-home`
3. Service：`lex-home`

### 镜像字段

各环境统一通过 `lex-home-image` 回填镜像。

常见位置：

1. `apps/clusters/office/images-configmap.yaml`
2. `apps/clusters/qcloud/images-configmap.yaml`
3. `apps/clusters/qcloud-hk/images-configmap.yaml`

### 镜像仓库差异

1. `office`：`code.lexmount.net/feixiang/lex-home`
2. `qcloud`：`lexmount.tencentcloudcr.com/cloud/lex-home`
3. `qcloud-hk`：`lexmoun-tcr-hk.tencentcloudcr.com/cloud/lex-home`

### 常用脚本

当前已有两个直接相关脚本：

1. `apps/tools/set_lex_home_image_tags.sh`
2. `apps/tools/sync_lex_home.sh`

## 前端最小联调路径

从前端视角理解最小链路，可以先按下面顺序：

1. 用户在 `lex-home` 页面触发操作
2. `lex-home` 的 server actions 调用内部依赖
3. 控制面请求进入 `browser-manager`
4. 如涉及 websocket / CDP，再进入 `browser-ws-gateway`
5. 如涉及实例创建，再进入 `k8s-chrome-daemon`

所以前端联调异常，不要只盯页面本身，要先区分是哪一层出问题：

1. `lex-home` 页面 / server actions
2. `browser-manager` HTTP 控制面
3. `browser-ws-gateway` websocket 链路
4. `lexmount-k8s-manifests` 环境配置

## office 环境最小验证

如果当前只做 `office` 联调，建议最少确认：

1. `frontend` namespace 下 `lex-home` Deployment 正常
2. `lex-home-image` 已切到目标镜像
3. `lex-home` 页面能打开
4. 页面触发的关键联调动作能命中后端

## 当前结论

如果只记住一句话，可以记住：

1. `lex-home` 是前端入口层
2. 代码看 `lex-home`
3. 部署看 `apps/lex-home`
4. 环境看 `images-configmap.yaml` 和 `lex-home-configmap.yaml`
5. 联调异常要沿“页面 -> server actions -> browser-manager -> ws / instance”往下拆
