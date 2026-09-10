# Render 部署

Render 是托管式容器平台，可以直接从 Docker 镜像或本仓库的 Dockerfile 部署服务。本页说明如何把 **CPA（CLIProxyAPI 本体）** 和 **CPAMP 完整模式（Manager Server）** 都部署到 Render，并让二者正常连接。

如果只是想先了解两种模式的区别，看[如何选择 CPA 面板](../guide/choosing-a-panel.md)；如果不需要在 Render 上跑 CPA 本体（已有别处运行的 CPA），可以跳过第一步，只做第二步。

## 部署前必读

- **两个服务都必须用支持 Disk 的付费实例**（Starter 及以上）。Render Free 实例既没有持久 Disk，空闲一段时间还会自动休眠；CPA 需要持久化 `config.yaml` 和账号认证文件，CPAMP 需要持久化 SQLite 和 `data.key`，缺一不可。
- 每个 Render 服务只能挂一个 Disk，且只支持单实例（不能自动扩缩容）——这正好符合 CPA 和 CPAMP 都要求单实例运行的前提，不算额外限制。
- Render 对外只通过 HTTPS 网关转发流量，不支持在公网地址上使用 RESP 需要的裸 TCP 协议，因此 CPAMP 的用量采集要显式设成 HTTP 队列模式（`USAGE_COLLECTOR_MODE=http`）。
- 官方 CPA 镜像 `eceasy/cli-proxy-api` 默认不包含 `config.yaml`（只打包了 `config.example.yaml`），需要在启动时生成并持久化这个文件。
- OAuth 登录在 Render 上只能走"远程浏览器回调"（手动粘贴回调 URL）：CPA 用来接收 OAuth 回调的额外端口（如 `8085` / `1455` / `54545` / `51121` / `11451`）不会公开，也不需要公开。

## 架构总览

| 服务 | 作用 | 端口 | 来源 |
| --- | --- | --- | --- |
| `cli-proxy-api`（CPA 本体） | 网关本体，接收并转发真实模型请求，Codex / Claude Code / OpenCode 等客户端直接连接这里 | `8317` | Docker 镜像 `eceasy/cli-proxy-api:latest` |
| `cpa-manager-plus`（CPAMP 完整模式） | Manager Server，负责请求历史、用量分析、账号健康和自动化 | `18317` | 本仓库 `Dockerfile.manager-server` 构建 |

如果想用 Blueprint 一次性创建两个服务，可以直接使用仓库根目录的 [`render.yaml`](https://github.com/seakee/CPA-Manager-Plus/blob/main/render.yaml)（Render Dashboard -> New -> Blueprint，选择本仓库）。字段名请对照 Render 当前文档核对；如果 Blueprint 部署失败，按下面的步骤手动创建两个 Web Service，效果完全一样。

## 第一步：部署 CPA 本体

1. Render Dashboard -> **New** -> **Web Service** -> 选择 "Existing Image"（部署已有镜像），填入：

   ```text
   docker.io/eceasy/cli-proxy-api:latest
   ```

2. Instance Type 选 **Starter 及以上**（需要 Disk）。
3. 添加 Disk：Mount Path 填 `/data`，大小从 1–2GB 起步即可（主要是认证文件和配置，体积很小）。
4. 覆盖启动命令（Docker Command）。因为 Disk 只能挂一个路径，而 CPA 需要 `config.yaml` 和认证目录两处持久化，所以用一段脚本把 `/data` 下的文件软链到镜像期望的路径：

   ```sh
   sh -c '
   mkdir -p /data/auths &&
   if [ ! -f /data/config.yaml ]; then cp /CLIProxyAPI/config.example.yaml /data/config.yaml; fi &&
   ln -sfn /data/config.yaml /CLIProxyAPI/config.yaml &&
   rm -rf /root/.cli-proxy-api &&
   ln -sfn /data/auths /root/.cli-proxy-api &&
   exec ./CLIProxyAPI
   '
   ```

5. 环境变量加一条 `PORT=8317`，告诉 Render 把公网流量转发到容器的 `8317` 端口（CPA 本身不读取 `PORT`，这只是 Render 路由用的）。
6. 部署完成后打开 Render 分配的地址（例如 `https://cli-proxy-api-xxxx.onrender.com`），查看 Logs 确认没有配置报错。
7. 打开该服务的 **Shell** 标签，编辑 `/data/config.yaml`，至少配置：

   ```yaml
   remote-management:
     secret-key: '一个足够长的随机字符串' # 这就是 CPA Management Key
     allow-remote: true

   usage-statistics-enabled: true
   redis-usage-queue-retention-seconds: 60

   api-keys:
     - 'sk-你自己生成的客户端 Key'
   ```

   保存后重启服务。CPA 启动时会把明文 `secret-key` 自动改写成 bcrypt hash 并写回 `config.yaml`，所以这个文件必须保持可写——挂在 Disk 上正好满足这一点。

8. 需要登录 Codex / Claude / Gemini 等 Provider 账号时，参考下面的 [OAuth 远程登录](#oauth-远程登录)。

## 第二步：部署 CPAMP 完整模式

1. Render Dashboard -> **New** -> **Web Service** -> 连接本仓库。
2. Runtime 选 **Docker**，Dockerfile Path 填 `Dockerfile.manager-server`，Docker Context 填仓库根目录 `.`。
3. Instance Type 选 **Starter 及以上**。
4. 添加 Disk：Mount Path 填 `/data`，大小建议从 5–10GB 起步（SQLite 会随请求历史增长，具体看请求量和历史保留时长）。
5. 环境变量：

   | Key | 建议值 | 说明 |
   | --- | --- | --- |
   | `PORT` | `18317` | 告诉 Render 转发到哪个端口 |
   | `HTTP_ADDR` | `0.0.0.0:18317` | Manager Server 监听地址 |
   | `USAGE_DB_PATH` | `/data/usage.sqlite` | SQLite 路径 |
   | `CPA_MANAGER_DATA_KEY_PATH` | `/data/data.key` | 数据加密 key 路径 |
   | `CPA_MANAGER_ADMIN_KEY` | 一串长随机字符串 | 显式设置，避免每次重启翻日志找生成的临时 key |
   | `USAGE_COLLECTOR_MODE` | `http` | Render 公网入口不支持 RESP 需要的裸 TCP |
   | `USAGE_BATCH_SIZE` | `100` | 单批最大采集记录数 |
   | `USAGE_POLL_INTERVAL_MS` | `500` | 空闲轮询间隔 |
   | `USAGE_QUERY_LIMIT` | `50000` | 最近用量事件返回上限 |
   | `USAGE_CORS_ORIGINS` | `*` 或你自己的域名 | 跨域来源 |

6. Health Check Path 填 `/health`。
7. 部署完成后打开：

   ```text
   https://<你的 CPAMP 服务>.onrender.com/management.html
   ```

   如果没有显式设置 `CPA_MANAGER_ADMIN_KEY`，去 Render Dashboard 的 Logs 里找启动时打印的一次性管理员密钥。
8. 首次 setup 填写：
   - **管理员密钥**：上一步取得的 key（或你在环境变量里设置的值）。
   - **CPA URL**：CPA 服务的 Render 公网地址，例如 `https://cli-proxy-api-xxxx.onrender.com`。
   - **CPA Management Key**：就是第一步 `config.yaml` 里 `remote-management.secret-key` 的明文（保存前的原始值）。

## 同区域内网连接（可选）

如果 CPA 和 CPAMP 部署在 **同一个 Render Region**，理论上可以让 CPAMP 通过 Render 私有网络直接访问 CPA，不必绕公网：

```text
http://<CPA 服务的 Render Name>:8317
```

这样做的前提和限制：

- 两个服务必须在同一个 Region，否则内网地址不可达。
- Render 私网是否原样透传 RESP 采集需要的裸 TCP 协议，请以 Render 当前文档或客服支持为准——不确定就继续用 `USAGE_COLLECTOR_MODE=http` 并走公网 HTTPS 地址，最稳妥，性能损失也可以忽略。
- 无论内网还是公网连接，CPA 本身仍然需要公开的 HTTPS 地址供 Codex / Claude Code 等客户端直接调用，所以这一步只是给 CPAMP 到 CPA 的管理流量做一个可选优化，不能替代 CPA 的公网入口。

## OAuth 远程登录

Render 只把 CPA 的 `8317` 端口暴露到公网；CPA 在登录流程中用来接收各家 Provider OAuth 回调的额外端口（`8085` / `1455` / `54545` / `51121` / `11451` 等）不会公开，也不需要公开：

1. 在 CPAMP 完整模式的 OAuth 登录页面发起登录。
2. 在 Provider 完成授权后，浏览器会尝试跳转到 `http://localhost:<port>/...`，这一步必然会失败——这个端口只存在于 Render 容器内部，你的浏览器访问不到。
3. 复制浏览器地址栏里 **完整** 的回调 URL，粘贴进 CPAMP 页面的回调 URL 输入框并提交，即可完成登录。

详见 [OAuth 登录](../manual/oauth.md)。Vertex 服务账号导入不走浏览器回调，不受影响。

## 备份

- **CPA**：定期通过 Render 的 Shell 把 `/data/config.yaml` 和 `/data/auths` 打包，上传到你自己的对象存储；Render 目前没有内建的 Disk 快照/一键下载功能。
- **CPAMP**：备份 `/data/usage.sqlite*` 和 `/data/data.key`，方法和 [备份与恢复](../operations/backup.md) 一致，只是执行位置换成 Render 的 Shell。`data.key` 丢失会导致已保存的 CPA Management Key 无法恢复，只能重新在面板里保存一次 CPA 连接。

## 常见问题

- **服务几分钟不用就打不开 / 冷启动很慢** — 说明用了 Free 实例。Free 实例空闲会休眠且没有 Disk，两个服务都至少要用 Starter。
- **CPAMP 一直显示未连接 CPA** — 确认 CPA URL 填的是完整 HTTPS 地址（带 `https://`），CPA 服务的 Logs 里没有报错，并且 `remote-management.allow-remote: true` 已经生效并重启过。
- **请求监控没有数据** — 确认 CPA 的 `usage-statistics-enabled: true` 已保存，CPAMP 的 `USAGE_COLLECTOR_MODE=http`，再参考[请求监控排障](../troubleshooting/request-monitoring.md)。
- **重启后 CPA 认证信息或 CPAMP 历史数据丢失** — 确认对应服务确实挂了 Disk，并且启动命令 / 环境变量把数据写到了 Disk 的挂载路径下（`/data`），而不是容器里其他会被重建的临时目录。
- **看到 `unsupported RESP prefix 'H'`** — 说明采集器仍在尝试 RESP 模式，请确认 `USAGE_COLLECTOR_MODE` 已经改成 `http` 并重新部署。
