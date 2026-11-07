# Google Cloud 试用 + Outline 自建 VPN：详细搭建指南（2026-11-07 实测版）

> **本文档所有步骤都是 2026-11-07 当天真实走通的**，不是照抄官方文档。照着做，全程约 30 分钟，其中你自己动手约 15 分钟，剩下由 Muse 在 SSH 里完成。
>
> 费用：**90 天内实际 $0**（所有花费都从 $300 试用额度里扣）；90 天后要么停服，要么升级按量付费，升级前必须先确认。
>
> ⚠️ **本文档里的 IP、端口、链接都是示例**（如 `203.0.113.10`、`12345`），换成你自己的真实值。不要把真实的 `ss://` 链接贴到任何公开地方。

## 总流程（6 步）

| 步骤 | 谁做 | 做什么 |
|---|---|---|
| 1. 注册试用 | 你 | cloud.google.com 开通 $300 试用（90 天），绑卡验证 |
| 2. 建实例 | 你 | 俄勒冈区建一台 e2-micro，系统默认 Debian 13 就行 |
| 3. 加 SSH 密钥 | 你 | 控制台 Metadata 里贴公钥（创建页里找不到入口，别找了） |
| 4. 装 Outline | Muse | SSH 上去一键安装，拿到管理端口 + 数据端口 + 密钥 |
| 5. 放行端口 + 固定 IP | 你 | 防火墙建两条规则；外网 IP 从临时转成静态 |
| 6. 连上验证 | 你 | 装 Outline Client，粘贴密钥，确认能翻墙 |

---

## 第 1 步：注册试用（你，约 10 分钟）

### 1.1 先知道三个坑

- **中国大陆不在账单国家列表里。** Google 官方把中国列为受限地区，注册页的国家下拉框里根本没有中国。大陆卡 + 大陆地址注册不了新试用，选香港/美国会因卡和账单地址对不上导致绑卡验证失败。**实测结论：这条路对大陆新用户基本不通**，除非你手头有境外地址和对应的卡。
- **验证金不是 $0–1，是 $50。** Google 官方文档写的是 "$0–$1 预授权"，但 2026-11-07 实测注册时直接扣了 **$50 验证金**。这笔钱是临时的：社区反馈验证完成后 3–4 天自动退回。如果超过一周没退，找 Google Cloud Billing 支持要。绑卡前先确认这张卡能接受一笔 $50 的临时冻结。
- **试用一人一次。** Google 按身份 + 卡 + 手机号认人，用同一个人的信息开新号大概率不给试用，甚至可能判滥用连累主账号。不要为了薅 $300 反复开号。

### 1.2 注册

1. 打开 [cloud.google.com](https://cloud.google.com)，点 **Get started for free**。
2. 用一个没开通过试用的 Google 账号登录（2026-11-07 实测用的是第二个 Google 账号，主账号的试用期已经过了）。
3. 按提示填国家、姓名（拼音，和卡上的名字一致）、绑一张真实的信用卡（Visa/Mastercard，实体卡最稳；预付卡、虚拟卡会被拒）。
4. 提交后等验证：页面会提示试用已开通，控制台顶部显示 **$300.00 credit and 90 days remaining**，并显示到期日（实测 2026-11-07 开通，到期日 2027-02-06）。
5. 注册完会默认建一个项目（名字如 `My First Project`），直接用它就行，不用新建。

> **只注册，不升级。** 试用账号默认就是免费试用状态，90 天内不用点任何"升级"按钮。点 Upgrade 才会变成付费账号，升级前必须先确认。

### 1.3 预算告警（注册完顺手做，2 分钟）

去 **Billing → Budgets & alerts → Create budget**，建一个 **$300** 的预算，告警阈值设 **50%、90%、100%**，告警发到邮箱。这样花到一半就知道，不会被动超支。

---

## 第 2 步：建实例（你，控制台操作，约 5 分钟）

1. 控制台 → **Compute Engine → VM instances → Create instance**（或者顶部搜索框搜 `VM instances`）。
2. 创建页左栏有几个分区：Machine configuration、OS and storage、Data protection、Networking、Observability、Security、Advanced。**系统在 OS and storage 里选，不在机型那一页**。
3. 按下面填（没提到的保持默认）：
   - **Name（名称）**：`outline-vpn`（随便起，好认就行）
   - **Region（区域）**：**us-west1（俄勒冈，离中国最近）**，Zone 选 **us-west1-b**
   - **Machine type（机型）**：**e2-micro**（共享核心，页面会显示月估价约 $7.11，其中机型 $6.11 + 10GB 磁盘 $1.00——试用期内从 $300 里扣，不用慌）
   - **OS and storage（系统和存储）**：点左栏这一项，默认就是 **Debian GNU/Linux 13**，直接用它就行，不用换（旧教程写的 Debian 12 是过时的）
   - **Boot disk（启动磁盘）**：默认 10GB balanced 磁盘。试用期内不用改；如果想严格走免费档，可以改成 Standard 磁盘 30GB
   - **Firewall（防火墙）**：勾不勾 Allow HTTP/HTTPS 都无妨，Outline 不靠这两个端口
4. 点页面底部的 **Create**，等实例状态变成 **Running**。
5. 在实例列表里记下两个 IP：**External IP（外网 IP，如 203.0.113.10）**和 **Internal IP（内网 IP，如 10.138.0.2）**。内网 IP 只能在 Google 内部用，SSH 和 VPN 都连不上它，**只认外网 IP**。

> **免费档只认三个区。** e2-micro 永久免费只限 **us-west1（俄勒冈）、us-central1（爱荷华）、us-east1（南卡）**，其他区建的 e2-micro 照常计费。认准 us-west1-b，别选错。

---

## 第 3 步：加 SSH 密钥（你，约 2 分钟）

**创建实例的页面里没有 SSH 密钥入口**——Security 分区里没有，Advanced 里也没有，别浪费时间找。密钥在建好实例之后补，步骤如下：

1. 控制台左上角菜单 → **Compute Engine → Settings**，点 **Metadata** 标签页，再点 **SSH Keys** 子标签。
2. 点 **Add item**（或 Edit → Add item），把 SSH **公钥**整行粘贴进去，点 Save。页面底部出现 `Successfully saved SSH keys` 就是加上了。密钥会自动同步到这台运行中的实例上，不用重启。
3. 没有现成密钥对的话，让 Muse 生成一对：Muse 会给你一个 **公钥**（`ssh-ed25519 AAAA... 你的邮箱`，贴到上面这一步）和一个 **私钥文件**（`.pem`，保存好，别发给别人）。公钥注释里的邮箱决定登录用户名（如注释是 `xxx@gmail.com`，用户名就是 `xxx`）。

> 私钥只用这一次：Muse 用它连上服务器装 Outline，装完即忘，不存档、不写进任何文件。

---

## 第 4 步：装 Outline Server（Muse，SSH 里执行）

这一步你不用动手，等 Muse 的消息。

1. Muse 用第 3 步的密钥 SSH 连上实例（用户名是公钥注释里的邮箱前缀）。
2. 跑一条一键安装命令（自动装 Docker）：

```bash
sudo bash -c "$(wget -qO- https://raw.githubusercontent.com/Jigsaw-Code/outline-server/master/src/server_manager/install_scripts/install_server.sh)"
```

> **实测注意：安装脚本中途会停在一个确认问题上**——`Would you like to install Docker? This will run 'curl https://get.docker.com/ | sh'. [Y/n]`。必须回答 `y` 才会继续，不回答就一直卡住。Muse 这边会用 `yes | ssh ...` 的方式自动回答。
>
> 在 e2-micro 上装 Docker 要几分钟，耐心等。装完输出里有三样关键东西：**管理端口**（TCP，如 24808）、**数据端口**（TCP+UDP，如 32362）、**apiUrl**（带一串随机路径的管理接口地址，如 `https://203.0.113.10:24808/AbCdEfGh...`）。

3. 安装输出里还会有一行 `Checking host firewall ... BLOCKED`，这是正常的——防火墙规则还没加，下一步再加。
4. Muse 拿 apiUrl 调接口建一条访问密钥（免装 Outline Manager 桌面端）：

```bash
curl -k -X POST "https://<外网IP>:<管理端口>/<apiPath>/access-keys" -H "Content-Type: application/json" -d '{"name":"你的名字"}'
```

返回的密钥会拼成一条 `ss://` 链接发给你，客户端粘贴即用。服务器上同时跑着两个容器：`shadowbox`（Outline 服务本体）和 `watchtower`（自动更新），用 `sudo docker ps` 能看到两个都是 Up 状态。

---

## 第 5 步：放行端口 + 固定 IP（你，控制台操作，约 5 分钟）

### 5.1 防火墙：建两条规则

Outline 的两个端口是安装时随机生成的，所以防火墙规则放在安装之后加。**VPC network 不在 Compute Engine 的左栏里**，进法有两种：

- 点左上角 ☰ 菜单，找 **VPC network → Firewall**；或
- 点顶部搜索框搜 `firewall`，点第一个结果 **Firewall（Product page · VPC Network）**。

进去点 **Create firewall rule**，按下面填（没提到的保持默认）：

| 字段 | 第一条（管理端口） | 第二条（数据端口） |
|---|---|---|
| Name | `outline-mgmt` | `outline-access` |
| Logs | Off | Off |
| Network | default | default |
| Priority | 1000 | 1000 |
| Direction of traffic | **Ingress** | **Ingress** |
| Action on match | **Allow** | **Allow** |
| Targets | All instances in the network | All instances in the network |
| Source filter | IPv4 ranges | IPv4 ranges |
| Source IPv4 ranges | `0.0.0.0/0` | `0.0.0.0/0` |
| Protocols and ports | 勾 tcp，填管理端口（如 `24808`） | 勾 tcp 和 udp，各填数据端口（如 `32362`） |

点 **Create**。两条都建好就生效，一般几秒内就能从外网连上。

> **Ingress 是什么意思？** Ingress = 入站流量，也就是外面连进你的服务器——放行的就是这种，选它没错。Egress = 出站流量（服务器往外连），不用管。

### 5.2 外网 IP：从临时转成静态

实例默认的外网 IP 是临时的（Ephemeral），机器一停再开就可能换 IP，换了 `ss://` 链接就失效。转成静态 IP 后地址永久不变：

1. 顶部搜索框搜 `IP addresses`，点第一个结果（VPC network 那个），进 IP 地址列表。
2. 找到你的外网 IP 那一行，看 **Type** 列：显示 **Static** 就是已经是静态的，不用管；显示 **Ephemeral** 才需要转。
3. 点那行最右边的 **⋮** 菜单，选把临时地址转成静态（Reserve static address），名字随便填，**IP 地址本身不变**。
4. 千万别点成新建地址——新建会换一个 IP，链接就失效了。

> 挂在运行中实例上的静态 IP 是免费的；只有闲置没挂任何实例的静态 IP 才收费。

---

## 第 6 步：连上验证（你，约 5 分钟）

### 6.1 装 Outline Client

- **Mac**：打开 App Store，搜 `Outline Client`，第一个 **Outline-Secure internet access** 就是，点 Get 安装。
  - **不要用 brew**：Homebrew 里只有管理端 `outline-manager`，没有客户端 `outline-client`。
  - **不要用 GitHub 上的旧 dmg 链接**：项目 2026 年换了组织（Jigsaw-Code → OutlineFoundation），旧仓库 `releases/latest/download/Outline-Client.dmg` 已经 404。Mac 客户端官方只走 App Store，没有单独的 dmg。
- **iPhone / iPad**：App Store 搜 Outline，同一个应用。
- **Android**：Google Play 搜 Outline；Play 上不了的话，去 Outline 官方仓库的 releases 说明页找 APK 直链。
- **Windows**：官网 [getoutline.org](https://getoutline.org) 下载。

### 6.2 连接

1. 打开 Outline Client，点右上角 **+**，粘贴 Muse 给你的 `ss://` 链接。
2. 点连接，状态变成 **Connected** 就是通了（实测 2026-11-07：连上 136.86.250.192:32362，显示 Connected，速度很快）。
3. 验证：打开浏览器搜"我的 IP"，确认 IP 地址在**美国俄勒冈**，再打开一个平时打不开的网站，能打开就是通了。

---

## 试用结束之后：花钱续用要多少钱

如果你用着满意、愿意花点钱继续用（2026-11-07 实测速度很快，George 表示可以接受付费），账是这么算的：

| 项目 | 费用 |
|---|---|
| e2-micro 实例（us-west1） | 约 $6.11/月 |
| 10GB 磁盘 | 约 $1.00/月 |
| 出站流量 | 每月前 1GB 免费，超出部分到中国约 **$0.19/GiB**（以 GCP 官网价格表为准） |
| 静态 IP（挂在运行中实例上） | $0 |

按用量估算（仅供参考，以实际账单为准）：

- 轻度用（每月出站 ~30GB）：约 $7.11 + $5.50 ≈ **$13/月**
- 中度用（每月出站 ~100GB）：约 $7.11 + $19 ≈ **$26/月**

关键规则：

- **试用结束不升级 = 自动停服，不扣钱。** 90 天到期（实测到期日 2027-02-06）如果没点 Upgrade，资源自动暂停，不会偷扣钱。想继续用才点 Upgrade，升级本身不收费，升级后才开始按量计费。
- **升级前先确认。** 升级成付费账号后，用量就直接扣卡了，记得第 1.3 步的预算告警要一直开着。
- 长期看，如果 Oracle 那条走通了可以切回去（永久免费，东京延迟更低）；Google 这条适合"先用 90 天免费额度，用着好再决定要不要付费"。

## 日常使用

- **多设备**：一台手机、电脑、平板可以共用同一条 `ss://` 密钥，也可以让 Muse 多建几条，一设备一条。
- **给家人用**：别转发你自己的密钥，让 Muse 单独建一条 key，单独给谁都能随时删。
- **密钥 = 这台服务器的全部访问权**：别发到群里、别贴到公开地方。
- **删密钥**：某条 key 不想用了，告诉 Muse 一声，从服务端删掉就行。
- **换 IP**：如果外网 IP 变了（没做 5.2 步），告诉 Muse 新 IP，重新拼一条链接，不用重装服务器。

## 费用边界

- 90 天内：所有花费都从 $300 里扣，实际支出 $0。
- 免费档每月只含 **1GB** 出站流量；刷视频多的话注意用量，预算告警会提醒。
- 别点的东西：付费机型、超出免费档的磁盘、按量计费的服务。升级为付费账号前必须先确认。

## 常见问题

| 现象 | 原因 / 办法 |
|---|---|
| 注册页国家列表里没有中国 | 正常，Google 把中国列为受限地区；大陆卡 + 大陆地址开不了新试用 |
| 绑卡扣了 $50，不是 $0–1 | 正常，实测就是 $50 验证金；验证完成后几天内自动退，超一周未退找 Google Cloud Billing 支持 |
| 建了 e2-micro 却在扣费 | 区域选错了：免费只限 us-west1 / us-central1 / us-east1，认准 us-west1-b |
| 创建页找不到 SSH 密钥入口 | 本来就没有，走第 3 步：建好后去 Settings → Metadata → SSH Keys 里贴 |
| 安装脚本卡住不动 | 它在等回答 `Would you like to install Docker? [Y/n]`，回 `y` 才能继续 |
| Muse 连不上 SSH | 确认密钥贴对了（Metadata 里能看到）；GCP 防火墙默认放行 22 端口，一般不用动 |
| 客户端连不上 / 一直转圈 | 先看防火墙规则：UDP 数据端口放行了没；再看实例是不是 Running 状态 |
| 重启后密钥失效了 | 临时外网 IP 变了，按第 5.2 步把 IP 转成静态 |
| 找不到 VPC network / Firewall 入口 | 不在 Compute Engine 左栏里，用顶部搜索框搜 `firewall` 或 `IP addresses`，点第一个结果 |
| 网速比 Oracle 慢 | 正常：俄勒冈到大连延迟 ~200ms，东京要低得多；刷网页看视频够用 |

## 分工一览

- **你**：注册试用、建实例、贴 SSH 密钥、放行两个端口、IP 转静态、装客户端连接。
- **Muse**：生成 SSH 密钥对、装 Outline Server、建密钥、拼 `ss://` 链接、日常加 key / 删 key、排障、到期前提醒你做决定。

## 2026-11-07 实测记录

- 账号：第二个 Google 账号（主账号 zhgao1981@gmail.com 的试用期已过），项目 My First Project
- 试用：$300 到账，90 天，2027-02-06 到期；验证金 $50 待退
- 实例：e2-micro、us-west1-b、Debian 13、外网 IP 136.86.250.192、内网 IP 10.138.0.2
- Outline：管理端口 24808（TCP）、数据端口 32362（TCP+UDP），防火墙两条规则已建、外网 TCP 实测已通
- 客户端：Mac App Store 装 Outline，粘贴链接后显示 Connected，速度快
- 待确认：外网 IP 是否已转静态；$50 验证金到账情况（11-11 检查一次）
