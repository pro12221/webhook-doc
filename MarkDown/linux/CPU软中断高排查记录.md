# CPU软中断高排查记录

## 基本信息

| 项目 | 内容 |
|---|---|
| 告警 | [P2-告警]越南胡志明01\|SEA05\|[uvmp\|vmzone-10.65.197.5-CPU软中断]当前值6.00%(>=6%) |
| 主机 | sea05-uvmp-197-5 |
| IP | 10.65.197.5 |
| 架构 | KVM 宿主机，48 vCPU，Mellanox ConnectX 双口网卡 |
| 角色 | 虚拟化宿主机，运行约 20+ VM，OVS 软件转发 |

## 根因结论

Mellanox 网卡 RSS 48 队列已开启（硬中断分散），但 **RPS 全部关闭**（`rps_cpus=0`），导致软中断只能在硬中断触发的那颗 CPU 上处理，无法跨核分流。前 22 颗核承担了绝大部分 NET_RX 软中断，后 26 颗核几乎空转，热点核软中断远超 6% 触发告警。

## 排查步骤

### 第 1 步：perf 定位 CPU 热点方向

**目的**：确认 CPU 时间花在哪里，是网络、磁盘还是其他。

```bash
perf record -g -a -- sleep 30
perf report
```

**结论**：

- CPU 空闲占 55.3%（整体不繁忙）
- 软中断入口 `__softirqentry_text_start` 占 5.62%（全核平均）
- 热点集中在网络收包路径：`net_rx_action` → `mlx5e_napi_poll` → `ovs_vport_receive` → `vhost_worker`
- 两台 VM 的 vhost 线程占大头：vhost-2393（7.67%）+ vhost-10069（5.79%）
- `copy_user_enhanced_fast_string` Self 占 1.78%，vhost 数据拷贝开销显著

**数据流路径还原**：

```
Mellanox NIC → 硬中断 → 软中断(NET_RX) → NAPI poll → OVS datapath → vhost → VM(tun)
    ↓              ↓           ↓              ↓            ↓           ↓         ↓
mlx5e_handle_rx_cq  do_IRQ  net_rx_action  mlx5e_napi_poll  ovs_dp_process_packet  vhost_worker  tun_get_user
                    ↓           ↓                            ↓           ↓
                irq_exit   __softirqentry              ovs_execute_actions  handle_tx/handle_rx
```

### 第 2 步：/proc/softirqs 确认软中断类型和分布

**目的**：确认是哪类软中断（NET_RX/BLOCK/TIMER 等），以及在各 CPU 间的分布是否均匀。

```bash
cat /proc/softirqs
```

**结论**：

- 软中断类型确认为 **NET_RX（网络收包）**
- **分布极度不均**：CPU3 有 38.36 亿次，CPU47 仅 2472 万次，**差距 155 倍**
- CPU0~21 承担了绝大部分 NET_RX，CPU22~47 几乎空闲

> 💡 量化验证：全核平均 softirq = 5.62%，softirq 总消耗 ≈ 270% 单核时间。CPU3 的 NET_RX 占全系统 ~11% → softirq ≈ 30%。CPU47 的 NET_RX 占 ~0.07% → softirq ≈ 0.2%。

### 第 3 步：/proc/interrupts 查看网卡中断分布

只看到管理面中断（pages_eq/cmd_eq/async_eq），全部绑在 CPU12。没有 `mlx5_comp*` 数据通道中断，Mellanox 驱动在 NAPI 模式下以 poll 方式工作。

### 第 4 步：确认网络拓扑

```
net0/1 (1a:00, 管理网卡) ── manbr ── net2 (af:00.0, Mellanox P0)
                                      │ 21个VM

net3 (af:00.1, Mellanox P1) ── wanbr ── 17个VM

GRE 隧道 (wildcard_gre) ── br0 ── 21个VM（无物理网卡，流量经 net2/net3 出去）

三个 OVS 桥共享 Mellanox 网卡带宽，纯软件转发，无硬件卸载。
```

### 第 5 步：确认 Mellanox RSS/RPS/GRO 配置

| 配置项 | 状态 | 说明 |
|---|---|---|
| RSS 多队列 | ✅ 已配 48 队列 | 硬中断分散到多核 |
| GRO/GSO/TSO | ✅ 已开启 | 减少包数，降低 CPU 开销 |
| hw-tc-offload | ✅ 已开启 | 硬件条件具备 |
| **RPS** | **❌ 全部关闭** | **rps_cpus 全部 = 0，核心问题** |

### 第 6 步：确认高流量 VM

vhost-2393：12 线程，每线程 6.9%~8.4% CPU，总计 ~95% CPU。vhost-10069：12 线程，每线程 4.2%~7.6% CPU，总计 ~70% CPU。

### 第 7 步：确认网络流量规模

| 网卡 | RX pps | TX pps | RX 带宽 | TX 带宽 |
|---|---|---|---|---|
| net2 | 486K pps | 529K pps | 581 MB/s | 681 MB/s |
| net3 | 103K pps | 37K pps | 127 MB/s | 2.7 MB/s |

## 根因分析

### 因果链条

1. **RPS 关闭**（rps_cpus=0）→ 软中断无法跨核迁移
2. **硬中断绑定 = 软中断绑定** → 硬中断在哪颗 CPU 触发，软中断就只能在那颗 CPU 处理
3. **RSS 只解决硬中断分布** → 48 个 RSS 队列分散硬中断，但软中断无法跨核迁移
4. **前 22 核包揽、后 26 核空闲** → 热点核软中断远超 6%，冷核接近 0%
5. **全核平均被冷核拉低** → perf 显示 5.62%，但热点核实际 20-30%

## 优化方案

### P0：开启 RPS（最关键，即时生效，风险极低）

```bash
# net2（主流量网卡）
for q in /sys/class/net/net2/queues/rx-*/rps_cpus; do
  echo "ffff,ffffffffffff" > $q
done
for q in /sys/class/net/net2/queues/rx-*/rps_flow_cnt; do
  echo 4096 > $q
done

# net3
for q in /sys/class/net/net3/queues/rx-*/rps_cpus; do
  echo "ffff,ffffffffffff" > $q
done
for q in /sys/class/net/net3/queues/rx-*/rps_flow_cnt; do
  echo 4096 > $q
done

# OVS 桥
echo "ffff,ffffffffffff" > /sys/class/net/br0/queues/rx-0/rps_cpus
echo 4096 > /sys/class/net/br0/queues/rx-0/rps_flow_cnt
echo "ffff,ffffffffffff" > /sys/class/net/wanbr/queues/rx-0/rps_cpus
echo 4096 > /sys/class/net/wanbr/queues/rx-0/rps_flow_cnt

# 全局 RPS 流表
echo 196608 > /proc/sys/net/core/rps_sock_flow_entries
```

| 指标 | 优化前 | 优化后（预期） |
|---|---|---|
| 参与 NET_RX 的 CPU | ~22 核（前半） | 48 核全部参与 |
| 热点核 CPU 软中断占比 | 20-30% | < 2% |
| CPU 软中断告警 | 6% ≥ 6% 触发 | < 3%，告警消除 |

### P1：持久化 RPS 配置

```bash
cat >> /etc/sysctl.conf << 'EOF'
net.core.rps_sock_flow_entries = 196608
EOF
sysctl -p
```

### P2：验证效果

```bash
watch -n2 'cat /proc/softirqs | head -1; cat /proc/softirqs | grep NET_RX'
mpstat -P ALL 2 3
cat /sys/class/net/net2/queues/rx-0/rps_cpus
```

### P3（后续优化）：OVS 硬件卸载

当前 OVS 2.10.5 较老，若后续升级可开启 `hw-offload=true`，预期软中断降低 60-80%。

## 附录：RPS 原理

**RPS（Receive Packet Steering）** 是 Linux 内核的软件级网络收包分流机制。

没有 RPS 时，硬中断在哪颗 CPU 触发，后续所有软中断处理就死死绑在这颗核上，不能迁移。

有 RPS 时，RPS 根据数据包的五元组（srcIP/dstIP/srcPort/dstPort/protocol）计算 hash，将包分发到不同 CPU 的 backlog 队列，让多核并行处理软中断。

| 机制 | 层级 | 作用 | 本机状态 |
|---|---|---|---|
| RSS | 硬件 | 多队列，硬中断分散到多核 | ✅ 已开（48 队列） |
| RPS | 软件 | 软中断跨核分流 | ❌ 全关（根因） |

RSS 解决硬中断分布，RPS 解决软中断分布。两者互补，缺一不可。

## 附录：perf 热点符号解读

| 符号 | 占比 | 含义 |
|---|---|---|
| `intel_idle` | 49.40% | CPU 执行 mwait/hlt 指令等待中断唤醒（空闲） |
| `vhost_worker` | 7.67%+4.74% | vhost 工作线程主循环，处理 VM 网络数据 |
| `__softirqentry_text_start` | 5.62% | 软中断入口，告警"CPU软中断"的计数点 |
| `net_rx_action` | 4.46% | NET_RX 软中断主函数，调用 NAPI poll 收包 |
| `mlx5e_napi_poll` | 3.40% | Mellanox 驱动 NAPI 轮询入口 |
| `handle_tx` | 3.73%+2.20% | vhost 发包方向（宿主机→VM） |
| `tun_get_user` | 3.25% | 从 tun 设备读取数据（vhost 拷贝数据） |
| `handle_rx` | 2.93% | vhost 收包方向（VM→宿主机） |
| `__netif_receive_skb_core` | 2.24% | 内核收包核心，将 skb 分发给协议栈或 OVS |
| `mlx5e_poll_rx_cq` | 1.97% | Mellanox RX 完成队列轮询 |
| `netdev_frame_hook` | 2.04% | OVS 注册的钩子，拦截进入 OVS 桥的包 |
| `ovs_vport_receive` | 2.02% | OVS 虚拟端口收包 |
| `ovs_dp_process_packet` | 1.94% | OVS 数据路径：查流表决定转发动作 |
| `copy_user_enhanced_fast_string` | 1.78% | vhost 将数据从内核拷贝到 VM 的 vring（Self 最高） |
| `ovs_execute_actions` | 1.52% | 执行 OVS 流表动作 |
