# FlClashTier 分步行动与目标

> 行动计划：从"方案评估"到"第一版代码"。当前阶段：**M0 开工**。
> 更新记录见文件末尾。

## 总原则

1. **M0 做得极其扎实** — M0 稳定后 ZeroTier 只是"接第二个出口"的工程问题
2. **改动最小化** — 不动 Mihomo Core / ZeroTier Core / VpnService，只写 Packet Pump
3. 每步有可验证目标，不靠理论推断收尾（MTU 必须实测）

## 阶段 0：工程准备（进行中 ✓）

- [x] 建本地项目目录 `~/FlClashTier/`
- [x] 归档讨论内容 → `docs/开发纪要.md`
- [x] 制定分步行动与目标 → `docs/milestones.md`
- [x] GitHub fork `chen08209/FlClash` → `ximalu/FlClash`
- [x] GitHub fork `zerotier/ZeroTierOne` → `ximalu/ZeroTierOne`（M1 准备）
- [x] GitHub 建项目仓库 `ximalu/FlClashTier`
- [ ] clone FlClash fork 到 `~/FlClashTier/FlClash`
- [ ] 本地构建验证：FlClash 原版能出 APK（建立基线）
  - [ ] Android SDK / NDK / Flutter / Go 工具链确认
  - [ ] 首次构建成功
- [ ] 确认 FlClash 版本与 mihomo vendored 快照版本（Clash.Meta 目录）
- [ ] 推送本项目文档到 `ximalu/FlClashTier`（README + docs）

## 阶段 1：M0 — 纯透传 Pump（当前主攻）

**目标**：真实 TUN → Pump → SOCK_DGRAM socketpair → Mihomo，全透传不分流，FlClash 原有功能零退化。

### 任务分解

- [ ] 定位插入点：`core/tun/tun.go` `tun.Start()` L50-61（LC.Tun 构造点）
- [ ] 实现 packet pump：
  - [ ] `unix.Socketpair(unix.AF_UNIX, unix.SOCK_DGRAM, 0)` 创建 DGRAM socketpair
  - [ ] 泵 goroutine：真 TUN fd → 读 packet → 写 socketpair（全透传）
  - [ ] 反向：socketpair → 读 datagram → 写真 TUN fd
  - [ ] 处理 NativeTun 持有端被 Close 时另一端的 EOF/EPIPE
- [ ] 用 socketpair 一端替换传给 sing_tun 的 fd（不碰 mihomo 核心）
- [ ] 保留真 TUN fd 的 MTU 9000 配置

### M0 验证清单（全部通过才算 M0 完成）

- [ ] HTTP/HTTPS 正常
- [ ] DNS 正常
- [ ] UDP 正常（含 QUIC/游戏类）
- [ ] IPv6 正常
- [ ] 大包正常（iperf3）
- [ ] 长时间运行稳定（≥数小时）
- [ ] Android 锁屏/解锁后连接不中断
- [ ] VPN 重启（开关 VPN）正常
- [ ] 网络 Wi-Fi → 5G → Wi-Fi 切换正常
- [ ] **MTU 压力测试**：1500 / 2000 / 4000 / 9000 分别跑 iperf3、HTTP download、UDP、ping -M do
  - 关注：EMSGSIZE、packet drop、fragmentation 异常、gVisor panic、throughput 下降
- [ ] 确认启动日志只有预期 Warn（`get tun name failed ... fallback to FlClash`），无其他异常

### 退出标准

> 已证明：**不修改 Mihomo 的情况下"截获并重新分发"FlClash 的全部 TUN 流量。**

## 阶段 2：M1 — ZeroTier 分流（M0 稳定后）

**目标**：Pump 分流 `dst ∈ 192.168.191.0/24 → ZeroTier，else → Mihomo`；Ubuntu ↔ Android ping 互通。

### 任务分解

- [ ] 引入 ZeroTier Core（C++，走 FlClash 现有 cgo 链路，静态链 libzerotiercore 或官方 Go 绑定；JNI 不加）
- [ ] 实现 adapter：ZeroTier Core `VirtualNetworkConfigCallback` 帧回调 → Pump
- [ ] 底层物理 socket 用 `VpnService.protect(socket)` 绕过 TUN（防环关键）
- [ ] 硬编码分流规则：`192.168.191.0/24 → ZT，else → Mihomo`
- [ ] 反向：ZeroTier frame → Pump → 真 TUN fd

### 验证清单

- [ ] `Ubuntu 192.168.191.1 │ ZeroTier │ Android 192.168.191.100` **ping 通**（第一验证目标）
- [ ] Android 上网（公网代理）不退化
- [ ] ZT 内网用 IP 访问正常（主机名 DNS 分流问题已知，先用 IP）

## 阶段 3：M2 — 防环 + 自动 Managed Routes

- [ ] 不硬编码网段：ZT core 的 managed routes 自动进 Pump 路由表
- [ ] 防环机制完善（ZT UDP 控制流量保护）
- [ ] 处理 ZT managed route 配 0.0.0.0/0 饿死 mihomo 的场景（路由优先级或警告）

## 风险登记

| 风险 | 应对 |
|------|------|
| socketpair 高并发下不等价于 TUN | M0 MTU/吞吐压力测试先行 |
| gVisor 栈对 socketpair 的兼容 | FlClash 默认 gvisor，实测；必要时对比 system 栈 |
| ZT 控制流量环路 | 复用官方 protect() 方案，不做 IP 白名单 |
| DNS 分流冲突 | ZT 内网先用 IP，M2 再处理 DNS |
| GitHub clone 超时 | codeload tarball / goproxy.cn 兜底 |

## 更新记录

- 2026-08-13：初版。基于 M0 定稿讨论（架构、SOCK_DGRAM 决策、MTU 压测、ZeroTier adapter 方式）。
