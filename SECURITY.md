# Security Policy

Free Google Disk Backup работает с секретами и production backup-процессами.
Безопасность здесь важнее удобства.

## Не публикуйте секреты

Никогда не вставляйте в GitHub issues, pull requests, README, screenshots,
chat prompts или logs:

- Restic password;
- `/etc/<project>-backup/restic-password`;
- `rclone.conf`;
- OAuth client ID/secret, если они относятся к вашему проекту;
- refresh tokens;
- Telegram bot token;
- SSH private keys;
- production `.env`;
- database dumps.

Если секрет случайно опубликован:

1. Сразу считайте его скомпрометированным.
2. Отзовите или пересоздайте token/secret.
3. Если утёк Restic password, создайте новый Restic repository и перенесите
   backup strategy. Старый repository нельзя надёжно "переобезопасить" простым
   удалением секрета из Git history.

## Restic password

Restic password нельзя восстановить. Если он потерян, encrypted repository на
Google Drive не поможет восстановить данные.

Проверить наличие файла:

```bash
sudo ls -l /etc/<project>-backup/restic-password
```

Проверить, что файл не пустой, не показывая пароль:

```bash
sudo wc -c /etc/<project>-backup/restic-password
```

Показать пароль можно только на своём сервере для сохранения в password manager:

```bash
sudo cat /etc/<project>-backup/restic-password
```

Не вставляйте результат этой команды в AI chat.

## Threat model

Эта схема помогает при:

- случайном удалении данных AI-агентом;
- плохом deploy;
- повреждённой миграции;
- потере VPS;
- необходимости достать старую версию `.env`, database dump или uploaded files.

Эта схема не спасает от:

- потери Restic password;
- компрометации Google account;
- компрометации сервера до создания backup;
- злоумышленника, который получил root и успел удалить/испортить backup config;
- отсутствия restore-test.

## Reporting vulnerabilities

Если вы нашли проблему безопасности в шаблонах или инструкциях, создайте GitHub
issue без секретов или откройте private advisory, если репозиторий это
поддерживает.

В описании укажите:

- какой файл или сценарий затронут;
- почему это опасно;
- безопасный пример воспроизведения без реальных секретов;
- предлагаемое исправление, если оно есть.

