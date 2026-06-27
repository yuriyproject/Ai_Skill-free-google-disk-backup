# Free Google Disk Backup — инструкция для человека и AI-агента

Эта инструкция описывает схему бесплатного регулярного бэкапирования проекта в
Google Drive через Restic + rclone. Идея: не покупать Hetzner Backup, S3,
Google Cloud Storage или отдельный backup-сервис, а использовать уже доступное
место на Google Drive.

## Что мы хотим получить

- Зашифрованные бэкапы на Google Drive.
- Почасовые, дневные, недельные и месячные точки восстановления.
- Telegram-уведомления об успехах, ошибках и ежедневном состоянии.
- Простое восстановление: список бэкапов, staging-распаковка, test restore,
  production restore только с явным подтверждением.
- Документ-handoff, чтобы другой AI-агент мог восстановить проект без гаданий.

## Готовый prompt для нового чата

```text
Use $free-google-disk-backup.

Мне нужно организовать бесплатные регулярные бэкапы этого production-проекта
на мой Google Drive. Используй Restic + rclone. Не используй платное хранилище
Hetzner, Google Cloud Storage, S3 или другие платные backup-сервисы.
Ничего не удаляй без явного разрешения.

Нужна гибридная глубина:
- почасовые бэкапы за 48 часов
- дневные бэкапы за 30 дней
- недельные бэкапы за 12 недель
- месячные бэкапы за 12 месяцев

Нужно бэкапить базу данных, production .env/secrets, пользовательские файлы,
важные deploy-файлы и source metadata/Git bundle. Не бэкапить кеши,
Docker images, зависимости и пересобираемые артефакты, если это не бизнес-данные.

Добавь Telegram-уведомления, daily summary, weekly maintenance и restore-test.
Оставь подробный handoff-документ: где лежит Restic password, rclone config,
как посмотреть статус, как расшифровать и восстановить бэкап.
```

## Что нужно дать агенту

По необходимости:

- SSH-доступ к серверу.
- Доступ к GitHub-репозиторию, если надо коммитить scripts/docs.
- Google OAuth flow для rclone.
- Отдельный Telegram BotFather token для backup-бота.
- Telegram chat ID для уведомлений.

Не отправляй в чат:

- пароли;
- `.env`;
- Restic password;
- rclone config;
- OAuth client secret;
- SSH private key;
- Telegram token.

Лучше вводить секреты через терминал, password manager, GitHub Secrets или
root-only файлы на сервере.

## Рекомендуемая архитектура

```text
Restic -> encrypted repository -> rclone -> Google Drive
systemd timers -> hourly/daily/weekly/monthly jobs
Telegram bot -> alerts and summaries
```

Типовые пути на сервере:

```text
/opt/<project>/ops/backup/          scripts in repo
/etc/<project>-backup/              root-only config and secrets
/var/lib/<project>-backup/          state/cache/staging
/var/lib/<project>-restore/         staged restores
```

Критически важный файл:

```text
/etc/<project>-backup/restic-password
```

Без него файлы на Google Drive останутся зашифрованными и бесполезными.
Его нужно сохранить в password manager.

## Google Drive API

Нужно:

1. Создать/выбрать Google Cloud project.
2. Enable Google Drive API.
3. Настроить OAuth consent screen.
4. Добавить scope:

```text
https://www.googleapis.com/auth/drive.file
```

5. Создать OAuth Client ID типа `Desktop app`.
6. Настроить rclone remote, обычно с именем `gdrive`.

Важно: это не должно использовать Google Cloud Storage, Compute Engine, Cloud
SQL или другие платные GCP-сервисы. Используется только Google Drive API и место
на твоём Google Drive.

## Telegram

Лучше создать отдельного backup-бота через BotFather, например:

```text
ProjectBackupBot
```

Не использовать основного customer-facing бота проекта. Backup bot должен
отправлять только operational alerts:

- backup succeeded;
- backup failed;
- daily summary;
- restore test result;
- maintenance result.

## Что бэкапить

Обычно нужно включить:

- database dump;
- production `.env` или аналогичный secrets file;
- пользовательские загрузки;
- OCR/images/documents, если их нельзя пересоздать;
- `docker-compose.yml`;
- `Caddyfile`/nginx config;
- Git bundle или точный commit metadata;
- `manifest.json`;
- `checksums.sha256`.

Обычно не нужно включать:

- Redis cache;
- Docker images;
- `node_modules`;
- virtualenv;
- build artifacts;
- временные файлы;
- логи, если нет отдельной причины.

## systemd timers

Ожидаемые таймеры:

```text
<project>-backup.timer              hourly backup
<project>-backup-summary.timer      daily Telegram summary
<project>-backup-maintenance.timer  weekly retention/prune/check
<project>-backup-full-check.timer   monthly repository check
<project>-backup-restore-test.timer monthly isolated restore test
```

Проверка:

```bash
sudo systemctl list-timers '*backup*' --all --no-pager
sudo systemctl status <project>-backup.service --no-pager -l
sudo journalctl -u <project>-backup.service -n 100 --no-pager
```

## Как читать Telegram summary

Пример:

```text
Latest: <snapshot-id> (OK)
Timestamp: 2026-06-18T08:03:06Z
Duration: 41s
Repository: ok
Maintenance: missing
Restore test: ok
Google Drive used/free: 5063760/5496985770348
```

Значение:

- `Latest (OK)` — последний бэкап успешен.
- `Repository: ok` — Restic видит репозиторий на Google Drive.
- `Maintenance: missing` — weekly maintenance ещё не запускался или нет state
  файла. До первого воскресного/недельного запуска это нормально.
- `Restore test: ok` — тест восстановления прошёл.
- `Google Drive used/free` — занято/свободно в байтах.

## Как доказать, что всё работает

Агент должен показать:

```bash
sudo systemctl list-timers '*backup*' --all --no-pager
sudo /opt/<project>/ops/backup/restore.sh list
sudo /opt/<project>/ops/backup/restore.sh test latest
sudo cat /var/lib/<project>-backup/state/latest-backup.json
sudo cat /var/lib/<project>-backup/state/latest-restore-test.json
```

Production restore нельзя запускать “просто проверить”. Сначала только isolated
test restore.

Для более строгой проверки лучше использовать отдельный Stage Restore Server:
не production VPS, а отдельную машину/restore-lab, которая читает Google Drive
Restic repository, восстанавливает snapshot, поднимает isolated database
container и проверяет, что данные действительно можно восстановить. Это
предпочтительный вариант для проектов, где страшно даже тестировать restore на
production-хосте.

Подробный сценарий: `docs/stage-restore-server.md`.

## Что должно быть в handoff-документе

- Что именно бэкапится.
- Что осознанно не бэкапится.
- Google Drive repository path.
- Retention schedule.
- systemd timer names.
- Telegram bot purpose.
- Server paths.
- Где лежит Restic password.
- Где лежит rclone config.
- Как посмотреть статус.
- Как сделать `list`, `stage`, `test restore`.
- Как делать production restore.
- Первый проверенный snapshot ID.
- Последний успешный restore-test.
- Предупреждение: не терять Restic password.

Документ не должен содержать секретные значения.
