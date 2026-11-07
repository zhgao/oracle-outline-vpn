# Google Cloud 试用 + Outline 自建 VPN：搭建指南（Oracle 的备用路径）

> 适用：Oracle 走不通时先用这条顶上。Google Cloud 新账号送 **$300，有效期 90 天**，到期未升级资源自动暂停，不会偷扣钱。
> 费用：**90 天内实际 $0**（所有花费都从 $300 里扣）；90 天后要么停服，要么升级按量付费，升级前必须先确认。
>
> **注意（2026-11-07 实测 George 的账号）：** 他的 GCP 账号（项目 schema-ai）试用期已经过了，$300 额度没了。所以这条路现在没有 $300 垫底：想继续用要先点 Upgrade 升成正式账号（升级本身不收费），免费档 e2-micro 还在，但每月只有 1GB 免费出站流量，VPN 用量超出部分到中国 $0.19/GiB，直接扣卡。做 $0 VPN 的话，Oracle 仍是首选；这条只适合愿意接受小额付费的情况。
> 全程约 20 分钟，其中 George 自己动手约 10 分钟，剩下由 Muse 在 SSH 里完成。

## 和 Oracle 那条的区别

|  | Oracle（主路径） | Google Cloud（这条） |
|---|---|---|
| 费用 | 永久 $0 | $300 额度，90 天 |
| 机房 | 东京，延迟低 | 俄勒冈，延迟 ~200ms，刷网页看视频够用 |
| 免费出站流量 | 每月 10TB | 每月只含 1GB，超出到中国 $0.19/GiB（100GB/月 ≈ $19） |
| 到期 | 无 | 90 天后停服 / 付费续 / 切回 Oracle |

## 总流程（5 步）

| 步骤 | 谁做 | 做什么 |
|---|---|---|
| 1. 注册 + 建项目 | George | cloud.google.com 免费试用注册，绑卡（预授权 $0–1，会退回） |
| 2. 建实例 | George | 俄勒冈区建一台免费 e2-micro，预留静态 IP |
| 3. 发 IP + SSH 密钥 | George | 把实例公网 IP 和 SSH 密钥发给 Muse（只用一次） |
| 4. 装 Outline + 放行端口 | Muse 装，George 放行 | 同 Oracle 那套流程 |
| 5. 连上验证 | George | 装 Outline Client，粘贴密钥，确认能翻墙 |

---

## 第 1 步：注册试用 + 建项目（George）

1. 打开 [cloud.google.com](https://cloud.google.com)，点 **Get started for free**。
2. 用 Gmail（zhgao1981@gmail.com）注册，按提示绑一张信用卡。绑卡时会有 **$0–1 的预授权，会退回**，不是扣费。
3. 注册完建一个项目（名字随便，如 `outline-vpn`）。

> **只注册，不升级。** 试用账号默认就是免费试用状态，90 天内不用点任何“升级”按钮。升级为付费账号前必须先确认。

## 第 2 步：建实例（George，控制台操作）

1. 控制台左上角菜单 → **Compute Engine → VM instances → Create instance**。
2. 按下面填（没提到的保持默认）：
   - **Name（名称）**：`outline-vpn`
   - **Region（区域）**：**us-west1（俄勒冈，离中国最近）**，Zone 选 **us-west1-b**
   - **Machine type（机型）**：**e2-micro**（1 核 1G，共享核心）
   - **Boot disk（启动磁盘）**：系统选 **Debian 12**，磁盘类型标准永久性磁盘，大小 **30GB**（免费档上限）
   - **Firewall（防火墙）**：勾选 Allow HTTP / Allow HTTPS 无妨（Outline 不靠这两个端口）
   - **SSH 密钥**：展开 Advanced options → Security → SSH keys，点 Add item，粘贴你的 SSH 公钥。没有现成密钥对的话，让 Muse 生成一对，私钥发你保存，公钥贴进去。
3. 点 **Create**，等实例状态变成 Running，记下它的 **External IP（外网 IP）**。

> **免费档只认三个区。** e2-micro 永久免费只限 **us-west1（俄勒冈）、us-central1（爱荷华）、us-east1（南卡）**，其他区建的 e2-micro 照常计费。认准 us-west1-b，别选错。

> **预留静态 IP（建议做）。** 实例默认的外网 IP 是临时的，重启可能换 IP，换了密钥就失效。去 VPC network → IP addresses → Reserve static address，选 us-west1，建好后挂到这台实例上。挂在运行中实例上的静态 IP 是免费的。

## 第 3 步：把外网 IP 发给 Muse（George）

把实例的 **External IP** 发给 Muse，Muse 用第 2 步的 SSH 密钥连上去装 Outline。密钥只用这一次，装完即忘，不存档。

## 第 4 步：装 Outline Server + 放行端口

### Muse 的步骤（SSH 里执行）

同 Oracle 那条，一键安装（自动装 Docker）：

```bash
sudo bash -c "$(wget -qO- https://raw.githubusercontent.com/Jigsaw-Code/outline-server/master/src/server_manager/install_scripts/install_server.sh)"
```

安装输出里有三样关键东西：**TCP 管理端口**、**UDP 数据端口**、**apiUrl**。Muse 拿 apiUrl 建访问密钥：

```bash
curl -k -X POST "https://<IP>:<管理端口>/<apiPath>/access-keys"
```

返回的密钥拼成 `ss://` 链接发给 George。

### George 的步骤（放行端口）

Outline 的两个端口是随机生成的，装完才知道，所以防火墙规则放在安装之后加：

1. 控制台 → **VPC network → Firewall → Create firewall rule**。
2. 名称填 `outline-vpn`，Targets 选 All instances in the network，Source IPv4 ranges 填 `0.0.0.0/0`。
3. Protocols and ports 填两条：`tcp:<管理端口>` 和 `udp:<数据端口>`（如 `tcp:12345`、`udp:23456`）。
4. 点 Create。SSH 的 22 端口默认已放行，不用动。

## 第 5 步：连上验证（George）

1. 手机/电脑装 **Outline Client**：官网 [getoutline.org](https://getoutline.org)。
2. 打开客户端 → 粘贴 Muse 给你的 `ss://` 链接 → 点连接。
3. 验证：打开浏览器搜“我的 IP”，确认 IP 地址在**美国俄勒冈**，再打开一个平时打不开的网站，能打开就是通了。

---

## 防扣费：预算告警（注册完顺手做）

去 **Billing → Budgets & alerts → Create budget**，建一个 **$300** 的预算，告警阈值设 **50%、90%、100%**，告警发到邮箱。这样 90 天内花到一半就知道，不会被动超支。

## 到期安排

- **第 80 天左右**提醒 George 做决定：停服 / 付费续 / 切回 Oracle。届时可建一次性提醒。
- 90 天到期未升级：资源自动暂停，不会偷扣钱。
- 90 天后如果继续用：超出 1GB/月的出站流量到中国 $0.19/GiB，100GB/月 ≈ $19，续用前先确认。

## 费用边界（$0 底线）

- 90 天内：所有花费都从 $300 里扣，实际支出 $0。
- 免费档每月只含 **1GB** 出站流量；刷视频多的话注意用量，预算告警会提醒。
- 别点的东西：付费机型、超出免费档的磁盘、按量计费的服务。升级为付费账号前必须先确认。

## 常见问题

| 现象 | 原因 / 办法 |
|---|---|
| 建了 e2-micro 却在扣费 | 区域选错了：免费只限 us-west1 / us-central1 / us-east1，认准 us-west1-b |
| Muse 连不上 SSH | 确认 SSH 密钥贴对了；GCP 防火墙默认放行 22 端口，一般不用动 |
| 客户端连不上 / 一直转圈 | 先看防火墙规则：UDP 数据端口放行了没；再看实例是不是 Running 状态 |
| 重启后密钥失效了 | 临时外网 IP 变了，按第 2 步末尾预留一个静态 IP 再挂上 |
| 网速比 Oracle 慢 | 正常：俄勒冈到大连延迟 ~200ms，东京要低得多；刷网页看视频够用 |

## 分工一览

- **George**：注册试用、建项目、建实例、发 IP、放行端口、装客户端连接。
- **Muse**：装 Outline Server、建密钥、拼 `ss://` 链接、日常加 key / 删 key、排障、第 80 天提醒你做决定。
