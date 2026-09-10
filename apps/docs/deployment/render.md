# Render 部署（免费方案：CPA + Supabase + 轻量面板）

本页说明如何把 **CPA（CLIProxyAPI 本体）** 部署到 Render 的 **Free 实例**，并使用免费的 **Supabase Postgres** 做持久化，完全不需要 Render 的付费 Disk。管理界面用 CPAMP **轻量面板**（由 CPA 直接托管，不需要额外的 Manager Server 服务）。

如果你需要请求历史、用量成本分析或账号自动化，这些能力只有 CPAMP **完整模式**（Manager Server）才有，而完整模式的 SQLite 存储没有对应的免费托管方案，需要 Render 付费 Disk，见 [Docker 部署](./docker.md)。本页只覆盖免费方案，也就是 CPA 本体 + 轻量面板。

## 为什么这样搭配

- CPA 自带一个可插拔的远程存储后端（`PGSTORE_DSN`），设置后会把 `config.yaml` 全文和账号认证 token 都存进 Postgres，容器本地磁盘只是一份可丢弃的缓存，重启时自动从数据库回填。这正好匹配 Render Free 实例"没有持久 Disk、随时可能重建文件系统"的特点，不需要改任何代码。
- CPAMP 轻量面板不需要独立服务、数据库或额外端口，只是让 CPA 托管一份不同的管理界面，天然免费。

代价：轻量面板没有持久化的请求历史、成本分析和服务端自动化；Free 实例空闲会休眠，冷启动有延迟。这些限制见下面的[已知限制](#免费方案的已知限制)。

## 架构总览

只有一个 Render 服务：

| 服务 | 作用 | 端口 | 来源 |
| --- | --- | --- | --- |
| `cli-proxy-api` | 网关本体 + 轻量管理面板，客户端和管理员都直接连它 | `8317` | Docker 镜像 `eceasy/cli-proxy-api:latest` |

持久化：Supabase Postgres（`config.yaml` + 账号认证 token）。

仓库根目录的 [`render.yaml`](https://github.com/seakee/CPA-Manager-Plus/blob/main/render.yaml) 就是按这个方案写的 Blueprint，可以直接用 Render Dashboard -> New -> Blueprint 一键创建；字段名请对照 Render 当前文档核对，跑不通就照着下面的步骤手动建服务。

## 第一步：创建 Supabase Postgres

1. 在 [Supabase](https://supabase.com/) 创建一个免费项目。
2. 进入 Project Settings -> Database -> **Connection pooling**，选择 **Transaction** 模式，复制连接串（形如 `postgresql://postgres.xxxx:[YOUR-PASSWORD]@aws-xxx.pooler.supabase.com:6543/postgres`）。

   推荐用连接池（端口通常是 `6543`）而不是直连（端口 `5432`）：Render Free 实例反复冷启动、反复建立新连接，连接池能避免占满 Supabase 免费项目有限的连接数。
3. 把 `[YOUR-PASSWORD]` 替换成你在创建项目时设置的数据库密码，得到完整 DSN，先记下来（下一步要填进 Render）。

## 第二步：把 CPA 部署到 Render

1. Render Dashboard -> **New** -> **Web Service** -> 选择 "Existing Image"，填入：

   ```text
   docker.io/eceasy/cli-proxy-api:latest
   ```

2. Instance Type 选 **Free**（本方案不需要 Disk）。
3. 环境变量：

   | Key | 值 |
   | --- | --- |
   | `PORT` | `8317`（告诉 Render 转发到哪个端口；CPA 本身不读取这个变量） |
   | `PGSTORE_DSN` | 第一步拿到的 Supabase 连接串 |
   | `TZ` | 可选，例如 `Asia/Shanghai` |

4. 部署完成后打开 Render 分配的地址（例如 `https://cli-proxy-api-xxxx.onrender.com`），查看 Logs：应该能看到 CPA 启动，并且没有连接 Postgres 失败的报错。

首次启动时，Postgres 里还没有任何配置，CPA 会用内置的 `config.example.yaml` 作为模板，生成一份默认配置并写进 Supabase；之后每次冷启动都会从 Supabase 回填这份配置，不再依赖 Render 容器本地磁盘。

## 第三步：配置 config.yaml

打开该服务的 **Shell** 标签，编辑本地镜像出来的配置文件（路径见镜像里的 `config.yaml`，通常在 `/CLIProxyAPI/config.yaml`），至少设置：

```yaml
remote-management:
  allow-remote: true # 浏览器从公网访问管理面板，必须开
  secret-key: '一个足够长的随机字符串' # 这就是 CPA Management Key
  disable-control-panel: false
  disable-auto-update-panel: false
  panel-github-repository: 'https://github.com/seakee/CPA-Manager-Plus'

api-keys:
  - 'sk-你自己生成的客户端 Key'
```

保存后重启服务。CPA 保存配置时会通过内置的文件监听把改动同步回 Supabase；如果重启后发现改动"丢了"（又变回了默认模板），说明这次同步没有生效，需要在 Shell 里再编辑一次并确认保存后日志里没有报错，必要时参考 CLIProxyAPI 自己的文档确认 `PGSTORE` 的同步时机（这是 CPA 项目自己实现的行为，不由 CPAMP 维护）。

不需要开启 `usage-statistics-enabled`：轻量面板不消费用量队列，开不开都不影响面板功能。

## 第四步：打开轻量面板

```text
https://<你的 CPA 服务>.onrender.com/management.html
```

用上一步设置的 CPA Management Key 登录。首次访问时 CPA 会从 `panel-github-repository` 指定的仓库的最新 Release 里拉取 `management.html` 并缓存，需要能访问 GitHub。

确认可以正常看到 Dashboard、配置中心、Provider、凭证管理、OAuth 和日志。

## OAuth 远程登录

Render 只把 CPA 的 `8317` 端口暴露到公网；CPA 在登录流程中用来接收各家 Provider OAuth 回调的额外端口（`8085` / `1455` / `54545` / `51121` / `11451` 等）不会公开，也不需要公开：

1. 在轻量面板的 OAuth 登录页面发起登录。
2. 在 Provider 完成授权后，浏览器会尝试跳转到 `http://localhost:<port>/...`，这一步必然会失败——这个端口只存在于 Render 容器内部，你的浏览器访问不到。
3. 复制浏览器地址栏里 **完整** 的回调 URL，粘贴进面板的回调 URL 输入框并提交，即可完成登录。

详见 [OAuth 登录](../manual/oauth.md)。Vertex 服务账号导入不走浏览器回调，不受影响。

## 免费方案的已知限制

- **Free 实例会休眠**：长时间没有请求后 Render 会让容器休眠，下一次请求需要冷启动（重新拉起容器 + 从 Supabase 回填配置），会有几秒到十几秒的延迟。如果对客户端首次请求的延迟敏感，需要升级到付费实例。
- **Supabase 免费项目也会暂停**：长时间（Supabase 目前是 7 天）没有任何数据库活动，免费项目会自动暂停，下次连接会失败，需要去 Supabase Dashboard 手动 Resume 项目。如果 CPA 本身也经常没人访问，两边的休眠/暂停可能叠加，第一次请求失败是预期行为，重试或先手动唤醒 Supabase 项目即可。
- **Render 的免费额度有限制**（实例数量、每月运行时长等），具体以 Render 当前的定价页面为准，这里不写死数字以免过时。
- **轻量面板没有请求历史 / 成本分析 / 账号自动化**，这是它的设计目标，不是 bug。需要这些能力时再考虑 CPAMP 完整模式（[Docker 部署](./docker.md)），但那需要 Render 付费 Disk（本仓库的 SQLite 存储没有免费托管方案）。

## 常见问题

- **面板打不开** — 确认 `secret-key` 非空、`disable-control-panel: false`，并且服务确实已经重启过、日志里没有配置解析错误。
- **连接 Supabase 失败 / 启动报数据库错误** — 确认 `PGSTORE_DSN` 用的是连接池地址（端口 `6543`）而不是直连地址、密码正确、Supabase 项目没有处于 Paused 状态。
- **编辑 config.yaml 后重启，改动又变回默认值** — 说明这次编辑没有成功同步回 Supabase，参考[第三步](#第三步-配置-config-yaml)重新确认。
- **仍显示官方面板而不是 CPAMP 界面** — 检查 `panel-github-repository` 拼写，并确认 CPA 能访问 GitHub Release。
