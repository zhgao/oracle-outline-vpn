# Oracle 永久免费 + Outline 自建 VPN：注册成功后的搭建指南

> 适用：Oracle Cloud 账号已注册成功，Home Region 为 **Japan East (Tokyo)**。
> 费用：**$0**，永久免费额度内（A1 实例 2 OCPU / 12GB + 每月 10TB 出站流量）。
> 全程约 20 分钟，其中 George 自己动手约 5 分钟，剩下由 Muse 在 SSH 里完成。

## 总流程（5 步）

| 步骤 | 谁做 | 做什么 |
|---|---|---|
| 1. 建实例 | George | 控制台建一台东京的免费 A1 实例 |
| 2. 发 IP + 私钥 | George | 把实例公网 IP 和 SSH 私钥发给 Muse（只用一次） |
| 3. 装 Outline Server | Muse | SSH 上去一键安装，拿到两个端口 |
| 4. 放行端口 | George | VCN 安全列表加两条入站规则 |
| 5. 连上验证 | George | 手机/电脑装 Outline Client，粘贴密钥，确认 IP 在东京 |

---

## 第 1 步：建实例（George，控制台操作）

1. 打开 [cloud.oracle.com](https://cloud.oracle.com)，登录账号。
2. 左上角菜单 → **Compute（计算）→ Instances（实例）→ Create instance（创建实例）**。
3. 按下面填（没提到的保持默认）：
   - **Name（名称）**：`outline-vpn`（随便起，好认就行）
   - **Placement（放置）**：Availability Domain 任选一个（如果提示容量不足，换另一个 AD 再试）
   - **Image（镜像）**：选 **Oracle Linux 9** 或 **Ubuntu 24.04**（两个都行，推荐 Oracle Linux 9）
   - **Shape（机型）**：点 Change shape → 系列选 **Ampere** → 选 **VM.Standard.A1.Flex**，改成 **2 OCPU / 12 GB 内存**（这就是永久免费额度，超出要收费，别多加）
   - **Networking（网络）**：用自动建好的 VCN 和子网就行；**勾选 Assign a public IPv4 address（分配公网 IPv4）**
   - **Add SSH keys（添加 SSH 密钥）**：选 **Generate a key pair（生成密钥对）**，点 **Save private key**，把下载的 `.key` 私钥文件保存好（第 2 步要用）
4. 点 **Create**，等实例状态变成 **Running**，记下它的 **Public IP（公网 IP）**。

> **建议多做一步：预留固定 IP。** 实例默认的公网 IP 是临时的，重启可能换 IP，换了密钥就失效。去 Networking → IP Management → Reserved Public IPs → Reserve，选同一个区域，建好后挂到这台实例上。挂在运行中实例上的预留 IP 是免费的。

> **如果建实例时提示容量不足（Out of capacity）：** 这是东京区 A1 免费机型常见情况，不是你的问题。换一个 Availability Domain 试试，或隔几小时再试，不用升级成付费机型。

## 第 2 步：把公网 IP 和私钥发给 Muse（George）

把这两样发给 Muse：

1. 实例的 **Public IP**（第 1 步记下的）
2. SSH 私钥文件的内容（第 1 步下载的 `.key` 文件，用文本打开，全文贴过来）

私钥只用这一次：Muse 用它连上服务器装 Outline，装完即忘，不存档、不写进任何文件。

## 第 3 步：装 Outline Server（Muse，SSH 里执行）

Muse 连上实例后跑一条一键安装命令（自动装 Docker）：

```bash
sudo bash -c "$(wget -qO- https://raw.githubusercontent.com/Jigsaw-Code/outline-server/master/src/server_manager/install_scripts/install_server.sh)"
```

安装输出里有三样关键东西：

- **TCP 管理端口**（如 12345，Outline Manager / API 用）
- **UDP 数据端口**（如 23456，VPN 流量走这个）
- **apiUrl**（带一串随机路径的管理接口地址）

Muse 拿 apiUrl 调接口建一条访问密钥（免装 Outline Manager 桌面端）：

```bash
curl -k -X POST "https://<IP>:<管理端口>/<apiPath>/access-keys"
```

返回的密钥会拼成一条 `ss://` 链接发给 George，客户端粘贴即用。

## 第 4 步：放行端口（George，控制台操作）

Outline 的两个端口是随机生成的，装完才知道，所以这一步放在安装之后：

1. 控制台左上角菜单 → **Networking（网络）→ Virtual Cloud Networks（虚拟云网络）** → 点进实例用的 VCN。
2. 左侧 **Security Lists（安全列表）** → 点进默认的那个 → **Add Ingress Rules（添加入站规则）**。
3. 加两条（Source CIDR 都填 `0.0.0.0/0`）：
   - **TCP**，Destination Port Range 填管理端口（如 `12345`）
   - **UDP**，Destination Port Range 填数据端口（如 `23456`）
4. 保存。SSH 的 22 端口默认已放行，不用动。

## 第 5 步：连上验证（George）

1. 手机/电脑装 **Outline Client**：官网 [getoutline.org](https://getoutline.org)。
2. 打开客户端 → 粘贴 Muse 给你的 `ss://` 链接 → 点连接。
3. 验证：打开浏览器搜“我的 IP”，确认 IP 地址在**日本东京**，再打开一个平时打不开的网站，能打开就是通了。

---

## 日常使用

- **多设备**：一台手机、电脑、平板可以共用同一条 `ss://` 密钥，也可以让 Muse 多建几条，一设备一条。
- **给家人用**：别转发你自己的密钥，让 Muse 单独建一条 key，单独给谁都能随时删。
- **密钥 = 这台服务器的全部访问权**：别发到群里、别贴到公开地方。
- **删密钥**：某条 key 不想用了，告诉 Muse 一声，从服务端删掉就行。

## 防回收（重要）

Oracle 有一条规则：免费实例连续 **7 天三项利用率都低于 20%**（CPU、内存、网络）可能被回收。正常当 VPN 用（每天上网、刷视频）不会触发。别建完长期关机不用，也别放着一个月不开机。

## 费用边界（$0 底线）

- 只在免费额度内：A1 实例 **2 OCPU / 12GB**、每月出站流量 **10TB 以内**，都是 $0。
- 别点的东西：付费机型、超出额度的磁盘、按量计费的服务。升级为付费账号前必须先确认。
- Oracle 不会在免费额度内偷扣钱，但绑的那张卡要留着做验证（只验证，不扣费）。

## 常见问题

| 现象 | 原因 / 办法 |
|---|---|
| 建实例提示 Out of capacity | 东京区 A1 免费机型暂时没容量，换 Availability Domain，或隔几小时再试 |
| Muse 连不上 SSH | 检查 VCN 安全列表 22 端口是否放行；确认发来的 IP 和私钥没贴错 |
| 客户端连不上 / 一直转圈 | 先看 VCN 安全列表：UDP 数据端口放行了没；再看实例是不是 Running 状态 |
| 重启后密钥失效了 | 临时公网 IP 变了，按第 1 步末尾预留一个固定 IP 再挂上 |
| 网速慢 | 东京机房到大连延迟本身很低；先确认连的是这台东京实例（查 IP 归属），再看是不是高峰期 |

## 分工一览

- **George**：建实例、发 IP 和私钥、放行两个端口、装客户端连接。
- **Muse**：装 Outline Server、建密钥、拼 `ss://` 链接、日常加 key / 删 key、排障。

## 备用路径

如果 Oracle 这条最终走不通，备用是 **Google Cloud 90 天 $300 试用**（e2-micro，俄勒冈区），同一套 Outline 安装流程通用。完整步骤见 [GOOGLE.md](GOOGLE.md)。
