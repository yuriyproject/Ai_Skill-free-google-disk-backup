# Checklists

Практические чеклисты для настройки и эксплуатации Free Google Disk Backup.

## Pre-install checklist

- [ ] Понятно, где находится production server.
- [ ] Понятно, какой проект бэкапим.
- [ ] Есть SSH-доступ к серверу.
- [ ] Известно, какая база данных используется.
- [ ] Найден production `.env` или аналогичный secrets/config file.
- [ ] Найдены durable user files: uploads, OCR/images, documents, media.
- [ ] Понятно, что не нужно бэкапить: cache, Docker images, dependencies.
- [ ] Создан отдельный Telegram backup bot.
- [ ] Получен Telegram chat ID.
- [ ] Создан/выбран Google Cloud project.
- [ ] Включён Google Drive API.
- [ ] Создан OAuth Client ID типа `Desktop app`.
- [ ] Выбран scope `drive.file`.

## Install verification checklist

- [ ] `rclone lsd gdrive:` работает.
- [ ] Restic repository initialized.
- [ ] `/etc/<project>-backup/restic-password` существует.
- [ ] Restic password сохранён в password manager.
- [ ] Sensitive files имеют owner `root` и mode `600`.
- [ ] Config directory имеет mode `700`.
- [ ] Manual backup завершился успешно.
- [ ] `restore.sh list` показывает snapshot.
- [ ] `restore.sh stage latest` распаковывает snapshot в staging.
- [ ] `restore.sh test latest` проходит isolated restore-test.
- [ ] Telegram success notification пришёл.
- [ ] systemd timers enabled.
- [ ] Daily summary enabled.

## Weekly maintenance checklist

- [ ] Проверить `systemctl list-timers '*backup*' --all --no-pager`.
- [ ] Убедиться, что maintenance timer имеет ожидаемый `NEXT`.
- [ ] После первого weekly запуска проверить latest maintenance state.
- [ ] Если summary пишет `Maintenance: missing` до первого weekly запуска, это нормально.
- [ ] Если `Maintenance: missing` остаётся после weekly запуска, проверить journal logs.

Команды:

```bash
sudo systemctl status <project>-backup-maintenance.timer --no-pager -l
sudo systemctl status <project>-backup-maintenance.service --no-pager -l
sudo journalctl -u <project>-backup-maintenance.service -n 100 --no-pager
```

## Change schedule checklist

- [ ] Понять, какой timer нужно изменить: backup, maintenance, full-check или restore-test.
- [ ] Посмотреть текущее расписание.
- [ ] Создать systemd override через `systemctl edit`, а не редактировать unit вслепую.
- [ ] Сбросить старый `OnCalendar` пустой строкой `OnCalendar=`.
- [ ] Добавить новый `OnCalendar`.
- [ ] Выполнить `daemon-reload`.
- [ ] Перезапустить timer.
- [ ] Проверить колонку `NEXT`.

Команды:

```bash
sudo systemctl cat <project>-backup-maintenance.timer --no-pager
sudo systemctl edit <project>-backup-maintenance.timer
sudo systemctl daemon-reload
sudo systemctl restart <project>-backup-maintenance.timer
sudo systemctl list-timers '*backup*' --all --no-pager
```

Пример maintenance 3 раза в неделю:

```ini
[Timer]
OnCalendar=
OnCalendar=Mon,Wed,Fri *-*-* 03:30:00
RandomizedDelaySec=10m
```

Пример restore-test 2 раза в месяц:

```ini
[Timer]
OnCalendar=
OnCalendar=*-*-01,15 05:30:00
RandomizedDelaySec=30m
```

## Monthly restore drill checklist

- [ ] Выбрать свежий snapshot.
- [ ] Выполнить isolated restore-test.
- [ ] Проверить, что database dump читается.
- [ ] Проверить, что manifest и checksums валидны.
- [ ] Проверить, что production `.env` есть в staged recovery, но не печатать его.
- [ ] Записать результат restore drill в handoff/runbook.

Команды:

```bash
sudo /opt/<project>/ops/backup/restore.sh list
sudo /opt/<project>/ops/backup/restore.sh test latest
sudo cat /var/lib/<project>-backup/state/latest-restore-test.json
```

## Stage restore server checklist

Используй этот сценарий, если не хочешь проверять восстановление на production
server.

- [ ] Поднять отдельный stage/test VPS или выделить постоянный restore-lab.
- [ ] Установить `restic`, `rclone`, `jq`, `git`, `docker`.
- [ ] Перенести backup config secrets root-only:
  `/etc/<project>-backup/restic-password`,
  `/etc/<project>-backup/rclone.conf`,
  `/etc/<project>-backup/backup.env`.
- [ ] Проверить права: config dir `700`, secret files `600`.
- [ ] Выполнить `restic snapshots`.
- [ ] Restore latest snapshot в stage folder.
- [ ] Проверить `manifest.json` и `checksums.sha256`.
- [ ] Восстановить database dump в isolated container.
- [ ] Проверить ключевые таблицы.
- [ ] Если поднимается app stack, использовать stage-safe `.env`.
- [ ] Отключить production Telegram bot/webhooks/payments/external writes.
- [ ] Записать результат restore-lab check в handoff.

Подробнее: [stage-restore-server.md](stage-restore-server.md).

## Manual trial restore checklist

Используй этот сценарий, когда хочешь убедиться в бэкапе руками, не доверяя
только Telegram summary.

- [ ] Выполнить `restore.sh list`.
- [ ] Выбрать snapshot или использовать `latest`.
- [ ] Выполнить `restore.sh test latest`.
- [ ] Проверить `latest-restore-test.json`.
- [ ] При необходимости выполнить `restore.sh stage latest`.
- [ ] Проверить `manifest.json`.
- [ ] Проверить `checksums.sha256`.
- [ ] Проверить наличие database dump.
- [ ] Проверить наличие production env file без вывода содержимого.
- [ ] Не запускать production restore.

Команды:

```bash
sudo /opt/<project>/ops/backup/restore.sh list
sudo /opt/<project>/ops/backup/restore.sh test latest
sudo cat /var/lib/<project>-backup/state/latest-restore-test.json
sudo /opt/<project>/ops/backup/restore.sh stage latest
```

В staged recovery можно проверять файлы, но не печатать secrets:

```bash
sudo find /var/lib/<project>-restore -maxdepth 4 -type f | sort
```

## Before production restore

- [ ] Есть явное разрешение владельца.
- [ ] Выбран конкретный snapshot ID, не просто `latest`.
- [ ] Isolated restore-test для этого snapshot прошёл успешно.
- [ ] Сделан свежий backup перед restore.
- [ ] Понятно, какие сервисы будут остановлены.
- [ ] Понятно, как откатиться.
- [ ] Production restore command содержит explicit confirmation token.
