# IPv6 VLESS-Reality 节点部署与 OpenClash 配置记录

---

## 一、 项目背景与网络拓扑

* **部署目标**：将新加坡纯 IPv6 VPS 部署为具备高隐蔽性的 **VLESS + Reality + Vision** 代理节点，并接入本地 **ImmortalWrt 旁路由（OpenClash）** 中，实现局域网客户端单跳低延迟直连。
* **网络拓扑**：
  ```
  [客户端设备 (PC/手机)]
         │ (纯 IPv4 Fake-IP 稳定分流，防 DNS 泄露)
         ▼
  [ImmortalWrt 旁路由 (192.168.31.63)] ── (OpenClash / Mihomo Meta 核心)
         │ (原生公网 IPv6 单跳直连，无需中间代理)
         ▼
  [小米主路由 (192.168.31.1)] ➔ [光猫 / 中国电信骨干网]
         │ (公网 IPv6 流量直达)
         ▼
  [新加坡 VPS (2407:d840:21:114:514::86:443)] ── (Xray-core 服务端)
         │
         ▼
  [全球互联网络]
  ```

---

## 二、 远程服务器（IPv6 VPS）改动记录

* **服务器 IP**：`2407:d840:21:114:514::86`
* **操作系统**：Debian GNU/Linux 11 (Bullseye) x86_64
* **系统与服务部署**：
  1. **软件源维护**：
     * 针对 Debian 11 归档源迁移导致的更新失败，在 `/etc/apt/sources.list` 中移除了失效的 backports 源，恢复基础包安装环境。
  2. **核心安装**：
     * 安装了最新版本 **Xray-core**（`v26.3.27`）至 `/usr/local/bin/xray`。
     * 配置并启用了 Systemd 开机自启服务 `/etc/systemd/system/xray.service`。
  3. **服务配置文件**（`/usr/local/etc/xray/config.json`）：
     * **监听协议与端口**：VLESS 协议，监听 `[::]:443`。
     * **流控与安全**：`xtls-rprx-vision`，Security 模式为 `reality`。
     * **伪装目标 (SNI)**：`addons.mozilla.org:443`（ServerNames: `addons.mozilla.org`）。
     * **流量探测 (Sniffing)**：开启 `http`、`tls`、`quic` 探测并重定向目标。
  4. **凭证安全性说明**：
     * 节点所需的 `UUID`、`Reality 私钥`、`Reality 公钥` 及 `Short ID` 均已在服务端随机生成，并加密保存在 VPS 的 `/usr/local/etc/xray/client_info.json` 中，此处做脱敏处理，不记录明文。

---

## 三、 小米主路由（192.168.31.1）人工操作方式

通过 Web 管理界面按以下步骤操作，为局域网激活 IPv6 公网能力：

1. 浏览器登录小米路由器后台：`http://192.168.31.1`；
2. 依次点击：**「常用设置」** ➔ **「上网设置」**；
3. 向下滚动至 **「IPv6 网络设置」** 区域：
   * 将开关切换为 **开启**；
   * 上网方式选择 **「Auto 模式」**；
4. 点击 **「保存」** 生效。

---

## 四、 ImmortalWrt 旁路由（192.168.31.63）改动与人工操作方式

### 1. 旁路由网络接口配置（LuCI 界面操作）
为使旁路由能接收主路由下发的 IPv6 前缀与默认网关：

1. 登录 ImmortalWrt 后台，进入 **「网络」** ➔ **「接口」**；
2. 点击 **「添加新接口」**：
   * **名称**：`wan6`
   * **协议**：选择 **DHCPv6 客户端**
   * **设备**：选择局域网桥接设备 **`br-lan`**
3. 点击 **「创建接口」** 并在高级设置中确认常规请求配置；
4. 点击 **「保存并应用」**，接口将自动获取到电信公网 IPv6 地址与默认网关。

---

### 2. OpenClash 节点配置文件更新
在 `/etc/openclash/config/vg+custom_ipv6.yaml` 中完成了以下改动：

1. **节点定义新增**（脱敏展示）：
   ```yaml
   - name: "[VLESS] 🇸🇬 新加坡-IPv6-Reality"
     server: 2407:d840:21:114:514::86
     port: 443
     type: vless
     uuid: <UUID_SECRET>
     tls: true
     flow: xtls-rprx-vision
     skip-cert-verify: false
     servername: addons.mozilla.org
     client-fingerprint: chrome
     reality-opts:
       public-key: <PUBLIC_KEY_SECRET>
       short-id: <SHORT_ID_SECRET>
     udp: true
   ```
2. **策略组模式调整**：
   * 将 **`🚀 Basic`** 分组从原有的自动测速（`url-test`）调整为**手动选择（`type: select`）**：
     ```yaml
     - name: 🚀 Basic
       type: select
       proxies:
         - "[VLESS] juyiting"
         - "[VLESS] 🇺🇸 美西洛杉矶-VLESS"
         - "[VLESS] 🇸🇬 新加坡-IPv6-Reality"
     ```
   * 同时将该节点收录至 `🚀 手动切换` 与 `🇸🇬 狮城节点` 列表中。
3. **配置头声明**：
   在配置头部显式声明了 `ipv6: true`。

---

### 3. OpenClash 运行参数设置（界面操作）
在 OpenClash Web 界面中调整运行参数，以使内核支持 IPv6 直连并保持旁路由稳定分流：

1. 进入 OpenClash 页面，点击 **「插件设置」** ➔ **「常规设置」**；
2. 切换到 **「IPv6 设置」** 选项卡：
   * 勾选 **「IPv6 流量代理」**（使内核支持 IPv6 出站拨号）；
   * **「IPv6 代理模式」**：选择 **「TProxy 模式」**（TCP + UDP 内核级透明代理，性能最优）；
   * **「允许 IPv6 类型 DNS 解析」**：**保持不勾选**（确保客户端 DNS 走纯 IPv4 Fake-IP，避免旁路由环境下分流异常或绕路）；
3. 点击底部 **「保存配置」** 并 **「应用配置」** 即可。
