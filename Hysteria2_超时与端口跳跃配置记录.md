# Hysteria 2 超时设置与端口跳跃配置记录

---

## 一、 背景与优化目标

在基于公网 UDP/QUIC 协议的代理传输（如 Hysteria 2）中，跨国网络链路常面临两大核心痛点：
1. **丢包抖动与连接僵死**：公网 UDP 偶发丢包或 NAT 映射老化可能导致单边断连；若超时时间设置过长，死连接（Dead Connection）将长时间占位导致断流假死；若过短又易因轻微波动频繁中断重连。
2. **运营商 QoS 限速与端口阻断**：部分运营商对长时间固定单端口（如 443）的大流量 UDP 进行限速、丢包或封锁。

本次改动主要围绕 **空闲超时优化** 与 **端口跳跃（Port Hopping）** 展开，确保：
* 服务端与客户端能及时识别并清理异常失效连接，断开后**尽快重新握手恢复**，避免客户端处于不可用的僵死状态；
* 传输层流量在宽泛端口区间内动态轮替，有效**绕过 ISP 对单端口 UDP 的针对性策略**。

---

## 二、 服务端配置（日本 IPv6 VPS: `<YOUR_SERVER_IPV6>`）

### 1. QUIC 空闲超时设置 (`maxIdleTimeout`)
编辑 `/etc/hysteria/config.yaml`，在 `quic` 块中配置：

```yaml
listen: :443

tls:
  cert: /etc/hysteria/server.crt
  key: /etc/hysteria/server.key

auth:
  type: password
  password: <PASSWORD>

masquerade:
  type: proxy
  proxy:
    url: https://bing.com/
    rewriteHost: true

quic:
  maxIdleTimeout: 60s
```

#### 作用与机制：
- 根据 RFC 9000 QUIC 规范，连接生效超时为客户端与服务端的协商最小值：
  $$\text{effective\_timeout} = \min(\text{client\_timeout}, \text{server\_timeout})$$
- 默认情况下客户端会每隔 10s 发送 PING 心跳保活。
- 设置 `maxIdleTimeout: 60s` 为服务端设定了 60 秒的断连清理上限：当发生运营商断网、光猫重拨号或严重丢包导致心跳中断时，服务端在 60 秒内准时释放服务端会话与相关系统资源，防止残留僵死会话阻碍客户端建立新会话。

---

### 2. 端口跳跃服务端实现（iptables REDIRECT 方案）

Hysteria 2 服务端采用安全沙盒化部署，服务进程以非 root 身份（`User=hysteria`）运行。由于系统底层的 `iptables-nft` 要求特权（root）身份，因此采用**系统级 PREROUTING REDIRECT 转发**是生产环境最稳健、推荐的方案：

#### 配置步骤：
```bash
# 1. 添加 IPv6 / IPv4 iptables 转发规则（将 20000-50000 UDP 重定向至 443）
iptables -t nat -A PREROUTING -p udp --dport 20000:50000 -j REDIRECT --to-ports 443
ip6tables -t nat -A PREROUTING -p udp --dport 20000:50000 -j REDIRECT --to-ports 443

# 2. 持久化规则保存（重启不丢失）
netfilter-persistent save
```

#### 方案优势：
* **权限安全隔离**：Hysteria 守护进程继续保持非特权运行，无提权风险；
* **平滑兼容无断网**：底层 443 端口依然开放，新旧客户端均可正常握手；
* **高性能透明转发**：内核 Netfilter 原生处理，几乎无额外资源消耗。

---

## 三、 客户端与订阅配置（OpenClash / Mihomo Meta）

### 1. 节点订阅定义 (`proxies.yaml`)
更新订阅节点配置，引入多端口区间与随机跳跃间隔：

```yaml
proxies:
  - name: "[HY2] 🇯🇵 日本-IPv6-Hysteria2"
    type: hysteria2
    server: <YOUR_SERVER_IPV6>
    ports: 20000-50000
    hop-interval: "30-60"
    password: <PASSWORD>
    sni: bing.com
    skip-cert-verify: true
```

### 2. 核心字段说明：
- **`ports: 20000-50000`**：定义客户端可以在 20000 至 50000 的端口池内随机选择端口发起连接。使用 `ports` 字段时，原单端口 `port: 443` 需省略或由该字段接管。
- **`hop-interval: "30-60"`**：配置端口切换周期。Mihomo Meta 核心支持区间范围语法，客户端将在 30 秒至 60 秒之间随机产生跳跃间隔，进一步消除固定时间周期的流量指纹特征。

### 3. 同步至订阅分发
通过 Cloudflare Workers KV 同步更新，OpenClash 订阅自动刷新拉取：
```bash
pnpm wrangler kv key put --binding=PROXIES_KV "proxies_yaml" --path=./proxies.yaml --remote
```

---

## 四、 实测抓包与行为验证

### 1. 端口跳跃连通性与时间轴实测
在客户端持续请求的同时，在 VPS 端运行 `tcpdump` 捕获来自 OpenClash 路由器的 3,160 个 UDP 数据包，端口分布如下：

| 阶段 | 目标端口 | 是否符合 [20000-50000] | 持续时间 | 捕获包数 | 状态 |
| :---: | :---: | :---: | :---: | :---: | :---: |
| **Hop 1** | **`33028`** | 是 | 23.68 秒 | 2,429 | 正常转发 (HTTP 200) |
| **Hop 2** | **`41700`** | 是 | 24.70 秒 | 160 | 正常转发 (HTTP 200) |
| **Hop 3** | **`41995`** | 是 | 41.02 秒 | 318 | 正常转发 (HTTP 200) |
| **Hop 4** | **`43460`** | 是 | 25.21 秒 | 253 | 正常转发 (HTTP 200) |

- **跳跃平滑性**：端口跳跃由底层 QUIC 协议透明完成，应用层连接（如网页加载、文件传输）完全不中断；
- **防火墙命中**：`ip6tables -t nat -L PREROUTING -v -n` 规则命中计数持续递增，验证通过。

### 2. 服务端 60 秒超时机制实测
- **测试方法**：客户端与服务端建立 QUIC 会话后，向客户端进程发送 `kill -STOP` 强制冻结其网络心跳包。
- **观测结果**：
  - `09:54:17 UTC`：客户端连接建立；
  - `09:54:47 UTC`（30s）：无心跳包，服务端保持观察；
  - `09:55:16 UTC`（精确 59-60s 时）：服务端触发超时清理并记录日志：
    ```text
    INFO client disconnected {"addr": "...", "id": "user", "error": "accepting stream failed: timeout: no recent network activity"}
    ```
- **结论**：服务端 `maxIdleTimeout: 60s` 严格生效，无心跳连接在 60 秒内被安全拆除。

---

## 五、 常用运维与排查命令

### 1. VPS 端检查
```bash
# 查看 Hysteria 服务端日志与连接/断连事件
journalctl -u hysteria-server -f

# 检查端口跳跃 iptables 规则命中包数
ip6tables -t nat -L PREROUTING -v -n --line-numbers

# 查看当前活跃的 UDP 连接跟踪
conntrack -L -p udp
```

### 2. OpenWrt / 客户端检查
```bash
# 查看 OpenClash 实时分流日志
tail -f /tmp/openclash.log

# 查询 Mihomo Meta 核心对节点的健康检查状态与延迟
curl -s -H "Authorization: Bearer <SECRET>" http://127.0.0.1:9090/providers/proxies | jq '.providers.Provider_xxxxxx.proxies[0]'
```
