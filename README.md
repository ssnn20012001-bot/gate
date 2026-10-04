# gate — VPN Gate SSTP 节点自动检测与订阅生成

用 GitHub Actions 定时跑 `vpngate.py`：抓取 VPN Gate 的公开节点、筛出 SSTP 节点、逐个做连通性检测，生成静态展示页和一份纯文本订阅，发布到 GitHub Pages。edgetunnel 后台再定时去拉这份订阅，拼上自己的 UUID 和节点域名，客户端因此能自动获得一批可用节点。

## 机制说明（先看这个）

两个部分，职责不重叠：

- **`vpngate.py`（本仓库，GitHub Actions 里跑）** 只做三件事：抓取节点、检测节点、生成文件。产物是三个文件，全部写进 `public/`：
  - `public/data.json` — 结构化数据（各国分组、延迟、出口 IP、住宅/机房标记）
  - `public/index.html` — 节点展示页（模板来自 `web/index.html`）
  - `public/nodes.txt` — 纯文本订阅，每行一个节点

  它**不接触**你的 UUID，也**不接触**你的节点域名，更不会去改 edgetunnel。

- **edgetunnel（Cloudflare Worker，单独部署）** 负责真正的代理服务。它在后台**主动拉取** `nodes.txt` 的 URL，解析每一行，然后自动把它自己的 **UUID** 和**节点域名**拼进去，生成最终订阅给客户端。

所以部署顺序是：**先部署 edgetunnel，拿到自己的域名和 UUID；再部署 GitHub 流水线；最后把 `nodes.txt` 的地址填回 edgetunnel 后台。** 因为订阅地址里带着你的 GitHub 用户名，只有 edgetunnel 先存在，你才知道最终要往后台填什么。

## 数据流

```
VPN Gate 数据源
  api/iphone/  或  fdciabdul/Vpngate-Scraper-API 镜像
        |
        v
筛选：只保留 SSTP (TCP) 节点  ->  按 host:port 去重
        |
        v
逐个调用 Cloudflare 检测 Worker
  GET /check?sstp=vpn:vpn@host:port
  返回 { success, colo, responseTime,
         exit: { ip, country, asn.org, privacy.is_datacenter } }
        |
        v
住宅 / 机房 判定
  优先看 is_datacenter，其次看 ASN org 关键字，最后回退到 vpnXXXXX 主机名推断
        |
        v
生成三个文件
  public/data.json      展示页数据
  public/index.html     展示页
  public/nodes.txt      订阅（每行一个节点）
        |
        v
GitHub Actions 上传 artifact -> 部署到 GitHub Pages
  https://<你的用户名>.github.io/gate/
        |
        v
edgetunnel 后台定时拉取 nodes.txt
  自动拼接自己的 UUID + 节点域名 -> 输出最终订阅 -> 客户端刷新即可
```

`nodes.txt` 每行的格式：

```
入口优选域名:443#国家-住宅-序号$sstp://vpn:vpn@host:port
```

举例：

```
hzytjy.cn:443#日本-住宅-01$sstp://vpn:vpn@123.45.67.89:443
saas.072159.xyz:443#美国-机房-01$sstp://vpn:vpn@98.76.54.32:5555
```

冒号前面的**入口优选域名**来自 `EDGE_HOSTS`，会轮询分配；`#` 后面是节点名（国家 + 住宅/机房 + 两位序号），排序时住宅节点优先、再按延迟从低到高。

## 部署步骤

### 1. 部署 edgetunnel（必须先做）

1. 打开 Cloudflare Dashboard → **Workers & Pages** → **Create** → 选 **Worker**，起名例如 `edgetunnel`，点 Deploy。
2. 进入这个 Worker → **Settings** → **Variables and Secrets**：
   - 新增一个 **KV Namespace**，绑定变量名填 `KV`，创建。
   - 新增一个 **Text** 变量，名字填 `ADMIN`，值填你自己设的**管理密码**（记牢，之后登录后台要用）。
3. **Edit code**，把本仓库 `workers/edgetunnel_worker.js` 的全部内容粘贴进去，覆盖默认代码，点 **Deploy**。
4. 访问 `https://你的Worker域名/admin`，输入刚才的 `ADMIN` 密码登录。
5. 在后台首页记下两样东西：
   - **UUID**（默认由 Worker 自动生成，也可以自己改）
   - **节点域名**（即这个 Worker 的域名）

> UUID 和节点域名只存在于 edgetunnel 这一侧。`vpngate.py` 不需要、也没有这两个变量。

### 2. 部署检测 Worker

1. 再建一个 Worker，例如 `nodecheck`，进入 **Edit code**，粘贴 `workers/check-socks5_worker.js` 的全部内容，部署。
2. 记下它的域名。
3. 浏览器里访问下面这个地址验证（把 host:port 换成 `nodes.txt` 里任意一行的后半段）：

   ```
   https://你的检测Worker域名/check?sstp=vpn:vpn@某节点:端口
   ```

   返回 JSON 就说明正常。成功时大致长这样：

   ```json
   {
     "success": true,
     "type": "sstp",
     "proxy": "sstp://vpn:vpn@某节点:端口",
     "colo": "NRT",
     "responseTime": 812,
     "exit": { "ip": "1.2.3.4", "country": "JP", "asn": { "asn": 12345, "org": "..." }, "is_datacenter": false }
   }
   ```

   返回 `"success": false` 或一段 HTML，说明代码没贴全或者 Worker 没部署好。

### 3. 创建 GitHub 仓库

1. 登录 GitHub（`ssnn20012001-bot`），点右上角 **+** → **New repository**。
2. Repository name 填 `gate`。
3. **必须是 Public**（GitHub Pages 的免费额度对私有仓库不适用，Actions 也因此无需任何 token 配置）。
4. 勾选 **Add a README file**。
5. 点 **Create repository**，然后把本仓库的文件上传上去：

   ```
   vpngate.py
   requirements.txt
   web/index.html
   workers/edgetunnel_worker.js
   workers/check-socks5_worker.js
   .github/workflows/check.yml
   .gitignore
   .gitattributes
   ```

   `.gitignore` 是隐藏文件，在网页上传界面可能看不到，可以用 Git 客户端或 GitHub 桌面端上传。

### 4. 确认两处配置

本仓库已经填好了默认值，**如果你沿用本文的部署可以直接跳过本节**；只有换仓库名、换用户名或换检测 Worker 时才需要改。

- **`.github/workflows/check.yml`** — 找到 `CHECK_WORKER`，确认是你的检测 Worker 地址：

  ```yaml
  CHECK_WORKER: "https://vpngate-check.ssnn20012001.workers.dev/check?sstp=vpn:vpn@"
  ```

- **`vpngate.py`** — 找到 `NODES_URL`，确认是你的 Pages 地址（如果你改了仓库名或用户名，这里也要跟着改）：

  ```python
  NODES_URL = os.environ.get("NODES_URL", "https://ssnn20012001-bot.github.io/gate/nodes.txt")
  ```

  这个值会展示在节点页的订阅入口上，所以必须是最终对外可访问的地址。

其余配置（`EDGE_HOSTS`、`CHECK_CONCURRENCY`、`CHECK_TIMEOUT`）在 workflow 里用环境变量传，不用动代码。

> **注意：`workers.dev` 在中国大陆被墙。** 检测 Worker 用 `xxx.workers.dev` 默认域名在国内**无法访问**（DNS 会被污染劫持，TLS 握手失败）。
> 这不影响流水线本身 —— `CHECK_WORKER` 是 GitHub Actions 的美国 runner 在调用，能正常连通；受影响的只有你本机想手动调 `check` 接口做调试。
> 如果你需要在国内直接调试检测接口，或者希望这个检测端长期稳定可用，建议给 Worker 绑定一个自定义域名（Cloudflare 控制台 → Workers 和 Pages → 你的 Worker → 设置 → 域和路由 → 添加自定义域）。

### 5. 设置 GitHub Pages

1. 进仓库 → **Settings** → **Pages**。
2. **Build and deployment** → **Source** 选 **GitHub Actions**。
3. 存好。

workflow 里已经用 `permissions:` 显式声明了：

```yaml
permissions:
  contents: read
  pages: write
  id-token: write
```

**不需要**再去 **Settings** → **Actions** → **General** → *Workflow permissions* 里改成 "Read and write permissions"。显式声明的 `permissions` 优先级更高，教程里说必须去改的那一步可以跳过。

如果仓库的 Actions 页顶部提示 "Get started with GitHub Actions"，按提示点启用即可。

### 6. 手动触发一次

1. 进仓库 → **Actions** → 左侧选 **VPN Gate Node Check**。
2. 右上 **Run workflow** → 再点一次 **Run workflow**。
3. 等 1-3 分钟，出现绿色对勾。日志最后一行会打印站点地址和部署结果。

第一次运行会顺带通过 API 自动启用 Pages，可能要多花一点时间。

### 7. 成果地址

| 用途 | 地址 |
| --- | --- |
| 节点展示页 | `https://ssnn20012001-bot.github.io/gate/` |
| 订阅文本 | `https://ssnn20012001-bot.github.io/gate/nodes.txt` |

### 8. 填回 edgetunnel 后台

1. 回到第 1 步的 `https://你的Worker域名/admin`。
2. 找到**自定义优选IP**（存进 KV 的 `ADD.txt`）那个输入框。
3. 把订阅地址粘进去：

   ```
   https://ssnn20012001-bot.github.io/gate/nodes.txt
   ```

4. 点保存。

之后每 30 分钟流水线自动更新一次 `nodes.txt`，edgetunnel 后台定时拉取，客户端只需刷新订阅，不用做任何操作。

## 配置速查表

| 配置项 | 在哪里 | 作用 | 备注 |
| --- | --- | --- | --- |
| `EDGE_HOSTS` | `vpngate.py`，或 workflow 的 `env` | 入口优选域名池 | 逗号分隔，**每项必须带 `:443`**。轮询分配给各节点。被墙就要换 |
| `WORKER_CHECK_URL` | `vpngate.py` 第 44 行 | 检测 Worker 地址 | 默认值是占位符，**必须改**；在 Action 里通过 `CHECK_WORKER` 覆盖即可，不用改代码 |
| `NODES_URL` | `vpngate.py` 第 307 行 | 展示给用户的订阅地址 | 改成你自己的 `https://<用户名>.github.io/gate/nodes.txt` |
| `CHECK_CONCURRENCY` | 环境变量，默认 `32` | 并发检测数 | 调大更快，但 Worker 侧和本地网络压力更大 |
| `CHECK_TIMEOUT` | 环境变量，默认 `90`（秒） | 单个节点检测超时 | 网络差可以调大 |
| `MAX_CHECK_NODES` | 环境变量，默认 `0` | 最多检测多少个节点 | `0` 表示不限制。只想快点出结果就设成比如 `40` |
| `HTTP_TIMEOUT` | 环境变量，默认 `60` | 抓取数据源的超时 | 抓取失败时调大 |
| `ADMIN` | edgetunnel Worker 的变量 | 后台管理密码 | 不是 token，纯粹是 `/admin` 的登录口令 |

## 常见问题

**全部节点都失败 / 存活数为 0**

打开 Actions 日志看 `Get VPN Gate nodes` 那一步的输出，对照 `stats`：

- `raw_nodes` 就是 0 或很小 → 抓取环节失败。VPN Gate 官方 API 经常抽风，脚本会自动回退到 `fdciabdul/Vpngate-Scraper-API` 镜像；两个源都挂时，隔几分钟手动重跑一次。
- `raw_nodes` 正常、`success` 为 0 → 检测环节全灭。多半是 `CHECK_WORKER` 域名填错或 Worker 没部署好。浏览器直接访问检测 URL（见第 2 步）确认能返回 JSON。
- `CHECK_TIMEOUT` 太短也会导致大面积失败，酌情调大。

**只有少数节点能连**

如果 `success` 不为 0，但客户端几乎连不上，问题通常不在节点，而在 `EDGE_HOSTS`：**入口优选域名被墙了**。检测是在 Cloudflare 侧发起的，那里能连通不代表你本地能连通。

用 bestcf 之类的优选工具重新测一批可用入口，替换掉 `EDGE_HOSTS` 里的域名，再重跑流水线。注意每项都要带 `:443`，格式是 `域名:443`，逗号分隔。

**edgetunnel 里所有节点延迟都是 -1**

- 确认 `nodes.txt` 的地址能在浏览器直接打开（返回纯文本，不是 404）。
- 确认后台填的订阅地址和仓库里 `NODES_URL` 一致，且仓库是 **Public**。
- 确认 edgetunnel 后台的 **UUID** 没被改动过，且客户端里填的 UUID 与后台一致。
- 确认客户端选择的传输协议与 edgetunnel 配置里的一致（`tcp` / `ws` / `grpc` 等要对上）。
- 确认节点域名没被墙，前面 `EDGE_HOSTS` 那一节的排查同样适用。

**30 分钟过去了，`nodes.txt` 没更新**

- 进 **Actions** 看最近一次 **VPN Gate Node Check** 运行是否成功。红色对钩说明当次失败了，点进去看失败步骤的日志。
- 确认 `schedule` 还在：`.github/workflows/check.yml` 里必须有

  ```yaml
  on:
    schedule:
      - cron: "*/30 * * * *"
  ```

  **只有 `workflow_dispatch` 的话就是只能手动触发，永远不会自动更新。**

- GitHub 的定时任务不是准点的，高峰期可能延迟 10-20 分钟。想立刻验证就手动 **Run workflow**。
- 连续一段时间仓库没有任何活动时，GitHub 可能会自动停用定时任务，重新手动跑一次即可恢复。

**检测 Worker 报错 / 返回不是 JSON**

直接浏览器访问：

```
https://你的检测Worker域名/check?sstp=vpn:vpn@某节点:端口
```

- 返回 HTML → Worker 代码没贴全，或者路由被改掉了。
- 返回 `"success": false` 带 `error` → Worker 正常，是这个节点本身连不上，换一个节点试。
- 404 → 域名或路径写错，注意是 `/check`，不是 `/api/check`。

## 重要澄清

网上不少教程（包括某些 AI 生成的教程）有几处与本仓库实际代码不符，下面逐条纠正。

**1. 生成的订阅文件叫 `nodes.txt`，不是 `hosts.txt`**

`vpngate.py` 里 `write_outputs()` 写出的三个文件是 `data.json`、`index.html`、`nodes.txt`，文件名是硬编码的。有的教程让你去填 `hosts.txt` 的地址，那是错的，填了必然 404。

**2. `vpngate.py` 里没有 `EDT_UUID` / `EDT_DOMAIN` / `EDT_FINGERPRINT`，也不需要**

UUID 和节点域名由 edgetunnel 后台自己掌握。它拉取 `nodes.txt` 时会自动把自己的 UUID 和域名拼到每行节点上。所以 `vpngate.py` 里不需要填，也不该填这三个变量。如果你在这份 `vpngate.py` 里搜不到它们，那是正常的，不是文件缺失或下载不完整。

**3. 网上教程给的那几个行号对应旧版本，已对不上当前文件**

教程里常说"改第 52~55 行""第 461~463 行""第 525~526 行"，那是旧版本的位置。当前文件里的实际位置是：

| 常量 | 当前行号 |
| --- | --- |
| `WORKER_CHECK_URL` / `CHECK_WORKER` | 第 44 行 |
| `EDGE_HOSTS` | 第 297 行 |
| `NODES_URL` | 第 307 行 |

文件一更新行号就变。**以搜索常量名为准，不要信行号。** 用编辑器的查找功能搜 `WORKER_CHECK_URL`、`EDGE_HOSTS`、`NODES_URL` 最稳妥。

**4. 定时刷新依赖 `check.yml` 里的 cron，上游原版没有**

上游原版仓库的 `check.yml` **只有 `workflow_dispatch`（手动触发），没有 `schedule`**。不补上这段 cron，整条流水线就永远不会自动更新：

```yaml
on:
  schedule:
    - cron: "*/30 * * * *"
  workflow_dispatch:
```

`workflow_dispatch` 是让你能手动触发，`schedule` 才是自动更新，两个都要有。

**5. 不需要配置任何 GitHub token 或改 workflow permissions**

workflow 里显式声明了 `contents: read / pages: write / id-token: write`，用的是 Actions 自带的 `GITHUB_TOKEN`，不需要在仓库里存任何 Personal Access Token。相应地，也**不需要**去 Settings → Actions → General 把 workflow permissions 改成 "Read and write permissions"。

## 引用与致谢

| 项目 | 地址 | 用途 |
| --- | --- | --- |
| [cmliu/edgetunnel](https://github.com/cmliu/edgetunnel) | https://github.com/cmliu/edgetunnel | VLESS 代理与链式代理，本项目的 Worker 端 |
| [lsh8848/cm-Workers-CheckSocks5](https://github.com/lsh8848/cm-Workers-CheckSocks5) | https://github.com/lsh8848/cm-Workers-CheckSocks5 | 检测 Worker，支持 sstp / socks5 / http 等协议的连通性与出口 IP 检测 |
| [fdciabdul/Vpngate-Scraper-API](https://github.com/fdciabdul/Vpngate-Scraper-API) | https://github.com/fdciabdul/Vpngate-Scraper-API | VPN Gate 数据镜像，官方 API 不可用时的备用数据源 |
| [VPN Gate](https://www.vpngate.net/) | https://www.vpngate.net/ | 节点数据的上游来源 |

本仓库的节点数据全部来自 VPN Gate 公开接口，仅供技术学习与个人测试使用。请遵守所在地区的法律法规，不要用于任何非法用途。
