# Contributing

Спасибо за интерес к Free Google Disk Backup.

Главная идея проекта: сделать понятный safety kit для AI-first разработки,
который помогает solo-builders и small teams настраивать восстановимые бэкапы
на Google Drive через Restic + rclone.

## Что особенно приветствуется

- Улучшения README и runbook.
- Более понятные инструкции по Google Cloud / OAuth / rclone.
- Шаблоны для PostgreSQL, MySQL/MariaDB, SQLite.
- Безопасные examples без реальных секретов.
- Restore-test сценарии.
- Улучшения Codex Skill.
- Английская версия документации.

## Правила безопасности

Не отправляйте в issues, pull requests, screenshots или examples:

- реальные `.env` файлы;
- Restic password;
- `rclone.conf`;
- OAuth client secret;
- Telegram bot token;
- SSH private keys;
- реальные production IP/домены, если они приватные;
- database dumps.

Если нужно показать config, заменяйте значения на placeholders:

```text
TELEGRAM_BACKUP_BOT_TOKEN=<secret>
RESTIC_PASSWORD_FILE=/etc/example-backup/restic-password
RESTIC_REPOSITORY=rclone:gdrive:Example-Backups/restic
```

## Стиль документации

- Пишите так, чтобы понял человек, который впервые открывает Google Cloud
  Console.
- Объясняйте не только команду, но и зачем она нужна.
- Для потенциально опасных команд явно пишите, что они делают.
- Production restore всегда должен быть guarded and explicit.
- Не обещайте, что backup работает, пока не описан restore-test.

## Pull request checklist

Перед PR проверьте:

- [ ] В документации нет реальных секретов.
- [ ] Все команды используют placeholders там, где нужны приватные значения.
- [ ] Restore flow не предлагает casual production restore.
- [ ] Новые examples не удаляют данные.
- [ ] README остаётся понятным для non-expert пользователя.

