# 腾讯云 CFS 挂载后无法创建子目录的处理经验

## 背景

在腾讯云 CFS 挂载到 Kubernetes / Linux 节点后，如果业务容器或初始化逻辑无法在挂载目录下创建子目录，优先检查 CFS 的 squash 策略与目录属主权限。

## 推荐处理顺序

1. 先把 CFS 的 squash 策略调整为默认 squash 策略。
2. 在挂载目录下创建业务需要的子目录。
3. 创建完成后，把对应目录权限调整为 `nobody:nogroup`。
4. 再把 CFS 的 squash 策略改回 `all squash`。

示例：

```bash
# 在默认 squash 策略下创建目录
mkdir -p /path/to/cfs/<subdir>

# 调整目录属主
chown -R nobody:nogroup /path/to/cfs/<subdir>

# 然后再到腾讯云 CFS 控制台把 squash 策略切回 all squash
```

## 原因

`all squash` 会把客户端访问映射到匿名用户。如果目录还不存在，或者父目录权限不匹配，容器/节点侧可能无法完成子目录创建。

先使用默认 squash 策略创建目录，再把目录属主调整为 `nobody:nogroup`，可以让后续 `all squash` 下的匿名用户继续正常访问该目录。

## 注意事项

- 不要直接在 `all squash` 状态下排查为应用自身权限问题，先确认 CFS squash 策略。
- 目录创建和 `chown` 应在切回 `all squash` 前完成。
- 该流程适合“挂载成功但无法创建子目录”的场景；如果挂载本身失败，应先排查网络、安全组、CFS 挂载点和 NFS 客户端配置。
