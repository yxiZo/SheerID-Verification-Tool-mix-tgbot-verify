# SheerID Telegram Bot Docker 部署文档

本文档适用于使用仓库内 `docker-compose.yml` 的部署方式。该方案会同时启动：

- `tgbot`：Telegram 机器人
- `mysql`：MySQL 8.0 数据库

MySQL 运行在 Docker 内部网络中，不需要在宿主机安装 MySQL，也不要求连接外部数据库。数据库数据保存在 Docker 命名卷 `mysql_data` 中。

---

## 1. 部署架构

```text
Telegram
   |
   v
tgbot 容器 ---> mysql 容器
                  |
                  v
             mysql_data 数据卷
```

机器人通过 Docker Compose 服务名 `mysql` 访问数据库。MySQL 端口默认不映射到宿主机，因此不会占用或暴露宿主机的 `3306` 端口。

---

## 2. 环境要求

- Linux、macOS 或安装了 Docker Desktop 的 Windows
- Docker Engine 24.0+
- Docker Compose v2
- 建议至少 2 GB 内存
- 建议至少 5 GB 可用磁盘空间
- 能够访问 Telegram API、Docker Hub 和 Playwright 下载源

检查 Docker 环境：

```bash
docker --version
docker compose version
```

---

## 3. 获取项目

```bash
git clone <你的仓库地址>
cd <项目目录>
```

确认当前目录包含以下文件：

```text
docker-compose.yml
Dockerfile
env.example
requirements.txt
bot.py
```

---

## 4. 准备环境变量

复制配置模板：

```bash
cp env.example .env
```

编辑 `.env`：

```bash
nano .env
```

必须修改的配置：

```env
# Telegram Bot
BOT_TOKEN=从BotFather获取的Token
ADMIN_USER_ID=你的Telegram数字ID

# MySQL
MYSQL_USER=tgbot_user
MYSQL_PASSWORD=请设置强密码
MYSQL_DATABASE=tgbot_verify
MYSQL_ROOT_PASSWORD=请设置另一个强密码
```

可选配置：

```env
CHANNEL_USERNAME=pk_oa
CHANNEL_URL=https://t.me/pk_oa
MYSQL_PORT=3306
```

配置说明：

| 变量 | 必填 | 说明 |
| --- | --- | --- |
| `BOT_TOKEN` | 是 | Telegram Bot Token |
| `ADMIN_USER_ID` | 是 | 管理员 Telegram 数字 ID |
| `CHANNEL_USERNAME` | 否 | 频道用户名，不需要 `@` |
| `CHANNEL_URL` | 否 | Telegram 频道链接 |
| `MYSQL_USER` | 是 | 业务数据库用户 |
| `MYSQL_PASSWORD` | 是 | 业务数据库用户密码 |
| `MYSQL_DATABASE` | 是 | 数据库名称 |
| `MYSQL_ROOT_PASSWORD` | 是 | MySQL root 密码 |
| `MYSQL_PORT` | 否 | 容器内 MySQL 端口，默认 `3306` |
| `TZ` | 否 | 时区，默认 `Asia/Shanghai` |

注意：

- 使用内置 MySQL 时，不要在 `.env` 中设置 `MYSQL_HOST=localhost`。
- `docker-compose.yml` 会在未设置 `MYSQL_HOST` 时自动使用 `MYSQL_HOST=mysql`。
- `.env` 包含密码，请勿提交到 Git 仓库。

---

## 5. 启动服务

构建镜像并启动：

```bash
docker compose up -d --build
```

首次启动需要下载 MySQL 镜像、安装 Python 依赖和 Playwright Chromium，耗时可能较长。

查看容器状态：

```bash
docker compose ps
```

正常情况下应看到：

```text
sheerid-mysql    ...    Up (healthy)
sheerid-tgbot    ...    Up
```

机器人会等待 MySQL 健康检查通过后再启动。

---

## 6. 查看日志

查看全部服务日志：

```bash
docker compose logs -f
```

只查看机器人日志：

```bash
docker compose logs -f tgbot
```

只查看 MySQL 日志：

```bash
docker compose logs -f mysql
```

查看最近 100 行日志：

```bash
docker compose logs --tail=100 tgbot
```

---

## 7. 验证部署

### 7.1 检查容器状态

```bash
docker compose ps
```

`mysql` 应处于 `healthy` 状态，`tgbot` 应处于 `Up` 状态。

### 7.2 检查数据库

进入 MySQL 容器：

```bash
docker compose exec mysql sh
```

在容器内登录 MySQL：

```bash
mysql -u"$MYSQL_USER" -p"$MYSQL_PASSWORD" "$MYSQL_DATABASE"
```

查看数据表：

```sql
SHOW TABLES;
```

应至少看到以下表：

```text
users
invitations
verifications
card_keys
card_key_usage
```

退出：

```sql
EXIT;
exit
```

### 7.3 检查机器人

在 Telegram 中向机器人发送：

```text
/start
```

然后检查积分：

```text
/balance
```

如果能够正常注册并返回积分，说明 Telegram API 和 MySQL 均连接正常。

---

## 8. 日常运维

### 重启服务

```bash
docker compose restart
```

只重启机器人：

```bash
docker compose restart tgbot
```

### 停止服务

```bash
docker compose stop
```

### 重新启动已停止的服务

```bash
docker compose start
```

### 停止并删除容器

```bash
docker compose down
```

该命令不会删除 `mysql_data` 数据卷，数据库数据仍会保留。

### 查看资源占用

```bash
docker stats
```

---

## 9. 更新版本

更新前先备份数据库：

```bash
docker compose exec -T mysql sh -c \
  'mysqldump -uroot -p"$MYSQL_ROOT_PASSWORD" --single-transaction "$MYSQL_DATABASE"' \
  > backup_before_update.sql
```

拉取代码并重新构建：

```bash
git pull
docker compose up -d --build
```

查看更新后的日志：

```bash
docker compose logs -f tgbot
```

`docker compose down` 不会删除数据库数据卷；不要使用 `docker compose down -v`，除非确认要删除全部数据库数据。

---

## 10. 数据库备份

创建备份目录：

```bash
mkdir -p backups
```

备份数据库：

```bash
docker compose exec -T mysql sh -c \
  'mysqldump -uroot -p"$MYSQL_ROOT_PASSWORD" \
  --single-transaction --routines --triggers "$MYSQL_DATABASE"' \
  > "backups/tgbot_verify_$(date +%Y%m%d_%H%M%S).sql"
```

检查备份文件：

```bash
ls -lh backups/
```

建议将备份文件同步到另一台机器或对象存储，避免宿主机故障导致数据丢失。

---

## 11. 数据库恢复

停止机器人，避免恢复期间继续写入：

```bash
docker compose stop tgbot
```

恢复数据库：

```bash
cat backups/your_backup.sql | docker compose exec -T mysql sh -c \
  'mysql -uroot -p"$MYSQL_ROOT_PASSWORD" "$MYSQL_DATABASE"'
```

重新启动机器人：

```bash
docker compose start tgbot
docker compose logs -f tgbot
```

恢复完成后，建议在 Telegram 中测试 `/start`、`/balance` 和管理员命令。

---

## 12. 常见问题

### 12.1 MySQL 一直处于 starting 或 unhealthy

查看状态和日志：

```bash
docker compose ps
docker compose logs mysql
```

常见原因：

- 宿主机内存不足
- `MYSQL_ROOT_PASSWORD`、`MYSQL_PASSWORD` 等配置为空
- Docker 数据卷权限异常
- MySQL 首次初始化尚未完成

首次启动通常需要等待 30 至 90 秒。

### 12.2 机器人提示数据库连接失败

检查 MySQL 是否健康：

```bash
docker compose ps mysql
```

检查机器人日志：

```bash
docker compose logs --tail=100 tgbot
```

确认 `.env` 中没有设置：

```env
MYSQL_HOST=localhost
```

内置 MySQL 部署下应删除该配置，或设置为：

```env
MYSQL_HOST=mysql
```

### 12.3 修改密码后数据库无法登录

`MYSQL_*` 初始化变量只在数据卷为空、MySQL 第一次初始化时生效。已有 `mysql_data` 数据卷时，修改 `.env` 中的密码不会自动修改数据库中的密码。

如果只是测试环境，可以删除数据卷后重新初始化：

```bash
docker compose down -v
docker compose up -d --build
```

警告：`docker compose down -v` 会永久删除数据库数据，执行前必须先备份。

生产环境应登录 MySQL 后使用 `ALTER USER` 修改密码，并同步更新 `.env`。

### 12.4 宿主机无法连接 3306 端口

这是预期行为。内置 MySQL 默认只在 Docker 内部网络提供服务，不映射宿主机端口。

如需临时执行数据库命令：

```bash
docker compose exec mysql sh -c \
  'mysql -u"$MYSQL_USER" -p"$MYSQL_PASSWORD" "$MYSQL_DATABASE"'
```

不建议将 MySQL 端口直接暴露到公网。

### 12.5 Playwright Chromium 启动失败

重新安装浏览器依赖：

```bash
docker compose exec tgbot playwright install --with-deps chromium
```

然后重启机器人：

```bash
docker compose restart tgbot
```

### 12.6 Bot Token 无效

检查 `.env` 中的 `BOT_TOKEN`：

- 不要包含引号或多余空格
- 确认 Token 未被 BotFather 撤销
- 修改后执行 `docker compose up -d`

---

## 13. 安全建议

1. 使用长度足够的随机密码，不要保留示例密码。
2. 不要将 `.env`、数据库备份提交到 Git。
3. 不要将 MySQL 端口映射到公网。
4. 定期备份 `mysql_data` 对应的数据库。
5. 服务器只开放 SSH 等必要端口。
6. 定期更新 Docker 镜像和项目代码。
7. 管理员账号应开启 Telegram 两步验证。

---

## 14. 卸载

停止并删除容器，但保留数据库数据：

```bash
docker compose down
```

删除容器和数据库数据：

```bash
docker compose down -v
```

删除项目镜像：

```bash
docker image rm sheerid-verification-tool-mix-tgbot-verify-tgbot
```

警告：删除数据卷后数据库无法恢复，除非提前完成备份。

---

## 15. 常用命令速查

```bash
# 启动或更新
docker compose up -d --build

# 查看状态
docker compose ps

# 查看机器人日志
docker compose logs -f tgbot

# 查看 MySQL 日志
docker compose logs -f mysql

# 重启机器人
docker compose restart tgbot

# 停止服务
docker compose stop

# 删除容器，保留数据库
docker compose down

# 备份数据库
docker compose exec -T mysql sh -c \
  'mysqldump -uroot -p"$MYSQL_ROOT_PASSWORD" --single-transaction "$MYSQL_DATABASE"' \
  > backup.sql

# 进入 MySQL 容器
docker compose exec mysql sh
```

