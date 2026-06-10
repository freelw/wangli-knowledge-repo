# 网关 iptables 运维

来源 Notion：https://app.notion.com/p/336e7e998fb28006a95be8400ea029c0

本文档记录 VPN 网关上的 `iptables` 转发规则，用于维护内网 Grafana / Kong 访问入口。

## 适用范围

维护两台 VPN 网关上的转发规则：

- `qcloud` 网关：`10.20.0.1`
- `qcloud-hk` 网关：`10.20.0.2`

目标是把 VPN 客户端访问网关 `443` 端口的流量，转发到对应环境的 `kong-internal` 端口。

## 现网参数

### qcloud

- 登录方式：`ssh root@10.20.0.1`
- VPN 网卡 IP：`10.20.0.1`
- 出口网卡 IP：`10.206.0.79`
- 转发目标：`10.206.0.23:31443`
- 验证 Host：`ops.lexmount.cn`

### qcloud-hk

- 登录方式：`ssh root@10.20.0.2`
- VPN 网卡 IP：`10.20.0.2`
- 出口网卡 IP：`172.19.0.6`
- 转发目标：`172.19.0.9:31443`
- 验证 Host：`ops.lexmount.com`

qcloud-hk 新增转发入口：

- 网关 VIP：`10.20.0.3`
- 转发目标：`172.19.0.7:30443`
- 规则用途：把访问 `10.20.0.3:443` 的流量转发到 `172.19.0.7:30443`。

## 转发原理

每个入口使用四类规则：

1. `PREROUTING DNAT`：把网关 `443` 入站流量改写到目标节点端口。
2. `FORWARD`：放行从 `wg0` 到目标节点的正向流量。
3. `FORWARD`：放行目标节点回包到 `wg0` 的反向流量。
4. `POSTROUTING SNAT`：把源地址改写为网关出口 IP。

主机还必须开启 IP 转发：

```bash
sysctl -w net.ipv4.ip_forward=1
```

## 查看规则

```bash
iptables -t nat -S
iptables -S
iptables -t nat -vnL
iptables -vnL
sysctl net.ipv4.ip_forward
```

转发开关预期输出：

```bash
net.ipv4.ip_forward = 1
```

## 临时添加规则

### qcloud

```bash
sysctl -w net.ipv4.ip_forward=1

iptables -t nat -A PREROUTING -d 10.20.0.1/32 -i wg0 -p tcp --dport 443 \
  -j DNAT --to-destination 10.206.0.23:31443

iptables -A FORWARD -d 10.206.0.23/32 -i wg0 -o eth0 -p tcp --dport 31443 \
  -j ACCEPT

iptables -A FORWARD -s 10.206.0.23/32 -i eth0 -o wg0 -p tcp --sport 31443 \
  -m state --state RELATED,ESTABLISHED -j ACCEPT

iptables -t nat -A POSTROUTING -d 10.206.0.23/32 -o eth0 -p tcp --dport 31443 \
  -j SNAT --to-source 10.206.0.79
```

### qcloud-hk

```bash
sysctl -w net.ipv4.ip_forward=1

iptables -t nat -A PREROUTING -d 10.20.0.2/32 -i wg0 -p tcp --dport 443 \
  -j DNAT --to-destination 172.19.0.9:31443

iptables -A FORWARD -d 172.19.0.9/32 -i wg0 -o eth0 -p tcp --dport 31443 \
  -j ACCEPT

iptables -A FORWARD -s 172.19.0.9/32 -i eth0 -o wg0 -p tcp --sport 31443 \
  -m state --state RELATED,ESTABLISHED -j ACCEPT

iptables -t nat -A POSTROUTING -d 172.19.0.9/32 -o eth0 -p tcp --dport 31443 \
  -j SNAT --to-source 172.19.0.6

# qcloud-hk 新增入口：10.20.0.3:443 -> 172.19.0.7:30443
iptables -t nat -A PREROUTING -d 10.20.0.3/32 -i wg0 -p tcp --dport 443 \
  -j DNAT --to-destination 172.19.0.7:30443

iptables -A FORWARD -d 172.19.0.7/32 -i wg0 -o eth0 -p tcp --dport 30443 \
  -j ACCEPT

iptables -A FORWARD -s 172.19.0.7/32 -i eth0 -o wg0 -p tcp --sport 30443 \
  -m state --state RELATED,ESTABLISHED -j ACCEPT

iptables -t nat -A POSTROUTING -d 172.19.0.7/32 -o eth0 -p tcp --dport 30443 \
  -j SNAT --to-source 172.19.0.6
```

## 删除临时规则

### qcloud

```bash
iptables -t nat -D PREROUTING -d 10.20.0.1/32 -i wg0 -p tcp --dport 443 \
  -j DNAT --to-destination 10.206.0.23:31443

iptables -D FORWARD -d 10.206.0.23/32 -i wg0 -o eth0 -p tcp --dport 31443 \
  -j ACCEPT

iptables -D FORWARD -s 10.206.0.23/32 -i eth0 -o wg0 -p tcp --sport 31443 \
  -m state --state RELATED,ESTABLISHED -j ACCEPT

iptables -t nat -D POSTROUTING -d 10.206.0.23/32 -o eth0 -p tcp --dport 31443 \
  -j SNAT --to-source 10.206.0.79
```

### qcloud-hk

```bash
iptables -t nat -D PREROUTING -d 10.20.0.2/32 -i wg0 -p tcp --dport 443 \
  -j DNAT --to-destination 172.19.0.9:31443

iptables -D FORWARD -d 172.19.0.9/32 -i wg0 -o eth0 -p tcp --dport 31443 \
  -j ACCEPT

iptables -D FORWARD -s 172.19.0.9/32 -i eth0 -o wg0 -p tcp --sport 31443 \
  -m state --state RELATED,ESTABLISHED -j ACCEPT

iptables -t nat -D POSTROUTING -d 172.19.0.9/32 -o eth0 -p tcp --dport 31443 \
  -j SNAT --to-source 172.19.0.6

# qcloud-hk 新增入口：10.20.0.3:443 -> 172.19.0.7:30443
iptables -t nat -D PREROUTING -d 10.20.0.3/32 -i wg0 -p tcp --dport 443 \
  -j DNAT --to-destination 172.19.0.7:30443

iptables -D FORWARD -d 172.19.0.7/32 -i wg0 -o eth0 -p tcp --dport 30443 \
  -j ACCEPT

iptables -D FORWARD -s 172.19.0.7/32 -i eth0 -o wg0 -p tcp --sport 30443 \
  -m state --state RELATED,ESTABLISHED -j ACCEPT

iptables -t nat -D POSTROUTING -d 172.19.0.7/32 -o eth0 -p tcp --dport 30443 \
  -j SNAT --to-source 172.19.0.6
```

## 持久化

两台网关当前都通过 `/etc/rc.local` 持久化规则。

已知备份文件：

- `/etc/rc.local.bak.codex-20260401`

持久化规则必须使用 `iptables -C ... || iptables -A ...` 这种幂等写法，避免机器重启或重复执行后产生重复规则。

### 持久化 qcloud-hk 新增入口

在 qcloud-hk 网关 `10.20.0.2` 上，把下面的 block 写入 `/etc/rc.local`。

如果 `/etc/rc.local` 末尾已经有 `exit 0`，必须把这个 block 放在 `exit 0` 之前。

```bash
# BEGIN codex-kong-internal-forward-10-20-0-3
sysctl -w net.ipv4.ip_forward=1 >/dev/null
iptables -t nat -C PREROUTING -d 10.20.0.3/32 -i wg0 -p tcp --dport 443 -j DNAT --to-destination 172.19.0.7:30443 2>/dev/null || \
  iptables -t nat -A PREROUTING -d 10.20.0.3/32 -i wg0 -p tcp --dport 443 -j DNAT --to-destination 172.19.0.7:30443
iptables -C FORWARD -d 172.19.0.7/32 -i wg0 -o eth0 -p tcp --dport 30443 -j ACCEPT 2>/dev/null || \
  iptables -A FORWARD -d 172.19.0.7/32 -i wg0 -o eth0 -p tcp --dport 30443 -j ACCEPT
iptables -C FORWARD -s 172.19.0.7/32 -i eth0 -o wg0 -p tcp --sport 30443 -m state --state RELATED,ESTABLISHED -j ACCEPT 2>/dev/null || \
  iptables -A FORWARD -s 172.19.0.7/32 -i eth0 -o wg0 -p tcp --sport 30443 -m state --state RELATED,ESTABLISHED -j ACCEPT
iptables -t nat -C POSTROUTING -d 172.19.0.7/32 -o eth0 -p tcp --dport 30443 -j SNAT --to-source 172.19.0.6 2>/dev/null || \
  iptables -t nat -A POSTROUTING -d 172.19.0.7/32 -o eth0 -p tcp --dport 30443 -j SNAT --to-source 172.19.0.6
# END codex-kong-internal-forward-10-20-0-3
```

推荐编辑流程：

```bash
ssh root@10.20.0.2
cp /etc/rc.local /etc/rc.local.bak.$(date +%Y%m%d%H%M%S)
vim /etc/rc.local
chmod +x /etc/rc.local
systemctl status rc-local || true
```

编辑完成后，可以手动执行一次 block，或者在维护窗口重启机器后验证：

```bash
iptables -t nat -S | grep '10.20.0.3\\|172.19.0.7'
iptables -S | grep '172.19.0.7'
curl -skI https://10.20.0.3/
```

持久化回滚方式：删除下面两个标记之间的 block：

```bash
# BEGIN codex-kong-internal-forward-10-20-0-3
# END codex-kong-internal-forward-10-20-0-3
```

然后执行“删除临时规则”章节中的删除命令。

## 业务验证

### qcloud

```bash
curl -skI -H 'Host: ops.lexmount.cn' https://10.20.0.1/grafana/
```

预期：

- `HTTP/1.1 302 Found`
- `Location: /grafana/login`

### qcloud-hk

```bash
curl -skI -H 'Host: ops.lexmount.com' https://10.20.0.2/grafana/
```

预期：

- `HTTP/1.1 302 Found`
- `Location: /grafana/login`

### qcloud-hk 新增入口

qcloud-hk 新增入口把 `10.20.0.3:443` 转发到 `172.19.0.7:30443`。

```bash
curl -skI https://10.20.0.3/
```

预期结果取决于 `172.19.0.7:30443` 暴露的具体服务；最低要求是请求能到达目标服务，而不是在网关侧超时。

## 常见故障排查

### 网关 443 端口不通

检查：

```bash
iptables -t nat -S
iptables -S
sysctl net.ipv4.ip_forward
```

### 请求超时

检查目标节点端口连通性：

```bash
# qcloud
nc -vz 10.206.0.23 31443

# qcloud-hk
nc -vz 172.19.0.9 31443

# qcloud-hk 新增入口
nc -vz 172.19.0.7 30443
```

### 请求返回 404

检查请求是否使用了预期 Host：

- `qcloud`: `ops.lexmount.cn`
- `qcloud-hk`: `ops.lexmount.com`

### 请求返回 403

检查：

- 是否误打到了旧公网 Kong。
- `kong-init-inner.js` 是否已经执行。
- `kong-internal` 路由是否已经创建。

### 重启后规则丢失

检查：

```bash
systemctl status rc-local
cat /etc/rc.local
```

## 变更原则

1. 先添加临时规则。
2. 验证业务流量。
3. 验证通过后再写入 `/etc/rc.local`。
4. 回滚时同时删除临时规则和 `/etc/rc.local` 中对应 block。
5. 未确认 `kong-internal` 可用前，不要切换转发目标。
