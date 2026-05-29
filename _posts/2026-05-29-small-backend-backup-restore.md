---
layout: post
title: "一次小型app后端的备份与恢复实践"
date: 2026-05-29 12:00:00 +0800
categories: [Development]
tags: [postgresql, docker-compose, backup, restore, rclone, cloudflare-r2]
---

很多小型后端项目在准备上线部署时，往往会把巨大的精力花在搭建与编排上，而最容易被侥幸忽略的则是**“灾难恢复”**。

定期生成的备份文件并不等于系统真的具备可恢复性。真正可靠的备份恢复方案，至少必须掷地有声地回答以下三个问题：

1. 如果生产数据库崩溃，现有的数据库冷备文件能不能迅速导入并运转起来？
2. 用户上传的文件目录（Uploads）能不能与数据库的快照点切切配合，回到基本吻合的时间点？
3. 如果整台基础宿主服务器彻底报废导致本机备份全灭，能不能利用远端仓库顺畅拉回所有物料重建生产？

本文整理了一套我在管理小型 Docker Compose 后端架构中长期使用的备份与恢复落地实践。以极简的结构示范：后端由 `app` 与 `db` 两个服务容器构成，数据库为 `PostgreSQL`，普通用户上传的多媒体资产放置于当前工作目录下的 `uploads/`。针对远端存放灾备库，将采用极具性价比的 Cloudflare R2（你完全可以平行替换为任何兼容标准 S3 的对象存储引擎）。

## 一、目录与约定

本文使用下面这些占位路径：

```bash
APP_DIR="/opt/example/example-backend"
BACKUP_DIR="/opt/example/backups"
RESTORE_ROOT="/opt/example/restore"
COMPOSE_FILE="docker-compose.prod.yml"
```

Docker Compose 中假设有两个服务：

```text
app   # 后端应用容器
db    # PostgreSQL 容器
```

PostgreSQL 的 `POSTGRES_USER` 和 `POSTGRES_DB` 来自 `db` 容器内部的环境变量，而不是宿主机的环境变量。这个细节很重要，后面会专门说明。

先创建本机备份目录：

```bash
sudo mkdir -p /opt/example/backups /opt/example/scripts /opt/example/restore
sudo chown -R deploy:deploy /opt/example/backups /opt/example/scripts /opt/example/restore
```

## 二、本机数据库备份

最基础的 PostgreSQL 备份可以直接用 `pg_dump`：

```bash
cd /opt/example/example-backend

docker compose -f docker-compose.prod.yml exec -T db sh -lc \
  'pg_dump -U "$POSTGRES_USER" "$POSTGRES_DB"' \
  > /opt/example/backups/postgres_$(date +%F_%H%M%S).sql
```

这里有一个容易踩的坑：

```bash
'pg_dump -U "$POSTGRES_USER" "$POSTGRES_DB"'
```

**注意：** 这段命令故意放在单引号里，让 `$POSTGRES_USER` 和 `$POSTGRES_DB` 在 `db` 容器内部展开，而不是在宿主机上展开。如果写成双引号且宿主机缺乏同名变量，PostgreSQL 客户端可能会退回使用当前系统用户（例如 `root`）连接，报出 `role "root" does not exist`。因此在容器化部署执行备份时，更推荐让数据库容器内部读取对应环境变量。

## 三、uploads 目录备份

很多小型后端的数据不只在数据库里。用户头像、图片、附件、导入文件等，通常还会落在本机目录中。

假设上传目录是项目根目录下的 `uploads/`，可以这样打包：

```bash
tar -C /opt/example/example-backend \
  -czf /opt/example/backups/uploads_$(date +%F_%H%M%S).tar.gz \
  uploads
```

**打包路径建议：** 使用 `tar -C APP_DIR uploads` 时，备份包里的路径会是相对路径 `uploads/...`，而不是宿主机上的绝对路径 `/opt/example/example-backend/uploads/...`。这样迁移到新服务器或新路径时更灵活，避免路径覆盖和多余层级。

## 四、先做不覆盖生产库的恢复演练

**恢复演练：** 备份最怕的是“看起来成功，但真的灾难降临时却恢复失败”。建议定期把最近的一份 SQL 备份导入到独立的**临时库**中，而不是直接覆盖生产库，用来验证最底层的“备份文件能否被 PostgreSQL 正常导入”。

```bash
cd /opt/example/example-backend

LATEST_SQL="$(ls -t /opt/example/backups/postgres_*.sql | head -n 1)"

docker compose -f docker-compose.prod.yml exec -T db sh -lc \
  'dropdb --if-exists -U "$POSTGRES_USER" restore_test'

docker compose -f docker-compose.prod.yml exec -T db sh -lc \
  'createdb -U "$POSTGRES_USER" restore_test'

docker compose -f docker-compose.prod.yml exec -T db sh -lc \
  'psql -v ON_ERROR_STOP=1 -U "$POSTGRES_USER" -d restore_test' < "${LATEST_SQL}"

docker compose -f docker-compose.prod.yml exec -T db sh -lc \
  'psql -U "$POSTGRES_USER" -d restore_test -c "\dt"'
```

如果能看到表列表，至少说明这个 SQL 文件可以被 PostgreSQL 正常导入。

演练结束后删除临时库：

```bash
docker compose -f docker-compose.prod.yml exec -T db sh -lc \
  'dropdb --if-exists -U "$POSTGRES_USER" restore_test'
```

这里的目标不是验证所有业务数据完全正确，而是先把最低层的“备份文件能不能导入”验证掉。

## 五、每日本机自动备份脚本

手工命令只适合验证流程。真正上线后，应该把它变成自动任务。

创建脚本：

```bash
sudo nano /opt/example/scripts/backup-local.sh
```

写入以下脚本内容：

```bash
#!/usr/bin/env bash
set -euo pipefail

PATH="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"

APP_DIR="/opt/example/example-backend"
BACKUP_DIR="/opt/example/backups"
COMPOSE_FILE="docker-compose.prod.yml"
LOCAL_RETENTION_DAYS="7"
LOCK_FILE="${BACKUP_DIR}/backup-local.lock"

mkdir -p "${BACKUP_DIR}"

command -v docker >/dev/null 2>&1 || {
  echo "Missing required command: docker" >&2
  exit 1
}

command -v flock >/dev/null 2>&1 || {
  echo "Missing required command: flock" >&2
  exit 1
}

exec 9>"${LOCK_FILE}"
flock -n 9 || {
  echo "Another backup job is already running." >&2
  exit 1
}

cd "${APP_DIR}"

# 如果生产环境变量文件不在 compose 默认位置，可以在这里指定：
# export APP_ENV_FILE="/opt/example/secrets/prod.env"

ts="$(date +%F_%H%M%S)"

# 1. 备份 PostgreSQL
docker compose -f "${COMPOSE_FILE}" exec -T db sh -lc \
  'pg_dump -U "$POSTGRES_USER" "$POSTGRES_DB"' \
  > "${BACKUP_DIR}/postgres_${ts}.sql"

# 2. 备份 uploads 目录
tar -C "${APP_DIR}" -czf "${BACKUP_DIR}/uploads_${ts}.tar.gz" uploads

# 3. 清理本机旧备份
find "${BACKUP_DIR}" -type f -name 'postgres_*.sql' -mtime +"${LOCAL_RETENTION_DAYS}" -delete
find "${BACKUP_DIR}" -type f -name 'uploads_*.tar.gz' -mtime +"${LOCAL_RETENTION_DAYS}" -delete
```
{: file="/opt/example/scripts/backup-local.sh" }

赋予执行权限并手动运行一次：

```bash
chmod 700 /opt/example/scripts/backup-local.sh
/opt/example/scripts/backup-local.sh
ls -lah /opt/example/backups
```

确认无误后，加入 `deploy` 用户的 crontab：

```bash
crontab -e
```

每天凌晨 3 点执行：

```cron
0 3 * * * /opt/example/scripts/backup-local.sh >> /opt/example/backups/backup-local.log 2>&1
```

本机备份能防误删、误改、升级失败，但不能防整台服务器损坏、系统盘异常、账号被误删等问题。因此生产环境最好再做一份异地备份。

## 六、远端备份到 Cloudflare R2

Cloudflare R2 可以当作一个兼容 S3 的对象存储，用来保存备份压缩包。这里它只作为备份仓库，不改变应用运行时的文件存储方式：

- 应用仍然读写本机 `uploads/`
- R2 只保存数据库和 uploads 的备份包
- R2 bucket 保持 private
- 不上传明文环境变量文件、私钥或第三方服务凭证

一个比较简单的保留策略是：

| 位置                        | 建议保留 | 作用                     |
| --------------------------- | -------- | ------------------------ |
| 本机 `/opt/example/backups` | 7 天     | 快速回滚、防误删         |
| Cloudflare R2               | 90 天    | 防服务器损坏、系统盘异常 |

### 准备 R2 bucket 和 rclone

在 Cloudflare R2 创建一个专门用于备份的 private bucket，例如：

```text
example-prod-backups
```

对象路径可以统一加环境前缀：

```text
prod/
```

最终结构类似：

```text
example-prod-backups/
  prod/
    2026-05-29_030000/
      postgres.sql.gz
      uploads.tar.gz
      backup_info.txt
      SHA256SUMS
```

**最小权限原则（Least Privilege）：** 创建 R2 API token 时务必遵守以下规则：
- 仅赋予该备份 Bucket 的 `Object Read & Write` 权限。
- 坚决避免使用全局 Account Token。
- 绝对不要将 Token 明文写入 Git 仓库。
- `Secret Access Key` 只应该存放到密码管理器及受限的服务器环境文件中。

安装 rclone：

```bash
sudo -v
curl https://rclone.org/install.sh | sudo bash
rclone version
```

配置 rclone：

```bash
mkdir -p ~/.config/rclone
chmod 700 ~/.config/rclone
nano ~/.config/rclone/rclone.conf
```

在此文件中写入对应对象存储的连接信息，示例配置如下：

```ini
[r2-backup]
type = s3
provider = Cloudflare
access_key_id = <R2_ACCESS_KEY_ID>
secret_access_key = <R2_SECRET_ACCESS_KEY>
endpoint = https://<ACCOUNT_ID>.r2.cloudflarestorage.com
acl = private
no_check_bucket = true
```
{: file="~/.config/rclone/rclone.conf" }

收紧权限：

```bash
chmod 600 ~/.config/rclone/rclone.conf
```

测试连接：

```bash
rclone lsf r2-backup:example-prod-backups

echo "r2 backup test $(date -Is)" > /tmp/r2-backup-test.txt
rclone copy /tmp/r2-backup-test.txt r2-backup:example-prod-backups/prod/_connection-test/
rclone lsf r2-backup:example-prod-backups/prod/_connection-test/
rclone delete r2-backup:example-prod-backups/prod/_connection-test/ --rmdirs
rm -f /tmp/r2-backup-test.txt
```

### R2 自动备份脚本

创建脚本：

```bash
sudo nano /opt/example/scripts/backup-r2.sh
```

写入以下脚本内容：

```bash
#!/usr/bin/env bash
set -euo pipefail
umask 077

PATH="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"

APP_DIR="/opt/example/example-backend"
COMPOSE_FILE="docker-compose.prod.yml"
LOCAL_BACKUP_ROOT="/opt/example/backups"
LOCK_FILE="${LOCAL_BACKUP_ROOT}/backup-r2.lock"

RCLONE_REMOTE="r2-backup"
R2_BUCKET="example-prod-backups"
R2_PREFIX="prod"

LOCAL_RETENTION_DAYS="7"
R2_RETENTION_DAYS="90"

TIMESTAMP="$(date +%F_%H%M%S)"
WORK_DIR="${LOCAL_BACKUP_ROOT}/${TIMESTAMP}"
R2_TARGET="${RCLONE_REMOTE}:${R2_BUCKET}/${R2_PREFIX}/${TIMESTAMP}"
R2_PRUNE_ROOT="${RCLONE_REMOTE}:${R2_BUCKET}/${R2_PREFIX}"

log() {
  echo "[$(date '+%F %T')] $*"
}

require_command() {
  if ! command -v "$1" >/dev/null 2>&1; then
    echo "Missing required command: $1" >&2
    exit 1
  fi
}

require_command docker
require_command rclone
require_command gzip
require_command tar
require_command sha256sum
require_command flock

mkdir -p "${LOCAL_BACKUP_ROOT}"

exec 9>"${LOCK_FILE}"
flock -n 9 || {
  echo "Another R2 backup job is already running." >&2
  exit 1
}

mkdir -p "${WORK_DIR}"
cd "${APP_DIR}"

# 如果生产环境变量文件不在 compose 默认位置，可以在这里指定：
# export APP_ENV_FILE="/opt/example/secrets/prod.env"

POSTGRES_USER_IN_CONTAINER="$(docker compose -f "${COMPOSE_FILE}" exec -T db sh -lc 'printf "%s" "$POSTGRES_USER"')"
POSTGRES_DB_IN_CONTAINER="$(docker compose -f "${COMPOSE_FILE}" exec -T db sh -lc 'printf "%s" "$POSTGRES_DB"')"

log "Dumping PostgreSQL database"
docker compose -f "${COMPOSE_FILE}" exec -T db sh -lc \
  'pg_dump -U "$POSTGRES_USER" "$POSTGRES_DB"' | gzip -9 > "${WORK_DIR}/postgres.sql.gz"

log "Archiving uploads directory"
tar -C "${APP_DIR}" -czf "${WORK_DIR}/uploads.tar.gz" uploads

log "Writing backup metadata"
{
  echo "timestamp=${TIMESTAMP}"
  echo "host=$(hostname)"
  echo "app_dir=${APP_DIR}"
  echo "compose_file=${COMPOSE_FILE}"
  echo "postgres_user=${POSTGRES_USER_IN_CONTAINER}"
  echo "postgres_db=${POSTGRES_DB_IN_CONTAINER}"
  echo "git_revision=$(git -C "${APP_DIR}" rev-parse HEAD 2>/dev/null || true)"
  echo "r2_bucket=${R2_BUCKET}"
  echo "r2_prefix=${R2_PREFIX}"
} > "${WORK_DIR}/backup_info.txt"

log "Generating SHA256 checksums"
(
  cd "${WORK_DIR}"
  sha256sum postgres.sql.gz uploads.tar.gz backup_info.txt > SHA256SUMS
)

log "Uploading backup to R2: ${R2_TARGET}"
rclone copy "${WORK_DIR}" "${R2_TARGET}" \
  --transfers 4 \
  --checkers 8 \
  --s3-upload-cutoff 64M \
  --s3-chunk-size 64M

log "Verifying uploaded objects by size"
rclone check "${WORK_DIR}" "${R2_TARGET}" --one-way --size-only

log "Pruning local backup directories older than ${LOCAL_RETENTION_DAYS} days"
find "${LOCAL_BACKUP_ROOT}" -mindepth 1 -maxdepth 1 -type d -mtime +"${LOCAL_RETENTION_DAYS}" -exec rm -rf {} +

log "Pruning R2 backup objects older than ${R2_RETENTION_DAYS} days"
rclone delete "${R2_PRUNE_ROOT}" --min-age "${R2_RETENTION_DAYS}d" --rmdirs

log "Backup job completed successfully"
```
{: file="/opt/example/scripts/backup-r2.sh" }

**备份策略建议：** 如果启用 R2 备份脚本，通常不需要再同时执行本机自动备份脚本。因为 R2 脚本本身也会在本机快照目录中保留最近几天的完整备份文件。

赋予权限并手动执行：

```bash
chmod 700 /opt/example/scripts/backup-r2.sh
/opt/example/scripts/backup-r2.sh
ls -lah /opt/example/backups
rclone lsf r2-backup:example-prod-backups/prod/
```

加入 `deploy` 用户的 crontab：

```cron
0 3 * * * /opt/example/scripts/backup-r2.sh >> /opt/example/backups/backup-r2.log 2>&1
```

第二天检查：

```bash
tail -n 100 /opt/example/backups/backup-r2.log
rclone lsf r2-backup:example-prod-backups/prod/
```

## 七、本机生产恢复脚本

真正恢复生产库是破坏性操作，所以我不建议只靠几条手工命令完成。更好的方式是写成脚本，并且加入以下保护：

- 恢复前检查备份文件是否存在、是否可读
- `.sql.gz` 先做 gzip 完整性检查
- `uploads.tar.gz` 先做 tar 完整性检查
- 使用 `flock` 防止多个恢复任务并发执行
- 从 `db` 容器内部读取生产数据库名
- 默认要求输入 `RESTORE <database_name>` 才继续
- 恢复前停止 app，避免继续写入
- 原 `uploads/` 不直接删除，而是先改名为 `uploads.before-restore.<timestamp>`
- 恢复失败时不盲目启动 app

创建脚本：

```bash
nano /opt/example/scripts/restore-prod.sh
```

脚本如下：

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

APP_DIR="${APP_DIR:-/opt/example/example-backend}"
BACKUP_DIR="${BACKUP_DIR:-/opt/example/backups}"
COMPOSE_FILE="${COMPOSE_FILE:-docker-compose.prod.yml}"
SQL_BACKUP=""
UPLOADS_BACKUP=""
SKIP_UPLOADS=0
ASSUME_YES=0
START_APP=1
CHOWN_TO="${CHOWN_TO:-deploy:deploy}"

usage() {
  cat <<'USAGE'
用法：
  restore-prod.sh [options]

常用示例：
  restore-prod.sh
  restore-prod.sh --sql /opt/example/backups/postgres_2026-05-29_030000.sql \
                  --uploads /opt/example/backups/uploads_2026-05-29_030000.tar.gz
  restore-prod.sh --skip-uploads

选项：
  --app-dir DIR        后端目录
  --backup-dir DIR     备份目录
  --compose-file FILE  compose 文件名或路径
  --sql FILE           SQL 备份，支持 .sql 和 .sql.gz
  --uploads FILE       uploads 压缩包
  --skip-uploads       只恢复数据库，不替换 uploads/
  --env-file FILE      导出 APP_ENV_FILE，供 compose 使用
  --no-start-app       恢复完成后不启动 app
  --yes                跳过交互式确认，谨慎使用
  -h, --help           显示帮助
USAGE
}

log() {
  printf '[%s] %s\n' "$(date '+%F %T')" "$*"
}

die() {
  printf 'ERROR: %s\n' "$*" >&2
  exit 1
}

require_command() {
  command -v "$1" >/dev/null 2>&1 || die "Missing required command: $1"
}

while [[ $# -gt 0 ]]; do
  case "$1" in
    --app-dir) APP_DIR="${2:?--app-dir requires a value}"; shift 2 ;;
    --backup-dir) BACKUP_DIR="${2:?--backup-dir requires a value}"; shift 2 ;;
    --compose-file) COMPOSE_FILE="${2:?--compose-file requires a value}"; shift 2 ;;
    --sql) SQL_BACKUP="${2:?--sql requires a value}"; shift 2 ;;
    --uploads) UPLOADS_BACKUP="${2:?--uploads requires a value}"; shift 2 ;;
    --skip-uploads) SKIP_UPLOADS=1; shift ;;
    --env-file) export APP_ENV_FILE="${2:?--env-file requires a value}"; shift 2 ;;
    --no-start-app) START_APP=0; shift ;;
    --yes) ASSUME_YES=1; shift ;;
    -h|--help) usage; exit 0 ;;
    *) usage >&2; die "Unknown option: $1" ;;
  esac
done

require_command docker
require_command find
require_command sort
require_command head
require_command cut
require_command tar
require_command flock
require_command grep
require_command mktemp

[[ -d "$APP_DIR" ]] || die "APP_DIR does not exist: $APP_DIR"
[[ -d "$BACKUP_DIR" ]] || die "BACKUP_DIR does not exist: $BACKUP_DIR"

cd "$APP_DIR"
[[ -f "$COMPOSE_FILE" ]] || die "Compose file not found from APP_DIR: $COMPOSE_FILE"

compose() {
  docker compose -f "$COMPOSE_FILE" "$@"
}

latest_sql_backup() {
  find "$BACKUP_DIR" -maxdepth 1 -type f \( -name 'postgres_*.sql' -o -name 'postgres_*.sql.gz' \) \
    -printf '%T@ %p\n' | sort -nr | head -n 1 | cut -d' ' -f2-
}

latest_uploads_backup() {
  find "$BACKUP_DIR" -maxdepth 1 -type f -name 'uploads_*.tar.gz' \
    -printf '%T@ %p\n' | sort -nr | head -n 1 | cut -d' ' -f2-
}

if [[ -z "$SQL_BACKUP" ]]; then
  SQL_BACKUP="$(latest_sql_backup)"
fi
[[ -n "$SQL_BACKUP" ]] || die "No SQL backup found in $BACKUP_DIR"
[[ -r "$SQL_BACKUP" ]] || die "SQL backup is not readable: $SQL_BACKUP"

if [[ "$SQL_BACKUP" == *.gz ]]; then
  require_command gzip
  gzip -t "$SQL_BACKUP" || die "SQL gzip backup failed integrity check: $SQL_BACKUP"
fi

if [[ "$SKIP_UPLOADS" -eq 0 ]]; then
  if [[ -z "$UPLOADS_BACKUP" ]]; then
    UPLOADS_BACKUP="$(latest_uploads_backup)"
  fi
  [[ -n "$UPLOADS_BACKUP" ]] || die "No uploads backup found. Use --skip-uploads to restore database only."
  [[ -r "$UPLOADS_BACKUP" ]] || die "Uploads backup is not readable: $UPLOADS_BACKUP"
  tar -tzf "$UPLOADS_BACKUP" >/dev/null || die "Uploads tarball failed integrity check: $UPLOADS_BACKUP"
fi

LOCK_FILE="${BACKUP_DIR}/restore-prod.lock"
exec 9>"$LOCK_FILE"
flock -n 9 || die "Another restore job is already running: $LOCK_FILE"

log "Checking docker compose configuration"
compose config >/dev/null

log "Ensuring database container is running"
compose up -d db

log "Waiting for PostgreSQL to accept connections"
for _ in {1..30}; do
  if compose exec -T db sh -lc 'pg_isready -U "$POSTGRES_USER" -d "$POSTGRES_DB" >/dev/null 2>&1'; then
    break
  fi
  sleep 2
done

compose exec -T db sh -lc 'pg_isready -U "$POSTGRES_USER" -d "$POSTGRES_DB" >/dev/null 2>&1' \
  || die "PostgreSQL did not become ready"

POSTGRES_USER_IN_CONTAINER="$(compose exec -T db sh -lc 'printf "%s" "$POSTGRES_USER"')"
POSTGRES_DB_IN_CONTAINER="$(compose exec -T db sh -lc 'printf "%s" "$POSTGRES_DB"')"

[[ -n "$POSTGRES_USER_IN_CONTAINER" ]] || die "POSTGRES_USER is empty inside db container"
[[ -n "$POSTGRES_DB_IN_CONTAINER" ]] || die "POSTGRES_DB is empty inside db container"

cat <<SUMMARY

About to restore production data.

App dir:        $APP_DIR
Compose file:   $COMPOSE_FILE
Database user:  $POSTGRES_USER_IN_CONTAINER
Database name:  $POSTGRES_DB_IN_CONTAINER
SQL backup:     $SQL_BACKUP
Uploads backup: $([[ "$SKIP_UPLOADS" -eq 1 ]] && printf 'SKIPPED' || printf '%s' "$UPLOADS_BACKUP")

This will DROP and recreate the configured production database.
SUMMARY

if [[ "$ASSUME_YES" -eq 0 ]]; then
  printf '\nType exactly "RESTORE %s" to continue: ' "$POSTGRES_DB_IN_CONTAINER"
  read -r confirmation
  [[ "$confirmation" == "RESTORE $POSTGRES_DB_IN_CONTAINER" ]] \
    || die "Confirmation did not match. Restore cancelled."
else
  log "--yes supplied; skipping interactive confirmation"
fi

RESTORE_TS="$(date +%F_%H%M%S)"

log "Stopping app before database restore"
compose stop app

restore_failed=1
trap 'rc=$?; if [[ $rc -ne 0 && $restore_failed -eq 1 ]]; then printf "ERROR: restore failed; app may still be stopped. Inspect docker compose logs before starting it again.\n" >&2; fi' EXIT

log "Dropping production database: $POSTGRES_DB_IN_CONTAINER"
compose exec -T db sh -lc 'dropdb --if-exists --force -U "$POSTGRES_USER" "$POSTGRES_DB"'

log "Creating production database: $POSTGRES_DB_IN_CONTAINER"
compose exec -T db sh -lc 'createdb -U "$POSTGRES_USER" "$POSTGRES_DB"'

log "Importing SQL backup"
if [[ "$SQL_BACKUP" == *.gz ]]; then
  gzip -dc "$SQL_BACKUP" | compose exec -T db sh -lc 'psql -v ON_ERROR_STOP=1 -U "$POSTGRES_USER" -d "$POSTGRES_DB"'
else
  compose exec -T db sh -lc 'psql -v ON_ERROR_STOP=1 -U "$POSTGRES_USER" -d "$POSTGRES_DB"' < "$SQL_BACKUP"
fi

if [[ "$SKIP_UPLOADS" -eq 0 ]]; then
  log "Replacing uploads directory from backup"
  UPLOADS_LISTING="$(mktemp)"
  tar -tzf "$UPLOADS_BACKUP" > "$UPLOADS_LISTING"

  if grep -qE '^uploads(/|$)' "$UPLOADS_LISTING"; then
    UPLOADS_LAYOUT="relative"
  elif grep -qE '^opt/example/example-backend/uploads(/|$)' "$UPLOADS_LISTING"; then
    UPLOADS_LAYOUT="legacy-absolute"
  else
    rm -f "$UPLOADS_LISTING"
    die "Uploads tarball does not contain uploads/ in a recognized layout: $UPLOADS_BACKUP"
  fi
  rm -f "$UPLOADS_LISTING"

  if [[ -e "$APP_DIR/uploads" ]]; then
    mv "$APP_DIR/uploads" "$APP_DIR/uploads.before-restore.$RESTORE_TS"
    log "Existing uploads moved to uploads.before-restore.$RESTORE_TS"
  fi

  if [[ "$UPLOADS_LAYOUT" == "relative" ]]; then
    tar -xzf "$UPLOADS_BACKUP" -C "$APP_DIR"
  else
    tar -xzf "$UPLOADS_BACKUP" -C /
  fi

  if [[ -n "$CHOWN_TO" ]]; then
    log "Setting uploads ownership to $CHOWN_TO"
    if [[ "$(id -u)" -eq 0 ]]; then
      chown -R "$CHOWN_TO" "$APP_DIR/uploads"
    elif command -v sudo >/dev/null 2>&1; then
      sudo chown -R "$CHOWN_TO" "$APP_DIR/uploads"
    else
      log "sudo not found; skipping ownership change. Set CHOWN_TO= to silence this step."
    fi
  fi
else
  log "Skipping uploads restore by request"
fi

if [[ "$START_APP" -eq 1 ]]; then
  log "Starting app"
  compose up -d app
  compose ps
else
  log "--no-start-app supplied; leaving app stopped"
fi

restore_failed=0
log "Production restore completed"
```
{: file="/opt/example/scripts/restore-prod.sh" }

常用方式：

```bash
chmod 700 /opt/example/scripts/restore-prod.sh
/opt/example/scripts/restore-prod.sh --help
```

默认恢复最新本机备份：

```bash
/opt/example/scripts/restore-prod.sh
```

事故时更推荐显式指定文件：

```bash
/opt/example/scripts/restore-prod.sh \
  --sql /opt/example/backups/postgres_2026-05-29_030000.sql \
  --uploads /opt/example/backups/uploads_2026-05-29_030000.tar.gz
```

如果只想恢复数据库，不替换上传目录：

```bash
/opt/example/scripts/restore-prod.sh --skip-uploads
```

## 八、从 R2 拉回并恢复

从 R2 恢复时，我不希望脚本直接把远端文件灌进生产库。更稳妥的流程是：

1. 选择一个 R2 备份时间戳目录
2. 下载到本机临时目录
3. 检查 `SHA256SUMS`
4. 检查 `postgres.sql.gz` gzip 完整性
5. 检查 `uploads.tar.gz` tar 完整性
6. 校验通过后移动到正式恢复目录
7. 调用本机恢复脚本 `restore-prod.sh`

这样可以复用同一套生产恢复逻辑，避免本机恢复和 R2 恢复出现两套实现。

创建脚本：

```bash
nano /opt/example/scripts/restore-prod-from-r2.sh
```

脚本如下：

```bash
#!/usr/bin/env bash
set -Eeuo pipefail
umask 077

PATH="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"

APP_DIR="${APP_DIR:-/opt/example/example-backend}"
COMPOSE_FILE="${COMPOSE_FILE:-docker-compose.prod.yml}"
RESTORE_ROOT="${RESTORE_ROOT:-/opt/example/restore}"
RCLONE_REMOTE="${RCLONE_REMOTE:-r2-backup}"
R2_BUCKET="${R2_BUCKET:-example-prod-backups}"
R2_PREFIX="${R2_PREFIX:-prod}"
RESTORE_SCRIPT="${RESTORE_SCRIPT:-/opt/example/scripts/restore-prod.sh}"

RESTORE_TS=""
LIST_ONLY=0
DOWNLOAD_ONLY=0
SKIP_UPLOADS=0
ASSUME_YES=0
START_APP=1
RCLONE_CONFIG_FILE=""

usage() {
  cat <<'USAGE'
用法：
  restore-prod-from-r2.sh [options]

常用示例：
  restore-prod-from-r2.sh --list
  restore-prod-from-r2.sh --timestamp 2026-05-29_030000
  restore-prod-from-r2.sh --timestamp 2026-05-29_030000 --download-only

选项：
  --timestamp TS       R2 备份目录名，例如 2026-05-29_030000
  --list               只列出 R2 上的备份目录
  --download-only      只下载并校验，不覆盖生产库
  --skip-uploads       只恢复数据库，不替换 uploads/
  --app-dir DIR        后端目录
  --restore-root DIR   R2 备份拉回后的本机根目录
  --compose-file FILE  compose 文件
  --restore-script FILE
                       本机生产恢复脚本路径
  --rclone-remote NAME rclone remote 名
  --r2-bucket NAME     R2 bucket 名
  --r2-prefix PREFIX   R2 环境前缀
  --rclone-config FILE 使用指定 rclone.conf
  --env-file FILE      导出 APP_ENV_FILE，供 compose 使用
  --no-start-app       恢复完成后不启动 app
  --yes                传给本机恢复脚本，跳过确认，谨慎使用
  -h, --help           显示帮助
USAGE
}

log() {
  printf '[%s] %s\n' "$(date '+%F %T')" "$*"
}

die() {
  printf 'ERROR: %s\n' "$*" >&2
  exit 1
}

require_command() {
  command -v "$1" >/dev/null 2>&1 || die "Missing required command: $1"
}

while [[ $# -gt 0 ]]; do
  case "$1" in
    --timestamp) RESTORE_TS="${2:?--timestamp requires a value}"; shift 2 ;;
    --list) LIST_ONLY=1; shift ;;
    --download-only) DOWNLOAD_ONLY=1; shift ;;
    --skip-uploads) SKIP_UPLOADS=1; shift ;;
    --app-dir) APP_DIR="${2:?--app-dir requires a value}"; shift 2 ;;
    --restore-root) RESTORE_ROOT="${2:?--restore-root requires a value}"; shift 2 ;;
    --compose-file) COMPOSE_FILE="${2:?--compose-file requires a value}"; shift 2 ;;
    --restore-script) RESTORE_SCRIPT="${2:?--restore-script requires a value}"; shift 2 ;;
    --rclone-remote) RCLONE_REMOTE="${2:?--rclone-remote requires a value}"; shift 2 ;;
    --r2-bucket) R2_BUCKET="${2:?--r2-bucket requires a value}"; shift 2 ;;
    --r2-prefix) R2_PREFIX="${2:?--r2-prefix requires a value}"; shift 2 ;;
    --rclone-config) RCLONE_CONFIG_FILE="${2:?--rclone-config requires a value}"; export RCLONE_CONFIG="$RCLONE_CONFIG_FILE"; shift 2 ;;
    --env-file) export APP_ENV_FILE="${2:?--env-file requires a value}"; shift 2 ;;
    --no-start-app) START_APP=0; shift ;;
    --yes) ASSUME_YES=1; shift ;;
    -h|--help) usage; exit 0 ;;
    *) usage >&2; die "Unknown option: $1" ;;
  esac
done

require_command rclone
require_command sort

R2_PREFIX="${R2_PREFIX#/}"
R2_PREFIX="${R2_PREFIX%/}"

if [[ -n "$R2_PREFIX" ]]; then
  R2_ROOT="${RCLONE_REMOTE}:${R2_BUCKET}/${R2_PREFIX}"
else
  R2_ROOT="${RCLONE_REMOTE}:${R2_BUCKET}"
fi

list_r2_backups() {
  rclone lsf --dirs-only "${R2_ROOT}/" | sort
}

latest_r2_timestamp() {
  rclone lsf --dirs-only "${R2_ROOT}/" \
    | sed 's:/$::' \
    | grep -E '^[0-9]{4}-[0-9]{2}-[0-9]{2}_[0-9]{6}$' \
    | sort \
    | tail -n 1 || true
}

if [[ "$LIST_ONLY" -eq 1 ]]; then
  log "Listing R2 backup directories from ${R2_ROOT}/"
  list_r2_backups
  exit 0
fi

require_command tail
require_command sed
require_command grep
require_command sha256sum
require_command gzip
require_command tar
require_command flock

if [[ -z "$RESTORE_TS" ]]; then
  log "No --timestamp supplied; selecting latest R2 backup under ${R2_ROOT}/"
  RESTORE_TS="$(latest_r2_timestamp)"
fi

RESTORE_TS="${RESTORE_TS%/}"
[[ -n "$RESTORE_TS" ]] || die "No timestamp backup directory found under ${R2_ROOT}/"
[[ "$RESTORE_TS" =~ ^[0-9]{4}-[0-9]{2}-[0-9]{2}_[0-9]{6}$ ]] \
  || die "Invalid timestamp format: $RESTORE_TS"

RESTORE_DIR="${RESTORE_ROOT}/${RESTORE_TS}"
R2_BACKUP_REMOTE="${R2_ROOT}/${RESTORE_TS}"
LOCK_FILE="${RESTORE_ROOT}/restore-prod-from-r2.lock"
RUN_TS="$(date +%F_%H%M%S)"
TEMP_RESTORE_DIR="${RESTORE_ROOT}/.${RESTORE_TS}.download.${RUN_TS}.$$"

if [[ "$DOWNLOAD_ONLY" -eq 0 ]]; then
  [[ -d "$APP_DIR" ]] || die "APP_DIR does not exist: $APP_DIR"
  [[ -r "$RESTORE_SCRIPT" ]] || die "Restore script is not readable: $RESTORE_SCRIPT"
fi

mkdir -p "$RESTORE_ROOT"
exec 9>"$LOCK_FILE"
flock -n 9 || die "Another R2 restore job is already running: $LOCK_FILE"

cleanup_temp_restore_dir() {
  if [[ -n "${TEMP_RESTORE_DIR:-}" && -d "$TEMP_RESTORE_DIR" ]]; then
    rm -rf "$TEMP_RESTORE_DIR"
  fi
}
trap cleanup_temp_restore_dir EXIT

cat <<SUMMARY

About to download production backup from R2.

R2 backup:       ${R2_BACKUP_REMOTE}/
Restore dir:     ${RESTORE_DIR}
App dir:         ${APP_DIR}
Restore script:  ${RESTORE_SCRIPT}
Uploads restore: $([[ "$SKIP_UPLOADS" -eq 1 ]] && printf 'SKIPPED' || printf 'ENABLED')

The downloaded backup must pass SHA256 verification before any production data is touched.
SUMMARY

log "Creating temporary download directory"
mkdir -p "$TEMP_RESTORE_DIR"

log "Downloading R2 backup to temporary directory"
rclone copy "${R2_BACKUP_REMOTE}/" "${TEMP_RESTORE_DIR}/" \
  --transfers 4 \
  --checkers 8

[[ -r "${TEMP_RESTORE_DIR}/SHA256SUMS" ]] || die "Missing SHA256SUMS"
[[ -r "${TEMP_RESTORE_DIR}/postgres.sql.gz" ]] || die "Missing postgres.sql.gz"

if [[ "$SKIP_UPLOADS" -eq 0 ]]; then
  [[ -r "${TEMP_RESTORE_DIR}/uploads.tar.gz" ]] || die "Missing uploads.tar.gz"
fi

log "Verifying downloaded backup with SHA256SUMS"
(
  cd "$TEMP_RESTORE_DIR"
  sha256sum -c SHA256SUMS
)

log "Checking PostgreSQL dump gzip integrity"
gzip -t "${TEMP_RESTORE_DIR}/postgres.sql.gz"

if [[ "$SKIP_UPLOADS" -eq 0 ]]; then
  log "Checking uploads tarball integrity"
  tar -tzf "${TEMP_RESTORE_DIR}/uploads.tar.gz" >/dev/null
fi

if [[ -e "$RESTORE_DIR" ]]; then
  OLD_RESTORE_DIR="${RESTORE_DIR}.before-r2-download.${RUN_TS}"
  mv "$RESTORE_DIR" "$OLD_RESTORE_DIR"
  log "Existing restore directory moved to ${OLD_RESTORE_DIR}"
fi

mv "$TEMP_RESTORE_DIR" "$RESTORE_DIR"
TEMP_RESTORE_DIR=""
log "Verified backup is ready at ${RESTORE_DIR}"

if [[ -r "${RESTORE_DIR}/backup_info.txt" ]]; then
  log "Downloaded backup metadata"
  sed -n '1,120p' "${RESTORE_DIR}/backup_info.txt"
fi

if [[ "$DOWNLOAD_ONLY" -eq 1 ]]; then
  log "--download-only supplied; production data was not changed"
  exit 0
fi

restore_args=(
  --app-dir "$APP_DIR"
  --backup-dir "$RESTORE_DIR"
  --compose-file "$COMPOSE_FILE"
  --sql "${RESTORE_DIR}/postgres.sql.gz"
)

if [[ "$SKIP_UPLOADS" -eq 1 ]]; then
  restore_args+=(--skip-uploads)
else
  restore_args+=(--uploads "${RESTORE_DIR}/uploads.tar.gz")
fi

if [[ "$START_APP" -eq 0 ]]; then
  restore_args+=(--no-start-app)
fi

if [[ "$ASSUME_YES" -eq 1 ]]; then
  restore_args+=(--yes)
fi

log "Calling local production restore script"
bash "$RESTORE_SCRIPT" "${restore_args[@]}"
```
{: file="/opt/example/scripts/restore-prod-from-r2.sh" }

使用方式：

```bash
chmod 700 /opt/example/scripts/restore-prod-from-r2.sh
```

列出 R2 上的恢复点：

```bash
/opt/example/scripts/restore-prod-from-r2.sh --list
```

只下载并校验，不覆盖生产数据：

```bash
/opt/example/scripts/restore-prod-from-r2.sh \
  --timestamp 2026-05-29_030000 \
  --download-only
```

确认要恢复时：

```bash
/opt/example/scripts/restore-prod-from-r2.sh \
  --timestamp 2026-05-29_030000
```

**最佳实践：** 如果不指定 `--timestamp`，脚本会默认选择 R2 指定前缀下最新的时间戳目录。但在事故恢复时，强烈建议**显式指定时间戳**，避免选中包含被污染数据的较新备份点。

## 九、恢复后的检查

恢复完成后，不要只看脚本是否退出成功，还要检查应用状态：

```bash
cd /opt/example/example-backend

docker compose -f docker-compose.prod.yml ps
docker compose -f docker-compose.prod.yml logs --tail=100 app
```

如果应用有健康检查接口，可以再请求一次：

```bash
curl -fsS http://127.0.0.1:<PORT>/health
```

确认业务正常后，再决定是否删除旧的上传目录：

```bash
ls -lah /opt/example/example-backend | grep uploads.before-restore
```

不要在刚恢复完成后立刻删除旧目录。最好等确认新数据没有问题，再清理这些回滚用的目录。

## 十、安全注意事项

在实施这套方案时，请特别注意以下边界与红线：

1. **R2 访问控制**：Bucket 必须保持 Private，决不开通公共访问。
2. **本地凭证安全**：`rclone.conf` 配置文件的权限严格设定为 `600`。
3. **最小化 Token**：R2 API Token 只授予该备份 Bucket 的读写权限。
4. **敏感内容隔离**：决不将 `.env`、JWT Secret、支付密钥、推送证书以及第三方 API 私钥等高敏信息明文打包上传至 R2。
5. **密钥独立存储**：环境变量及安全密钥应单独离线存入 1Password 等密码管理器或专用的加密硬件备份体系。
6. **阻断恶意恢复**：恢复脚本要求高阻力的交互式确认，不推荐常规流程中使用 `--yes` 免硬确认。
7. **常规灾难演练**：每个月或每季度，必须至少开展一次在隔离（不覆盖生产）环境下的灾难恢复演练。
8. **云端生命周期**：远端对象存储的 Lifecycle 规则（自动过期）不要短于脚本里定义的 `R2_RETENTION_DAYS` 天数。
9. **日志脱敏防范**：切勿在备份日志任务中 echo 或是明文打印涉及秘钥的环境变量。
10. **强一致性要求**：如果业务本身有着银行级别的强一致性要求，则数据库与上传目录需要升级至更严苛的基于块快照 (Volume Snapshot) 结合应用停写机制的灾备方式。

## 十一、适用场景

这套方案主要针对并非常适合以下情况：

- 小型架构后端
- 单机（单节点）Docker Compose 部署模型
- 组合方案：PostgreSQL 配合宿主机卷载（Volume）的 `uploads` 目录
- 个人项目、早期初创业务或内部基础工具
- 期望以最低实施复杂度获取一套可靠底线级别灾备能力的初创场景

请注意：**它不等于企业级高可用（HA）架构**。它没有通过分布式锁去解决多节点并发一致性，也没有处理跨地域多活节点、秒级别的 RPO 或者无缝的自动故障漂移。

但回归现实，对于绝大多数资源受限的小中型项目来说，先做到以下三件事，已经足以击败世界上 80% “只有上线文档，没有灾难急救方案”的系统：

1. 每日定时且全自动地备份结构化数据与非结构化上传目录。
2. 本机驻留可用作迅速退闪的快照包；Cloudflare R2 保留可以抵御物理宕机长线灾难备份。
3. 定期将这份备份档实际导入另一台空白试验库，证明**“该备份的确可以用”**。

**写在最后：** 备份的价值不在于每天产生了多少打包文件，而在于致命事故降临（如误删数据库、磁盘报废）的那个凌晨里，你能否凭借键盘，既冷静、且可重复、更明确可验证地把系统稳稳恢复回来。