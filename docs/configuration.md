# 配置参考

Seally 复用 Seafile 的 Compose 项目、数据库、缓存、公开域名和数据卷。所有变量都
应写入 Seafile 部署目录原有的 `.env`，不要维护第二份互相覆盖的生产配置。

## 必填配置

| 变量 | 来源 | 说明 |
|---|---|---|
| `SEALLY_IMAGE` | 新增 | 固定版本标签或 digest；不要使用 `latest` |
| `SEAFILE_VOLUME` | 复用 Seafile | Seafile 与 Seally 必须挂载相同目录到 `/shared` |
| `SEAFILE_SERVER_HOSTNAME` | 复用 Seafile | 用户访问的域名或 IP，不包含协议和路径 |
| `SEAFILE_SERVER_PROTOCOL` | 复用 Seafile | `http` 或 `https` |
| `SEAFILE_MYSQL_DB_PASSWORD` | 复用 Seafile | Seafile 数据库用户密码 |
| `JWT_PRIVATE_KEY` | 复用 Seafile | Seahub 第三方 SSO 签名密钥；已有部署不要更换 |
| `SM_SESSION_SECRET` | 新增 | Seally 独立高熵密钥，执行 `openssl rand -hex 32` 生成 |
| `CACHE_PROVIDER` | 明确设置 | Seafile 12 为 `memcached`，Seafile 13 为 `redis` |

数据库名和连接参数默认遵循 Seafile 官方值。只有原部署使用了自定义值时才需要覆盖：

```dotenv
SEAFILE_MYSQL_DB_HOST=db
SEAFILE_MYSQL_DB_PORT=3306
SEAFILE_MYSQL_DB_USER=seafile
SEAFILE_MYSQL_DB_CCNET_DB_NAME=ccnet_db
SEAFILE_MYSQL_DB_SEAFILE_DB_NAME=seafile_db
SEAFILE_MYSQL_DB_SEAHUB_DB_NAME=seahub_db
```

## 密钥职责

`JWT_PRIVATE_KEY` 和 `SM_SESSION_SECRET` 不能混用：

| 密钥 | 使用方 | 更换影响 |
|---|---|---|
| `JWT_PRIVATE_KEY` | Seafile、Seahub、Seally SSO | 现有跨服务令牌失效，可能影响 SeaDoc/通知服务 |
| `SM_SESSION_SECRET` | 仅 Seally | Seally 会话失效、授权指纹变化、已加密网盘凭据无法解密 |

`SM_SESSION_SECRET` 同时用于派生网盘凭据加密密钥。部署后应保持不变，并与数据库
备份分开保存。如果丢失，不能从数据库恢复。

## 缓存

Seafile 12 官方 Compose 使用 Memcached：

```dotenv
CACHE_PROVIDER=memcached
MEMCACHED_HOST=memcached
MEMCACHED_PORT=11211
```

Seafile 13 官方 Compose 默认使用 Redis：

```dotenv
CACHE_PROVIDER=redis
REDIS_HOST=redis
REDIS_PORT=6379
REDIS_PASSWORD=
```

Seafile 13 的缓存变化及环境变量定义见 [官方环境变量参考](https://manual.seafile.com/13.0/config/env/)。
如果现有部署为 Redis 设置了密码，Seally 的 `REDIS_PASSWORD` 必须复用同一个值。

## Seafile 底层对象存储

Seally 的同步引擎目前按 Seafile 本地文件对象后端（`fs`）读取 commit、fs 和 block
对象。Seafile Pro 使用 `s3`、`oss`、`ceph` 或 `multiple` 等非本地后端时：

- Seally 容器仍可启动，SSO 和独立网盘管理功能仍可使用。
- 不要创建涉及 Seafile 资料库的同步任务。
- 双向同步会在装配阶段明确拒绝；其他同步方向也不属于支持范围。

全新 Seafile 13 Pro 部署应设置 `SEAF_SERVER_STORAGE_TYPE=disk`；全新 Seafile 12
Pro 部署应保持 `INIT_S3_STORAGE_BACKEND_CONFIG=false`。已有 Pro 部署请使用
[接入指南中的安全预检命令](deploy-with-existing-seafile.md#1-确认现有部署)，该命令只
打印后端名称，不会打印对象存储凭据。

## 地址与反向代理

```dotenv
SM_SEAFILE_INNER_URL=http://seafile:80
```

该地址只在 `seafile-net` 内使用，不应改成公网域名。公开地址由
`SEAFILE_SERVER_PROTOCOL` 与 `SEAFILE_SERVER_HOSTNAME` 组合生成。仓库提供的
Caddy 标签会把同一域名下的 `/mate/*` 转发到 Seally，并移除 `/mate` 前缀。

若不用官方 Caddy，而是自建 Nginx、Traefik 或外部负载均衡器，需要自行实现等价的
路径转发，并保证浏览器看到的地址仍为 `https://域名/mate/`。本仓库未提供这些代理的
受支持模板。

## 数据目录与备份

Seally 容器内使用以下路径：

- `/shared/seafile/conf`：读取 Seafile 配置，也用于 Seally 授权文件。
- `/shared/seafile/seafile-data`：读取 Seafile 资料库对象。
- `/shared/seafile/seafile-mate-data`：保存 Seally 自身持久化数据。
- `/shared/seafile/logs/seally.log`：应用日志文件；容器内对应
  `/opt/seafile/logs/seally.log`，同时也会输出到容器标准输出。

备份至少应覆盖 Seafile 数据库、整个 `SEAFILE_VOLUME`、Compose 文件和密钥。数据库
备份与 `SM_SESSION_SECRET` 分开存放。

## NFS（可选）

推荐先通过宿主机的 `/etc/fstab` 或 autofs 挂载 NFS，再把挂载点 bind mount 到
Seally 容器。取消 `seally.yml` 中这一行的注释：

```yaml
- ${NFS_VOLUME:-/mnt/nfs}:/mnt/nfs
```

然后在 `.env` 设置真实宿主机路径，例如 `NFS_VOLUME=/mnt/nfs`。确认
`mountpoint /mnt/nfs` 成功后再重建 Seally 容器。NFS 挂载根和用户授权在 Seally
“设置”页面中管理。

## 可选网盘

| 变量 | 用途 |
|---|---|
| `ONEDRIVE_CLIENT_ID` | OneDrive 公共客户端 ID |
| `GDRIVE_CLIENT_ID` | Google Drive OAuth 客户端 ID |
| `GDRIVE_CLIENT_SECRET` | Google Drive OAuth 客户端密钥 |
| `GDRIVE_REDIRECT_URI` | Google Drive 回调地址，通常为 `https://域名/mate/api/v1/drives/oauth/callback` |
| `DROPBOX_CLIENT_ID` | Dropbox 应用客户端 ID |
| `BAIDU_APP_ROOT` | 百度应用目录，默认 `/apps/seally`，必须匹配开放平台应用名 |

凭据不要提交到 Git。Google Drive 的三个 `GDRIVE_*` 变量必须成组配置；非 localhost
部署应使用 HTTPS 回调，并与 Google Cloud Console 中的回调地址完全一致。

## 日志与会话

```dotenv
SM_RUN_MODE=release
SM_SESSION_TTL=23h
TIME_ZONE=Asia/Shanghai
```

`SM_SESSION_TTL` 使用 Go duration 格式，例如 `30m`、`12h`、`23h`。
