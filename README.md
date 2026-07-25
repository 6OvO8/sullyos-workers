# 自部署 SullyOS 后端 Worker

这个仓库放的是 SullyOS 几个后端 Worker 打好的成品代码。你 **fork 一份、在 Cloudflare 连上它**，之后每次上游更新，你只要在 GitHub 点一下「Sync fork」，Cloudflare 就会自动重新部署——不用再复制粘贴几百 KB 的代码，手机上也能操作。

内容物：

| 目录 | 是什么 | 需要 D1 数据库 |
|------|--------|---------------|
| `amsg/` | 主动消息 2.0：角色到点主动发消息给你 | 需要 |
| `instant-push/` | Instant Push：聊天回复走后台推送，关掉页面也能收 | 不需要（可选） |
| `mcp-proxy/` | MCP 工具代理：让角色能连你自己配的 MCP 工具服务器 | 不需要 |

每个都是独立的，只部署你要用的那个就行。

---

## 一次性准备

### 1. Fork 这个仓库

页面右上角「Fork」。这一步之后你就有了自己的一份副本。

### 2. 建 D1 数据库（只有 `amsg/` 需要，其余跳过）

Cloudflare 面板 → 左侧 **Storage & Databases** → **D1** → **Create database**，名字随便起（比如 `sullyos-amsg`）。

建好后进去，把 **Database ID** 复制下来（一串 uuid，长这样 `3f2b1c8a-9d4e-...`）。

> 表结构不用管：SullyOS 里点「连接」时会自动建表。

### 3. 在 Cloudflare 连上仓库

Cloudflare 面板 → **Workers & Pages** → **Create** → 选 **Import a repository**（连 GitHub 那个），授权后选中你 fork 的仓库。

两项要填：

- **Root directory**：你要部署的那个子目录，比如 `amsg`
- **构建命令**：`sh ./deploy-prepare.sh`

> 代码已经是打包好的，这条构建命令不编译任何东西——它只做一件事：把下一步那个
> Database ID 填进配置文件。这样你就不用去 GitHub 上编辑代码了。

### 4. 设变量和密钥

部署设置里找到环境变量 / Secrets，按你部署的 Worker 加：

**`amsg/`**

| 名字 | 哪来的 | 必填 | 类型 |
|------|--------|------|------|
| `D1_DATABASE_ID` | 上一步复制的 Database ID | 是 | 构建变量 |
| `AMSG_MASTER_KEY` | SullyOS 设置 → 主动消息 2.0 里能一键生成 | 是 | Secret |
| `VAPID_PUBLIC_KEY` | SullyOS 设置 →「推送凭据 (VAPID)」面板 | 是 | Secret |
| `VAPID_PRIVATE_KEY` | 同上 | 是 | Secret |
| `VAPID_EMAIL` | 随便一个 `mailto:你的邮箱` | 否 | Secret |
| `AMSG_SERVER_TOKEN` | 自己起一个密码，填了就要求所有请求带上它 | 否 | Secret |

> `D1_DATABASE_ID` 放**构建变量**（Settings → Build → Variables），不是 Secret——
> 它要在构建阶段被读到，而且 Database ID 本身不是敏感信息。其余几个放 Secret。

> ⚠️ VAPID 那一对**必须和 SullyOS 面板里的是同一对**。整个站点共用一个浏览器推送订阅，Worker 用别的密钥对去签，推送会被浏览器拒掉（403），表现是「一切正常但就是收不到」。

**`instant-push/`** 需要 `VAPID_PUBLIC_KEY` / `VAPID_PRIVATE_KEY`（同一对），`VAPID_EMAIL` 可选。

**`mcp-proxy/`** 不需要密钥。

### 5. 部署，把地址填回 SullyOS

部署完 Cloudflare 会给你一个 `https://xxx.workers.dev` 地址，复制它，填进 SullyOS 对应的设置项里，点「连接」。

---

## 以后怎么更新

上游发了新版本之后：

1. 打开你 fork 的仓库页面
2. 点 **Sync fork** → **Update branch**
3. 完事——Cloudflare 检测到新提交会自动重新部署

密钥、D1 绑定、你填的 Database ID 都不会丢。

---

## 常见问题

**Sync fork 提示冲突？**
正常情况不会——你的 Database ID 和密钥都存在 Cloudflare，不在仓库里，所以 fork 里没有你改过的文件。如果真冲突了（比如你手动编辑过），删掉 fork 重新 fork 一遍就行，Cloudflare 那边的连接、变量和密钥都不受影响。

**构建失败，日志里说 `D1_DATABASE_ID 是空的`？**
构建变量没设，或者设成 Secret 了。它必须放在 **Settings → Build → Variables**（构建阶段才读得到），不是运行时的 Secret。照日志里的四步做一遍即可。

**部署成功了但 SullyOS 连不上？**
先在浏览器直接打开 `https://你的地址/capabilities`。能返回一段 JSON 说明 Worker 活着，问题多半在地址填错或 `AMSG_SERVER_TOKEN` 不一致；打不开就去 Cloudflare 看部署日志。

**主动消息到点了没反应？**
`amsg/` 靠定时触发器每分钟检查一次，配置里已经写好了（`crons = ["* * * * *"]`）。去 Worker 的 **Settings → Trigger Events** 确认 Cron 那条在；不在的话通常是 Root directory 填错、部署的不是 `amsg` 目录。

**想用 wrangler 命令行而不是网页？**

```bash
cd amsg
wrangler d1 create sullyos-amsg     # 拿 database_id 填进 wrangler.toml
wrangler secret put AMSG_MASTER_KEY # 其余密钥同理
wrangler deploy
```
