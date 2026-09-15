# Seafile 与 Seally 全新联合部署

本指南从 Seafile 官方 Compose 文件开始，适用于 Seafile 12/13 的社区版（CE）和
专业版（Pro）单机部署。Seafile 的基础文件始终从对应版本的官方手册下载，本仓库
只额外提供 `seally.yml`。

## 前置条件

- Linux AMD64 主机，已安装 Docker Engine 与 Compose v2 插件。
- 域名已解析到该主机；若使用 HTTPS，公网需能访问 80/443 端口以便 Caddy 申请证书。
- 资源满足 [Seafile 13 官方系统要求](https://manual.seafile.com/13.0/setup/system_requirements/)。
  启用 Pro 搜索、SeaDoc 或其他扩展时应预留更多内存与磁盘。
- 已决定持久化目录和备份位置；不要把真实 `.env` 提交到 Git。

## 1. 选择版本组合

| 组合 | `SEAFILE_MAJOR` | `SEAFILE_EDITION` | 缓存 | 额外搜索文件 |
|---|---|---|---|---|
| Seafile 12 CE | `12.0` | `ce` | Memcached | 无 |
| Seafile 12 Pro | `12.0` | `pro` | Memcached | 已包含在 `seafile-server.yml` |
| Seafile 13 CE | `13.0` | `ce` | Redis | 无 |
| Seafile 13 Pro | `13.0` | `pro` | Redis | `elasticsearch.yml` |

Seally 对 Seafile 资料库的同步目前只支持本地文件对象后端（`fs`）。选择 Pro 时必须
使用 `disk`：Seafile 13 Pro 保持 `SEAF_SERVER_STORAGE_TYPE=disk`；Seafile 12 Pro
保持 `INIT_S3_STORAGE_BACKEND_CONFIG=false`。S3/multiple 等设置可以运行 Seafile，
但不在 Seally 同步功能的支持范围内。

Seafile 12 从官方版本开始采用 `.env`、`seafile-server.yml` 与 `caddy.yml` 的
Compose 结构；Seafile 13 默认缓存改为 Redis，且 Pro 的 Elasticsearch 从
`seafile-server.yml` 拆出。参见 [Seafile 12 CE 部署说明](https://manual.seafile.com/12.0/setup/setup_ce_by_docker/)
和 [Seafile 13 升级说明](https://manual.seafile.com/13.0/upgrade/upgrade_docker/)。

## 2. 下载官方文件

创建部署目录，然后把下面两个变量改成上表中的目标组合：

```bash
sudo install -d -m 0750 -o "$(id -un)" -g "$(id -gn)" /opt/seafile
cd /opt/seafile

SEAFILE_MAJOR=13.0
SEAFILE_EDITION=ce

curl -fL -o .env \
  "https://manual.seafile.com/${SEAFILE_MAJOR}/repo/docker/${SEAFILE_EDITION}/env"
curl -fLO \
  "https://manual.seafile.com/${SEAFILE_MAJOR}/repo/docker/${SEAFILE_EDITION}/seafile-server.yml"
curl -fLO \
  "https://manual.seafile.com/${SEAFILE_MAJOR}/repo/docker/caddy.yml"
curl -fLO \
  "https://manual.seafile.com/${SEAFILE_MAJOR}/repo/docker/seadoc.yml"
```

仅 Seafile 13 Pro 还需要：

```bash
curl -fLO \
  "https://manual.seafile.com/13.0/repo/docker/pro/elasticsearch.yml"
```

然后下载 Seally Compose 文件：

```bash
SEALLY_COMMUNITY_REF=YYYY.MM.PATCH
curl -fL -o seally.yml \
  "https://raw.githubusercontent.com/ysf002/Seally/${SEALLY_COMMUNITY_REF}/seally.yml"
```

把 `SEALLY_COMMUNITY_REF` 替换为本社区仓库的发布 tag 或完整 commit，不要从可变的
`main` 分支下载生产配置。

对应的官方部署说明：

- [12 CE](https://manual.seafile.com/12.0/setup/setup_ce_by_docker/)
- [12 Pro](https://manual.seafile.com/12.0/setup/setup_pro_by_docker/)
- [13 CE](https://manual.seafile.com/13.0/setup/setup_ce_by_docker/)
- [13 Pro](https://manual.seafile.com/13.0/setup/setup_pro_by_docker/)

## 3. 配置 `.env`

先按 Seafile 官方说明填写 `.env`。以下值必须使用真实、安全的配置：

```dotenv
SEAFILE_SERVER_HOSTNAME=seafile.example.com
SEAFILE_SERVER_PROTOCOL=https
TIME_ZONE=Asia/Shanghai
JWT_PRIVATE_KEY=替换为随机值
SEAFILE_MYSQL_DB_PASSWORD=替换为随机值
INIT_SEAFILE_MYSQL_ROOT_PASSWORD=替换为随机值
INIT_SEAFILE_ADMIN_EMAIL=admin@example.com
INIT_SEAFILE_ADMIN_PASSWORD=替换为强密码
```

可分别执行 `openssl rand -hex 32` 生成 `JWT_PRIVATE_KEY` 和各自独立的
`SM_SESSION_SECRET`。不要让两个变量共用同一个值。

在 `COMPOSE_FILE` 原值末尾追加 `seally.yml`：

```dotenv
# 12 CE、12 Pro、13 CE 的典型值
COMPOSE_FILE='seafile-server.yml,caddy.yml,seadoc.yml,seally.yml'

# 13 Pro 使用 Elasticsearch 时的典型值
COMPOSE_FILE='seafile-server.yml,caddy.yml,seadoc.yml,elasticsearch.yml,seally.yml'
```

如果你增减了 SeaDoc、通知服务、SeaSearch 等组件，以官方 `.env` 的原值为基准，
只在末尾追加 `seally.yml`。

再把 [`seally.env.example`](../seally.env.example) 中的配置合并到 `.env`。最少需要：

```dotenv
SEALLY_IMAGE=ghcr.io/ysf002-project/seally:YYYY.MM.PATCH
SM_SESSION_SECRET=替换为另一个随机值
```

缓存类型无需额外配置。Seafile 13 会复用官方 `.env` 中的
`CACHE_PROVIDER=redis`；Seafile 12 的官方 `.env` 没有该变量，`seally.yml` 会自动
使用 `memcached`。Redis/Memcached 的地址、端口和密码也直接复用 Seafile 原有配置，
不要在 Seally 配置段中重复定义这些变量。

把 `SEALLY_IMAGE` 替换成发布页中的完整版本标签或 digest。生产环境禁止使用
`latest`。

## 4. 检查并首次启动

```bash
docker compose config --quiet
docker compose config --services
docker compose pull
docker compose up -d
docker compose logs -f seafile
```

`config --services` 必须同时列出 `db`、`seafile`、`caddy`、目标版本的缓存服务和
`seally`。等待 Seafile 日志显示初始化及启动完成后，按 `Ctrl+C` 退出日志跟踪，
不会停止容器。

检查 Seally：

```bash
docker compose ps
docker compose logs --tail=100 seally
docker compose exec seally tail -n 100 /opt/seafile/logs/seally.log
```

首次初始化较慢时，Seally 可能先等待 `/shared/seafile/conf`；在 Seafile 完成初始化后
会自动恢复。若持续重启，参阅 [故障排查](troubleshooting.md)。

## 5. 增加 Seahub SSO 入口

Seafile 首次成功启动后，编辑
`${SEAFILE_VOLUME}/seafile/conf/seahub_settings.py`，在文件末尾加入：

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

然后重新加载 Seahub：

```bash
docker compose restart seafile
docker compose logs --tail=100 seafile
```

Seahub 的自定义导航机制可参考 [官方说明](https://manual.seafile.com/latest/config/seahub_customization/)。

## 6. 验收与备份

1. 打开 `https://你的域名/` 并使用管理员账号登录。
2. 确认左侧出现 **Seally**，点击后进入 `/mate/` 且无需再次登录。
3. 确认 Seally 可以读取当前账号的资料库。
4. 如需 Google Drive 或 Dropbox，以管理员身份打开“设置 → 云盘应用设置”填写应用
   身份；OneDrive 已使用内置公共客户端，无需部署变量或 Client Secret。
5. 运行 `docker compose ps`，确认所有必需服务都在运行。
6. 备份 `.env` 中的密钥，但不要与数据库备份存放在同一位置。
7. 将数据库、`${SEAFILE_VOLUME}` 和 Compose 目录纳入定期备份；Google Client
   Secret 已加密保存在数据库中，恢复时必须同时保留原 `SM_SESSION_SECRET`。

Seafile Pro 授权与 Seally 授权是两套独立机制。使用 Pro 镜像时仍需按 Seafile 官方
要求安装 Seafile Pro 授权；这不会替代 Seally 自身的授权。
