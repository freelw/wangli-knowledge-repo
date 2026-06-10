# Gateway iptables Ops

Source Notion: https://app.notion.com/p/336e7e998fb28006a95be8400ea029c0

This runbook records the VPN gateway `iptables` forwarding rules for internal Grafana/Kong access.

## Scope

Maintain forwarding rules on two VPN gateways:

- `qcloud` gateway: `10.20.0.1`
- `qcloud-hk` gateway: `10.20.0.2`

The goal is to forward VPN client traffic sent to gateway port `443` to each environment's `kong-internal` port `31443`.

## Current Parameters

### qcloud

- Login: `ssh root@10.20.0.1`
- VPN interface IP: `10.20.0.1`
- Egress interface IP: `10.206.0.79`
- Forward target: `10.206.0.23:31443`
- Verify host: `ops.lexmount.cn`

### qcloud-hk

- Login: `ssh root@10.20.0.2`
- VPN interface IP: `10.20.0.2`
- Egress interface IP: `172.19.0.6`
- Forward target: `172.19.0.9:31443`
- Verify host: `ops.lexmount.com`

Additional qcloud-hk forwarding entry:

- Gateway VIP: `10.20.0.3`
- Forward target: `172.19.0.7:30443`
- Rule purpose: forward traffic sent to `10.20.0.3:443` to `172.19.0.7:30443`.

## Forwarding Model

Each gateway uses four rules:

1. `PREROUTING DNAT`: rewrite gateway `443` traffic to target node `31443`.
2. `FORWARD`: allow traffic from `wg0` to target node `31443`.
3. `FORWARD`: allow established return traffic from target node back to `wg0`.
4. `POSTROUTING SNAT`: rewrite source IP to the gateway egress IP.

The host must also enable IP forwarding:

```bash
sysctl -w net.ipv4.ip_forward=1
```

## Inspect Rules

```bash
iptables -t nat -S
iptables -S
iptables -t nat -vnL
iptables -vnL
sysctl net.ipv4.ip_forward
```

Expected forwarding switch:

```bash
net.ipv4.ip_forward = 1
```

## Add Temporary Rules

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

# Additional qcloud-hk entry: 10.20.0.3:443 -> 172.19.0.7:30443
iptables -t nat -A PREROUTING -d 10.20.0.3/32 -i wg0 -p tcp --dport 443 \
  -j DNAT --to-destination 172.19.0.7:30443

iptables -A FORWARD -d 172.19.0.7/32 -i wg0 -o eth0 -p tcp --dport 30443 \
  -j ACCEPT

iptables -A FORWARD -s 172.19.0.7/32 -i eth0 -o wg0 -p tcp --sport 30443 \
  -m state --state RELATED,ESTABLISHED -j ACCEPT

iptables -t nat -A POSTROUTING -d 172.19.0.7/32 -o eth0 -p tcp --dport 30443 \
  -j SNAT --to-source 172.19.0.6
```

## Delete Temporary Rules

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

# Additional qcloud-hk entry: 10.20.0.3:443 -> 172.19.0.7:30443
iptables -t nat -D PREROUTING -d 10.20.0.3/32 -i wg0 -p tcp --dport 443 \
  -j DNAT --to-destination 172.19.0.7:30443

iptables -D FORWARD -d 172.19.0.7/32 -i wg0 -o eth0 -p tcp --dport 30443 \
  -j ACCEPT

iptables -D FORWARD -s 172.19.0.7/32 -i eth0 -o wg0 -p tcp --sport 30443 \
  -m state --state RELATED,ESTABLISHED -j ACCEPT

iptables -t nat -D POSTROUTING -d 172.19.0.7/32 -o eth0 -p tcp --dport 30443 \
  -j SNAT --to-source 172.19.0.6
```

## Persistence

Both gateways currently persist rules through `/etc/rc.local`.

Known backup file:

- `/etc/rc.local.bak.codex-20260401`

Use an idempotent block guarded by `iptables -C ... || iptables -A ...`.

### Persist qcloud-hk additional entry

Append the following block to `/etc/rc.local` on the qcloud-hk gateway (`10.20.0.2`).

Place it before the final `exit 0` line if `exit 0` already exists.

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

Recommended edit flow:

```bash
ssh root@10.20.0.2
cp /etc/rc.local /etc/rc.local.bak.$(date +%Y%m%d%H%M%S)
vim /etc/rc.local
chmod +x /etc/rc.local
systemctl status rc-local || true
```

After editing, run the block once manually or reboot during a maintenance window, then verify:

```bash
iptables -t nat -S | grep '10.20.0.3\\|172.19.0.7'
iptables -S | grep '172.19.0.7'
curl -skI https://10.20.0.3/
```

Rollback persistence by removing the block between:

```bash
# BEGIN codex-kong-internal-forward-10-20-0-3
# END codex-kong-internal-forward-10-20-0-3
```

Then delete the temporary rules in the `Delete Temporary Rules` section.

## Business Verification

### qcloud

```bash
curl -skI -H 'Host: ops.lexmount.cn' https://10.20.0.1/grafana/
```

Expected:

- `HTTP/1.1 302 Found`
- `Location: /grafana/login`

### qcloud-hk

```bash
curl -skI -H 'Host: ops.lexmount.com' https://10.20.0.2/grafana/
```

Expected:

- `HTTP/1.1 302 Found`
- `Location: /grafana/login`

### qcloud-hk additional entry

The additional qcloud-hk entry forwards `10.20.0.3:443` to `172.19.0.7:30443`.

```bash
curl -skI https://10.20.0.3/
```

Expected result depends on the service exposed by `172.19.0.7:30443`; at minimum, the request should reach the target instead of timing out at the gateway.

## Troubleshooting

### Gateway port 443 is unreachable

Check:

```bash
iptables -t nat -S
iptables -S
sysctl net.ipv4.ip_forward
```

### Request times out

Check target node connectivity:

```bash
# qcloud
nc -vz 10.206.0.23 31443

# qcloud-hk
nc -vz 172.19.0.9 31443

# qcloud-hk additional entry
nc -vz 172.19.0.7 30443
```

### Request returns 404

Check whether the request uses the expected Host header:

- `qcloud`: `ops.lexmount.cn`
- `qcloud-hk`: `ops.lexmount.com`

### Request returns 403

Check:

- Whether the request is hitting old public Kong by mistake.
- Whether `kong-init-inner.js` has been executed.
- Whether the `kong-internal` route exists.

### Rules are lost after reboot

Check:

```bash
systemctl status rc-local
cat /etc/rc.local
```

## Change Principles

1. Apply temporary rules first.
2. Verify business traffic.
3. Persist to `/etc/rc.local` only after validation.
4. Rollback both temporary rules and the matching `/etc/rc.local` block.
5. Do not switch the target until `kong-internal` is confirmed healthy.
