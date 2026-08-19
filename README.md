# FlClashTier

**FlClash + ZeroTier Core 双网络出口** — 在 FlClash（Android VpnService + Mihomo 内核）中嵌入 ZeroTier Core，实现单 VPN 双出口：

- **ZeroTier** 管私网（ZT managed routes，如 192.168.191.0/24）
- **Mihomo** 管代理/公网（DIRECT / PROXY）

> [!IMPORTANT]
> **本仓库是项目文档仓库（docs only），不包含代码。**
> **代码仓库（FlClash fork）：https://github.com/ximalu/FlClash**

## 核心思路

不改 Mihomo、不改 Android VpnService、不加 JNI —— 只在 `tun.Start()` 与 `sing_tun.New()` 之间插入一个 **Packet Pump**（TUN 数据分流层）：

```
Android VpnService
      │  real TUN fd
      ▼
┌───────────┐
│ ZT Pump   │  ← FlClashTier 真正新增的核心
└─────┬─────┘
      │  SOCK_DGRAM socketpair
      ▼
sing_tun / Mihomo
      │
      ▼
    Proxy
```

M0 的本质：把 Mihomo 原来直接拿到的 TUN fd 换成一个行为上足够像 TUN 的 **Unix DGRAM fd**。

## 里程碑

| 阶段 | 内容 | 目标 |
|------|------|------|
| **M0** | 真实 TUN → Pump → socketpair → Mihomo，全透传不分流，ZeroTier=0 | FlClash 原有功能零退化（浏览器/DNS/TCP/UDP/IPv6） |
| **M1** | Pump 分流：`dst ∈ ZT 网段 → ZeroTier，else → Mihomo`（先硬编码 192.168.191.0/24） | Ubuntu ↔ Android ping 互通 + Android 上网正常 |
| **M2** | 防环 + 自动 Managed Routes（不硬编码网段） | ZT core 的 managed routes 自动进 Pump 路由表 |

## 仓库结构

本仓库仅包含文档：

- `docs/开发纪要.md` — 全部讨论内容归档
- `docs/milestones.md` — 分步行动与目标（任务清单）

代码在独立仓库（fork）：**[https://github.com/ximalu/FlClash](https://github.com/ximalu/FlClash)**。

## 关联仓库

- 上游：https://github.com/chen08209/FlClash
- **代码（fork）：https://github.com/ximalu/FlClash**（M0/M1 工作副本，活跃开发）
- fork：https://github.com/ximalu/ZeroTierOne （M1 准备，ZeroTier Core C++）
- 依赖：sagernet/sing-tun、metacubex/mihomo（vendored 于 FlClash/Clash.Meta）、sagernet/gvisor

## 许可证注意

- ZeroTier Core：BSL 1.1（源码可得；个人/内部/学术免费；每版本 4 年后转 Apache 2.0）
- FlClash：GPL-3.0（fork 需保持 GPL）
