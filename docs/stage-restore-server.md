# Stage Restore Server

Stage Restore Server, или restore-lab, — это отдельный сервер, на котором
проверяется восстановление backup repository без риска для production.

Это предпочтительный вариант для владельца проекта, если restore-test на
production-хосте психологически или архитектурно кажется слишком рискованным.

## Идея

Production server только создаёт backups.

Stage Restore Server только проверяет, что backups можно восстановить.

```text
Production server
  |
  | creates encrypted Restic snapshots
  v
Google Drive Restic repository
  ^
  |
Stage Restore Server
  |
  | restores latest snapshot
  | restores database into isolated container
  | optionally boots app stack with safe stage env
  v
Telegram: restore-lab OK / failed
```

## Почему это лучше, чем проверять на production

- Production database не трогается.
- Production `.env` не заменяется.
- Production containers не останавливаются.
- Можно проверить disaster recovery почти как в реальной аварии.
- Можно смелее тестировать restore scripts.
- Ошибка restore-test не ломает рабочий сервис.

## Два режима

### Постоянный restore-lab

Небольшой VPS живёт постоянно и по расписанию проверяет backup.

Плюсы:

- удобно автоматизировать;
- можно получать регулярный Telegram restore-lab report;
- подходит, если проектов несколько.

Минусы:

- стоит денег каждый месяц;
- на нём тоже нужно защищать secrets.

### Временный test VPS

VPS создаётся только на время restore drill, например раз в месяц.

Плюсы:

- дешевле;
- ближе к реальному disaster recovery from scratch.

Минусы:

- больше ручных действий;
- сложнее автоматизировать.

## Что нужно на Stage Restore Server

Минимум:

```bash
sudo apt-get update
sudo apt-get install -y restic rclone jq git docker.io
```

Если проект использует Docker Compose:

```bash
docker compose version
```

Если compose plugin отсутствует, установи его способом, подходящим для ОС.

## Какие секреты нужны

Stage Restore Server должен уметь читать Google Drive Restic repository и
расшифровывать backups.

Нужны:

```text
/etc/<project>-backup/restic-password
/etc/<project>-backup/rclone.conf
/etc/<project>-backup/backup.env
```

Важно:

- эти файлы дают доступ к encrypted backups;
- если stage server скомпрометирован, backups можно расшифровать;
- храни их root-only;
- не печатай их в чат или logs.

Права:

```bash
sudo install -d -m 700 /etc/<project>-backup
sudo chown root:root /etc/<project>-backup
sudo chmod 700 /etc/<project>-backup

sudo chown root:root /etc/<project>-backup/*
sudo chmod 600 /etc/<project>-backup/*
```

## Безопасная stage env

Если ты поднимаешь приложение целиком, не запускай его с production `.env` как
есть. Production `.env` может содержать:

- Telegram bot token;
- webhook URL;
- payment/API keys;
- email/SMS providers;
- production domain;
- background job flags.

На stage server сделай отдельный `stage.env`:

- database points to stage database;
- Telegram bot token disabled or replaced with test bot;
- webhooks disabled;
- payments disabled;
- external writes disabled;
- background jobs disabled unless they are part of restore test;
- domain/local URL stage-only.

Правило:

> Restore production secrets only for inspection/recovery, not for casually
> booting a live stage app.

## Level 1: проверить repository

```bash
sudo bash -c '
set -euo pipefail
source /etc/<project>-backup/backup.env
export RCLONE_CONFIG RESTIC_PASSWORD_FILE RESTIC_REPOSITORY
export RESTIC_CACHE_DIR="${RESTIC_CACHE_DIR:-/var/lib/<project>-backup/restic-cache}"
install -d -m 700 "$RESTIC_CACHE_DIR"
restic -r "$RESTIC_REPOSITORY" snapshots
'
```

Если snapshots видны, stage server может читать Google Drive repository и
расшифровывать metadata.

## Level 2: restore snapshot в stage folder

```bash
sudo bash -c '
set -euo pipefail
source /etc/<project>-backup/backup.env
export RCLONE_CONFIG RESTIC_PASSWORD_FILE RESTIC_REPOSITORY
export RESTIC_CACHE_DIR="${RESTIC_CACHE_DIR:-/var/lib/<project>-backup/restic-cache}"
SNAPSHOT_ID="latest"
TARGET="/var/lib/<project>-restore/manual-${SNAPSHOT_ID}"
install -d -m 700 "$TARGET"
restic -r "$RESTIC_REPOSITORY" restore "$SNAPSHOT_ID" --target "$TARGET"
find "$TARGET" -maxdepth 5 -type f | sort | head -200
'
```

Проверь, что внутри есть:

- `manifest.json`;
- `checksums.sha256`;
- database dump;
- production env backup;
- user files/media;
- deploy files.

Не печатай production env contents.

## Level 3: восстановить database в isolated container

Пример для PostgreSQL:

```bash
sudo docker run -d \
  --name restore-lab-postgres \
  -e POSTGRES_PASSWORD=restore-lab-password \
  postgres:16-alpine

for i in $(seq 1 30); do
  sudo docker exec restore-lab-postgres pg_isready -U postgres && break
  sleep 1
done
```

Создать роль, если dump её ожидает:

```bash
sudo docker exec restore-lab-postgres psql \
  -U postgres \
  -d postgres \
  -v ON_ERROR_STOP=1 \
  -c "CREATE ROLE <project_db_user> LOGIN;"
```

Восстановить dump:

```bash
sudo docker exec -i restore-lab-postgres pg_restore \
  -U postgres \
  -d postgres \
  --clean \
  --if-exists \
  < /path/to/restored/database.dump
```

Проверить таблицы:

```bash
sudo docker exec restore-lab-postgres psql -U postgres -d postgres -Atc \
  "SELECT table_name FROM information_schema.tables WHERE table_schema='public' ORDER BY table_name;"
```

Cleanup:

```bash
sudo docker rm -f restore-lab-postgres
```

## Level 4: поднять application stack на stage server

Это самый сильный тест, но требует осторожности.

Порядок:

1. Restore source/deploy files или clone repository.
2. Создать stage `.env`, не production `.env`.
3. Поднять только безопасные сервисы:
   - database;
   - backend;
   - frontend/admin;
   - optionally worker with external writes disabled.
4. Не запускать production Telegram bot с реальным token.
5. Не включать production webhooks.
6. Проверить health endpoints.
7. Проверить login/admin read-only сценарии.
8. Проверить, что restored data видны.

Пример:

```bash
cd /opt/<project-stage>
sudo docker compose --env-file .env.stage up -d db backend admin_frontend caddy
```

Проверки:

```bash
curl -fsS http://localhost/health
curl -fsS http://localhost/health/db
```

Если проект имеет admin panel, лучше открыть её на stage URL или через SSH
tunnel, но не публиковать stage наружу без необходимости.

## Level 5: disaster recovery rehearsal

Это полноценная репетиция аварии:

1. Новый чистый VPS.
2. Установка runtime dependencies.
3. Восстановление backup config.
4. Restore latest snapshot.
5. Restore database.
6. Restore durable files.
7. Boot app stack with stage-safe env.
8. Проверка health/admin/core flows.
9. Telegram report.
10. Удаление VPS или очистка restore-lab.

## Автоматизация restore-lab

На постоянном stage server можно сделать отдельный timer:

```text
<project>-stage-restore-test.timer
```

Например 1-го и 15-го числа:

```ini
[Timer]
OnCalendar=*-*-01,15 05:30:00
Persistent=true
RandomizedDelaySec=30m
Unit=<project>-stage-restore-test.service
```

Service должен:

1. получить latest snapshot;
2. восстановить snapshot в fresh staging dir;
3. восстановить database в isolated container;
4. проверить manifest/checksums/tables;
5. отправить Telegram report;
6. удалить временные containers;
7. оставить state JSON с результатом.

## Telegram report example

```text
Restore-lab check OK
Project: example-project
Snapshot: 8b9ad277
Timestamp: 2026-06-19T05:35:12Z
Repository: ok
Database restore: ok
Manifest/checksums: ok
Stage app boot: skipped
Duration: 84s
```

## Что записывать в handoff

- Stage Restore Server hostname/IP.
- Где лежит stage backup config.
- Какой snapshot проверялся последним.
- Когда был последний successful restore-lab check.
- Какие уровни проверки включены: Level 1/2/3/4/5.
- Запускается ли app stack или только database restore.
- Какие внешние интеграции отключены на stage.

## Главное правило

Stage Restore Server должен доказывать восстановимость backup, но не должен
становиться вторым production.

