# 接入已有 Seafile

本指南适用于已经通过 Seafile 12/13 官方 Docker Compose 文件运行的单机实例，
同时支持社区版（CE）和专业版（Pro）。操作会新增一个 `seally` 容器，不会重建
Seafile 数据库或资料库。宿主机需为 Linux AMD64，并已安装 Docker Engine 与
Compose v2 插件。

## 1. 确认现有部署

进入 Seafile 的 Compose 目录（通常为 `/opt/seafile`）：

```bash
cd /opt/seafile
docker compose config --services
docker compose ps
```

服务列表必须至少包含 `db`、`seafile` 和 `caddy`，网络名称应为 `seafile-net`。
如果服务名或网络名被自定义过，请先同步修改 `seally.yml`，不要直接照抄本指南。

根据 `.env` 中的 `SEAFILE_IMAGE` 判断组合：

| 镜像 | 组合 | `CACHE_PROVIDER` |
|---|---|---|
| `seafileltd/seafile-mc:12.0-*` | 12 CE | `memcached` |
| `seafileltd/seafile-pro-mc:12.0-*` | 12 Pro | `memcached` |
| `seafileltd/seafile-mc:13.0-*` | 13 CE | `redis` |
| `seafileltd/seafile-pro-mc:13.0-*` | 13 Pro | `redis` |

如果是 Pro，先检查 Seafile 自身的对象存储后端。下面的命令只输出相关段的 `name`，
不会打印 S3 密钥：

```bash
docker compose exec seafile awk '
  /^\[(commit_object_backend|fs_object_backend|block_backend)\]$/ { section=$0; next }
  /^\[/ { section="" }
  section && /^[[:space:]]*name[[:space:]]*=/ { print section, $0 }
' /shared/seafile/conf/seafile.conf
```

没有输出，或输出的 `name` 均为 `fs`，才属于完整支持范围。如果任一项为 `s3`、
`oss`、`ceph` 等非本地后端，Seally 可以用于登录和网盘管理，但不要创建涉及 Seafile
资料库的同步任务；当前双向同步会明确拒绝，其他方向也不受支持。

先备份 `.env`、数据库、`${SEAFILE_VOLUME}`，以及
`${SEAFILE_VOLUME}/seafile/conf/seahub_settings.py`。不要在没有备份的生产实例上
直接试部署。

## 2. 添加 Compose 文件

下载本仓库的 `seally.yml` 到当前目录，或从本仓库复制该文件：

```bash
SEALLY_COMMUNITY_REF=YYYY.MM.PATCH
curl -fL -o seally.yml \
  "https://raw.githubusercontent.com/ysf002/Seally/${SEALLY_COMMUNITY_REF}/seally.yml"
```

把 `SEALLY_COMMUNITY_REF` 替换为本社区仓库的发布 tag 或完整 commit，不要从可变的
`main` 分支下载生产配置。建议让它与本次部署记录中的 Seally 镜像版本一同留档。

编辑现有 `.env`，在原有 `COMPOSE_FILE` 的末尾追加 `seally.yml`。必须保留已有
组件。例如：

```dotenv
# 原值
COMPOSE_FILE='seafile-server.yml,caddy.yml,seadoc.yml'

# 修改后
COMPOSE_FILE='seafile-server.yml,caddy.yml,seadoc.yml,seally.yml'
```

Seafile 13 Pro 如果使用 Elasticsearch，原值通常还包含 `elasticsearch.yml`；
保留它，只在最后追加 `seally.yml`。

## 3. 配置环境变量

把 [`seally.env.example`](../seally.env.example) 中需要的变量追加到现有 `.env`，
至少填写以下内容：

```dotenv
SEALLY_IMAGE=ghcr.io/ysf002-project/seally:YYYY.MM.PATCH
CACHE_PROVIDER=redis
SM_SESSION_SECRET=替换为随机值
SM_RUN_MODE=release
```

- 把 `SEALLY_IMAGE` 替换为发布页给出的完整版本标签或 digest，不要使用 `latest`。
- Seafile 12 把 `CACHE_PROVIDER` 改为 `memcached`；Seafile 13 使用 `redis`。
- 使用 `openssl rand -hex 32` 生成 `SM_SESSION_SECRET`。
- 确认现有 `JWT_PRIVATE_KEY` 非空。Seally 会直接复用它；如果已有值，不得重新生成
  或修改，否则现有 Seafile/SeaDoc/通知服务的令牌可能失效。
- `SEAFILE_MYSQL_DB_PASSWORD`、`SEAFILE_VOLUME`、域名和协议均复用现有值，不要
  为 Seally 建第二套。

运行配置检查：

```bash
docker compose config --quiet
docker compose config --services
```

第二条命令的输出应包含 `seally`。

## 4. 启动 Seally

```bash
docker compose pull seally
docker compose up -d seally
docker compose ps seally
docker compose logs --tail=100 seally
docker compose exec seally tail -n 100 /opt/seafile/logs/seally.log
```

首次启动会等待 `/shared/seafile/conf`，并自动创建 Seally 所需的数据表。Seally 日志
同时写入容器标准输出和共享卷内的 `/opt/seafile/logs/seally.log`；两处均不应出现
数据库、缓存或 `/shared` 相关错误。

## 5. 配置 Seafile 登录入口

编辑 `${SEAFILE_VOLUME}/seafile/conf/seahub_settings.py`，在文件末尾加入以下内容。
域名、协议和密钥必须替换为当前部署的真实值：

```python
ENABLE_SSO_TO_THIRDPART_WEBSITE = True
THIRDPART_WEBSITE_SECRET_KEY = '与 .env 中 JWT_PRIVATE_KEY 完全相同的值'
THIRDPART_WEBSITE_URL = 'https://seafile.example.com/mate/'

CUSTOM_NAV_ITEMS = [
    {
        'icon': 'sf3-font-share-from-other-servers sf3-font',
        'desc': 'Seally',
        'link': '/sso-to-thirdpart/',
    },
]
```

如果已有 `CUSTOM_NAV_ITEMS`，把 Seally 条目合并到现有列表，不要定义第二个同名变量。
如果文件中已有第三方网站 SSO 设置，先确认它是否被其他应用占用；Seahub 的这组
单一目标配置不能同时指向两个不同应用。

重启 Seafile 让 Seahub 重新加载配置：

```bash
docker compose restart seafile
docker compose logs --tail=100 seafile
```

## 6. 验证

1. 登录 Seafile，确认左侧出现 **Seally**。
2. 点击入口，确认浏览器进入 `https://你的域名/mate/` 且不要求再次登录。
3. 在 Seally 中确认能列出当前用户可访问的 Seafile 资料库。
4. 运行 `docker compose ps`，确认 `seafile`、`caddy`、数据库、缓存和 `seally`
   都处于运行状态。

如果直接访问 `/mate/` 正常但点击侧边栏登录失败，优先核对
`THIRDPART_WEBSITE_SECRET_KEY`、`JWT_PRIVATE_KEY` 和 `SM_JWT_SECRET`（后者由
`seally.yml` 自动取 `JWT_PRIVATE_KEY`）是否完全一致。

## 更新与移除

更新时只修改 `SEALLY_IMAGE` 为新的固定版本或 digest，然后执行：

```bash
docker compose pull seally
docker compose up -d seally
```

移除时，从 `COMPOSE_FILE` 删除 `seally.yml`，删除 Seahub 中的 Seally SSO/导航配置，
再停止并移除 `seally` 容器。`${SEAFILE_VOLUME}/seafile/seafile-mate-data` 和数据库
中的 Seally 表不会自动删除，以便重新接入；如需清理数据，请先备份并单独评估。
