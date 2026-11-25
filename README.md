# FlClashTier

**FlClash + ZeroTier Core 双网络出口** — 在 FlClash（Android VpnService + mihomo 内核）中嵌入 ZeroTier Core，用**一个** VPN 槽位同时提供私网访问与代理出口：

- **ZeroTier** 管私网（ZT managed routes）
- **mihomo** 管代理 / 公网（DIRECT / PROXY）

本仓库是**项目文档 + 安装包发布页**；**源码**在 [ximalu/FlClash](https://github.com/ximalu/FlClash)（上游 [chen08209/FlClash](https://github.com/chen08209/FlClash) 的 fork）。

## 下载 / Install

APK 统一发布在本仓库的 [Releases](https://github.com/ximalu/FlClashTier/releases)。

- 平台：Android **arm64-v8a**（只有 arm64 有 ZeroTier 静态库）
- 包名：`com.flclashtier`
- 升级方式：**覆盖安装，不要卸载** —— 卸载会清掉 ZeroTier identity，节点地址改变，需要在 Central 重新授权
- 签名：正式 release key，证书 SHA-256 `b719cbf2…70370f`
- 资源名形如 `FlClashTier-<version>-arm64-release.apk`，每个 release 正文附 `sha256`

首次使用顺序：① 配置文件页导入 Clash 配置 → ② 设置 → ZeroTier 填 Network ID → ③ 首页开启 VPN → ④（只需要一次）在 ZeroTier Central 授权这个节点。identity 会持久化，之后升级、重启都不用重新授权。

## 架构

```
Android VpnService
      │  real TUN fd
      ▼
┌───────────────┐
│  Flow Router  │  ← 按 ZeroTier Managed Routes 分流
└───┬───────┬───┘
    │       │
    ▼       ▼
 ZT core  mihomo（socketpair → sing_tun）
```

- 分流依据只有一条：ZeroTier core 下发的 managed routes（不硬编码网段）
- ZeroTier 的物理 UDP socket 绑定当前物理网络，换网时重建并重置对端路径（路径级自愈）
- 诊断：分层计数、sendto errno 直方图、`zt-diag.log` / `app-diag.log`

设计过程文档（开发纪要、里程碑清单）只在本地保存，不随仓库发布；
源码与提交历史见 [ximalu/FlClash](https://github.com/ximalu/FlClash)。

## 已知限制

- 仅 arm64
- 需要系统 VPN 槽位（所有 VpnService 应用都一样）
- 对端列表不显示「直连 / 中继」：ZeroTier C API 给不出可靠证据
- 冷启动后第一次换网，ZT 通路可能需要约 90 秒恢复；后续换网通常无感

## 许可与致谢

- 本项目遵循 **GPL-3.0**（见 `LICENSE`），与上游一致
- 基于 [FlClash](https://github.com/chen08209/FlClash) 二次开发（GPL-3.0）
- 代理内核：[mihomo / Clash.Meta](https://github.com/MetaCubeX/mihomo)（GPL-3.0）
- 虚拟网络内核：[ZeroTierOne](https://github.com/zerotier/ZeroTierOne)，BSL 1.1（源码可得；个人/内部/学术免费；每版本 4 年后转 Apache 2.0）
- 依赖：sagernet/sing-tun、sagernet/gvisor
