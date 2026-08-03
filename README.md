# EasyTier（ws/wss 打洞增强版）

> 本仓库是 [EasyTier](https://github.com/EasyTier/EasyTier) 的增强 fork。EasyTier 是一个由 Rust 和 Tokio 驱动的简单、安全、去中心化的异地组网方案；本 fork 在其之上重点增强了 **WebSocket（ws/wss）TCP 打洞** 能力，改善复杂 NAT 环境下的 P2P 直连成功率，并定期与上游保持同步。

<p align="center">
<img src="assets/config-page.png" width="300" alt="配置页面">
<img src="assets/running-page.png" width="300" alt="运行页面">
</p>

---

## 这个仓库做了什么

EasyTier 原版支持 UDP / IPv6 NAT 穿透，但在部分受限网络（NAT4-NAT4、UDP 被限速或拦截）下，TCP 打洞是更可靠的直连手段。原版仅支持裸 TCP 打洞，本 fork 将其扩展为完整的 **ws/wss 打洞链路**：

1. **打洞传输方式自动协商**：发起打洞前，通过 RPC 探测本端与对端是否监听 ws/wss，自动选择「裸 TCP」或「打洞后升级为 WebSocket」两种传输方式之一（`TcpHolePunchTunnel`）。
2. **STUN 动态端口映射**：打洞成功后，本地监听端口通过 STUN 换取公网映射地址（`TcpPortMappingLease`），并以「动态映射监听器」的形式注册进节点状态，供对端回连。
3. **映射地址交换 RPC**：新增 `WsHolePunchRpc.ExchangeMappedAddr`，让打洞双方交换映射后的可达地址。
4. **TCP 连接升级为 WS 隧道**：监听端（`WsTunnelListener`）通过 peek 首字节识别 TLS ClientHello 与 HTTP Upgrade，将打洞建立的裸 TCP 流无缝升级为 ws/wss 隧道；非法连接（无效 HTTP 升级、错误协议）会被显式重置，避免连接悬挂。
5. **完善的诊断能力**：全链路带 `ws_hole_punch` 前缀的 tracing 日志，便于排查打洞过程。

对使用者而言这一切是透明的：只要双端运行本增强版，打洞协商会自动进行；与原版节点互通时自动回退到原版兼容的打洞逻辑。

### 主要改动文件

| 模块 | 文件 | 作用 |
|---|---|---|
| 打洞核心 | `easytier/src/connector/tcp_hole_punch.rs` | 传输方式选择、ws 映射地址交换、打洞隧道构建 |
| STUN 映射 | `easytier/src/common/stun.rs` | TCP 端口映射租约（`TcpPortMappingLease`） |
| 隧道层 | `easytier/src/tunnel/websocket.rs` | TCP 流协议识别与 WS 隧道升级 |
| 监听器 | `easytier/src/instance/listeners.rs`、`easytier/src/common/global_ctx.rs` | 动态映射监听器的注册 / 更新 / 撤销 |
| RPC 协议 | `easytier/src/proto/peer_rpc.proto` | `WsHolePunchRpc` 服务定义 |

---

## EasyTier 是什么

EasyTier 是一个去中心化的虚拟组网工具，节点之间平等、独立，无需中心化服务器：

- 🔒 **去中心化**：节点平等且独立，无需中心化服务
- 🌍 **跨平台**：支持 Win / macOS / Linux / FreeBSD / Android 与 X86 / ARM / MIPS 架构
- 🔐 **安全**：AES-GCM 或 WireGuard 加密，防止中间人攻击
- 🔌 **高效 NAT 穿透**：UDP / IPv6 / TCP 穿透，可在 NAT4-NAT4 网络中工作；本 fork 进一步增强 ws/wss 打洞
- 🌐 **子网代理**：节点可以共享子网供其他节点访问
- 🔄 **智能路由**：延迟优先和自动路由选择
- ⚡ **高性能**：全链路零拷贝，支持 TCP / UDP / WSS / WG 协议

📚 上游完整文档：[easytier.cn](https://easytier.cn) ｜ 🖥️ Web 控制台：[easytier.cn/web](https://easytier.cn/web) ｜ 📝 上游 Releases：[EasyTier/EasyTier Releases](https://github.com/EasyTier/EasyTier/releases)

---

## 快速开始

### 使用共享节点组网

没有公网 IP 时，可借助社区共享节点进入网络，节点会自动尝试 NAT 穿透建立 P2P 连接；P2P 失败时数据经共享节点中继。同一网络的节点需使用相同的 `--network-name` 与 `--network-secret`：

```bash
# 节点 A（以管理员权限运行）
sudo easytier-core -d --network-name abc --network-secret abc -p tcp://<共享节点IP>:11010

# 节点 B
sudo easytier-core -d --network-name abc --network-secret abc -p tcp://<共享节点IP>:11010
```

验证网络状态与连通性：

```bash
easytier-cli peer     # 查看已连接的对等节点
easytier-cli route    # 查看路由信息
ping 10.126.126.2     # 测试虚拟网段连通性
```

### 去中心化直连组网

无需任何公共节点，两台设备能互相通信即可组网：

```bash
# 节点 A：第一个节点，直接启动
sudo easytier-core -i 10.144.144.1

# 节点 B：连接节点 A 的公网地址
sudo easytier-core -i 10.144.144.2 -p udp://<节点A公网IP>:11010
```

默认监听端口：TCP 11010、UDP 11010、WebSocket 11011、WebSocket SSL 11012、WireGuard 11013。

> 提示：若无法 ping 通，通常是防火墙拦截了入站流量，请放行对应端口。

---

## 从源码构建

构建前需安装 Rust 工具链；GUI 构建还需要 Node.js 与 pnpm。

```bash
# 构建核心程序（产物：target/debug/easytier-core）
cargo build -p easytier --bin easytier-core

# 构建 GUI（先在仓库根目录执行 pnpm -r install 安装前端依赖）
pnpm --dir easytier-gui tauri build --debug --no-bundle
```

Windows 下建议显式指定 target（如 `--target x86_64-pc-windows-msvc`），避免与隐式 host target 产生两套编译缓存。其他平台的构建方式请参考上游仓库。

---

## 与上游的关系

- 上游仓库：[EasyTier/EasyTier](https://github.com/EasyTier/EasyTier)，本 fork 定期合并 `upstream/main`。
- 本 fork 的净改动聚焦于 ws/wss 打洞增强（见上文「这个仓库做了什么」）；上游的通用问题请优先在上游 issue 中反馈。

## 许可证

EasyTier 在 [LGPL-3.0](https://github.com/EasyTier/EasyTier/blob/main/LICENSE) 许可下发布，本 fork 的增强代码遵循同一许可证。
