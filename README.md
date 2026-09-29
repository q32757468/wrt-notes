# wrt-notes

> 个人软路由网络优化、节点部署与云服务运维实战笔记。专注于家庭局域网无感分流、原生 IPv6 穿透加速、轻量节点运维及网络诊断排错。

---

## 📖 目录索引

| 文档名称 | 分类 | 核心内容与知识点 |
| :--- | :--- | :--- |
| **[OpenClash 旁路由部署与配置记录](./OpenClash_旁路由部署与配置记录.md)** | 路由与分流 | • 单臂旁路由（Bypass Gateway）网络拓扑规划<br>• OpenWrt 基础配置（静态 IP / 关闭 DHCP）<br>• 防火墙规则与 NAT 伪装优化（保留局域网源 IP）<br>• OpenClash 内核选择、Fake-IP 与防 DNS 泄露<br>• 常见问题排查与避坑指南 |
| **[IPv6 VLESS 节点部署与配置记录](./IPv6_VLESS_部署与配置记录.md)** | 节点与穿透 | • 纯 IPv6 VPS 极低成本部署实战<br>• Xray-core (VLESS + Reality + Vision) 服务端配置<br>• 小米等家用主路由 IPv6 原生穿透与公网开通<br>• OpenClash 接入纯 IPv6 节点实现端到端单跳直连 |
| **[云服务器与运维平台汇总](./云服务器与运维平台汇总.md)** | 资源与运维 | • 优质线路与超低价 VPS 推荐（SadIDC / PoloCloud / ByteVirt）<br>• 纯 IPv6 机器特性及 WARP 出口补齐技巧<br>• 在线网络排错平台对比（ITDog 与 阿里云拨测 BOCE）<br>• 路由追踪、丢包排查与节点诊断最佳实践 |

---

## 🌐 核心网络拓扑

仓库中记录的典型家庭双栈/单臂旁路由拓扑结构如下：

```text
       [ 局域网终端 (PC / 手机 / 智能家居) ]
                        │
                        │ 1. DHCP 网关 & DNS 指向旁路由
                        ▼
       [ ImmortalWrt / OpenWrt 旁路由 ]
             (OpenClash / Fake-IP 模式)
                        │
         ┌──────────────┴──────────────┐
         │ (国内直连流量)               │ (海外代理流量)
         │ 保留客户端源 IP              │ 原生公网 IPv6 单跳直连
         ▼                             ▼
   [ 家用主路由 (LAN) ]         [ 新加坡/海外纯 IPv6 VPS ]
         │                             │ (Xray VLESS-Reality)
         │ PPPoE / 光猫出站             │
         ▼                             ▼
   [ 运营商骨干网 ]             [ 全球互联网 / 外部资源 ]
```

