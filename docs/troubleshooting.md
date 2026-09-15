# 故障排查

先收集状态，不要立即删除容器、数据库或数据目录：

```bash
cd /opt/seafile
docker compose config --services
docker compose ps
docker compose logs --tail=200 seally
docker compose exec seally tail -n 200 /opt/seafile/logs/seally.log
docker compose logs --tail=100 seafile
docker compose logs --tail=100 caddy
```

## Compose 提示变量未设置

`seally.yml` 会主动拒绝缺少关键变量的配置。根据错误补齐：

- `SEALLY_IMAGE`：使用发布页中的固定版本标签或 digest。
- `SEAFILE_MYSQL_DB_PASSWORD`：复用现有 Seafile 数据库密码。
- `SEAFILE_SERVER_HOSTNAME`：复用现有域名或 IP。
- `JWT_PRIVATE_KEY`：复用 Seafile 已有值。
- `SM_SESSION_SECRET`：执行 `openssl rand -hex 32` 新生成，仅供 Seally 使用。
- `CACHE_PROVIDER`：Seafile 12 填 `memcached`，Seafile 13 填 `redis`。

修改 `.env` 后先运行 `docker compose config --quiet`。

## 拉取镜像提示 denied 或 unauthorized

先确认 `SEALLY_IMAGE` 与发布页给出的仓库、版本标签完全一致。如果镜像包不是公开的，
需要使用具有 `read:packages` 权限的 GitHub Personal Access Token 登录 GHCR：

```bash
docker login ghcr.io -u 你的GitHub用户名
```

在交互提示中输入 token，不要把 token 直接写进命令、`.env` 或 Issue。

## 找不到 `/shared/seafile/conf`

日志出现 `Waiting for /shared/seafile/conf` 或超时，通常是 Seally 与 Seafile 没有复用
同一个数据卷。比较两个容器的 `/shared` 挂载来源：

```bash
docker inspect seafile --format '{{range .Mounts}}{{if eq .Destination "/shared"}}{{.Source}}{{end}}{{end}}'
docker inspect seally --format '{{range .Mounts}}{{if eq .Destination "/shared"}}{{.Source}}{{end}}{{end}}'
```

两行必须完全相同。修正 `.env` 中的 `SEAFILE_VOLUME` 后执行
`docker compose up -d seally`。

## 数据库连接失败

确认 `db`、`seafile`、`seally` 位于同一个 `seafile-net`，并检查数据库参数是否与
Seafile 一致：

```bash
docker network inspect seafile-net
docker compose ps db
```

不要把 `SEAFILE_MYSQL_DB_HOST` 设置成 `localhost`；在容器内它应为官方服务名 `db`，
除非你的数据库确实位于 Compose 之外。

## 缓存连接失败

- Seafile 12 官方 Compose 提供 `memcached`，应设置 `CACHE_PROVIDER=memcached`。
- Seafile 13 官方 Compose 提供 `redis`，应设置 `CACHE_PROVIDER=redis`。
- Redis 使用密码时，Seafile 与 Seally 的 `REDIS_PASSWORD` 必须相同。

修改后重建而不是只重启容器，确保新环境变量生效：

```bash
docker compose up -d seally
```

## `/mate/` 返回 404 或 502

依次检查：

1. `docker compose ps seally caddy` 是否在运行。
2. `seally` 是否加入 `seafile-net`。
3. `seally.yml` 中 Caddy 标签是否保留。
4. `.env` 的 `SEAFILE_SERVER_HOSTNAME` 和 `SEAFILE_SERVER_PROTOCOL` 是否与浏览器地址一致。
5. 使用自建反向代理时，是否将 `/mate/*` 去前缀后转发到 `seally:8080`。

## 点击 Seally 后又回到登录页

核对以下三个值是否完全相同：

- `.env` 中的 `JWT_PRIVATE_KEY`
- `seahub_settings.py` 中的 `THIRDPART_WEBSITE_SECRET_KEY`
- Seally 容器中的 `SM_JWT_SECRET`

不要用会一次打印容器全部环境变量的命令排查，因为其中包含数据库密码和密钥。
请在服务器本地逐项核对，且不要把真实值粘贴到终端记录、聊天或 Issue。修正后执行
`docker compose up -d seally` 和 `docker compose restart seafile`。

## Seafile 左侧没有 Seally

确认配置写在实际使用的
`${SEAFILE_VOLUME}/seafile/conf/seahub_settings.py` 末尾，且没有后续同名变量覆盖它。
如果已有 `CUSTOM_NAV_ITEMS`，应把 Seally 条目合并进原列表。修改后重启 `seafile`
容器并重新登录。

## 提交 Issue 前脱敏

可以提交服务名、状态、错误栈和镜像版本；必须删除或替换以下内容：数据库密码、
`JWT_PRIVATE_KEY`、`SM_SESSION_SECRET`、OAuth client secret、access/refresh token、
对象存储 Access Key/Secret Key、真实用户邮箱和内部域名。
