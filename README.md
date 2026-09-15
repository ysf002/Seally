# Seally 社区

本仓库提供 Seally 的社区部署文档与 Docker Compose 配置。Seally 与 Seafile
共用用户、数据库和数据卷，并通过 Seafile 的第三方网站 SSO 登录；部署完成后，
用户可以从 Seafile 左侧导航栏进入 Seally。

> 本仓库当前面向 Seafile 官方的**单机 Docker Compose 部署**。二进制安装、
> Kubernetes 和 Seafile 集群不在这些模板的支持范围内。当前 Seally 发布镜像为
> Linux AMD64；ARM64 主机不在本模板的验证范围内。

## 兼容范围

| Seafile | 版本 | 官方镜像 | 缓存 | Seally 配置 |
|---|---:|---|---|---|
| 社区版（CE） | 12 | `seafileltd/seafile-mc:12.0-latest` | Memcached | 支持 |
| 专业版（Pro） | 12 | `seafileltd/seafile-pro-mc:12.0-latest` | Memcached | 支持 |
| 社区版（CE） | 13 | `seafileltd/seafile-mc:13.0-latest` | Redis | 支持 |
| 专业版（Pro） | 13 | `seafileltd/seafile-pro-mc:13.0-latest` | Redis | 支持 |

四种组合共用仓库根目录的 [`seally.yml`](seally.yml)。CE/Pro 的差异由 Seafile
官方 Compose 文件处理。Seally 会复用 Seafile 13 的 `CACHE_PROVIDER=redis`；
Seafile 12 的官方环境文件没有该变量，`seally.yml` 会自动回退到 `memcached`，
无需为 Seally 单独配置缓存类型。

### Seafile 底层存储限制

上表的完整支持以 Seafile 使用本地文件对象后端（`fs`）为前提。Seafile Pro 如果把
自身的 commit、fs 或 block 对象存入 S3/OSS/Ceph，Seally 仍可启动并进行登录及网盘
管理，但**涉及 Seafile 资料库的同步任务不受支持**；双向任务会明确拒绝启动，其他
同步方向也不应创建。已有 Pro 部署请先执行部署指南中的对象后端预检。全新 Pro 部署
应保持 `SEAF_SERVER_STORAGE_TYPE=disk`，不要启用 S3/multiple 后端。

## 选择部署场景

- **Seafile 已经运行**：阅读 [接入已有 Seafile](docs/deploy-with-existing-seafile.md)。
- **Seafile 和 Seally 一起全新部署**：阅读 [全新联合部署](docs/deploy-with-new-seafile.md)。

两个场景的核心流程都是：把 `seally.yml` 放入 Seafile Compose 目录，在原有
`COMPOSE_FILE` 末尾追加 `seally.yml`，补充两个 Seally 部署变量，再配置 Seahub
SSO 入口。云盘应用身份在部署完成后的 Web 设置页管理，不写入 `.env`。

## 仓库内容

| 文件 | 用途 |
|---|---|
| [`seally.yml`](seally.yml) | 可叠加到 Seafile 官方 Compose 项目的 Seally 服务定义 |
| [`seally.env.example`](seally.env.example) | 需要追加到 Seafile `.env` 的配置模板 |
| [接入已有 Seafile](docs/deploy-with-existing-seafile.md) | 不重装 Seafile，向现有实例增加 Seally |
| [全新联合部署](docs/deploy-with-new-seafile.md) | 按版本和版本类型下载官方文件后一次部署 |
| [配置参考](docs/configuration.md) | 必填变量、缓存、域名、存储和可选网盘配置 |
| [故障排查](docs/troubleshooting.md) | 容器、网络、数据库、缓存、SSO 常见问题 |

## 部署前须知

1. 生产环境必须使用 Seally 的完整版本标签或镜像 digest，不要使用 `latest`。
2. `JWT_PRIVATE_KEY` 与 Seafile 共用，不要修改已有部署中的非空值。
3. `SM_SESSION_SECRET` 是 Seally 独立密钥，使用 `openssl rand -hex 32` 生成。
   它同时参与会话签名、网盘凭据加密和授权指纹生成；上线后不要更换，并与数据库
   备份分开保管。
4. Seally 必须与 Seafile 挂载同一个 `SEAFILE_VOLUME`，且都映射到容器内 `/shared`。
5. Seally 会在 Seafile 数据库中创建自身所需的表。首次接入前请先完成数据库和
   `${SEAFILE_VOLUME}` 的备份。

## 上游资料

本仓库的 Seafile 版本差异以官方资料为准：

- [Seafile 12 CE Docker 部署](https://manual.seafile.com/12.0/setup/setup_ce_by_docker/)
- [Seafile 12 Pro Docker 部署](https://manual.seafile.com/12.0/setup/setup_pro_by_docker/)
- [Seafile 13 CE Docker 部署](https://manual.seafile.com/13.0/setup/setup_ce_by_docker/)
- [Seafile 13 Pro Docker 部署](https://manual.seafile.com/13.0/setup/setup_pro_by_docker/)
- [Seafile 13 环境变量参考](https://manual.seafile.com/13.0/config/env/)
- [Seahub 自定义导航](https://manual.seafile.com/latest/config/seahub_customization/)

## 获取帮助

部署问题请在本仓库提交 Issue，并附上以下脱敏信息：Seafile 大版本、CE/Pro、
`docker compose config --services`、`docker compose ps`，以及 `docker compose logs
--tail=200 seally` 和 `/opt/seafile/logs/seally.log` 中相关错误的脱敏输出。请勿提交
`.env`、数据库密码、OAuth 凭据或任何密钥。
